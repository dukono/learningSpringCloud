# 6.6 Spring Cloud Stream — RabbitMQ binder: configuración completa

← [6.5 Kafka binder](sc-stream-kafka-binder.md) | [Índice](README.md) | [6.7 Consumer Groups](sc-stream-consumer-groups.md) →

---

## Introducción

El RabbitMQ binder implementa la abstracción Binder de Spring Cloud Stream sobre el protocolo AMQP. Resuelve el problema de integrar aplicaciones Spring con exchanges y queues de RabbitMQ sin usar directamente `RabbitTemplate` o `AmqpTemplate`. Existe porque RabbitMQ tiene un modelo de routing basado en exchanges, bindings y routing keys que es distinto al modelo de topics de Kafka, y el binder abstrae esas diferencias. Se necesita cuando el broker de mensajería corporativo es RabbitMQ, o cuando se requiere routing flexible por atributos del mensaje.

## Modelo de recursos en RabbitMQ binder

El RabbitMQ binder crea y gestiona exchanges y queues automáticamente. La relación entre el binding de Stream y los recursos AMQP es la siguiente:

```
spring.cloud.stream.bindings.[nombre].destination  →  Exchange en RabbitMQ
spring.cloud.stream.bindings.[nombre].group        →  Queue: [destination].[group]
(sin group)                                         →  Queue anónima y efímera
DLQ (con autoBindDlq=true)                         →  Queue: [destination].[group].dlq
```

## Ejemplo central — configuración completa del RabbitMQ binder

El siguiente ejemplo muestra una aplicación con consumer y producer configurados con las propiedades específicas de RabbitMQ, incluyendo DLQ, acknowledgement manual y routing key expression:

```java
package com.example.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.util.function.Function;

@SpringBootApplication
public class RabbitStreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(RabbitStreamApplication.class, args);
    }

    @Bean
    public Function<String, String> routeAlert() {
        return message -> "ROUTED:" + message;
    }
}
```

```yaml
# application.yml — configuración exhaustiva del RabbitMQ binder
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

  cloud:
    function:
      definition: routeAlert

    stream:
      bindings:
        routeAlert-in-0:
          destination: alerts-exchange
          group: alert-processors
          content-type: application/json
        routeAlert-out-0:
          destination: notifications-exchange
          content-type: application/json

      # Configuración específica del RabbitMQ binder (consumer)
      rabbit:
        bindings:
          routeAlert-in-0:
            consumer:
              auto-bind-dlq: true           # Crea automáticamente la DLQ
              republish-to-dlq: true        # Añade cabeceras de error al mensaje en DLQ
              acknowledge-mode: AUTO        # AUTO | MANUAL | NONE
              prefetch: 10                  # Mensajes prefetched por consumer
              durable-subscription: true    # Queue durable (sobrevive reinicios)
              max-attempts: 3              # Reintentos antes de DLQ
              binding-routing-key: "alerts.#"  # Routing key para el binding AMQP
              exchange-type: topic         # topic | fanout | direct | headers

          # Configuración específica del producer
          routeAlert-out-0:
            producer:
              routing-key-expression: "headers['severity']"  # SpEL para routing key
              exchange-type: topic                            # Tipo de exchange
              declare-exchange: true                         # Declara el exchange si no existe
              delivery-mode: PERSISTENT                      # PERSISTENT | NON_PERSISTENT
              mandatory: true                               # Retorna mensajes no enrutados
```

## Tabla de propiedades del RabbitMQ binder

Las propiedades más relevantes del RabbitMQ binder, separadas por consumer y producer:

| Propiedad (consumer) | Descripción | Valor por defecto |
|---------------------|-------------|-------------------|
| `auto-bind-dlq` | Crea queue DLQ automáticamente | `false` |
| `republish-to-dlq` | Añade cabeceras x-exception-* al mensaje en DLQ | `true` (si auto-bind-dlq=true) |
| `acknowledge-mode` | Modo de confirmación: AUTO, MANUAL, NONE | `AUTO` |
| `prefetch` | Mensajes prefetched (QoS) por consumer | `1` |
| `durable-subscription` | Queue durable que sobrevive reinicios | `true` si hay group |
| `binding-routing-key` | Routing key del binding AMQP (para topic exchange) | `#` (todo) |
| `exchange-type` | Tipo de exchange: topic/fanout/direct/headers | `topic` |

