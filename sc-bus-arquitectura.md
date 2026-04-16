# 7.1 Arquitectura de Spring Cloud Bus

← [6.13 Testing / Verificación de Spring Cloud Stream](sc-stream-testing.md) | [Índice](README.md) | [7.2 Setup y dependencias de Spring Cloud Bus](sc-bus-setup.md) →

## Introducción

En un sistema de microservicios con configuración externalizada, propagar un cambio de configuración a cien instancias en producción manualmente es inviable: llamar a `/actuator/refresh` en cada pod genera acoplamiento operacional, introduce ventanas de inconsistencia y no escala. Spring Cloud Bus resuelve este problema añadiendo una capa de broadcast sobre un message broker existente: cuando un nodo publica un `RemoteApplicationEvent`, el bus lo entrega a todos los nodos suscritos simultáneamente sin que el emisor conozca quiénes son los receptores. Bus no sustituye a Spring Cloud Stream; los dos módulos coexisten sobre el mismo broker, pero con propósitos distintos: Bus transporta eventos de infraestructura (refresh de configuración, cambios de entorno), mientras que Stream transporta mensajes de negocio (pedidos, pagos, eventos de dominio). Confundir ambos modelos es el error más frecuente al incorporar Bus en un proyecto que ya usa Stream.

> [PREREQUISITO] Este fichero asume conocimiento de Spring Cloud Config (sc-config-refresh.md) y de los conceptos básicos de Spring Cloud Stream (sc-stream-bindings.md).

## Representación visual

La topología de Spring Cloud Bus coloca a todos los nodos como consumidores del mismo topic o exchange. El diagrama siguiente muestra el flujo completo desde el disparo del endpoint hasta la re-instanciación de los beans con `@RefreshScope`.

```
  Git Repository
       │ push
       ▼
  Config Server  ──POST /actuator/busrefresh──►  springCloudBus topic/exchange
       │                                              │
       │                                    ┌─────────┴──────────┐
       │                                    ▼                    ▼
       │                            Service-A:inst-1    Service-A:inst-2
       │                            Service-B:inst-1    Service-C:inst-1
       │                                    │
       │                         RefreshRemoteApplicationEvent
       │                                    │
       │                         EnvironmentChangeEvent
       │                                    │
       │                         @RefreshScope beans re-instanciados
       │
       └─► Config Server puede también suscribirse para auto-actualizar su caché
```

La tabla siguiente contrasta Bus y Stream para evitar confusiones habituales:

| Dimensión | Spring Cloud Bus | Spring Cloud Stream |
|---|---|---|
| Propósito | Eventos de infraestructura (refresh, env change) | Mensajes de negocio entre microservicios |
| Clase base | `RemoteApplicationEvent` | `Message<T>` genérico |
| Topic por defecto | `springCloudBus` (único) | Definido por el desarrollador por binding |
| Receptor previsto | Todos los nodos del clúster | Consumidores específicos del canal |
| Configuración | `spring.cloud.bus.*` | `spring.cloud.stream.bindings.*` |
| API de envío | `ApplicationEventPublisher` | `StreamBridge` o función reactiva |

## Ejemplo central

El siguiente ejemplo muestra una aplicación Spring Boot 4.0.x mínima que actúa como nodo participante en un bus Kafka. Al recibir un `RefreshRemoteApplicationEvent`, Spring re-instancia todos los beans anotados con `@RefreshScope`.

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
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
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

```yaml
# application.yml
spring:
  application:
    name: inventory-service
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
    config:
      uri: http://config-server:8888
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv, refresh, health, info
```

```java
// src/main/java/com/example/inventory/InventoryApplication.java
package com.example.inventory;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@SpringBootApplication
public class InventoryApplication {
    public static void main(String[] args) {
        SpringApplication.run(InventoryApplication.class, args);
    }
}

// Bean que se re-instancia al recibir RefreshRemoteApplicationEvent
@Component
@RefreshScope
class InventoryConfig {

    private final String warehouseLocation;

    public InventoryConfig(@Value("${warehouse.location:default-location}") String warehouseLocation) {
        this.warehouseLocation = warehouseLocation;
        System.out.println("InventoryConfig instanciado con location: " + warehouseLocation);
    }

    public String getWarehouseLocation() {
        return warehouseLocation;
    }
}
```

Con esta configuración, al enviar `POST /actuator/busrefresh` a cualquier nodo del clúster, todos los nodos reciben el evento y re-instancian `InventoryConfig` con los valores actualizados del Config Server.

