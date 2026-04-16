# 7.2 Setup y dependencias de Spring Cloud Bus

← [7.1 Arquitectura de Spring Cloud Bus](sc-bus-arquitectura.md) | [Índice](README.md) | [7.3 Configuración del broker subyacente para Spring Cloud Bus](sc-bus-broker-config.md) →

## Introducción

Añadir Spring Cloud Bus a un proyecto es engañosamente sencillo: basta un starter para que `BusAutoConfiguration` registre todos los beans necesarios. Sin embargo, la simplicidad del setup oculta dos decisiones críticas que impactan directamente en producción: la elección del broker (RabbitMQ vs Kafka) y la configuración explícita del `spring.cloud.bus.id`. En entornos contenedorizados con varias instancias del mismo servicio, el `bus.id` automático puede generar colisiones silenciosas que hacen que un nodo descarte los propios eventos que él mismo publicó, produciendo un síntoma que parece un bug del broker pero es un problema de identidad del nodo.

> [PREREQUISITO] Es necesario tener un broker Kafka o RabbitMQ accesible y la dependencia de Spring Cloud Config si se quiere usar el refresh distribuido. Consulta sc-bus-broker-config.md para la configuración detallada del broker.

## Representación visual

El árbol siguiente muestra la relación entre los starters disponibles, la autoconfiguración que activan y los beans que registran.

```
spring-cloud-starter-bus-amqp
  └── spring-cloud-bus (núcleo)
  │     ├── BusAutoConfiguration
  │     │     ├── BusConsumerProperties
  │     │     ├── BusProperties (spring.cloud.bus.*)
  │     │     ├── RefreshListener (si spring-cloud-context en classpath)
  │     │     └── EnvironmentChangeListener
  │     └── RemoteApplicationEventScan (escanea @RemoteApplicationEvent)
  └── spring-cloud-stream-binder-rabbit
        └── Bindings: springCloudBusInput / springCloudBusOutput

spring-cloud-starter-bus-kafka
  └── spring-cloud-bus (núcleo, idéntico)
  └── spring-cloud-stream-binder-kafka
        └── Bindings: springCloudBusInput / springCloudBusOutput
```

La tabla siguiente compara los dos starters para facilitar la elección:

| Criterio | bus-amqp | bus-kafka |
|---|---|---|
| Dependencia transitiva | spring-cloud-stream-binder-rabbit | spring-cloud-stream-binder-kafka |
| Broker requerido | RabbitMQ 3.x+ | Apache Kafka 2.x+ |
| Propiedad de conexión | `spring.rabbitmq.host` | `spring.kafka.bootstrap-servers` |
| Tipo de canal | Fanout exchange | Topic (1 partición por defecto) |

## Ejemplo central

El ejemplo siguiente configura un servicio `order-service` con Bus sobre Kafka, incluye la gestión de dependencias BOM correcta y una configuración de `bus.id` robusta para entornos Kubernetes.

```xml
<!-- pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.0</version>
    </parent>

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

    <dependencies>
        <!-- Bus sobre Kafka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-kafka</artifactId>
        </dependency>
        <!-- Alternativa para RabbitMQ:
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
</project>
```

```yaml
# application.yml
spring:
  application:
    name: order-service
  profiles:
    active: production

  cloud:
    bus:
      enabled: true
      # bus.id explícito: evita colisiones en Kubernetes con múltiples réplicas
      # En K8s, usar el nombre del pod como parte del id:
      # id: ${spring.application.name}:${spring.profiles.active}:${HOSTNAME:localhost}
      # En entornos no-K8s, random.value garantiza unicidad:
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
    config:
      uri: http://config-server:8888

  kafka:
    bootstrap-servers: kafka-broker:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv, health, info, refresh
  endpoint:
    health:
      show-details: always
```

