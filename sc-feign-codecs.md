# 4.5 Codecs — Encoder, Decoder, FormEncoder, multipart y ErrorDecoder

<- [4.4 Configuración por propiedades YAML](sc-feign-configuracion-propiedades.md) | [Índice](README.md) | [4.6 Interceptores y observabilidad](sc-feign-interceptores.md) ->

---

## Introducción

Los codecs de Feign son los responsables de transformar los objetos Java en bytes para la petición saliente (Encoder) y de convertir los bytes de la respuesta en objetos Java (Decoder). El `ErrorDecoder` se ocupa de un caso especial: las respuestas con código HTTP de error (4xx/5xx), que Feign trata como una ruta de procesamiento separada del Decoder normal.

Los codecs por defecto (`SpringEncoder` y `SpringDecoder`) delegan en los `HttpMessageConverter` de Spring MVC y cubren el 90% de los casos de uso: JSON con Jackson, XML con JAXB. El problema aparece cuando el servicio remoto devuelve un formato no estándar, cuando hay que enviar formularios HTML (`application/x-www-form-urlencoded`), cuando el payload es un fichero binario (multipart), o cuando una respuesta con cuerpo JSON representa un error de negocio que debe mapearse a una excepción de dominio en lugar de a `FeignException`.

> [ADVERTENCIA] Los codecs de Feign son distintos de los codecs de Spring WebFlux (`CodecConfigurer`). No se comparten. Configurar un `HttpMessageConverter` en `WebMvcConfigurer` no afecta a los clientes Feign; hay que registrar el `Encoder`/`Decoder` explícitamente en la configuración de Feign.

> [ADVERTENCIA] `ErrorDecoder` y `Decoder` cubren casos mutuamente excluyentes: `Decoder` procesa respuestas 2xx (y 404 si `decode404=true`); `ErrorDecoder` procesa el resto de errores HTTP. Confundirlos es un error frecuente: si se intenta manejar un 404 en el `ErrorDecoder` pero `decode404=true` está activo, el 404 nunca llega al `ErrorDecoder`.

---

## Diagrama: flujo de procesamiento de respuesta en Feign

El siguiente diagrama muestra en qué punto del pipeline de Feign interviene cada codec.

```
 Respuesta HTTP del servicio remoto
         │
         ▼
 ┌─────────────────────────────────────┐
 │  ¿Código HTTP de respuesta?         │
 │                                     │
 │  2xx ──────────────────────────────►│ Decoder (SpringDecoder por defecto)
 │                                     │   → deserializa body → objeto Java
 │  404 con decode404=true ───────────►│ Decoder (mismo flujo)
 │                                     │
 │  404 con decode404=false ──────────►│ ErrorDecoder
 │  4xx (excepto 404) ────────────────►│ ErrorDecoder
 │  5xx ──────────────────────────────►│ ErrorDecoder
 └─────────────────────────────────────┘
         │
         ▼ ErrorDecoder
 ¿Tiene Retry-After header?
   Sí → lanza RetryableException (Feign puede reintentar)
   No → lanza Exception de dominio o FeignException
```

---

## Ejemplo central

El siguiente ejemplo muestra cuatro escenarios: Encoder/Decoder por defecto, `FormEncoder` para formularios, `SpringFormEncoder` para multipart, y `ErrorDecoder` personalizado que convierte respuestas de error en excepciones de dominio.

**Dependencias adicionales necesarias para FormEncoder:**

```xml
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <!-- La versión la gestiona el BOM de Spring Cloud -->
</dependency>
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
</dependency>
```

**Configuración de Feign con todos los codecs personalizados:**

```java
package com.example.orderservice.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import feign.codec.Decoder;
import feign.codec.Encoder;
import feign.codec.ErrorDecoder;
import feign.form.spring.SpringFormEncoder;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringDecoder;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.http.HttpStatus;

public class CatalogFeignConfig {

    /**
     * SpringFormEncoder delega en SpringEncoder para tipos no-form.
     * Permite que el mismo cliente envíe tanto JSON como multipart/form-data.
     */
    @Bean
    public Encoder feignFormEncoder(ObjectFactory<HttpMessageConverters> converters) {
        return new SpringFormEncoder(new SpringEncoder(converters));
    }

    /**
     * Decoder personalizado: usa el ObjectMapper de la aplicación para
     * deserializar campos con tipos polimórficos o fechas en formato ISO no estándar.
     */
    @Bean
    public Decoder feignDecoder(ObjectFactory<HttpMessageConverters> converters) {
        // SpringDecoder ya usa los HttpMessageConverter registrados en la app,
        // incluyendo el ObjectMapper configurado con los módulos Jackson necesarios.
        return new SpringDecoder(converters);
    }

    /**
     * ErrorDecoder personalizado: convierte códigos HTTP de error en excepciones de dominio.
     */
    @Bean
    public ErrorDecoder catalogErrorDecoder() {
        return new CatalogErrorDecoder();
    }
}
```

**CatalogErrorDecoder.java — implementación del ErrorDecoder:**

