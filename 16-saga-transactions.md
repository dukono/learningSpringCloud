# Parte 16 — Transacciones Distribuidas y Patrón Saga

← [Parte 15 — Testing](./15-testing.md) | [Volver al índice](./README.md)

---

## 16.1 El problema: ACID en microservicios

En un monolito, una transacción que toca varias tablas es trivial: una sola transacción de BD lo abarca todo. En microservicios, cada servicio tiene su propia base de datos — no hay una sola conexión que abarque todo.

```
MONOLITO — una transacción:
┌─────────────────────────────────────────┐
│  BEGIN TRANSACTION                      │
│    INSERT pedidos ...                   │
│    UPDATE inventario SET stock = stock-1│
│    INSERT pagos ...                     │
│  COMMIT  ← todo o nada                 │
└─────────────────────────────────────────┘

MICROSERVICIOS — tres BDs distintas:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ pedidos-svc  │  │inventario-svc│  │  pagos-svc   │
│  BD Pedidos  │  │ BD Inventario│  │  BD Pagos    │
└──────────────┘  └──────────────┘  └──────────────┘
        ↑                ↑                 ↑
   No hay una transacción que los una
```

**El problema concreto:** el pedido se crea, el inventario se descuenta, pero el pago falla. ¿Cómo revertir los dos primeros pasos?

---

## 16.2 Por qué 2PC (Two-Phase Commit) no funciona

**2PC** es el protocolo clásico para transacciones distribuidas. Tiene un coordinador que pregunta a todos los participantes si pueden hacer commit y luego ordena el commit o rollback global.

**Problemas en microservicios:**
- Acoplamiento fuerte entre servicios — todos deben estar disponibles
- El coordinador es punto único de fallo
- Bloquea recursos hasta que todos responden — baja disponibilidad
- No funciona bien con servicios asíncronos o colas de mensajes

> `[CONCEPTO]` El teorema CAP dice que en un sistema distribuido no puedes tener Consistencia + Disponibilidad + Tolerancia a particiones a la vez. 2PC sacrifica disponibilidad para mantener consistencia. En microservicios se prefiere disponibilidad y **consistencia eventual**.

---

## 16.3 Patrón Saga

Una **Saga** es una secuencia de transacciones locales donde cada paso publica un evento o mensaje que dispara el siguiente paso. Si un paso falla, se ejecutan **transacciones compensatorias** para deshacer los pasos anteriores.

```
Saga de "Crear Pedido":

paso 1: pedidos-svc    → crea pedido en estado PENDIENTE
paso 2: inventario-svc → reserva stock
paso 3: pagos-svc      → procesa pago
paso 4: pedidos-svc    → actualiza pedido a CONFIRMADO

Si paso 3 falla:
  compensación 2: inventario-svc → libera reserva de stock
  compensación 1: pedidos-svc    → cancela pedido (estado CANCELADO)
```

---

## 16.4 Coreografía vs Orquestación

### Coreografía — eventos sin coordinador central

Cada servicio reacciona a eventos y emite los suyos. No hay nadie que dirija.

```
pedidos-svc                inventario-svc              pagos-svc
    │                            │                          │
    │── PedidoCreado ──────────→ │                          │
    │                            │── StockReservado ──────→ │
    │                            │                          │── PagoProcessado ──→ ...
    │                            │
    │ (si falla pago):
    │                            │ ←── PagoFallido ─────────│
    │ ←── StockLiberado ─────────│
    │── PedidoCancelado ────────→ ...
```

**Ventajas:** bajo acoplamiento, servicios independientes.
**Desventajas:** difícil de rastrear el flujo completo, riesgo de ciclos de eventos.

### Orquestación — un orquestador central dirige los pasos

```
                     ┌─────────────────┐
                     │   Orquestador   │
                     │  (Saga Manager) │
                     └────────┬────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ↓                   ↓                   ↓
   pedidos-svc          inventario-svc         pagos-svc
   (paso 1)              (paso 2)              (paso 3)
```

**Ventajas:** flujo explícito y visible, fácil de depurar.
**Desventajas:** el orquestador puede convertirse en un "dios" con demasiada lógica.

---

## 16.5 Implementación con Spring + Kafka (Coreografía)

### Paso 1: Pedidos crea el pedido y publica evento

```java
@Service
@Transactional
public class PedidoService {

    @Autowired private PedidoRepository pedidoRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;

    public Pedido crearPedido(CrearPedidoRequest request) {
        // Transacción LOCAL — solo afecta a la BD de pedidos
        Pedido pedido = pedidoRepository.save(
            Pedido.builder()
                .productoId(request.getProductoId())
                .cantidad(request.getCantidad())
                .estado(EstadoPedido.PENDIENTE)
                .build()
        );

        // Publicar evento para que inventario reaccione
        kafkaTemplate.send("pedidos.creados", new PedidoCreadoEvento(
            pedido.getId(),
            pedido.getProductoId(),
            pedido.getCantidad()
        ));

        return pedido;
    }

    // Transacción compensatoria
    public void cancelarPedido(Long pedidoId, String motivo) {
        Pedido pedido = pedidoRepository.findById(pedidoId).orElseThrow();
        pedido.setEstado(EstadoPedido.CANCELADO);
        pedido.setMotivoCancelacion(motivo);
        pedidoRepository.save(pedido);
    }
}
```

### Paso 2: Inventario escucha y reserva stock

