# 7.8 Patrones de fallo y troubleshooting de Spring Cloud Bus

← [7.7 Tracing y observabilidad de Spring Cloud Bus](sc-bus-observabilidad.md) | [Índice](README.md) | [7.9 Testing de Spring Cloud Bus](sc-bus-testing.md) →

## Introducción

Los fallos de Spring Cloud Bus en producción tienen un patrón característico: el sistema falla silenciosamente. No hay excepciones visibles, el endpoint responde con 200, pero la configuración no se actualiza en uno o varios nodos. Esto ocurre porque Bus separa el mecanismo de entrega (el broker) del mecanismo de procesamiento (el listener en cada nodo): si el broker entrega el evento pero el nodo no lo procesa correctamente, no hay error en el emisor. Los cinco patrones de fallo más frecuentes —nodo que no recibe el refresh, beans que no se actualizan a pesar de recibir el evento, eventos personalizados no recibidos, loop de refresh y diagnóstico activo— tienen cada uno una causa raíz distinta y requieren una estrategia de diagnóstico específica.

> [ADVERTENCIA] Antes de diagnosticar un fallo de Bus, verifica que `spring.cloud.bus.ack.enabled=true` y `spring.cloud.bus.trace.enabled=true` están activos en el entorno donde falla. Sin estos, el diagnóstico es ciego y consume mucho más tiempo (ver sc-bus-observabilidad.md).

## Representación visual

La tabla siguiente es la referencia de diagnóstico rápido: síntoma observable, causa raíz más probable y acción correctiva.

| Síntoma | Causa raíz más probable | Diagnóstico | Corrección |
|---|---|---|---|
| Nodo no recibe refresh, no aparece en ACKs | `bus.id` duplicado — el nodo descarta su propio evento | Comparar `bus.id` de todos los nodos | Fijar `bus.id` único con `${HOSTNAME}` |
| Nodo no recibe refresh, binding incorrecto | `springCloudBusInput` no conectado al broker | Ver `/actuator/bindings`; buscar errores de binder | Verificar configuración de broker y binder |
| Refresh recibido (en ACKs) pero bean no actualiza | Bean sin `@RefreshScope` | Buscar el bean en logs de Context refresh | Añadir `@RefreshScope` al bean |
| Evento personalizado no llega a ningún nodo | Paquete no declarado en `@RemoteApplicationEventScan` | `InvalidDefinitionException` en logs del receptor | Añadir el paquete al escaneo |
| Loop de refresh infinito | Nodo se auto-envía eventos que él mismo procesa | Ver eventos con `origin=` del mismo nodo repetidos | Verificar `bus.id` único; usar `busrefresh` no local |
| Refresh funciona una vez pero no las siguientes | Consumer group Kafka sin offset commit | Ver lag del consumer group en Kafka | Configurar `enable-auto-commit=true` o manual commit |

El árbol de decisión siguiente guía el diagnóstico paso a paso:

```
¿Los ACKs llegan al nodo emisor para el evento problemático?
│
├── NO → El nodo receptor no está conectado al bus
│        ├── ¿Aparece springCloudBusInput en /actuator/bindings?
│        │   ├── NO → Problema de binder: broker no accesible o mal configurado
│        │   └── SÍ → ¿bus.id duplicado? (comparar logs de inicio)
│        │            ├── SÍ → Nodo descarta evento porque parece propio
│        │            └── NO → Revisar consumer group y particiones Kafka
│        └── ¿Hay errores de conexión al broker en el log?
│            ├── SÍ → Broker caído o credenciales incorrectas
│            └── NO → Problema de serialización del evento
│
└── SÍ → El nodo recibió el evento pero no actualizó el bean
         ├── ¿El bean tiene @RefreshScope?
         │   ├── NO → Añadir @RefreshScope
         │   └── SÍ → ¿El bean se instancia en el constructor (no en @PostConstruct)?
         │            ├── NO → Mover la lectura de propiedades al constructor
         │            └── SÍ → ¿Hay un proxy CGLIB que impide la re-instanciación?
         └── Ver logs de ContextRefresher para confirmación del refresh
```

