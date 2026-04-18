# 6.9 Spring Cloud Stream — Error handling, reintentos y DLQ

← [6.8 Particionado](sc-stream-particionado.md) | [Índice](README.md) | [6.10 Serialización](sc-stream-serializacion.md) →

---

## Introducción

El manejo de errores en Spring Cloud Stream garantiza que los mensajes que fallan durante el procesamiento no se pierdan silenciosamente. Resuelve el problema de qué hacer cuando un handler lanza una excepción: reintentar, aplicar backoff exponencial, clasificar la excepción como retryable o no, y finalmente enviar el mensaje fallido a una Dead Letter Queue (DLQ). Existe porque en sistemas distribuidos los fallos transitorios son esperables y necesitan ser manejados con políticas predecibles. Se necesita en cualquier consumer de producción donde la pérdida de mensajes sea inaceptable.

## Flujo de error handling en Spring Cloud Stream

El flujo de error handling sigue una secuencia determinista cuando un consumer handler lanza una excepción:

```
Mensaje recibido
      │
  Handler lanza excepción
      │
  ¿Es retryable?
    NO → Envía inmediatamente a DLQ (si está configurada) o descarta
    SÍ → Reintento #1 (backOffInitialInterval)
             │
         ¿Sigue fallando?
             SÍ → Reintento #2 (interval * multiplier)
                     │
                 ¿Sigue fallando?
                     SÍ → Reintento #3
                             │
                     ¿Agotó maxAttempts?
                         SÍ → Envía a DLQ
```

## Ejemplo central — configuración completa de error handling

El siguiente ejemplo muestra un consumer con configuración completa de reintentos, backoff exponencial, clasificación de excepciones y DLQ tanto para Kafka como para RabbitMQ:

```java
package com.example.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.util.function.Consumer;

@SpringBootApplication
public class ErrorHandlingApplication {

    public static void main(String[] args) {
        SpringApplication.run(ErrorHandlingApplication.class, args);
    }

    @Bean
    public Consumer<String> processOrder() {
        return order -> {
            if (order.contains("FAIL")) {
                throw new RuntimeException("Processing failed for order: " + order);
            }
            System.out.println("Processed: " + order);
        };
    }
}
```

```yaml
# application.yml — error handling completo para Kafka binder
spring:
  cloud:
    function:
      definition: processOrder

    stream:
      bindings:
        processOrder-in-0:
          destination: orders-topic
          group: order-service
          consumer:
            max-attempts: 3                    # Intentos totales (1 original + 2 reintentos)
            back-off-initial-interval: 1000    # 1 segundo en el primer reintento
            back-off-max-interval: 10000       # Máximo 10 segundos de espera
            back-off-multiplier: 2.0           # 1s → 2s → 4s (capped a 10s)
            default-retryable: true            # Todas las excepciones son retryable por defecto
            retryable-exceptions:
              java.net.ConnectException: true  # Explícitamente retryable
            non-retryable-exceptions:
              java.lang.IllegalArgumentException: true  # NO reintenta: va directo a DLQ

      kafka:
        binder:
          brokers: localhost:9092
        bindings:
          processOrder-in-0:
            consumer:
              enable-dlq: true       # Activa DLQ en Kafka
              # DLT generado: orders-topic.DLT
```

```yaml
# Para RabbitMQ binder (mismo bean Java, distinta configuración):
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: orders-exchange
          group: order-service
          consumer:
            max-attempts: 3
            back-off-initial-interval: 1000
            back-off-multiplier: 2.0

      rabbit:
        bindings:
          processOrder-in-0:
            consumer:
              auto-bind-dlq: true      # Crea: orders-exchange.order-service.dlq
              republish-to-dlq: true   # Añade x-exception-message y x-exception-stacktrace
```

## Tabla de propiedades de error handling

