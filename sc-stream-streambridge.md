# 6.10 StreamBridge — producción de mensajes programática

← [6.9 Polling y batch consumers](sc-stream-polling-batch.md) | [Índice (README.md)](README.md) | [6.11 Multi-binder](sc-stream-multi-binder.md) →

## Introducción

`StreamBridge` resuelve el problema de producir mensajes desde código imperativo que no pertenece al modelo funcional reactivo. El modelo funcional de Spring Cloud Stream (6.1) gestiona el ciclo de vida completo de los bindings: los `Supplier<O>` producen mensajes periódicamente, los `Function<I,O>` producen como reacción a mensajes entrantes. Sin embargo, hay escenarios donde la producción de mensajes ocurre en lugares donde no existe un ciclo de vida de binding natural: un REST controller que produce un evento al recibir una petición HTTP, un job de Quartz que genera alertas, o un scheduler que emite estados periódicos de forma imperativa.

`StreamBridge` es un componente inyectable que permite enviar mensajes a cualquier binding de salida de forma programática, sin necesidad de un bean funcional. No es un reemplazo del modelo `Supplier`; es un complemento para cuando el productor está controlado por lógica imperativa externa al pipeline de mensajería.

> [CONCEPTO] StreamBridge no reemplaza al modelo funcional. Si el caso de uso es "producir mensajes en respuesta a otros mensajes", usar `Function<I,O>`. Si es "producir mensajes desde código imperativo (REST, scheduler)", usar `StreamBridge`.

## Representación visual

La diferencia entre el modelo Supplier y StreamBridge radica en quién controla el ciclo de vida de la producción. El diagrama siguiente muestra ambos enfoques.

```
MODELO SUPPLIER (modelo funcional):
────────────────────────────────────────────────────────
Spring Scheduler (poller)
    │  fixed-delay: 1000ms
    ▼
@Bean Supplier<T>  ──► statusSupplier-out-0 ──► Broker
(framework controla cuándo se invoca)

STREAMBRIDGE (código imperativo):
────────────────────────────────────────────────────────
REST Controller  ──► streamBridge.send("orders-out", event)
                                │
                                ▼
                 ┌──────────────────────────┐
                 │  StreamBridge            │
                 │  Caché de bindings       │
                 │  dinámicos               │
                 └──────────┬───────────────┘
                            │
                            ▼
                    orders-out ──► Broker

(la aplicación controla cuándo se produce)
```

## Ejemplo central

El siguiente ejemplo implementa un servicio REST que usa `StreamBridge` para producir eventos de pedido al recibir peticiones HTTP, y un ejemplo de envío a destinos dinámicos determinados en runtime.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**application.yml**
```yaml
spring:
  cloud:
    stream:
      bindings:
        orders-out:                       # binding de salida para StreamBridge
          destination: orders
          content-type: application/json
        # Para destinos dinámicos, no es necesario declarar el binding en yml
        # SpringBridge los crea en la caché la primera vez que se usan
      dynamic-destination-cache-size: 10  # tamaño máximo de la caché de bindings dinámicos
      kafka:
        binder:
          brokers: localhost:9092
```

**OrderEvent.java**
```java
package com.example.streambridge;

public record OrderEvent(String orderId, String customerId, double amount, String region) {}
```

**OrderController.java**
```java
package com.example.streambridge;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.http.ResponseEntity;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/orders")
public class OrderController {

    private final StreamBridge streamBridge;

    public OrderController(StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }

    /**
     * Envío estático: usa el binding 'orders-out' configurado en application.yml.
     * El binding se crea una sola vez y se reutiliza en todos los envíos posteriores.
     */
    @PostMapping
    public ResponseEntity<String> createOrder(@RequestBody OrderEvent event) {
        boolean sent = streamBridge.send(
            "orders-out",
            MessageBuilder.withPayload(event)
                .setHeader("X-Source", "REST")
                .setHeader("X-Region", event.region())
                .build()
        );

        if (sent) {
            return ResponseEntity.ok("Order accepted: " + event.orderId());
        } else {
            return ResponseEntity.internalServerError().body("Failed to send order");
        }
    }

    /**
     * Envío dinámico: el binding destino se determina en runtime según la región del pedido.
     * StreamBridge crea el binding dinámicamente si no existe en la caché.
     * La caché tiene un límite de 'dynamic-destination-cache-size' (default: 10).
     *
     * Útil para fan-out a múltiples topics según propiedades del mensaje.
     */
    @PostMapping("/regional/{region}")
    public ResponseEntity<String> createRegionalOrder(
            @PathVariable String region,
            @RequestBody OrderEvent event) {

        // El binding 'orders-eu', 'orders-us', etc. se crea dinámicamente
        String bindingName = "orders-" + region.toLowerCase();

        streamBridge.send(
            bindingName,
            event  // StreamBridge serializa automáticamente con contentType por defecto
        );

        return ResponseEntity.ok("Regional order sent to " + bindingName);
    }
}
```