## Tabla de elementos clave

Los parámetros siguientes controlan el comportamiento fundamental de Spring Cloud Bus y son los que aparecen con mayor frecuencia en entrevistas técnicas.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.bus.enabled` | Boolean | `true` | Activa o desactiva completamente el bus. Útil para deshabilitar en tests sin broker. |
| `spring.cloud.bus.id` | String | `${spring.application.name}:${spring.profiles.active}:${random.uuid}` | Identificador único del nodo en el bus. Colisiones causan que un nodo descarte sus propios eventos. |
| `spring.cloud.bus.destination` | String | `springCloudBus` | Nombre del topic Kafka o exchange RabbitMQ donde se publican los eventos. |
| `spring.cloud.bus.trace.enabled` | Boolean | `false` | Registra cada evento propagado por el bus en el log de la aplicación. |
| `spring.cloud.bus.ack.enabled` | Boolean | `false` | Cada nodo confirma la recepción del evento publicando un `AckRemoteApplicationEvent`. |
| `spring.cloud.bus.refresh.enabled` | Boolean | `true` | Habilita el listener de `RefreshRemoteApplicationEvent` en este nodo. |
| `spring.cloud.bus.env.enabled` | Boolean | `true` | Habilita el listener de `EnvironmentChangeRemoteApplicationEvent` en este nodo. |

## Buenas y malas prácticas

**Hacer:**

- Fijar `spring.cloud.bus.id` explícitamente en entornos Kubernetes usando `$(POD_NAME)` o una combinación de `${spring.application.name}:${spring.profiles.active}:${random.value}`. El default automático puede repetirse entre pods del mismo Deployment si `random.value` no está disponible al arranque, causando que un nodo descarte los eventos que él mismo envió.
- Separar el topic de Bus del resto de topics de negocio configurando `spring.cloud.bus.destination=infrastructure.bus` para aislar el tráfico de infraestructura del de negocio en sistemas de alta carga.
- Anotar con `@RefreshScope` únicamente los beans que leen propiedades de configuración en su constructor. Anotar todos los beans indiscriminadamente genera re-instanciaciones innecesarias que pueden provocar latencia visible en cada refresh.
- Verificar que el módulo Bus está activo comprobando el log de inicio: `BusAutoConfiguration` debe aparecer entre las autoconfiguraciones activas.

**Evitar:**

- No mezclar la lógica de mensajería de negocio con eventos de Bus en el mismo topic. Si el topic `springCloudBus` recibe mensajes de dominio por error, los nodos intentarán deserializarlos como `RemoteApplicationEvent` y producirán errores de deserialización en cascada.
- No asumir que Bus garantiza exactamente una entrega. Con RabbitMQ en modo fanout, si un nodo está caído durante el broadcast, el evento se pierde para ese nodo. El diseño debe tolerar configuraciones temporalmente desactualizadas.
- No invocar `/actuator/busrefresh` directamente en producción sin autenticación. Este endpoint dispara operaciones en todos los nodos del clúster simultáneamente y debe estar protegido (ver sc-bus-seguridad-endpoint.md).

## Comparación: Bus sobre RabbitMQ vs Bus sobre Kafka

La elección del broker afecta el comportamiento de entrega y la retención de eventos. Si el proyecto ya usa uno de los dos, la decisión es trivial: Bus usa el mismo broker sin coste adicional.

| Criterio | Bus sobre RabbitMQ | Bus sobre Kafka |
|---|---|---|
| Starter | `spring-cloud-starter-bus-amqp` | `spring-cloud-starter-bus-kafka` |
| Tipo de intercambiador | Fanout exchange `springCloudBus` | Topic `springCloudBus` (1 partición) |
| Retención de eventos | No (mensaje se descarta si no hay consumidor) | Sí (configurable con `retention.ms`) |
| Nodo que arranca tarde | No recibe eventos anteriores | Puede recuperar eventos si retention > 0 |
| Configuración adicional | `spring.rabbitmq.*` | `spring.kafka.bootstrap-servers` |
| Adecuado para | Entornos ya con RabbitMQ; baja retención necesaria | Entornos ya con Kafka; audit de cambios útil |

---

← [6.13 Testing / Verificación de Spring Cloud Stream](sc-stream-testing.md) | [Índice](README.md) | [7.2 Setup y dependencias de Spring Cloud Bus](sc-bus-setup.md) →