## Ejemplo central

El ejemplo siguiente muestra configuración de diagnóstico activo y los logs esperados para cada patrón de fallo.

```xml
<!-- pom.xml — dependencias para diagnóstico -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml — configuración de diagnóstico activo
spring:
  application:
    name: payment-service
  cloud:
    bus:
      enabled: true
      # bus.id robusto para evitar colisiones en Kubernetes
      id: ${spring.application.name}:${spring.profiles.active:default}:${HOSTNAME:${random.value}}
      trace:
        enabled: true
      ack:
        enabled: true
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: ${spring.application.name}-bus-consumers
      auto-offset-reset: latest
      enable-auto-commit: true

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, bindings, health, info, env

logging:
  level:
    # Ver todos los eventos Bus
    org.springframework.cloud.bus: DEBUG
    # Ver el proceso de refresh del contexto
    org.springframework.cloud.context.refresh: DEBUG
    # Ver la conexión al binder de Kafka
    org.springframework.cloud.stream.binder.kafka: INFO
```

```java
// Patrón de fallo 1: bus.id duplicado
// INCORRECTO — puede generar duplicados en contenedores
// spring.cloud.bus.id=payment-service:production (igual en todos los pods)

// CORRECTO — único por pod en Kubernetes
// spring.cloud.bus.id=${spring.application.name}:${spring.profiles.active:default}:${HOSTNAME}

// Logs que indican bus.id duplicado:
// WARN  o.s.c.b.BusAutoConfiguration : Discarding event from self: id=evt-123, origin=payment-service:production:pod-abc
// (el nodo descarta el evento porque su origin coincide con su propio bus.id)
```

```java
// src/main/java/com/example/payment/config/PaymentConfig.java
package com.example.payment.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

// CORRECTO: @RefreshScope en el bean
@Component
@RefreshScope
public class PaymentConfig {

    private final double feeRate;
    private final String provider;

    // CORRECTO: leer propiedades en el constructor
    // Al recibir RefreshRemoteApplicationEvent, Spring llama a este constructor
    // con los nuevos valores del Config Server
    public PaymentConfig(
            @Value("${payment.fee-rate:0.015}") double feeRate,
            @Value("${payment.provider:stripe}") String provider) {
        this.feeRate = feeRate;
        this.provider = provider;
    }

    public double getFeeRate() { return feeRate; }
    public String getProvider() { return provider; }
}

// INCORRECTO: sin @RefreshScope — el bean no se re-instancia nunca
// @Component
// public class PaymentConfigWrong {
//     @Value("${payment.fee-rate:0.015}")
//     private double feeRate; // este valor no cambia tras el refresh
// }
```

```java
// src/main/java/com/example/payment/events/PaymentModeChangeEvent.java
package com.example.payment.events;

import org.springframework.cloud.bus.event.RemoteApplicationEvent;

// Evento personalizado — DEBE estar en paquete registrado con @RemoteApplicationEventScan
public class PaymentModeChangeEvent extends RemoteApplicationEvent {

    private String newMode;

    protected PaymentModeChangeEvent() {}

    public PaymentModeChangeEvent(Object source, String originService, String newMode) {
        super(source, originService);
        this.newMode = newMode;
    }

    public String getNewMode() { return newMode; }
    public void setNewMode(String newMode) { this.newMode = newMode; }
}
```

```java
// src/main/java/com/example/payment/PaymentApplication.java
package com.example.payment;

import com.example.payment.events.PaymentModeChangeEvent;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.bus.jackson.RemoteApplicationEventScan;

@SpringBootApplication
// SIN ESTO: InvalidDefinitionException al deserializar PaymentModeChangeEvent
// CON ESTO: Bus puede deserializar correctamente el evento personalizado
@RemoteApplicationEventScan(basePackages = "com.example.payment.events")
public class PaymentApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentApplication.class, args);
    }
}
```

