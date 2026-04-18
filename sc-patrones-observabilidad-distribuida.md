# 13.8 Observabilidad distribuida: trazas, métricas, logs y health checks

← [13.7 Patrones de resiliencia](sc-patrones-resilience-design-patterns.md) | [Índice](README.md) | [13.9 Patrones de seguridad](sc-patrones-seguridad-patrones.md) →

---

## Introducción

En un sistema distribuido, un fallo raramente tiene una causa única y localizada: puede ser la combinación de latencia en un servicio, saturación de conexiones en otro y un despliegue defectuoso en un tercero. La observabilidad distribuida — los tres pilares de trazas, métricas y logs correlacionados — es lo que transforma la depuración de un sistema de "intentar adivinar" a "seguir la evidencia". Spring Boot Actuator con Micrometer y Prometheus es el stack estándar en el ecosistema Spring Cloud.

## Los tres pilares de la observabilidad

> [CONCEPTO] **Distributed Tracing (Micrometer Tracing)**: el tracing distribuido propaga un identificador de traza (`traceId`) a través de todas las llamadas de un flujo — HTTP, mensajería, bases de datos. Cada operación dentro de ese flujo genera un `spanId` hijo. Juntos, el árbol de spans reconstruye el flujo completo de una petición a través de múltiples servicios, con tiempos exactos de cada operación.

Si un servicio intermedio no propaga las cabeceras de tracing (`traceparent` en W3C TraceContext, o `X-B3-TraceId` en B3), la traza se rompe y es imposible correlacionar las llamadas downstream con la petición original.

> [CONCEPTO] **Log Aggregation**: los logs de cada microservicio son locales al pod/contenedor. En producción, los pods se destruyen y crean constantemente, por lo que los logs locales se pierden. La agregación de logs (ELK: Elasticsearch + Logstash + Kibana, o Loki + Grafana) recopila los logs de todos los servicios en un almacén central. El `traceId` en cada línea de log permite correlacionar los logs de diferentes servicios pertenecientes al mismo flujo.

> [CONCEPTO] **Metrics (Micrometer + Prometheus)**: Micrometer es la capa de abstracción de métricas de Spring Boot, análoga a SLF4J para logs. Exporta métricas en el formato que Prometheus entiende mediante el endpoint `/actuator/prometheus`. Los cuatro tipos de métricas de Micrometer: **Counter** (cuenta de eventos), **Gauge** (valor actual de un estado), **Timer** (distribución de duraciones), **DistributionSummary** (distribución de valores como tamaño de payload).

## Health Check API

> [CONCEPTO] **Health Check API**: Spring Boot Actuator expone `/actuator/health` con el estado de salud del servicio y sus dependencias (BD, broker, servicios externos). En Kubernetes, este endpoint se usa como base para las **liveness probe** (¿el proceso está vivo?) y **readiness probe** (¿el servicio puede aceptar tráfico?). La diferencia es crítica: liveness failure reinicia el pod; readiness failure lo saca del pool de carga pero no lo reinicia.

| Probe | Endpoint típico | Acción en fallo | Cuándo falla |
|---|---|---|---|
| Liveness | `/actuator/health/liveness` | Restart del pod | Deadlock, estado corrupto irrecuperable |
| Readiness | `/actuator/health/readiness` | Saca del balanceador | BD no disponible, calentamiento inicial |

## Ejemplo central: instrumentación completa con Micrometer Tracing y Actuator

El siguiente ejemplo configura la observabilidad completa en un microservicio Spring Boot: tracing distribuido con Micrometer + Zipkin, métricas custom, logs estructurados con traceId y health checks separados para liveness y readiness.

```xml
<!-- pom.xml — dependencias de observabilidad -->
<!-- Spring Boot Actuator -->
<!-- <dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency> -->
<!-- Micrometer Tracing con OpenTelemetry bridge -->
<!-- <dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency> -->
<!-- Exportador OTLP para enviar trazas a Zipkin/Jaeger -->
<!-- <dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency> -->
<!-- Micrometer Registry Prometheus -->
<!-- <dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency> -->
```

```yaml
# application.yml — configuración de observabilidad completa

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true              # habilita /actuator/health/liveness y /readiness
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  tracing:
    sampling:
      probability: 1.0             # 100% en desarrollo; 0.1 (10%) en producción
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

spring:
  application:
    name: orders-service           # aparece como 'service.name' en las trazas

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%X{traceId:-},%X{spanId:-}] %-5level %logger{36} - %msg%n"
```

