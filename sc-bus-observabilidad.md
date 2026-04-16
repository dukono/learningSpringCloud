# 7.7 Tracing y observabilidad de Spring Cloud Bus

← [7.6 Seguridad del endpoint actuator de Spring Cloud Bus](sc-bus-seguridad-endpoint.md) | [Índice](README.md) | [7.8 Patrones de fallo y troubleshooting de Spring Cloud Bus](sc-bus-troubleshooting.md) →

## Introducción

Diagnosticar por qué un nodo concreto no recibió un evento de refresh es uno de los problemas más frustrantes de Spring Cloud Bus en producción: el evento se publicó, el broker lo enrutó, pero el bean sigue con valores antiguos. Sin observabilidad activa, el diagnóstico requiere revisar logs de todos los nodos simultáneamente y correlacionar manualmente los eventos. Spring Cloud Bus proporciona dos mecanismos específicos para este problema: `spring.cloud.bus.trace.enabled` registra cada evento que el nodo propaga, y `spring.cloud.bus.ack.enabled` hace que cada nodo publique un `AckRemoteApplicationEvent` al procesar un evento, creando un rastro auditble de qué nodos procesaron qué eventos. Sobre estos mecanismos básicos, la integración con Micrometer Tracing y Zipkin permite correlacionar el ciclo de vida completo de un evento Bus con las trazas distribuidas del resto del sistema.

> [CONCEPTO] `spring.cloud.bus.trace.enabled` y `spring.cloud.bus.ack.enabled` son complementarios: trace registra el evento cuando el nodo lo propaga al bus, y ack registra cuando otro nodo lo recibe. Juntos permiten ver el viaje completo del evento.

## Representación visual

El diagrama siguiente muestra qué eventos se registran con trace y ack activados, y cómo se correlacionan en un sistema de trazas distribuidas.

```
Nodo emisor (config-server)
│
│ POST /actuator/busrefresh
│
├─► Publica RefreshRemoteApplicationEvent en springCloudBus
│   [trace] BusRefreshListener: Sending RefreshRemoteApplicationEvent id=abc-123
│
│                    Broker
│                       │
│          ┌────────────┼────────────┐
│          ▼            ▼            ▼
│    service-a:1   service-a:2   service-b:1
│          │
│   [trace] Recibe RefreshRemoteApplicationEvent id=abc-123
│   [ack]   Publica AckRemoteApplicationEvent
│           { ackId=abc-123, ackDestination=service-a:1 }
│
│                    Broker
│                       │
│          ┌────────────┼────────────┐
│          ▼            ▼            ▼
│    config-server  service-a:2  service-b:1
│   [trace] Recibe AckRemoteApplicationEvent para id=abc-123

Con Micrometer Tracing:
  traceId: xyz-789 (propagado en el header del mensaje Bus)
    └── span: busrefresh-publish (config-server)
    └── span: busrefresh-receive (service-a:1)
    └── span: busrefresh-receive (service-a:2)
    └── span: busrefresh-receive (service-b:1)
```

## Ejemplo central

El ejemplo siguiente configura observabilidad completa de Bus: trace, ack y trazas distribuidas con Micrometer y Zipkin.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Micrometer Tracing con Brave (implementación Zipkin) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
    <!-- Exportador de spans a Zipkin vía HTTP -->
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-sender-urlconnection</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  application:
    name: config-server
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
      # Activar log de eventos propagados por este nodo
      trace:
        enabled: true
      # Activar confirmaciones de recepción (AckRemoteApplicationEvent)
      ack:
        enabled: true
        # destination-service vacío = escucha acks de todos los nodos
        destination-service:
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv, health, info, metrics
  tracing:
    # 1.0 = 100% de peticiones trazadas; reducir a 0.1 en producción de alto volumen
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

logging:
  level:
    # Logs de traza del bus: muestra cada evento propagado/recibido
    org.springframework.cloud.bus: DEBUG
    # Correlación de logs con traceId de Micrometer
    org.springframework.cloud.stream: INFO
  pattern:
    # Incluir traceId y spanId en cada línea de log
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

