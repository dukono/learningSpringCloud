# 4.8 Manejo de errores HTTP — FeignException, ErrorDecoder personalizado y excepciones de dominio

<- [4.7 Logging](sc-feign-logging.md) | [Índice](README.md) | [4.9 Fallback y Circuit Breaker](sc-feign-fallback.md) ->

---

## Introducción

Cuando un servicio remoto devuelve un código HTTP de error (4xx o 5xx), Feign no lanza automáticamente la excepción correcta de dominio. Por defecto, cualquier respuesta no-2xx pasa por `ErrorDecoder.Default`, que la convierte en una `FeignException` genérica con el código de estado y el cuerpo de respuesta como string. Esta excepción llega al llamador sin semántica de dominio: el código que recibe la llamada debe inspeccionar el status de la `FeignException` para saber si fue un 404, un 409 o un 503, acoplando el código de negocio a los detalles del protocolo HTTP.

El `ErrorDecoder` personalizado es la solución: intercepta cada respuesta de error antes de que llegue al llamador y la convierte en la excepción de dominio adecuada. El resultado es que el servicio consumidor trabaja con `ProductNotFoundException` o `PaymentDeclinedException` en lugar de con `FeignException` + inspección de status.

> [ADVERTENCIA] `ErrorDecoder` y `Decoder` son mutuamente excluyentes por código de estado. El `Decoder` procesa respuestas 2xx. El `ErrorDecoder` procesa el resto. Si `decode404=true` está activo en el cliente, las respuestas 404 van al `Decoder`, no al `ErrorDecoder`. Ver 4.2 para detalles sobre `decode404`.

---

## Diagrama: jerarquía de FeignException

El siguiente diagrama muestra la jerarquía de excepciones que `ErrorDecoder.Default` puede lanzar según el rango del código HTTP.

```
 RuntimeException
 └── FeignException
      ├── FeignClientException  (4xx — errores del cliente)
      │    ├── BadRequest         (400)
      │    ├── Unauthorized       (401)
      │    ├── Forbidden          (403)
      │    ├── NotFound           (404)
      │    ├── MethodNotAllowed   (405)
      │    ├── Gone               (410)
      │    ├── UnprocessableEntity(422)
      │    └── TooManyRequests    (429)
      └── FeignServerException  (5xx — errores del servidor)
           ├── InternalServerError (500)
           ├── NotImplemented      (501)
           ├── BadGateway          (502)
           ├── ServiceUnavailable  (503)
           └── GatewayTimeout      (504)

 RetryableException  (no extiende FeignException; activa reintentos en Feign)
```

---

## Ejemplo central

El siguiente ejemplo muestra un `ErrorDecoder` completo que convierte respuestas de error en excepciones de dominio, incluyendo el manejo del header `Retry-After` para errores 429 y 503 retryables.

**Excepciones de dominio (deben extender `RuntimeException` para ser unchecked):**

```java
package com.example.orderservice.exception;

public class ProductNotFoundException extends RuntimeException {
    private final String productId;

    public ProductNotFoundException(String productId) {
        super("Producto no encontrado: " + productId);
        this.productId = productId;
    }

    public String getProductId() { return productId; }
}

public class CatalogConflictException extends RuntimeException {
    public CatalogConflictException(String message) { super(message); }
}

public class CatalogUnavailableException extends RuntimeException {
    public CatalogUnavailableException(String message) { super(message); }
}
```

**CatalogErrorDecoder.java — implementación completa:**

```java
package com.example.orderservice.feign;

import com.example.orderservice.exception.CatalogConflictException;
import com.example.orderservice.exception.CatalogUnavailableException;
import com.example.orderservice.exception.ProductNotFoundException;
import feign.Response;
import feign.RetryableException;
import feign.codec.ErrorDecoder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Collection;
import java.util.Date;
import java.util.Map;

public class CatalogErrorDecoder implements ErrorDecoder {

    private static final Logger log = LoggerFactory.getLogger(CatalogErrorDecoder.class);

    @Override
    public Exception decode(String methodKey, Response response) {
        String body = readBody(response);
        int status   = response.status();

        return switch (status) {
            case 404 -> new ProductNotFoundException(extractProductId(response.request().url()));
            case 409 -> new CatalogConflictException(
                    "Conflicto al procesar " + methodKey + ": " + body);
            case 429, 503 -> buildRetryableException(response, methodKey, status);
            default -> {
                log.error("Error no esperado desde catalog-service. status={}, method={}, body={}",
                        status, methodKey, body);
                yield new ErrorDecoder.Default().decode(methodKey, response);
            }
        };
    }

    /**
     * Construye RetryableException respetando el header Retry-After si existe.
     * Feign solo reintentará si la excepción es de tipo RetryableException.
     */
    private RetryableException buildRetryableException(Response response, String methodKey, int status) {
        Date retryAfter = extractRetryAfter(response.headers());
        log.warn("Servicio no disponible temporalmente ({}). method={}, retryAfter={}",
                status, methodKey, retryAfter);

        return new RetryableException(
                status,
                "Servicio no disponible (status=" + status + ")",
                response.request().httpMethod(),
                retryAfter,
                response.request()
        );
    }

    private Date extractRetryAfter(Map<String, Collection<String>> headers) {
        Collection<String> retryAfterHeader = headers.get("Retry-After");
        if (retryAfterHeader == null || retryAfterHeader.isEmpty()) return null;

        String value = retryAfterHeader.iterator().next();
        try {
            // Retry-After puede ser segundos (integer) o fecha HTTP-date
            long seconds = Long.parseLong(value);
            return new Date(System.currentTimeMillis() + seconds * 1000);
        } catch (NumberFormatException e) {
            try {
                ZonedDateTime zdt = ZonedDateTime.parse(value, DateTimeFormatter.RFC_1123_DATE_TIME);
                return Date.from(zdt.toInstant());
            } catch (Exception ex) {
                return null;
            }
        }
    }

    private String readBody(Response response) {
        if (response.body() == null) return "[sin cuerpo]";
        try {
            return new String(response.body().asInputStream().readAllBytes(), StandardCharsets.UTF_8);
        } catch (IOException e) {
            return "[error leyendo cuerpo: " + e.getMessage() + "]";
        }
    }

    private String extractProductId(String url) {
        // Extrae el último segmento de la URL como ID del producto
        String[] parts = url.split("/");
        return parts.length > 0 ? parts[parts.length - 1] : "desconocido";
    }
}
```

