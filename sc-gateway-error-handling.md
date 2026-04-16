# 3.14 Manejo de errores y respuestas de error

← [3.13 Actuator, administración y observabilidad](sc-gateway-actuator.md) | [Índice](README.md) | [3.15 Testing / Verificación de Gateway](sc-gateway-testing.md) →

---

## Introducción

Sin manejo de errores explícito, Spring Cloud Gateway devuelve respuestas de error en el formato por defecto de Spring Boot WebFlux: un JSON con campos `timestamp`, `path`, `status`, `error` y `message`. Este formato puede no ser coherente con el contrato de la API ni con el formato de error de los servicios backend. Peor aún, ante errores de enrutamiento (backend inaccesible, timeout, 404 de ruta no encontrada) el gateway puede exponer información interna de la infraestructura. El desarrollador necesita personalizar las respuestas de error de Gateway para retornar formatos coherentes con el contrato API ante fallos de enrutamiento y de servicios backend. Este fichero cubre `DefaultErrorWebExceptionHandler`, `AbstractErrorWebExceptionHandler` y `ErrorAttributes`.

> [ADVERTENCIA] Spring Cloud 2025.1.1 (Oakwood) / Spring Boot 4.0.x puede haber modificado `ErrorWebFluxAutoConfiguration` respecto a versiones anteriores. Verificar las Release Notes antes de personalizar `AbstractErrorWebExceptionHandler` en producción.

## Diagrama: flujo de error en Spring Cloud Gateway

El siguiente diagrama muestra qué componente maneja cada tipo de error y cómo fluye hasta el cliente.

```
Petición → Gateway

  Tipo de error 1: No hay ruta que haga match
  → RoutePredicateHandlerMapping lanza NoRouteFoundException
  → DefaultErrorWebExceptionHandler → 404 {status, error, path, message}

  Tipo de error 2: Backend inaccesible (ConnectTimeoutException)
  → NettyRoutingFilter lanza la excepción
  → DefaultErrorWebExceptionHandler → 504 {status, error, path, message}

  Tipo de error 3: CircuitBreaker abierto (con fallbackUri)
  → CircuitBreakerGatewayFilter redirige → forward:/fallback/...
  → Controlador de fallback → respuesta personalizada

  Tipo de error 4: CircuitBreaker abierto (sin fallbackUri)
  → CircuitBreakerGatewayFilter propaga la excepción
  → DefaultErrorWebExceptionHandler → 503

  Personalización:
  DefaultErrorWebExceptionHandler
       ↓ se reemplaza con
  AbstractErrorWebExceptionHandler (implementación propia)
       ↓ consulta
  ErrorAttributes (DefaultErrorAttributes o implementación propia)
```

## Ejemplo central

El siguiente ejemplo muestra cómo personalizar el formato de error del gateway implementando `AbstractErrorWebExceptionHandler`.

### Implementación del manejador de errores personalizado

```java
package com.example.gateway;

import org.springframework.boot.autoconfigure.web.WebProperties;
import org.springframework.boot.autoconfigure.web.reactive.error.AbstractErrorWebExceptionHandler;
import org.springframework.boot.web.reactive.error.ErrorAttributes;
import org.springframework.context.ApplicationContext;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerCodecConfigurer;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

import java.util.Map;

/**
 * Manejador de errores personalizado para Spring Cloud Gateway.
 * Reemplaza DefaultErrorWebExceptionHandler con un formato JSON canónico.
 *
 * @Order(-1): se ejecuta antes que DefaultErrorWebExceptionHandler (@Order(-2) en algunas versiones).
 * Ajustar el orden si hay conflicto con otros manejadores de error.
 */
@Component
@Order(-1)
public class GatewayErrorWebExceptionHandler extends AbstractErrorWebExceptionHandler {

    public GatewayErrorWebExceptionHandler(
            ErrorAttributes errorAttributes,
            WebProperties webProperties,
            ApplicationContext applicationContext,
            ServerCodecConfigurer configurer) {
        super(errorAttributes, webProperties.getResources(), applicationContext);
        this.setMessageWriters(configurer.getWriters());
        this.setMessageReaders(configurer.getReaders());
    }

    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(ErrorAttributes errorAttributes) {
        return RouterFunctions.route(RequestPredicates.all(), this::renderErrorResponse);
    }

    private Mono<ServerResponse> renderErrorResponse(ServerRequest request) {
        // Obtener atributos del error (incluye la excepción original)
        Map<String, Object> errorAttributes = getErrorAttributes(request,
            org.springframework.boot.web.error.ErrorAttributeOptions.defaults());

        int statusCode = (int) errorAttributes.getOrDefault("status", 500);
        HttpStatus status = HttpStatus.resolve(statusCode);
        if (status == null) status = HttpStatus.INTERNAL_SERVER_ERROR;

        // Construir respuesta con formato canónico del API
        Map<String, Object> errorResponse = Map.of(
            "error", Map.of(
                "code", mapStatusToCode(statusCode),
                "message", errorAttributes.getOrDefault("message", "Error interno"),
                "path", errorAttributes.getOrDefault("path", request.path()),
                "requestId", request.exchange().getRequest().getId()
            )
        );

        return ServerResponse
            .status(status)
            .contentType(MediaType.APPLICATION_JSON)
            .body(BodyInserters.fromValue(errorResponse));
    }

    private String mapStatusToCode(int status) {
        return switch (status) {
            case 404 -> "ROUTE_NOT_FOUND";
            case 503 -> "SERVICE_UNAVAILABLE";
            case 504 -> "GATEWAY_TIMEOUT";
            case 429 -> "RATE_LIMIT_EXCEEDED";
            case 401 -> "UNAUTHORIZED";
            case 403 -> "FORBIDDEN";
            default  -> "INTERNAL_ERROR";
        };
    }
}
```