```java
// src/main/java/com/example/config/observability/BusEventObserver.java
package com.example.config.observability;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.bus.event.AckRemoteApplicationEvent;
import org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent;
import org.springframework.cloud.bus.event.SentApplicationEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class BusEventObserver {

    private static final Logger log = LoggerFactory.getLogger(BusEventObserver.class);

    // Registrar cuando este nodo envía un evento al bus
    // (se activa con spring.cloud.bus.trace.enabled=true)
    @EventListener
    public void onBusEventSent(SentApplicationEvent event) {
        log.info("Bus event ENVIADO: type={}, id={}, destination={}",
                 event.getType().getSimpleName(),
                 event.getId(),
                 event.getDestinationService());
    }

    // Registrar cuando este nodo recibe una confirmación de recepción
    // (se activa con spring.cloud.bus.ack.enabled=true)
    @EventListener
    public void onBusEventAcknowledged(AckRemoteApplicationEvent event) {
        log.info("Bus event ACK recibido: ackId={}, ackDestination={}, from={}",
                 event.getAckId(),
                 event.getAckDestination(),
                 event.getOriginService());
    }

    // Registrar cuando este nodo recibe un evento de refresh
    @EventListener
    public void onRefreshEvent(RefreshRemoteApplicationEvent event) {
        log.info("RefreshRemoteApplicationEvent recibido: id={}, from={}",
                 event.getId(),
                 event.getOriginService());
    }
}
```

Con `trace.enabled=true` y `ack.enabled=true`, el log del nodo emisor mostrará:

```
INFO  [config-server,abc123,def456] Bus event ENVIADO: type=RefreshRemoteApplicationEvent, id=evt-789, destination=**
INFO  [config-server,abc123,def456] Bus event ACK recibido: ackId=evt-789, ackDestination=service-a:production:node1, from=service-a:production:node1
INFO  [config-server,abc123,def456] Bus event ACK recibido: ackId=evt-789, ackDestination=service-a:production:node2, from=service-a:production:node2
INFO  [config-server,abc123,def456] Bus event ACK recibido: ackId=evt-789, ackDestination=service-b:production:node1, from=service-b:production:node1
```

Si un nodo no aparece en los ACKs, está desconectado del broker o tiene un problema de configuración del binding.

## Tabla de elementos clave

La tabla siguiente recoge todas las propiedades de observabilidad de Bus.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.bus.trace.enabled` | Boolean | `false` | Activa el log de cada evento propagado por este nodo. Registra `SentApplicationEvent`. |
| `spring.cloud.bus.ack.enabled` | Boolean | `false` | Cada nodo que recibe un evento publica un `AckRemoteApplicationEvent`. |
| `spring.cloud.bus.ack.destination-service` | String | `""` (todos) | Patrón de qué nodos deben enviar ACK. Vacío = todos los nodos. |
| `management.tracing.sampling.probability` | Double | `0.1` | Porcentaje de peticiones trazadas con Micrometer. `1.0` = 100%. |
| `management.zipkin.tracing.endpoint` | String | — | URL del servidor Zipkin para exportar trazas de Bus. |
| `logging.level.org.springframework.cloud.bus` | Level | `INFO` | Nivel de log del bus. `DEBUG` muestra el payload completo de cada evento. |
| `AckRemoteApplicationEvent.ackId` | String | — | ID del evento original confirmado. Permite correlacionar envío y recepción. |
| `AckRemoteApplicationEvent.ackDestination` | String | — | `bus.id` del nodo que confirma la recepción. |

## Buenas y malas prácticas

**Hacer:**

- Activar `ack.enabled=true` durante las primeras semanas de operación de Bus en producción para verificar que todos los nodos reciben los eventos correctamente. Una vez verificado el funcionamiento, se puede desactivar si el volumen de ACKs genera overhead en el broker.
- Incluir el patrón de log con `traceId` y `spanId` en la configuración de logging. Sin este patrón, correlacionar los logs de Bus de diferentes nodos para un mismo evento requiere buscar por `id` del evento manualmente, lo que consume tiempo en un incidente.
- Configurar un dashboard en Grafana o Kibana que muestre el tiempo entre el envío de un `RefreshRemoteApplicationEvent` y la recepción de todos los ACKs esperados. Una degradación en este tiempo indica problemas de rendimiento del broker antes de que se manifiesten como errores visibles.

**Evitar:**

- No activar `trace.enabled=true` en producción de forma permanente con `logging.level.org.springframework.cloud.bus=DEBUG`. Cada evento genera una línea de log por nodo, y en un clúster grande con refreshes frecuentes, esto puede saturar el sistema de logs y dificultar la búsqueda de errores reales.
- No confundir `spring.cloud.bus.trace.enabled` con Micrometer Tracing. El primero es logging de eventos Bus en texto plano; el segundo es trazado distribuido con spans y contextos de traza. Ambos son complementarios y resuelven necesidades diferentes: trace para diagnóstico operacional, Micrometer para correlación entre servicios.
- No asumir que la ausencia de errores en el log significa que todos los nodos recibieron el evento. Sin `ack.enabled=true`, no hay confirmación de recepción. Un nodo puede estar desconectado del broker sin generar ningún error en el nodo emisor.

---

← [7.6 Seguridad del endpoint actuator de Spring Cloud Bus](sc-bus-seguridad-endpoint.md) | [Índice](README.md) | [7.8 Patrones de fallo y troubleshooting de Spring Cloud Bus](sc-bus-troubleshooting.md) →
