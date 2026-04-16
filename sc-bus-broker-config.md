# 7.3 Configuración del broker subyacente para Spring Cloud Bus

← [7.2 Setup y dependencias de Spring Cloud Bus](sc-bus-setup.md) | [Índice](README.md) | [7.4 Refresh de configuración distribuido con Spring Cloud Bus](sc-bus-refresh-distribuido.md) →

## Introducción

Spring Cloud Bus no implementa su propio protocolo de mensajería: delega completamente en Spring Cloud Stream, que a su vez usa el binder del broker elegido. Esto significa que configurar Bus para producción requiere configurar correctamente dos capas: las propiedades del broker (`spring.rabbitmq.*` o `spring.kafka.*`) y las propiedades específicas de los bindings internos de Bus (`springCloudBusInput` y `springCloudBusOutput`). El comportamiento del sistema ante la desconexión del broker es una de las preguntas más frecuentes en entrevistas: con RabbitMQ en modo fanout, los eventos emitidos mientras un nodo está caído se pierden irrecuperablemente; con Kafka, la retención configurable permite que el nodo los recupere al reconectarse. Esta distinción impacta directamente en el diseño de la estrategia de alta disponibilidad del bus.

> [PREREQUISITO] Comprende cómo Bus usa los bindings `springCloudBusInput` y `springCloudBusOutput` internamente antes de modificar propiedades de Spring Cloud Stream para esos canales. Consulta sc-stream-bindings.md para el modelo de bindings de Stream.

## Representación visual

El diagrama siguiente muestra cómo Bus se apoya en Stream para conectarse al broker, evidenciando las dos capas de configuración.

```
  Aplicación Spring Boot
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  BusAutoConfiguration                               │
  │    ├── BusProperties (spring.cloud.bus.*)           │
  │    └── Registra listeners de RemoteApplicationEvent │
  │                                                     │
  │  Spring Cloud Stream                                │
  │    ├── springCloudBusOutput ──► Binder ──► Broker   │
  │    └── springCloudBusInput  ◄── Binder ◄── Broker   │
  │                                                     │
  │  Binder RabbitMQ                Binder Kafka        │
  │    spring.rabbitmq.host           spring.kafka.     │
  │    spring.rabbitmq.port           bootstrap-servers │
  │    spring.rabbitmq.username                         │
  │    spring.rabbitmq.password                         │
  │    spring.rabbitmq.virtual-host                     │
  └─────────────────────────────────────────────────────┘
```

La tabla siguiente resume las diferencias de comportamiento entre los dos brokers en escenarios de fallo:

| Escenario | RabbitMQ (fanout) | Kafka (topic con retención) |
|---|---|---|
| Nodo caído durante refresh | Evento perdido para ese nodo | Evento recuperable si retention > uptime |
| Broker caído | Publicación falla; los listeners no reciben nada | Publicación falla; retención en broker no aplica |
| Reconexión automática | Sí, via spring-retry | Sí, vía KafkaConsumer auto-reconnect |
| Nodo que arranca tarde | No recibe eventos anteriores | Recibe eventos desde su último offset |

## Ejemplo central

El ejemplo siguiente muestra la configuración completa para Bus sobre RabbitMQ y Bus sobre Kafka, incluyendo propiedades de broker, bindings internos y comportamiento en fallo. Se incluyen ambas variantes en el mismo fichero con perfiles para facilitar la comparación.

```xml
<!-- pom.xml — elige uno de los dos starters -->
<dependencies>
    <!-- Opción A: RabbitMQ -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>

    <!-- Opción B: Kafka (comentar el de arriba y descomentar este) -->
    <!--
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
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
</dependencies>
```

```yaml
# application.yml — configuración base común
spring:
  application:
    name: notification-service
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
      # Nombre del topic/exchange en el broker
      destination: springCloudBus
      # Activar ack y trace en diagnóstico
      trace:
        enabled: false
      ack:
        enabled: false

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, health, info

---
# Perfil: rabbitmq
spring:
  config:
    activate:
      on-profile: rabbitmq
  rabbitmq:
    host: rabbitmq-host
    port: 5672
    username: bususer
    password: buspassword
    virtual-host: /production
    # Reconexión automática en caso de caída del broker
    connection-timeout: 60000
  cloud:
    stream:
      bindings:
        springCloudBusInput:
          destination: springCloudBus
          group: ${spring.application.name}
        springCloudBusOutput:
          destination: springCloudBus

---
# Perfil: kafka
spring:
  config:
    activate:
      on-profile: kafka
  kafka:
    bootstrap-servers: kafka-broker-1:9092,kafka-broker-2:9092
    consumer:
      group-id: ${spring.application.name}-bus
      auto-offset-reset: latest
      enable-auto-commit: true
    producer:
      acks: 1
  cloud:
    stream:
      kafka:
        binder:
          brokers: kafka-broker-1:9092,kafka-broker-2:9092
          auto-create-topics: true
        bindings:
          springCloudBusInput:
            consumer:
              # Número de particiones del topic springCloudBus
              # Si hay más nodos que particiones, algunos no recibirán mensajes
              # Por eso el topic de Bus debe tener al menos tantas particiones
              # como instancias máximas esperadas del servicio con mayor escala
              start-offset: latest
      bindings:
        springCloudBusInput:
          destination: springCloudBus
          group: ${spring.application.name}
          consumer:
            # Con auto-offset-reset: latest, los nodos que arrancan tarde
            # no reciben eventos anteriores a su inicio
            # Cambiar a earliest solo si se quiere replay de eventos de infraestructura
            concurrency: 1
        springCloudBusOutput:
          destination: springCloudBus
```