**StreamBridgeApplication.java**
```java
package com.example.streambridge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StreamBridgeApplication {
    public static void main(String[] args) {
        SpringApplication.run(StreamBridgeApplication.class, args);
    }
}
```

## Tabla de elementos clave

`StreamBridge` tiene pocas propiedades de configuración propias, pero interacciona con la configuración de bindings y binders. La tabla siguiente cubre los puntos de configuración relevantes.

| Parámetro / Método | Tipo | Default | Descripción |
|-------------------|------|---------|-------------|
| `StreamBridge.send(String bindingName, Object data)` | Método | — | Envía `data` al binding especificado. Serializa con el `contentType` del binding o `application/json` por defecto. Retorna `true` si el envío fue exitoso. |
| `StreamBridge.send(String bindingName, Message<?> message)` | Método | — | Envía un `Message` completo con headers personalizados al binding especificado. |
| `StreamBridge.send(String bindingName, Object data, MimeType mimeType)` | Método | — | Envía con un MIME type específico, ignorando el `contentType` del binding. |
| `spring.cloud.stream.dynamic-destination-cache-size` | `int` | `10` | Número máximo de bindings dinámicos que StreamBridge mantiene en caché. Superar este límite elimina los bindings más antiguos de la caché (LRU). |
| `spring.cloud.stream.bindings.<name>.contentType` | `String` | `application/json` | Tipo de contenido usado por StreamBridge al serializar el payload del binding. |

## Buenas y malas prácticas

**Hacer:**

- Inyectar `StreamBridge` directamente donde se necesite (REST controller, job scheduler). Es un bean de Spring estándar, thread-safe, y reutilizable en múltiples componentes sin configuración adicional.
- Declarar en `application.yml` los bindings de salida que `StreamBridge` usará frecuentemente. Aunque `StreamBridge` puede crear bindings dinámicamente, los bindings declarados en yml tienen configuración de producer explícita (particionado, sync, compresión) que los bindings dinámicos no tienen.
- Usar `send(bindingName, Message<?>)` cuando se necesita propagar cabeceras de correlación o tracing. El método con Object solo serializa el payload; el método con Message preserva las cabeceras originales.

**Evitar:**

- Usar destinos dinámicos con un número ilimitado de valores distintos (por ejemplo, un `orderId` diferente por pedido). La caché de bindings dinámicos tiene un tamaño máximo (`dynamic-destination-cache-size`). Con muchos destinos distintos, el LRU desaloja bindings constantemente, generando overhead de creación de conexiones en el broker.
- Usar `StreamBridge` como sustituto de `Function<I,O>` en un pipeline de transformación de mensajes. Si el productor imperativo está dentro del mismo servicio que el consumer, la comunicación debería ser directa (llamada de método) o mediante un bean funcional, no a través del broker (ida y vuelta innecesaria con latencia de red y serialización).

## Comparación: Supplier vs StreamBridge

Ambos mecanismos producen mensajes, pero tienen semánticas y casos de uso diferentes. Elegir el incorrecto complica innecesariamente el código.

| Característica | `Supplier<O>` | `StreamBridge` |
|----------------|--------------|---------------|
| Controlado por | Framework (poller con schedule) | Código imperativo de la aplicación |
| Trigger | Temporal (fixed-delay, cron) | Evento del negocio (HTTP, DB, scheduler propio) |
| Binding | Declarativo e implícito por naming | Explícito en cada llamada `send()` |
| Caso de uso | Health beats, reportes periódicos, CDC polling | REST API, Quartz jobs, respuesta a eventos externos |
| Configuración de rate | `spring.cloud.stream.poller.fixed-delay` | El llamador controla cuándo invoca `send()` |

> [EXAMEN] Pregunta frecuente: "Tienes un REST endpoint que debe publicar un evento en Kafka al recibir una petición POST. ¿Usarías Supplier o StreamBridge?" Respuesta correcta: `StreamBridge`. Un `Supplier` se invoca periódicamente por el framework; no puede reaccionar a eventos externos como una petición HTTP. `StreamBridge` es el mecanismo diseñado para este caso.

---

← [6.9 Polling y batch consumers](sc-stream-polling-batch.md) | [Índice (README.md)](README.md) | [6.11 Multi-binder](sc-stream-multi-binder.md) →
