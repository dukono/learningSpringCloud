# 12.6 Spring Cloud Function — Message<T> y MessageHeaders

← [12.5 Composición de funciones](sc-function-composicion.md) | [Índice](README.md) | [12.7 Routing dinámico →](sc-function-routing.md)

---

## Introducción

Spring Cloud Function permite declarar funciones que trabajan con `Message<T>` en lugar del tipo puro `T`. Esta forma de declaración proporciona acceso completo a los metadatos del mensaje (headers) tanto para lectura como para enriquecimiento. Los headers de la petición HTTP, los atributos del mensaje de Kafka o RabbitMQ, y los metadatos de SCF se exponen uniformemente a través de `MessageHeaders`.

> [CONCEPTO] `Message<T>` es una abstracción de Spring Messaging que encapsula un payload de tipo `T` junto con un mapa inmutable de `MessageHeaders`. Permite desacoplar el contenido del mensaje de sus metadatos sin cambiar el contrato funcional.

> [CONCEPTO] Cuando una función declara `Function<Message<T>, Message<R>>`, SCF mapea automáticamente los headers de la fuente (HTTP, Kafka, etc.) al `MessageHeaders` del mensaje de entrada.

> [PREREQUISITO] Las clases `Message`, `MessageHeaders` y `MessageBuilder` pertenecen a `spring-messaging`, incluido transitivamente en todos los starters de SCF.

## Diagrama de propagación de headers

El siguiente diagrama muestra cómo los headers fluyen desde la petición HTTP hacia la función y de vuelta a la respuesta.

```mermaid
sequenceDiagram
    participant CLI as Cliente HTTP
    participant CTL as FunctionController
    participant FN as enrichedProcess()

    CLI->>CTL: POST /enrichedProcess\nX-Correlation-Id: abc-123\nBody: {"name":"order-1"}

    rect rgb(0, 80, 160)
        Note over CTL: Construye Message&lt;String&gt;\npayload = body\nheaders = HTTP headers mapeados
    end

    CTL->>FN: Message&lt;String&gt; con headers incluidos
    Note over FN: getHeaders().get("X-Correlation-Id") → "abc-123"
    Note over FN: MessageBuilder.withPayload(result)\n.copyHeadersIfAbsent(...)\n.setHeader("X-Processed","true")
    FN-->>CTL: Message&lt;String&gt; respuesta enriquecida

    CTL-->>CLI: HTTP 200\nX-Correlation-Id: abc-123\nX-Processed: true\nBody: resultado
```
*Propagación de headers HTTP a través de Message&lt;T&gt;: entrada mapeada automáticamente, salida construida con MessageBuilder.*

## Ejemplo central

El siguiente ejemplo muestra una función que lee un header de entrada, enriquece el mensaje con nuevos headers y propaga los headers originales usando `MessageBuilder`.

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@SpringBootApplication
public class MessageHeadersApplication {

    public static void main(String[] args) {
        SpringApplication.run(MessageHeadersApplication.class, args);
    }

    /**
     * Función T puro: no tiene acceso a headers.
     * Simple y suficiente cuando solo importa el payload.
     */
    @Bean
    public Function<String, String> simplePure() {
        return String::toUpperCase;
    }

    /**
     * Función con Message<T>: acceso completo a headers.
     * Lee X-Correlation-Id, procesa el payload y propaga todos los headers.
     */
    @Bean
    public Function<Message<String>, Message<String>> enrichedProcess() {
        return message -> {
            MessageHeaders headers = message.getHeaders();

            // Leer header de entrada
            String correlationId = (String) headers.get("X-Correlation-Id");
            String contentType = headers.get("contentType", String.class);

            // Procesar payload
            String result = message.getPayload().toUpperCase();

            // Construir respuesta con headers enriquecidos
            return MessageBuilder.withPayload(result)
                    .copyHeadersIfAbsent(headers)          // propaga headers originales
                    .setHeader("X-Processed-By", "enrichedProcess")
                    .setHeader("X-Correlation-Id", correlationId)
                    .build();
        };
    }

    /**
     * Función que enriquece sin modificar el payload.
     * Útil para añadir metadatos de auditoría o trazabilidad.
     */
    @Bean
    public Function<Message<String>, Message<String>> auditDecorator() {
        return message -> MessageBuilder.fromMessage(message)
                .setHeader("X-Audit-Timestamp", System.currentTimeMillis())
                .setHeader("X-Audit-Service", "demo-service")
                .build();
    }
}
```

Configuración en `application.yml`:

```yaml
spring:
  cloud:
    function:
      definition: enrichedProcess
```

> [ADVERTENCIA] `MessageHeaders` es inmutable — no se puede modificar directamente. Para crear un mensaje con headers modificados siempre se debe usar `MessageBuilder`. Intentar modificar `getHeaders()` lanzará `UnsupportedOperationException`.

## Tabla de elementos clave

La siguiente tabla resume los métodos más usados de `Message`, `MessageHeaders` y `MessageBuilder`.

| Clase / Método | Descripción |
|---|---|
| `message.getPayload()` | Obtiene el cuerpo del mensaje (tipo `T`) |
| `message.getHeaders()` | Retorna el mapa inmutable `MessageHeaders` |
| `headers.get("key", Class<T>)` | Obtiene un header con conversión de tipo segura |
| `MessageBuilder.withPayload(p)` | Crea un nuevo builder con el payload indicado |
| `MessageBuilder.fromMessage(m)` | Crea un builder copiando payload y headers del mensaje original |
| `.setHeader("key", value)` | Añade o sobreescribe un header |
| `.copyHeadersIfAbsent(headers)` | Copia headers solo si no existen ya en el builder |
| `.build()` | Materializa el `Message<T>` resultante |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `Function<T, R>` (tipo puro) cuando no se necesitan headers — simplifica la firma y facilita el testing unitario.
- Usar `Function<Message<T>, Message<R>>` solo cuando se necesita leer o escribir headers.
- Usar `copyHeadersIfAbsent()` para propagar headers de trazabilidad sin sobreescribir los ya establecidos.
- Usar `headers.get("key", TargetType.class)` con el tipo explícito para evitar casts inseguros.

**Malas prácticas:**
- Intentar modificar el mapa devuelto por `getHeaders()` directamente — es inmutable y lanzará excepción.
- Devolver un mensaje de salida sin propagar el `X-Correlation-Id` — rompe la trazabilidad distribuida.
- Usar `Message<T>` en lugar de `T` cuando no se necesitan headers — añade complejidad innecesaria.

## Verificación y práctica

> [EXAMEN] ¿Por qué declarar `Function<Message<String>, Message<String>>` en lugar de `Function<String, String>` y cuándo es necesario?

> [EXAMEN] ¿Por qué no se puede modificar directamente el mapa devuelto por `message.getHeaders()` y qué clase se debe usar para crear un mensaje con headers modificados?

> [EXAMEN] ¿Qué método de `MessageBuilder` permite copiar los headers de un mensaje original solo cuando no existen ya en el nuevo mensaje?

> [EXAMEN] ¿Cómo se obtiene de forma segura un header con un tipo concreto desde `MessageHeaders`?

---

← [12.5 Composición de funciones](sc-function-composicion.md) | [Índice](README.md) | [12.7 Routing dinámico →](sc-function-routing.md)