Los logs esperados para un refresh correcto son:

```
# En el nodo emisor (con trace.enabled=true):
DEBUG o.s.c.b.BusAutoConfiguration : Publishing RefreshRemoteApplicationEvent id=evt-abc123, destination=**

# En cada nodo receptor:
DEBUG o.s.c.b.BusAutoConfiguration : Received RefreshRemoteApplicationEvent id=evt-abc123, origin=config-server:default:xyz
DEBUG o.s.c.c.r.ContextRefresher : Keys changed: [payment.fee-rate]
INFO  c.e.payment.config.PaymentConfig : PaymentConfig instanciado con feeRate=0.018

# ACK de vuelta al emisor:
DEBUG o.s.c.b.BusAutoConfiguration : Sending AckRemoteApplicationEvent for evt-abc123 from payment-service:production:pod-1
```

## Tabla de elementos clave

La tabla siguiente recoge los parámetros de diagnóstico más útiles en troubleshooting de Bus.

| Elemento | Valor/Tipo | Descripción |
|---|---|---|
| `spring.cloud.bus.trace.enabled=true` | Boolean | Activa log de eventos enviados y recibidos. Primer paso de diagnóstico. |
| `spring.cloud.bus.ack.enabled=true` | Boolean | Verifica qué nodos recibieron el evento. Permite detectar nodos aislados. |
| `/actuator/bindings` | Endpoint Stream | Muestra el estado de `springCloudBusInput` y `springCloudBusOutput`. |
| `/actuator/env` | Endpoint | Verifica si los valores del Config Server están actualizados en este nodo. |
| `logging.level.org.springframework.cloud.bus=DEBUG` | Log level | Log detallado de todos los eventos Bus. |
| `logging.level.org.springframework.cloud.context.refresh=DEBUG` | Log level | Log del proceso de refresh del contexto de Spring. |
| `spring.cloud.bus.id` duplicado | Síntoma | El nodo descarta eventos con log `Discarding event from self`. |
| `@RemoteApplicationEventScan` ausente | Síntoma | `InvalidDefinitionException` en logs del receptor del evento personalizado. |

## Buenas y malas prácticas

**Hacer:**

- Verificar `/actuator/bindings` como primer paso de diagnóstico cuando un nodo no recibe eventos. Si `springCloudBusInput` no aparece en estado `running`, el problema está en la conectividad con el broker, no en la configuración de Bus.
- Comparar los `bus.id` de todos los nodos al inicio de un diagnóstico. En Kubernetes, ejecutar `kubectl logs -l app=my-service | grep 'spring.cloud.bus.id'` en todos los pods muestra inmediatamente si hay duplicados.
- Usar `/actuator/env` en el nodo sospechoso para verificar si los valores del Config Server están actualizados. Si `/actuator/env` muestra los nuevos valores pero el bean sigue con los antiguos, el problema es la ausencia de `@RefreshScope`.

**Evitar:**

- No reiniciar pods como primera respuesta a un fallo de refresh. El reinicio oculta el problema real (bus.id duplicado, binding incorrecto) y no lo resuelve: el pod reiniciado puede volver a tener el mismo problema si la configuración no se corrige.
- No deshabilitar `ack.enabled` durante el diagnóstico activo. Los ACKs son la única forma objetiva de saber qué nodos procesaron el evento. Sin ellos, el diagnóstico se basa en asumir qué nodos están funcionando, lo que prolonga el tiempo de resolución.
- No ignorar el patrón de loop de refresh. Si un nodo aparece continuamente en los logs con `Sending RefreshRemoteApplicationEvent` y luego `Received RefreshRemoteApplicationEvent` en ciclo, está procesando sus propios eventos, lo que indica un `bus.id` no configurado correctamente que hace que el nodo no se reconozca a sí mismo como origen.

---

← [7.7 Tracing y observabilidad de Spring Cloud Bus](sc-bus-observabilidad.md) | [Índice](README.md) | [7.9 Testing de Spring Cloud Bus](sc-bus-testing.md) →