```java
@Service
public class InventarioSagaListener {

    @Autowired private InventarioRepository inventarioRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "pedidos.creados")
    public void onPedidoCreado(PedidoCreadoEvento evento) {
        try {
            Inventario inv = inventarioRepository
                .findByProductoId(evento.getProductoId())
                .orElseThrow();

            if (inv.getStockDisponible() < evento.getCantidad()) {
                throw new StockInsuficienteException();
            }

            inv.reservar(evento.getCantidad());
            inventarioRepository.save(inv);

            // Éxito: notificar al siguiente paso
            kafkaTemplate.send("inventario.reservado", new StockReservadoEvento(
                evento.getPedidoId(), evento.getProductoId(), evento.getCantidad()
            ));

        } catch (Exception e) {
            // Fallo: publicar evento de compensación
            kafkaTemplate.send("inventario.fallido", new StockFallidoEvento(
                evento.getPedidoId(), e.getMessage()
            ));
        }
    }

    // Compensación: liberar reserva si el pago falla
    @KafkaListener(topics = "pagos.fallidos")
    public void onPagoFallido(PagoFallidoEvento evento) {
        inventarioRepository.findByProductoId(evento.getProductoId())
            .ifPresent(inv -> {
                inv.liberarReserva(evento.getCantidad());
                inventarioRepository.save(inv);
            });

        kafkaTemplate.send("inventario.liberado",
            new StockLiberadoEvento(evento.getPedidoId()));
    }
}
```

### Paso 3: Pedidos escucha resultados finales

```java
@Service
public class PedidoResultadoListener {

    @Autowired private PedidoService pedidoService;

    @KafkaListener(topics = "pagos.procesados")
    public void onPagoExitoso(PagoProcesadoEvento evento) {
        pedidoService.confirmarPedido(evento.getPedidoId());
    }

    @KafkaListener(topics = {"inventario.fallido", "inventario.liberado"})
    public void onFallo(SagaFallidoEvento evento) {
        pedidoService.cancelarPedido(evento.getPedidoId(), evento.getMotivo());
    }
}
```

---

## 16.6 Patrón Outbox — garantizar que el evento se publica

**Problema:** si el servicio guarda en BD y luego falla antes de publicar el evento en Kafka, la saga se rompe.

```java
// MAL — no atómico: si Kafka falla, el pedido queda en BD sin evento
pedidoRepository.save(pedido);          // paso 1: BD ok
kafkaTemplate.send("pedidos.creados");  // paso 2: ¿y si Kafka falla aquí?
```

**Solución — Outbox pattern:** guardar el evento en la misma transacción de BD, en una tabla `outbox`. Un proceso separado lee la tabla y publica en Kafka.

```java
@Service
@Transactional
public class PedidoService {

    @Autowired private PedidoRepository pedidoRepository;
    @Autowired private OutboxRepository outboxRepository;  // misma BD

    public Pedido crearPedido(CrearPedidoRequest request) {
        Pedido pedido = pedidoRepository.save(...);

        // Guardar evento en la misma transacción — atómico
        outboxRepository.save(OutboxEvent.builder()
            .aggregateType("Pedido")
            .aggregateId(pedido.getId().toString())
            .eventType("PedidoCreado")
            .payload(toJson(new PedidoCreadoEvento(pedido)))
            .build());

        return pedido;
        // La transacción hace COMMIT de pedido + outbox juntos
    }
}
```

```java
// Publicador — lee la tabla outbox y publica en Kafka
@Component
public class OutboxPublisher {

    @Autowired private OutboxRepository outboxRepository;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)  // cada segundo
    @Transactional
    public void publicarEventosPendientes() {
        List<OutboxEvent> eventos = outboxRepository.findByPublicadoFalse();
        eventos.forEach(evento -> {
            kafkaTemplate.send(evento.getEventType(), evento.getPayload());
            evento.setPublicado(true);
        });
        outboxRepository.saveAll(eventos);
    }
}
```

> `[ADVERTENCIA]` El Outbox pattern tiene una limitación: el publicador puede enviar el mismo evento dos veces si falla entre el envío y el mark como publicado. Los consumidores deben ser **idempotentes**.

---

## 16.7 Idempotencia en consumidores

```java
@Service
public class InventarioSagaListener {

    @Autowired private EventoProceadoRepository procesados;

    @KafkaListener(topics = "pedidos.creados")
    @Transactional
    public void onPedidoCreado(PedidoCreadoEvento evento) {
        // Verificar si ya procesamos este evento (idempotencia)
        if (procesados.existsByEventoId(evento.getEventoId())) {
            log.warn("Evento duplicado ignorado: {}", evento.getEventoId());
            return;
        }

        // Procesar...
        reservarStock(evento);

        // Marcar como procesado en la misma transacción
        procesados.save(new EventoProcesado(evento.getEventoId()));
    }
}
```

---

## 16.8 Resumen de patrones

| Patrón | Cuándo usarlo |
|--------|--------------|
| **Saga Coreografía** | Flujos simples con pocos pasos, equipos autónomos |
| **Saga Orquestación** | Flujos complejos, necesidad de visibilidad central |
| **Outbox** | Siempre que publiques eventos desde una transacción de BD |
| **Idempotencia** | Siempre en consumidores de mensajes |
| **2PC** | Evitarlo en microservicios — solo en sistemas legacy muy controlados |

> `[EXAMEN]` La diferencia clave entre Saga y 2PC: Saga usa **transacciones compensatorias** (deshacer lo ya hecho), no rollback global. La consistencia es **eventual**, no inmediata.

---

← [Parte 15 — Testing](./15-testing.md) | [Volver al índice](./README.md)
