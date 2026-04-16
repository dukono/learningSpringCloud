# 4.6 Interceptores y observabilidad — RequestInterceptor, BasicAuth y métricas Micrometer

<- [4.5 Codecs](sc-feign-codecs.md) | [Índice](README.md) | [4.7 Logging](sc-feign-logging.md) ->

---

## Introducción

Cuando un microservicio llama a otro, cada petición HTTP debe portar contexto transversal: el token de autenticación del usuario original, un identificador de correlación para rastrear la llamada en los logs, y los identificadores de traza distribuida para correlacionar spans en Zipkin o Jaeger. Hacer esto manualmente en cada método de la interfaz Feign es inmanejable y propenso a omisiones.

`RequestInterceptor` es el mecanismo de Feign para esta necesidad. La interfaz tiene un único método — `apply(RequestTemplate)` — que recibe la plantilla de la petición antes de que se envíe al cliente HTTP subyacente. Aquí se pueden añadir, modificar o eliminar cabeceras, parámetros de query o incluso el path. Los interceptores son la extensión correcta para cualquier lógica transversal que no depende del contenido del método concreto.

La observabilidad de Feign se integra en esta misma capa: el soporte de Micrometer Tracing en Feign funciona mediante un interceptor interno que propaga automáticamente las cabeceras de traza (`traceparent` W3C o `X-B3-TraceId` B3) sin que el desarrollador tenga que escribir código. Cuando se necesita un span personalizado, el lugar correcto también es un `RequestInterceptor`.

> [PREREQUISITO] Para la propagación automática de trazas, `spring-cloud-starter-openfeign` junto con `micrometer-tracing-bridge-brave` o `micrometer-tracing-bridge-otel` debe estar en el classpath. Activar con `spring.cloud.openfeign.micrometer.enabled=true`.

---

## Diagrama: pipeline de interceptores en una petición Feign

El siguiente diagrama muestra el orden de ejecución de los interceptores dentro del pipeline de Feign.

```
 Llamada al método de la interfaz @FeignClient
         │
         ▼
 ReflectiveFeign.invoke()
         │
         ▼
 SynchronousMethodHandler.invoke()
         │   construye RequestTemplate con path, verb, body
         │
         ▼
 Por cada RequestInterceptor registrado (en orden de registro):
   │
   ├─ BasicAuthRequestInterceptor    → añade cabecera Authorization: Basic …
   ├─ BearerTokenInterceptor         → añade cabecera Authorization: Bearer …
   ├─ CorrelationIdInterceptor       → añade cabecera X-Correlation-Id
   └─ MicrometerObservationInterceptor → añade traceparent / X-B3-TraceId  (automático)
         │
         ▼
 Client.execute(Request, Options)    → envío HTTP al servidor
```

---

## Ejemplo central

El siguiente ejemplo cubre tres interceptores de uso frecuente: propagación de Bearer token desde el contexto de seguridad de Spring, propagación de un correlation ID, y creación de un span personalizado para medir la latencia de llamadas Feign a servicios externos.

**BearerTokenInterceptor.java — propagación del token del usuario:**

```java
package com.example.orderservice.feign;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;

/**
 * Propaga el JWT del usuario autenticado en el SecurityContext
 * a todas las peticiones salientes del cliente Feign que usa esta configuración.
 *
 * Registrado como bean en la configuración local del cliente (no global)
 * para no añadir la cabecera a clientes que llaman a servicios externos sin auth.
 */
public class BearerTokenInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        if (authentication instanceof JwtAuthenticationToken jwtAuth) {
            String token = jwtAuth.getToken().getTokenValue();
            template.header("Authorization", "Bearer " + token);
        }
        // Si no hay autenticación activa, se deja pasar sin cabecera.
        // El servicio remoto devolverá 401 y el ErrorDecoder lo gestionará.
    }
}
```

**CorrelationIdInterceptor.java — propagación de correlation ID:**

```java
package com.example.orderservice.feign;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.slf4j.MDC;

/**
 * Lee el correlation ID del MDC (puesto por un filtro de entrada en el servicio)
 * y lo propaga como cabecera HTTP al servicio remoto.
 */
public class CorrelationIdInterceptor implements RequestInterceptor {

    public static final String CORRELATION_HEADER = "X-Correlation-Id";
    public static final String MDC_KEY            = "correlationId";

    @Override
    public void apply(RequestTemplate template) {
        String correlationId = MDC.get(MDC_KEY);
        if (correlationId != null) {
            template.header(CORRELATION_HEADER, correlationId);
        }
    }
}
```

**GlobalFeignConfig.java — registro de interceptores globales:**

```java
package com.example.orderservice.config;

import com.example.orderservice.feign.CorrelationIdInterceptor;
import feign.RequestInterceptor;

// Sin @Configuration para evitar doble registro si está dentro del package escaneado
public class GlobalFeignConfig {

    @Bean
    public RequestInterceptor correlationIdInterceptor() {
        return new CorrelationIdInterceptor();
    }
    // BearerTokenInterceptor NO va aquí (global): solo en la config local del cliente interno
}
```