```java
// Métricas custom de negocio con Micrometer
package com.example.orders.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;
import java.util.concurrent.TimeUnit;

@Component
public class OrderMetrics {

    private final Counter ordersPlaced;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;

    public OrderMetrics(MeterRegistry registry) {
        // Counter: cuántos pedidos se han creado
        this.ordersPlaced = Counter.builder("orders.placed.total")
            .description("Total number of orders placed")
            .tag("service", "orders")
            .register(registry);

        // Counter con tag dinámico: pedidos fallidos por tipo de fallo
        this.ordersFailed = Counter.builder("orders.failed.total")
            .description("Total number of failed orders")
            .tag("service", "orders")
            .register(registry);

        // Timer: distribución del tiempo de procesamiento de pedidos
        this.orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Time taken to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
            .register(registry);
    }

    public void recordOrderPlaced() {
        ordersPlaced.increment();
    }

    public void recordOrderFailed(String reason) {
        Counter.builder("orders.failed.total")
            .tag("reason", reason)
            .register(ordersPlaced.getId().getRegistry())
            .increment();
    }

    public <T> T recordProcessingTime(java.util.concurrent.Callable<T> supplier) throws Exception {
        return orderProcessingTime.recordCallable(supplier);
    }
}
```

```java
// Health Indicator custom: verifica la disponibilidad del servicio de inventario
package com.example.orders.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import java.time.Duration;

@Component("inventoryService")  // aparece en /actuator/health como "inventoryService"
public class InventoryServiceHealthIndicator implements HealthIndicator {

    private final WebClient inventoryClient;

    public InventoryServiceHealthIndicator(WebClient.Builder webClientBuilder) {
        this.inventoryClient = webClientBuilder
            .baseUrl("http://inventory-service")
            .build();
    }

    @Override
    public Health health() {
        try {
            inventoryClient.get()
                .uri("/actuator/health")
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofSeconds(2))
                .block();
            return Health.up()
                .withDetail("url", "http://inventory-service/actuator/health")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .withDetail("url", "http://inventory-service/actuator/health")
                .build();
        }
    }
}
```

```java
// Uso de tracing manual con Observation API de Micrometer
package com.example.orders.service;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final ObservationRegistry observationRegistry;
    private final OrderMetrics orderMetrics;

    public OrderService(ObservationRegistry observationRegistry, OrderMetrics orderMetrics) {
        this.observationRegistry = observationRegistry;
        this.orderMetrics = orderMetrics;
    }

    public Order placeOrder(CreateOrderCommand command) {
        // Observation crea automáticamente un span hijo en el contexto de tracing actual
        return Observation.createNotStarted("order.place", observationRegistry)
            .lowCardinalityKeyValue("customerId.hashed",
                String.valueOf(command.getCustomerId().hashCode()))
            .observe(() -> {
                Order order = Order.place(command.getCustomerId(), command.getLines());
                orderMetrics.recordOrderPlaced();
                return order;
            });
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Incluir el `traceId` en todas las líneas de log para correlacionar logs entre servicios.
- Usar sampling probabilístico en producción (10-20%) para reducir el volumen de trazas sin perder visibilidad estadística.
- Separar los health checks en liveness y readiness con semánticas correctas — un servicio sin BD debería fallar readiness, no liveness.
- Usar tags de alta cardinalidad (userId, orderId) en métricas solo con `DistributionSummary` o bajo control estricto — cada valor único crea una nueva serie temporal en Prometheus.

**Malas prácticas:**
- Tags de alta cardinalidad en contadores (tag `userId` con millones de usuarios = millones de series en Prometheus → explosión de memoria).
- No propagar las cabeceras de tracing en mensajes de Kafka/RabbitMQ — rompe la visibilidad del flujo asíncrono.
- Exponer `/actuator/prometheus` sin autenticación en producción.

## Verificación y práctica

> [EXAMEN] 1. ¿Cómo propaga Micrometer Tracing el traceId y spanId a través de llamadas HTTP, y qué pierdes si un servicio no instrumenta las cabeceras?

> [EXAMEN] 2. ¿Por qué el logging local es insuficiente en microservicios, y qué arquitectura de log aggregation permite correlacionar trazas entre servicios?

> [EXAMEN] 3. ¿Cuál es la diferencia entre liveness probe y readiness probe en Kubernetes, y qué endpoint de Actuator se usa para cada una?

> [EXAMEN] 4. ¿Qué cuatro tipos de métricas provee Micrometer, y cuándo usarías un Timer en lugar de un Counter?

> [EXAMEN] 5. ¿Por qué los tags de alta cardinalidad en métricas de Prometheus son problemáticos, y cómo los evitas?

---

← [13.7 Patrones de resiliencia](sc-patrones-resilience-design-patterns.md) | [Índice](README.md) | [13.9 Patrones de seguridad](sc-patrones-seguridad-patrones.md) →