### ErrorAttributes personalizado (para añadir campos extra)

```java
package com.example.gateway;

import org.springframework.boot.web.error.ErrorAttributeOptions;
import org.springframework.boot.web.reactive.error.DefaultErrorAttributes;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.ServerRequest;

import java.util.Map;

/**
 * ErrorAttributes personalizado que añade el campo "gatewayVersion"
 * a todos los errores y elimina el campo "trace" (stack trace) en producción.
 */
@Component
public class GatewayErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(
            ServerRequest request,
            ErrorAttributeOptions options) {
        Map<String, Object> attributes = super.getErrorAttributes(request, options);

        // Añadir información de contexto de gateway
        attributes.put("gatewayVersion", "1.0");
        attributes.put("service", "api-gateway");

        // Eliminar stack trace en producción (no exponer detalles internos)
        attributes.remove("trace");
        attributes.remove("exception");

        return attributes;
    }
}
```

### Configuración YAML (sin cambios requeridos para el manejo de errores básico)

```yaml
# application.yml
# El AbstractErrorWebExceptionHandler se registra automáticamente.
# Solo se necesita ajustar si hay conflictos de orden.

spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            # CircuitBreaker sin fallbackUri → el error llega a GatewayErrorWebExceptionHandler
            - name: CircuitBreaker
              args:
                name: orderCB
                # sin fallbackUri: GatewayErrorWebExceptionHandler maneja el 503
```

## Tabla de elementos clave

La siguiente tabla describe los componentes del manejo de errores en Gateway.

| Componente | Responsabilidad | Cómo personalizar |
|-----------|-----------------|-------------------|
| `DefaultErrorWebExceptionHandler` | Maneja excepciones no capturadas y genera la respuesta de error | Reemplazar con `AbstractErrorWebExceptionHandler` |
| `AbstractErrorWebExceptionHandler` | Clase base para handlers de error personalizados | Extender e implementar `getRoutingFunction()` |
| `DefaultErrorAttributes` | Provee los campos del error (status, message, path, timestamp) | Extender y sobreescribir `getErrorAttributes()` |
| `ErrorAttributes` | Interfaz para proveedores de atributos de error | Implementar la interfaz |
| `@Order(-1)` | Prioridad del manejador personalizado sobre el default | Ajustar según otros manejadores en el contexto |

> [EXAMEN] `DefaultErrorWebExceptionHandler` vs `AbstractErrorWebExceptionHandler`: el primero es la implementación concreta que usa Spring Boot por defecto en contextos WebFlux; el segundo es la clase abstracta base que se extiende para personalizar. Al extender `AbstractErrorWebExceptionHandler` y registrarlo con `@Order` mayor (número más negativo) que el del default, el personalizado toma precedencia.

## Buenas y malas prácticas

**Hacer:**
- Eliminar los campos `trace` y `exception` de los atributos de error en producción: exponer el stack trace completo en la respuesta HTTP revela la estructura interna del gateway, versiones de librerías y posibles vectores de ataque.
- Incluir un campo `requestId` en la respuesta de error: permite correlacionar errores del cliente con logs del gateway sin exponer información sensible.
- Usar códigos de error semánticos (`GATEWAY_TIMEOUT`, `ROUTE_NOT_FOUND`) además del status HTTP: los clientes pueden implementar manejo de errores específico basado en el código semántico sin depender del status HTTP que puede ser ambiguo (ej: varios tipos de error usan 500).

**Evitar:**
- Usar `@Order(-2)` o menor en el manejador personalizado sin verificar el orden del `DefaultErrorWebExceptionHandler`: el orden puede variar entre versiones de Spring Boot; usar un número razonablemente negativo (ej: `-1`, `-5`) y verificar el comportamiento.
- Devolver 200 OK con un body de error en el manejador personalizado: confunde a los clientes que usan el status HTTP para detectar fallos y dificulta el alerting basado en códigos de respuesta en el API gateway o en los proxies upstream.

---

← [3.13 Actuator, administración y observabilidad](sc-gateway-actuator.md) | [Índice](README.md) | [3.15 Testing / Verificación de Gateway](sc-gateway-testing.md) →