**application.yml — activación de Micrometer Tracing en Feign:**

```yaml
spring:
  cloud:
    openfeign:
      micrometer:
        enabled: true   # activa propagación automática de traceId/spanId en todas las peticiones Feign

management:
  tracing:
    sampling:
      probability: 1.0  # 100% de muestreo en desarrollo; reducir en producción
```

**CustomSpanInterceptor.java — creación de span personalizado (cuando se necesita más granularidad):**

```java
package com.example.orderservice.feign;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import io.micrometer.tracing.Span;
import io.micrometer.tracing.Tracer;

/**
 * Crea un span personalizado para cada llamada Feign saliente.
 * Útil cuando se quiere añadir atributos propios al span (p.ej. nombre del cliente,
 * versión de la API, tenant ID).
 *
 * NOTA: con spring.cloud.openfeign.micrometer.enabled=true, Feign ya crea spans
 * automáticamente. Este interceptor añade atributos extra al span activo.
 */
public class CustomSpanInterceptor implements RequestInterceptor {

    private final Tracer tracer;

    public CustomSpanInterceptor(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public void apply(RequestTemplate template) {
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            // Añadir el nombre del servicio remoto como atributo del span
            currentSpan.tag("feign.target", template.feignTarget().name());
            currentSpan.tag("feign.path", template.path());
        }
    }
}
```

> [CONCEPTO] Un `RequestInterceptor` se ejecuta síncronamente en el thread que realiza la llamada Feign. Si el interceptor bloquea (p. ej. hace una llamada HTTP para refrescar un token), bloquea también el thread del llamador. Para tokens de corta duración, usar caché con expiración corta en el interceptor.

---

## Tabla de elementos clave

| Elemento | Descripción | Cuándo registrar |
|---|---|---|
| `RequestInterceptor` | Interfaz de un único método; modifica el `RequestTemplate` antes del envío | Siempre que se necesite contexto transversal en peticiones salientes |
| `BasicAuthRequestInterceptor` | Implementación lista; añade cabecera `Authorization: Basic` | Servicios internos con autenticación básica |
| `BearerTokenInterceptor` (custom) | Lee el JWT del `SecurityContext` | Servicios que requieren propagación de identidad OAuth2 |
| `CorrelationIdInterceptor` (custom) | Lee del MDC y añade cabecera | Todos los clientes para trazabilidad en logs |
| `MicrometerObservationInterceptor` | Añadido automáticamente por Spring Cloud | Al activar `spring.cloud.openfeign.micrometer.enabled=true` |
| `spring.cloud.openfeign.micrometer.enabled` | `boolean` | `true` para propagar traceId/spanId automáticamente |
| `Tracer.currentSpan()` | Acceso al span activo | Para añadir atributos personalizados al span de tracing |

---

## Buenas y malas prácticas

**Hacer:**
- Registrar `CorrelationIdInterceptor` como interceptor global (en `defaultConfiguration`) y `BearerTokenInterceptor` como interceptor local (solo en clientes de servicios internos que requieren autenticación). Esta separación evita que peticiones a servicios externos (sin auth) reciban cabeceras de token que podrían filtrar credenciales.
- Usar `spring.cloud.openfeign.micrometer.enabled=true` en lugar de crear manualmente un interceptor de tracing. La integración automática de Micrometer Tracing garantiza la propagación correcta de contexto en escenarios reactivos y asíncronos donde el MDC manual puede perderse.
- Hacer los interceptores `@Bean` de prototype o stateless. Un interceptor con estado compartido (p. ej. un contador mutable) puede producir data races si múltiples threads Feign lo invocan concurrentemente.

**Evitar:**
- Lanzar excepciones checked desde `RequestInterceptor.apply()`. Feign no las declara en la firma del método; cualquier excepción checked lanzada desde un interceptor produce una `UndeclaredThrowableException` que enmascara la causa real en los logs.
- Llamar a servicios externos dentro de un `RequestInterceptor` para obtener tokens de acceso sin caché. Cada petición Feign generaría una llamada adicional al servidor de autenticación, multiplicando la latencia y el riesgo de saturar el endpoint de tokens.
- Registrar el mismo tipo de interceptor tanto en la configuración local del cliente como en `defaultConfiguration`. Feign acumula todos los interceptores registrados; el resultado es que la cabecera se añade dos veces, lo que puede causar errores en servidores que validan cabeceras duplicadas.

---

## Referencias externas

**Micrometer Tracing** — `spring.cloud.openfeign.micrometer.enabled=true` activa la propagación automática de `traceId` y `spanId` en todas las peticiones Feign. Sustituyó a Spring Cloud Sleuth a partir de Spring Cloud 2022.0. Las cabeceras B3 (`X-B3-TraceId`) o W3C TraceContext (`traceparent`) se propagan según el bridge configurado (`micrometer-tracing-bridge-brave` para B3, `micrometer-tracing-bridge-otel` para W3C). Ver módulo sc-tracing para configuración de exportadores y samplers.

---

<- [4.5 Codecs](sc-feign-codecs.md) | [Índice](README.md) | [4.7 Logging](sc-feign-logging.md) ->
