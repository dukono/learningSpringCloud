# 7.5 Spring Cloud Bus — Configuración de brokers RabbitMQ y Kafka

← [7.4 Spring Cloud Bus — Eventos personalizados con RemoteApplicationEvent](sc-bus-eventos-personalizados.md) | [Índice](README.md) | [7.6 Spring Cloud Bus — Trazabilidad y destination pattern](sc-bus-observabilidad.md) →

---

## Introducción

Spring Cloud Bus abstrae el broker subyacente pero expone las propiedades de conexión y configuración de RabbitMQ y Kafka a través de sus respectivos namespaces de Spring Boot y Spring Cloud Stream. Conocer las propiedades específicas de cada broker es necesario para configurar correctamente entornos de producción y para resolver problemas de conectividad o comportamiento inesperado del Bus.

> [PREREQUISITO] Spring Cloud Bus delega el transporte en Spring Cloud Stream. La configuración del broker se realiza a través de las propiedades de Spring Boot (`spring.rabbitmq.*`, `spring.kafka.*`) y de Spring Cloud Stream (`spring.cloud.stream.bindings.*`). No existe un namespace propio del Bus para la configuración del broker.

## Configuración de RabbitMQ

RabbitMQ es el broker más común en despliegues de Spring Cloud Bus. El Bus crea automáticamente un exchange de tipo `fanout` con el nombre definido en `spring.cloud.bus.destination` (por defecto `springCloudBus`). Cada instancia crea una cola anónima que se liga a este exchange.

Las propiedades de conexión a RabbitMQ se configuran bajo el namespace `spring.rabbitmq`:

| Propiedad | Valor por defecto | Descripción |
|-----------|-------------------|-------------|
| `spring.rabbitmq.host` | `localhost` | Hostname del servidor RabbitMQ |
| `spring.rabbitmq.port` | `5672` | Puerto AMQP del servidor |
| `spring.rabbitmq.username` | `guest` | Usuario de autenticación |
| `spring.rabbitmq.password` | `guest` | Contraseña de autenticación |
| `spring.rabbitmq.virtual-host` | `/` | VirtualHost de RabbitMQ |
| `spring.rabbitmq.ssl.enabled` | `false` | Habilita conexión SSL/TLS |

## Configuración de Apache Kafka

Con Apache Kafka, el Bus usa un topic llamado `springCloudBus` (o el valor de `spring.cloud.bus.destination`). La diferencia fundamental respecto a RabbitMQ es que Kafka usa consumer groups para distribuir mensajes, lo que requiere configuración explícita del grupo en el binding de entrada del Bus.

Las propiedades clave para Kafka son:

| Propiedad | Descripción |
|-----------|-------------|
| `spring.cloud.stream.kafka.binder.brokers` | Dirección(es) de los brokers Kafka |
| `spring.cloud.stream.bindings.springCloudBusInput.group` | Consumer group único por instancia |
| `spring.cloud.stream.kafka.binder.auto-create-topics` | Crear topics automáticamente (default: true) |
| `spring.cloud.stream.kafka.binder.replication-factor` | Factor de replicación del topic del Bus |

## Ejemplo central — Configuración completa con ambos brokers

El siguiente ejemplo muestra la configuración completa para entornos de producción con RabbitMQ y Kafka, incluyendo SSL, consumer groups y ajustes de rendimiento.

```yaml
# application-rabbitmq.yml — configuración de producción con RabbitMQ
spring:
  application:
    name: payment-service
  profiles:
    active: rabbitmq

  rabbitmq:
    host: rabbitmq.internal.company.com
    port: 5671                           # Puerto SSL/TLS
    username: bus-user
    password: ${RABBITMQ_PASSWORD}       # Inyectar desde variable de entorno
    virtual-host: /spring-cloud-bus
    ssl:
      enabled: true
      key-store: classpath:rabbit-client.p12
      key-store-password: ${KEYSTORE_PASS}
      trust-store: classpath:rabbit-ca.p12
      trust-store-password: ${TRUSTSTORE_PASS}

  cloud:
    bus:
      enabled: true
      destination: springCloudBus        # Nombre del exchange fanout
      id: ${spring.application.name}:${spring.profiles.active}:${server.port:8080}

    stream:
      bindings:
        # El binding de entrada del Bus en RabbitMQ NO requiere group
        # (cada instancia tiene cola anónima propia, recibe todos los mensajes)
        springCloudBusInput:
          destination: springCloudBus
        springCloudBusOutput:
          destination: springCloudBus

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env, health
```

