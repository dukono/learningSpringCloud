# 6.12 Actuator y monitorización de bindings

← [6.11 Multi-binder](sc-stream-multi-binder.md) | [Índice (README.md)](README.md) | [6.13 Testing / Verificación de Spring Cloud Stream](sc-stream-testing.md) →

## Introducción

Spring Cloud Stream expone los bindings de mensajería como recursos operacionales gestionables mediante Spring Boot Actuator. El problema que resuelve es la rigidez operacional de los sistemas de mensajería: sin esta integración, detener el consumo de mensajes de un binding requiere reiniciar el servicio completo. Con el endpoint `/actuator/bindings`, los operadores pueden pausar, reanudar, detener o iniciar el consumo de cualquier binding individualmente sin interrupción de servicio.

Esto es especialmente valioso en escenarios de mantenimiento (pausar el consumo mientras se realiza una migración de base de datos), en gestión de incidentes (detener el procesamiento de un binding que está generando errores en cascada sin detener los demás bindings del servicio), y en operaciones controladas de deployment (drainar mensajes en vuelo antes de un despliegue).

> [PREREQUISITO] `spring-boot-starter-actuator` debe estar en el classpath. Spring Cloud Stream no activa los endpoints de Actuator automáticamente; requiere la dependencia de Actuator.

## Representación visual

Los endpoints de Actuator de Spring Cloud Stream exponen los bindings del servicio con su estado en tiempo real y permiten operaciones de ciclo de vida sobre ellos.

```
GET  /actuator/bindings
─────────────────────────────────────────────────────
[
  { "name": "processOrder-in-0",  "state": "running",  "paused": false },
  { "name": "processOrder-out-0", "state": "running",  "paused": false },
  { "name": "auditEvent-in-0",    "state": "stopped",  "paused": false }
]

POST /actuator/bindings/{bindingName}
─────────────────────────────────────────────────────
Body: { "state": "PAUSED" }   → pausa el consumo sin desconectar del broker
Body: { "state": "RESUMED" }  → reanuda el consumo tras una pausa
Body: { "state": "STOPPED" }  → detiene el binding y desconecta del broker
Body: { "state": "STARTED" }  → inicia un binding que estaba STOPPED

Estados posibles:
  running → PAUSED  → running  (pausa/reanuda sin desconexión)
  running → STOPPED → STARTED  (detiene/inicia con desconexión del broker)
```

## Ejemplo central

El siguiente ejemplo configura un servicio con el endpoint de bindings habilitado, métricas de Micrometer y un controlador de administración personalizado que usa el `BindingsEndpoint` programáticamente.

**pom.xml**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream-binder-kafka</artifactId>
    </dependency>
    <!-- Actuator requerido para los endpoints de bindings -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Para métricas con Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
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
    function:
      definition: processOrder;auditEvent
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processors
          producer:
            error-channel-enabled: true   # habilitar error channel para el binding de salida
        processOrder-out-0:
          destination: orders-processed
        auditEvent-in-0:
          destination: orders-processed
          group: audit-service
      kafka:
        binder:
          brokers: localhost:9092

management:
  endpoints:
    web:
      exposure:
        include: bindings,health,metrics,prometheus   # exponer endpoints relevantes
  endpoint:
    bindings:
      enabled: true                     # habilitar endpoint de bindings explícitamente
    health:
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}
```

**OrderEvent.java**
```java
package com.example.actuator;

public record OrderEvent(String orderId, String type, double amount) {}
```

**StreamConfig.java**
```java
package com.example.actuator;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;

import java.util.function.Consumer;
import java.util.function.Function;

@Configuration
public class StreamConfig {

    @Bean
    public Function<Message<OrderEvent>, Message<OrderEvent>> processOrder() {
        return message -> {
            System.out.printf("[processOrder] Processing: %s%n", message.getPayload().orderId());
            return message;
        };
    }

    @Bean
    public Consumer<Message<OrderEvent>> auditEvent() {
        return message -> {
            System.out.printf("[auditEvent] Auditing: %s%n", message.getPayload().orderId());
        };
    }
}
```

**BindingAdminController.java**
```java
package com.example.actuator;