| Propiedad (producer) | Descripción | Valor por defecto |
|---------------------|-------------|-------------------|
| `routing-key-expression` | SpEL para calcular la routing key | `[destination]` |
| `exchange-type` | Tipo de exchange destino | `topic` |
| `declare-exchange` | Declara el exchange si no existe | `true` |
| `delivery-mode` | Persistencia del mensaje: PERSISTENT/NON_PERSISTENT | `PERSISTENT` |

> [CONCEPTO] La diferencia entre `auto-bind-dlq` y `republish-to-dlq`: `auto-bind-dlq: true` crea la queue DLQ y el dead-letter exchange (DLX) de forma automática. `republish-to-dlq: true` hace que los mensajes que agotan reintentos se re-publiquen activamente a la DLQ con cabeceras adicionales de diagnóstico (`x-exception-message`, `x-exception-stacktrace`, `x-original-exchange`). Sin `republish-to-dlq`, los mensajes van a la DLQ via el mecanismo nativo de dead-letter de RabbitMQ sin las cabeceras extra.

> [CONCEPTO] El nombre de la DLQ en RabbitMQ sigue el patrón `[destination].[group].dlq`. Por ejemplo, si `destination=alerts-exchange` y `group=alert-processors`, la DLQ se llama `alerts-exchange.alert-processors.dlq`. Este patrón es diferente al de Kafka (`[destination].DLT`).

> [EXAMEN] `acknowledge-mode: MANUAL` requiere que el handler del mensaje confirme explícitamente el mensaje usando `Channel.basicAck()` o `Channel.basicNack()`. Si el handler no confirma, el mensaje permanece en la queue como "unacked" y no se reparte a otros consumers hasta que el canal se cierre.

> [ADVERTENCIA] Si se usa `exchange-type: fanout`, la `routing-key-expression` del producer es ignorada porque los exchanges fanout enrutan a todas las queues ligadas independientemente de la routing key.

## Comparación — DLQ en Kafka vs RabbitMQ

| Aspecto | Kafka binder | RabbitMQ binder |
|---------|--------------|-----------------|
| Habilitar DLQ | `enable-dlq: true` | `auto-bind-dlq: true` |
| Nombre de la DLQ | `[destination].DLT` | `[destination].[group].dlq` |
| Cabeceras de error | No por defecto | Con `republish-to-dlq: true` |
| Namespace propiedad | `kafka.bindings.[b].consumer` | `rabbit.bindings.[b].consumer` |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `auto-bind-dlq: true` junto con `republish-to-dlq: true` en producción para tener cabeceras de diagnóstico en la DLQ.
- Configurar `durable-subscription: true` y `group` para garantizar que las queues sobreviven reinicios de la aplicación.
- Especificar `exchange-type` explícitamente para evitar depender del tipo por defecto.

**Malas prácticas:**
- Usar `acknowledge-mode: NONE` en producción (los mensajes se confirman automáticamente sin importar si el procesamiento falló).
- Confundir el nombre de la DLQ de RabbitMQ (`destination.group.dlq`) con el de Kafka (`destination.DLT`).
- Usar `republish-to-dlq: true` sin `auto-bind-dlq: true` (la DLQ no existiría).

## Verificación y práctica

1. ¿Cuál es el nombre de la DLQ generada para `destination=payments` y `group=payment-service` con `auto-bind-dlq: true`?

2. ¿Cuál es la diferencia práctica entre `auto-bind-dlq` y `republishToDlq` en el RabbitMQ binder?

3. Si `acknowledge-mode: MANUAL`, ¿qué debe hacer el handler del mensaje para confirmar el procesamiento correcto?

4. ¿Por qué `routing-key-expression` no tiene efecto cuando `exchange-type: fanout`?

5. ¿En qué namespace de propiedades se configura `auto-bind-dlq`: en `spring.cloud.stream.bindings.*` o en `spring.cloud.stream.rabbit.bindings.*`?

---

← [6.5 Kafka binder](sc-stream-kafka-binder.md) | [Índice](README.md) | [6.7 Consumer Groups](sc-stream-consumer-groups.md) →