```java
// src/main/java/com/example/notification/NotificationApplication.java
package com.example.notification;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@SpringBootApplication
public class NotificationApplication {
    public static void main(String[] args) {
        SpringApplication.run(NotificationApplication.class, args);
    }
}

@Component
@RefreshScope
class NotificationConfig {

    private final String smtpHost;
    private final int smtpPort;

    public NotificationConfig(
            @Value("${notification.smtp.host:localhost}") String smtpHost,
            @Value("${notification.smtp.port:587}") int smtpPort) {
        this.smtpHost = smtpHost;
        this.smtpPort = smtpPort;
        System.out.printf("NotificationConfig: smtp=%s:%d%n", smtpHost, smtpPort);
    }

    public String getSmtpHost() { return smtpHost; }
    public int getSmtpPort() { return smtpPort; }
}
```

## Tabla de elementos clave

La tabla siguiente recoge las propiedades de broker más relevantes para Bus en producción, agrupadas por broker.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.bus.destination` | String | `springCloudBus` | Nombre del topic Kafka o exchange RabbitMQ. Cambiar para aislar entornos. |
| `spring.rabbitmq.host` | String | `localhost` | Host del broker RabbitMQ usado por Bus. |
| `spring.rabbitmq.port` | Int | `5672` | Puerto AMQP de RabbitMQ. |
| `spring.rabbitmq.username` | String | `guest` | Usuario de autenticación en RabbitMQ. |
| `spring.rabbitmq.password` | String | `guest` | Contraseña de autenticación en RabbitMQ. |
| `spring.rabbitmq.virtual-host` | String | `/` | Virtual host de RabbitMQ. Aislar entornos con virtual hosts distintos. |
| `spring.kafka.bootstrap-servers` | String | `localhost:9092` | Dirección del broker Kafka usado por Bus. |
| `spring.cloud.stream.bindings.springCloudBusInput.group` | String | — | Consumer group del binding de entrada. Debe ser único por servicio para garantizar broadcast a todos los nodos. |
| `spring.cloud.stream.bindings.springCloudBusInput.destination` | String | `springCloudBus` | Destino del binding de entrada. Debe coincidir con `spring.cloud.bus.destination`. |

## Buenas y malas prácticas

**Hacer:**

- Configurar `spring.cloud.stream.bindings.springCloudBusInput.group` con el nombre del servicio. En Kafka, si varios nodos del mismo servicio comparten el mismo `group`, el broker distribuye las particiones entre ellos (cada evento llega a un solo nodo del grupo). Si cada nodo tiene un `group` distinto (o no tiene group), todos los nodos reciben todos los eventos, que es el comportamiento correcto para Bus.
- Separar `spring.cloud.bus.destination` de los topics de negocio. Un nombre como `infra.bus.production` evita que herramientas de monitorización de topics de negocio confundan los eventos de Bus con mensajes de dominio.
- Monitorizar el lag del consumer group de Bus en Kafka. Si el lag crece indefinidamente, los nodos no están consumiendo los eventos de refresh, y la configuración del clúster puede estar desactualizada durante periodos prolongados.

**Evitar:**

- No configurar `auto-offset-reset: earliest` para el binding `springCloudBusInput` en Kafka en producción. Al reiniciar el consumer group, todos los eventos de refresh históricos se reprocesarán, causando que todos los beans con `@RefreshScope` se re-instancien en cascada y generando un pico de carga en el Config Server.
- No reutilizar las credenciales de RabbitMQ de las colas de negocio para Bus. Si esas credenciales rotan o se restringen, Bus deja de funcionar silenciosamente: los nodos no reciben refresh pero tampoco logean errores obvios hasta que el broker rechaza la conexión.
- No asumir que Bus crea automáticamente el topic de Kafka con las particiones correctas. En producción, crear el topic `springCloudBus` manualmente con tantas particiones como instancias máximas esperadas del servicio con mayor escala, para garantizar que cada nodo tiene su propia partición asignada.

---

← [7.2 Setup y dependencias de Spring Cloud Bus](sc-bus-setup.md) | [Índice](README.md) | [7.4 Refresh de configuración distribuido con Spring Cloud Bus](sc-bus-refresh-distribuido.md) →