| Propiedad | Namespace | Descripción | Default |
|-----------|-----------|-------------|---------|
| `max-attempts` | `bindings.[b].consumer` | Intentos totales incluyendo el original | `3` |
| `back-off-initial-interval` | `bindings.[b].consumer` | Pausa inicial entre reintentos (ms) | `1000` |
| `back-off-max-interval` | `bindings.[b].consumer` | Pausa máxima entre reintentos (ms) | `10000` |
| `back-off-multiplier` | `bindings.[b].consumer` | Factor multiplicador del backoff | `2.0` |
| `default-retryable` | `bindings.[b].consumer` | Si las excepciones son retryable por defecto | `true` |
| `retryable-exceptions` | `bindings.[b].consumer` | Mapa excepción→boolean para retryable | `{}` |
| `enable-dlq` | `kafka.bindings.[b].consumer` | Activa DLT en Kafka | `false` |
| `auto-bind-dlq` | `rabbit.bindings.[b].consumer` | Crea DLQ en RabbitMQ | `false` |
| `republish-to-dlq` | `rabbit.bindings.[b].consumer` | Añade cabeceras de error en DLQ | `true` |

> [CONCEPTO] `max-attempts: 3` significa 1 intento original más 2 reintentos. El valor `1` desactiva los reintentos completamente (solo el intento original). El valor `0` o negativo es inválido.

> [CONCEPTO] El canal de errores de Spring Integration (`[binding-name].errors`) recibe un `ErrorMessage` cada vez que el procesamiento de un mensaje falla, incluso durante los reintentos. Se puede suscribir a este canal para logging centralizado de errores.

> [EXAMEN] `enable-dlq` está en el namespace `spring.cloud.stream.kafka.bindings.[nombre].consumer`, no en `spring.cloud.stream.bindings.[nombre].consumer`. El primero es del Kafka binder; el segundo es del core de Stream. Esta distinción aparece frecuentemente como pregunta trampa.

> [ADVERTENCIA] Si no se configura DLQ y se agotan los reintentos, el comportamiento depende del binder: en Kafka, el offset se confirma de todas formas y el mensaje se pierde; en RabbitMQ, el mensaje puede rechazarse y volver a la queue indefinidamente si `acknowledge-mode: MANUAL` y no se llama a `basicAck`.

## Comparación — DLQ en Kafka vs RabbitMQ

| Aspecto | Kafka | RabbitMQ |
|---------|-------|----------|
| Propiedad para activar | `enable-dlq: true` | `auto-bind-dlq: true` |
| Nombre del destino DLQ | `[destination].DLT` | `[destination].[group].dlq` |
| Cabeceras de error | No por defecto | Con `republish-to-dlq: true` |
| Namespace | `kafka.bindings.[b].consumer` | `rabbit.bindings.[b].consumer` |

## Buenas y malas prácticas

**Buenas prácticas:**
- Siempre configurar DLQ en consumers de producción para garantizar que los mensajes fallidos no se pierdan.
- Usar `non-retryable-exceptions` para excepciones de validación (evita reintentos inútiles en datos incorrectos).
- Monitorizar la DLQ para detectar errores sistemáticos de procesamiento.

**Malas prácticas:**
- Usar `max-attempts: 1` sin DLQ (los mensajes fallidos se pierden silenciosamente).
- Configurar `back-off-multiplier: 1.0` (backoff lineal, no exponencial — se satura rápidamente).
- Asumir que `enable-dlq` está en `spring.cloud.stream.bindings.*` cuando está en `spring.cloud.stream.kafka.bindings.*`.

## Verificación y práctica

1. ¿Qué configuración permite reintentar 3 veces un mensaje con backoff exponencial (1s, 2s, 4s) antes de enviarlo a la DLQ en Kafka?

2. ¿Cuál es la diferencia entre `retryable-exceptions` y `non-retryable-exceptions` y qué prioridad tiene `default-retryable`?

3. ¿En qué namespace de propiedades se configura `enable-dlq` para Kafka y `auto-bind-dlq` para RabbitMQ?

4. ¿Cuál es el nombre del Dead Letter Topic generado para `destination: inventory-updates` en el Kafka binder con `enable-dlq: true`?

5. Si `max-attempts: 1`, ¿cuántos reintentos se realizan y qué ocurre si el procesamiento falla y no hay DLQ configurada?

---

← [6.8 Particionado](sc-stream-particionado.md) | [Índice](README.md) | [6.10 Serialización](sc-stream-serializacion.md) →