**Registro del ErrorDecoder en la configuración del cliente:**

```java
package com.example.orderservice.config;

import com.example.orderservice.feign.CatalogErrorDecoder;
import feign.codec.ErrorDecoder;
import org.springframework.context.annotation.Bean;

public class CatalogFeignConfig {

    @Bean
    public ErrorDecoder catalogErrorDecoder() {
        return new CatalogErrorDecoder();
    }
}
```

> [CONCEPTO] `RetryableException` es la única excepción que hace que Feign active su mecanismo de reintento nativo (`Retryer`). Cualquier otra excepción lanzada por el `ErrorDecoder` se propaga directamente al llamador sin reintento, independientemente de la configuración del `Retryer`.

---

## Tabla de elementos clave

| Elemento | Tipo | Descripción |
|---|---|---|
| `ErrorDecoder` | interfaz | Contrato para convertir respuestas HTTP de error en excepciones |
| `ErrorDecoder.Default` | implementación | Convierte errores HTTP en `FeignException` o sus subtipos |
| `FeignException` | `RuntimeException` | Excepción base para errores de Feign; contiene status y body |
| `FeignClientException` | `FeignException` | Para códigos 4xx; tiene subtipos por status concreto |
| `FeignServerException` | `FeignException` | Para códigos 5xx; tiene subtipos por status concreto |
| `RetryableException` | `RuntimeException` | Activa el `Retryer` de Feign; acepta fecha `Retry-After` |
| `methodKey` en `decode()` | `String` | Identificador del método Feign: `"InterfaceName#methodName(ParamTypes)"` |
| `Response.body()` | `Response.Body` | InputStream de un solo uso; leer antes de cerrar la respuesta |

---

## Buenas y malas prácticas

**Hacer:**
- Usar el parámetro `methodKey` en el `ErrorDecoder` para incluir el contexto en el mensaje de error. En un servicio con 20 clientes Feign, saber que el error viene de `CatalogClient#getProduct(String)` en lugar de solo "404" reduce el tiempo de diagnóstico.
- Leer el cuerpo de la respuesta en el `ErrorDecoder` y guardarlo en el mensaje de la excepción. Es la única oportunidad de acceder al cuerpo de error; una vez que el `decode()` retorna, el `InputStream` se cierra.
- Crear `RetryableException` para errores 503 con cabecera `Retry-After`. Feign puede respetar ese tiempo de espera si el `Retryer` está configurado con un `maxAttempts > 0`.

**Evitar:**
- Lanzar excepciones checked desde el `ErrorDecoder`. La firma del método es `Exception decode(...)`, lo que técnicamente permite excepciones checked, pero Feign las envuelve en `UndeclaredThrowableException`. El llamador las recibe como `RuntimeException` con causa anidada, lo que dificulta el `catch` por tipo.
- Registrar el mismo `ErrorDecoder` como bean global (`defaultConfiguration`) cuando su lógica es específica de un servicio. Un `ProductNotFoundException` no tiene sentido cuando el error viene del servicio de pagos. Los `ErrorDecoder` deben ser locales al cliente que los justifica.
- Ignorar el status 429 (Too Many Requests) sin `RetryableException`. Un 429 sin reintento implica que el consumidor trata el rate limiting como un error permanente. Si la API del servicio remoto está correctamente diseñada, respetará el `Retry-After` y el reintento tendrá éxito.

---

<- [4.7 Logging](sc-feign-logging.md) | [Índice](README.md) | [4.9 Fallback y Circuit Breaker](sc-feign-fallback.md) ->