```java
package com.example.orderservice.config;

import com.example.orderservice.exception.ProductNotFoundException;
import com.example.orderservice.exception.CatalogServiceException;
import feign.Response;
import feign.codec.ErrorDecoder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class CatalogErrorDecoder implements ErrorDecoder {

    private static final Logger log = LoggerFactory.getLogger(CatalogErrorDecoder.class);
    private final ErrorDecoder defaultDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        String body = extractBody(response);

        return switch (response.status()) {
            case 404 -> new ProductNotFoundException(
                    "Producto no encontrado. Método: " + methodKey + ". Body: " + body
            );
            case 409 -> new CatalogServiceException(
                    "Conflicto en catálogo. Método: " + methodKey + ". Detalle: " + body
            );
            case 503 -> {
                // 503 es retryable: Feign lanzará RetryableException si hay Retryer configurado
                log.warn("Catalog-service no disponible (503). Método: {}", methodKey);
                yield defaultDecoder.decode(methodKey, response);
            }
            default -> {
                log.error("Error inesperado desde catalog-service. Status: {}, Método: {}",
                        response.status(), methodKey);
                yield defaultDecoder.decode(methodKey, response);
            }
        };
    }

    private String extractBody(Response response) {
        if (response.body() == null) return "[sin cuerpo]";
        try {
            return new String(response.body().asInputStream().readAllBytes(), StandardCharsets.UTF_8);
        } catch (IOException e) {
            return "[error leyendo cuerpo]";
        }
    }
}
```

**Ejemplo de método multipart en la interfaz Feign:**

```java
package com.example.orderservice.client;

import com.example.orderservice.config.CatalogFeignConfig;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestPart;
import org.springframework.web.multipart.MultipartFile;

@FeignClient(
        name          = "catalog-service",
        contextId     = "catalogMedia",
        configuration = CatalogFeignConfig.class
)
public interface CatalogMediaClient {

    /**
     * Sube una imagen de producto como multipart/form-data.
     * Requiere SpringFormEncoder en la configuración del cliente.
     */
    @PostMapping(
            value    = "/api/products/images",
            consumes = MediaType.MULTIPART_FORM_DATA_VALUE
    )
    void uploadProductImage(
            @RequestPart("productId") String productId,
            @RequestPart("image")     MultipartFile image
    );
}
```

---

## Tabla de elementos clave

Codecs disponibles y sus casos de uso.

| Codec / Clase | Tipo | Cuándo usar |
|---|---|---|
| `SpringEncoder` | `Encoder` | Default; serializa con `HttpMessageConverter` de Spring (Jackson JSON, etc.) |
| `SpringDecoder` | `Decoder` | Default; deserializa con `HttpMessageConverter` de Spring |
| `SpringFormEncoder` | `Encoder` | Envío de `application/x-www-form-urlencoded` o `multipart/form-data` |
| `JacksonEncoder` | `Encoder` | Serialización Jackson sin contexto Spring MVC |
| `JacksonDecoder` | `Decoder` | Deserialización Jackson sin contexto Spring MVC |
| `GsonEncoder` / `GsonDecoder` | `Encoder`/`Decoder` | Proyectos que usan Gson en lugar de Jackson |
| `ErrorDecoder.Default` | `ErrorDecoder` | Default; convierte errores HTTP en `FeignException` |
| `ErrorDecoder` (custom) | `ErrorDecoder` | Convertir 4xx/5xx en excepciones de dominio específicas |

---

## Buenas y malas prácticas

**Hacer:**
- Leer el cuerpo de la respuesta en `ErrorDecoder` antes de cerrar el objeto `Response`. El cuerpo de Feign es un `InputStream` de un solo uso; si no se lee en el `decode()`, se pierde. Guardar el contenido en un `String` para incluirlo en el mensaje de la excepción facilita el diagnóstico en logs.
- Usar `SpringFormEncoder` envolviendo `SpringEncoder` como delegado. El delegado gestiona los tipos no-form (JSON, XML) y `SpringFormEncoder` solo interviene para los tipos `multipart` y `form-urlencoded`. Sin el delegado, el cliente perdería la capacidad de enviar JSON.
- Lanzar `RetryableException` desde el `ErrorDecoder` para errores transitorios (503, 429 con `Retry-After`). Feign solo reintentará si la excepción es `RetryableException`; cualquier otro tipo detiene el reintento.

**Evitar:**
- Compartir un `ErrorDecoder` con estado (campos mutables) entre clientes. El `ErrorDecoder` se instancia una vez por configuración de cliente y puede recibir llamadas concurrentes. Un estado mutable produciría data races.
- Devolver `null` desde un `Decoder` personalizado cuando el cuerpo de respuesta está vacío. Feign no valida el resultado del `Decoder`; el `null` propagará silenciosamente hasta el llamador y puede causar `NullPointerException` lejos del punto de origen.
- Registrar el `SpringFormEncoder` como bean global (en `defaultConfiguration`) si solo un cliente del módulo necesita soporte multipart. Los demás clientes cargarán el encoder innecesariamente y podrían serializar accidentalmente tipos que deberían ir como JSON como `form-data`.

---

<- [4.4 Configuración por propiedades YAML](sc-feign-configuracion-propiedades.md) | [Índice](README.md) | [4.6 Interceptores y observabilidad](sc-feign-interceptores.md) ->