```java
// src/main/java/com/example/order/OrderApplication.java
package com.example.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/order/config/OrderProperties.java
package com.example.order.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Component
@RefreshScope
public class OrderProperties {

    private final int maxOrderItems;
    private final String processingMode;

    public OrderProperties(
            @Value("${order.max-items:100}") int maxOrderItems,
            @Value("${order.processing-mode:sync}") String processingMode) {
        this.maxOrderItems = maxOrderItems;
        this.processingMode = processingMode;
    }

    public int getMaxOrderItems() {
        return maxOrderItems;
    }

    public String getProcessingMode() {
        return processingMode;
    }
}
```

Para verificar que `BusAutoConfiguration` está activa, busca en el log de inicio:

```
INFO o.s.c.b.BusAutoConfiguration : Registering beans for Spring Cloud Bus
INFO o.s.c.stream.binder.kafka.KafkaMessageChannelBinder : Binding [springCloudBusInput]
INFO o.s.c.stream.binder.kafka.KafkaMessageChannelBinder : Binding [springCloudBusOutput]
```

## Tabla de elementos clave

La tabla siguiente recoge los parámetros que un senior debe conocer de memoria al configurar Bus desde cero.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.bus.enabled` | Boolean | `true` | Activa o desactiva el bus completo. Poner a `false` en tests sin broker. |
| `spring.cloud.bus.id` | String | `${app}:${profile}:${random.uuid}` | Identidad única del nodo. Colisiones causan auto-descarte de eventos. |
| `spring.cloud.bus.destination` | String | `springCloudBus` | Nombre del topic/exchange en el broker. |
| `spring.cloud.bus.refresh.enabled` | Boolean | `true` | Habilita el listener de `RefreshRemoteApplicationEvent`. |
| `spring.cloud.bus.env.enabled` | Boolean | `true` | Habilita el listener de `EnvironmentChangeRemoteApplicationEvent`. |
| `spring.cloud.bus.trace.enabled` | Boolean | `false` | Activa log de traza de eventos Bus. Útil en diagnóstico. |
| `spring.cloud.bus.ack.enabled` | Boolean | `false` | Cada nodo publica un `AckRemoteApplicationEvent` al recibir un evento. |

## Buenas y malas prácticas

**Hacer:**

- Fijar `spring.cloud.bus.id` usando `${HOSTNAME}` en entornos Kubernetes. Kubernetes garantiza que el nombre del pod es único dentro del namespace, eliminando el riesgo de colisión. Sin esto, dos pods con el mismo `random.value` descartarán mutuamente sus eventos porque cada nodo ignora los eventos cuya fuente tiene el mismo `bus.id`.
- Verificar el binding de Bus consultando el endpoint `/actuator/bindings` (si Spring Cloud Stream Actuator está habilitado). Si `springCloudBusInput` y `springCloudBusOutput` no aparecen, el bus no está conectado al broker.
- Incluir `busrefresh` en `management.endpoints.web.exposure.include` explícitamente. Desde Spring Boot 3.x, todos los endpoints de actuator excepto `health` e `info` están deshabilitados por defecto.
- Deshabilitar Bus en perfiles de test con `spring.cloud.bus.enabled=false` para evitar la necesidad de un broker en CI.

**Evitar:**

- No usar `spring-cloud-starter-bus-amqp` y `spring-cloud-starter-bus-kafka` simultáneamente en el mismo módulo. Si ambos están en el classpath, Spring Cloud Stream intentará registrar dos binders para el mismo binding `springCloudBus`, generando un fallo de arranque con un error de binder ambiguo.
- No confiar en el `bus.id` automático en entornos con orquestadores de contenedores. En Kubernetes, si el pod reinicia rápidamente, puede recibir el mismo `random.value` que un pod anterior aún en proceso de terminación, causando dos nodos con el mismo `bus.id` durante la ventana de transición.
- No añadir Bus a un servicio que no necesita refresh de configuración ni eventos de infraestructura. Bus añade un consumer group al broker; en sistemas con decenas de microservicios esto multiplica los consumer groups y el overhead de rebalanceo de particiones en Kafka.

---

← [7.1 Arquitectura de Spring Cloud Bus](sc-bus-arquitectura.md) | [Índice](README.md) | [7.3 Configuración del broker subyacente para Spring Cloud Bus](sc-bus-broker-config.md) →