import org.springframework.boot.actuate.endpoint.web.annotation.ControllerEndpoint;
import org.springframework.cloud.stream.endpoint.BindingsEndpoint;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Controlador que usa el BindingsEndpoint de Spring Cloud Stream
 * para operaciones de administración en tiempo real.
 *
 * Este componente muestra cómo interactuar con los bindings programáticamente.
 * En producción, las operaciones se realizan típicamente vía HTTP directamente
 * al endpoint /actuator/bindings/{bindingName}.
 */
@RestController
@RequestMapping("/admin/bindings")
public class BindingAdminController {

    private final BindingsEndpoint bindingsEndpoint;

    public BindingAdminController(BindingsEndpoint bindingsEndpoint) {
        this.bindingsEndpoint = bindingsEndpoint;
    }

    @GetMapping
    public ResponseEntity<Object> listBindings() {
        return ResponseEntity.ok(bindingsEndpoint.queryStates());
    }

    @PostMapping("/{bindingName}/pause")
    public ResponseEntity<String> pauseBinding(@PathVariable String bindingName) {
        // Pausa el consumo del binding sin desconectarse del broker
        // El offset no avanza mientras está pausado
        bindingsEndpoint.changeState(bindingName,
            BindingsEndpoint.State.PAUSED);
        return ResponseEntity.ok("Binding " + bindingName + " paused");
    }

    @PostMapping("/{bindingName}/resume")
    public ResponseEntity<String> resumeBinding(@PathVariable String bindingName) {
        bindingsEndpoint.changeState(bindingName,
            BindingsEndpoint.State.RESUMED);
        return ResponseEntity.ok("Binding " + bindingName + " resumed");
    }
}
```

**ActuatorApplication.java**
```java
package com.example.actuator;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ActuatorApplication {
    public static void main(String[] args) {
        SpringApplication.run(ActuatorApplication.class, args);
    }
}
```

## Tabla de elementos clave

Las propiedades de Actuator de Spring Cloud Stream se configuran bajo `management.*`. La tabla siguiente cubre las más relevantes para monitorización en producción.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `management.endpoints.web.exposure.include` | `String` | `health,info` | Lista de endpoints expuestos por HTTP. Incluir `bindings` para las operaciones de ciclo de vida. |
| `management.endpoint.bindings.enabled` | `boolean` | `true` | Habilita/deshabilita el endpoint `/actuator/bindings`. |
| `spring.cloud.stream.bindings.<name>.producer.error-channel-enabled` | `boolean` | `false` | Habilita el error channel para el binding de salida del producer. Permite recibir errores de envío en un canal de Spring Integration. |
| Métrica `spring.cloud.stream.binder.kafka.offset` | Gauge (Micrometer) | — | Lag actual del consumer group en Kafka. Registrada automáticamente por el binder Kafka. Visible en `/actuator/metrics/spring.cloud.stream.binder.kafka.offset`. |

## Buenas y malas prácticas

**Hacer:**

- Exponer el endpoint `/actuator/bindings` solo en redes internas o protegido con autenticación. Pausar o detener un binding de producción es una operación con impacto directo en el sistema; no debe ser accesible desde el exterior.
- Usar `PAUSED` en lugar de `STOPPED` para mantenimientos breves. `PAUSED` mantiene la conexión al broker y el consumer group activo; el offset no avanza pero el rebalanceo no se dispara. `STOPPED` desconecta del broker y puede provocar un rebalanceo del consumer group en Kafka que redistribuye particiones a otras instancias.
- Configurar alertas sobre la métrica de lag del consumer group (`spring.cloud.stream.binder.kafka.offset`). Un lag creciente es el indicador más temprano de un problema de throughput antes de que impacte en la latencia de extremo a extremo.

**Evitar:**

- Dejar el endpoint de Actuator de bindings habilitado sin autenticación en entornos de producción. Cualquier cliente que conozca la URL puede detener el consumo de mensajes del servicio, produciendo un incidente de disponibilidad sin necesidad de acceso al clúster Kafka o RabbitMQ.
- Usar `STOPPED` en scripts automatizados de deployment sin un mecanismo para volver a `STARTED` si el deployment falla. Si el deployment falla después de `STOPPED` y el servicio no se reinicia correctamente, el binding permanece detenido indefinidamente.

---

← [6.11 Multi-binder](sc-stream-multi-binder.md) | [Índice (README.md)](README.md) | [6.13 Testing / Verificación de Spring Cloud Stream](sc-stream-testing.md) →