```yaml
# application-kafka.yml — configuración de producción con Kafka
spring:
  application:
    name: payment-service
  profiles:
    active: kafka

  cloud:
    stream:
      kafka:
        binder:
          brokers: kafka1.internal:9092, kafka2.internal:9092, kafka3.internal:9092
          auto-create-topics: true
          replication-factor: 3         # Replicación en clúster de 3 nodos
          configuration:
            # Seguridad SSL para Kafka
            security.protocol: SSL
            ssl.truststore.location: /etc/ssl/kafka/truststore.jks
            ssl.truststore.password: ${KAFKA_TRUSTSTORE_PASS}
            ssl.keystore.location: /etc/ssl/kafka/keystore.jks
            ssl.keystore.password: ${KAFKA_KEYSTORE_PASS}
      bindings:
        springCloudBusInput:
          destination: springCloudBus
          # CRÍTICO: cada instancia debe tener un consumer group ÚNICO
          # para recibir TODOS los mensajes (comportamiento broadcast)
          group: ${spring.application.name}-${random.uuid}
        springCloudBusOutput:
          destination: springCloudBus

    bus:
      enabled: true
      destination: springCloudBus
      id: ${spring.application.name}:${spring.profiles.active}:${server.port:8080}

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env, health
```

```java
// BrokerHealthCheck.java — verificar conectividad al broker en arranque
package com.example.paymentservice.health;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
public class BrokerHealthCheck {

    private static final Logger log = LoggerFactory.getLogger(BrokerHealthCheck.class);

    @EventListener(ApplicationReadyEvent.class)
    public void verifyBusConnectivity() {
        log.info("Spring Cloud Bus initialized. Broker connectivity verified by BusAutoConfiguration.");
        log.info("Bus endpoint available at: POST /actuator/bus-refresh");
    }
}
```

## Diferencias clave entre RabbitMQ y Kafka en el Bus

El comportamiento del Bus difiere entre los dos brokers por la arquitectura de mensajería de cada uno. Estas diferencias son relevantes para el examen y para decidir qué broker usar en producción.

| Aspecto | RabbitMQ | Apache Kafka |
|---------|----------|--------------|
| Tipo de canal | Exchange `fanout` | Topic con particiones |
| Consumer group requerido | No (cola anónima por instancia) | Sí (único por instancia para broadcast) |
| Persistencia de mensajes | No (por defecto, colas anónimas son transitorias) | Sí (mensajes persistidos en log) |
| Broadcast natural | Automático con exchange fanout | Requiere consumer group único |
| Mensajes al reconectar | No recibe mensajes previos | Puede recibir mensajes anteriores según `auto.offset.reset` |
| Escalado horizontal | Simple | Requiere cuidado con particiones |

> [ADVERTENCIA] Con Kafka, si se usa un consumer group compartido entre instancias del mismo servicio (por ejemplo, `group: payment-service`), Kafka asignará particiones del topic a cada instancia. Solo una instancia recibirá cada mensaje, rompiendo el comportamiento de broadcast del Bus.

## Consumer groups en Kafka — el problema de duplicados

El consumer group en Kafka define qué instancias compiten por los mensajes de un topic. Para el Bus se necesita el comportamiento opuesto a la competencia: cada instancia debe recibir TODOS los mensajes.

La solución es asignar un consumer group único a cada instancia del Bus usando un identificador aleatorio o el puerto:

```yaml
spring:
  cloud:
    stream:
      bindings:
        springCloudBusInput:
          group: ${spring.application.name}-${random.uuid}
          # Alternativa usando el puerto:
          # group: ${spring.application.name}-${server.port:8080}
```

## Buenas y malas prácticas

**Buenas prácticas:**

- En producción, usar variables de entorno o un secrets manager (Vault, Kubernetes Secrets) para las credenciales del broker, nunca hardcodearlas en el YAML.
- Configurar `spring.cloud.bus.destination` con un nombre descriptivo y diferente por entorno para evitar que múltiples entornos compartan el mismo topic/exchange accidentalmente.
- Usar SSL/TLS para la conexión al broker en entornos de producción y staging.

**Malas prácticas:**

- Compartir el mismo `spring.cloud.bus.destination` entre entornos de desarrollo y producción. Un refresh en dev podría propagarse a producción si ambos apuntan al mismo broker y exchange.
- Configurar `spring.cloud.stream.bindings.springCloudBusInput.group` con un valor estático y compartido entre instancias en Kafka. Esto rompe el broadcast.
- Ignorar los logs de arranque del BusAutoConfiguration. Contienen información sobre los bindings activos y posibles errores de conexión.

## Verificación y práctica

> [EXAMEN] **1.** ¿Por qué RabbitMQ no requiere configurar un consumer group para el Bus, mientras que Kafka sí lo requiere?

> [EXAMEN] **2.** ¿Qué tipo de exchange crea Spring Cloud Bus en RabbitMQ y cómo garantiza que todas las instancias reciban los mensajes?

> [EXAMEN] **3.** ¿Qué valor debe tener `spring.cloud.stream.bindings.springCloudBusInput.group` en Kafka para garantizar que cada instancia reciba todos los mensajes del Bus?

> [EXAMEN] **4.** ¿A través de qué namespace de Spring Boot se configura la conexión AMQP a RabbitMQ para el Bus?

---

← [7.4 Spring Cloud Bus — Eventos personalizados con RemoteApplicationEvent](sc-bus-eventos-personalizados.md) | [Índice](README.md) | [7.6 Spring Cloud Bus — Trazabilidad y destination pattern](sc-bus-observabilidad.md) →
