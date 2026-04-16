# 3.4.6 Retry: política de reintentos automáticos

← [3.4.5 Circuit Breaker en Gateway con fallback](sc-gateway-circuitbreaker.md) | [Índice](README.md) | [3.4.7 Filtros de seguridad y sesión](sc-gateway-filtros-seguridad.md) →

---

## Introducción

Los fallos transitorios (desconexión de red, restart de un pod, GC pause) en servicios backend suelen resolverse si se reintenta la petición unos milisegundos después. Sin `RetryGatewayFilterFactory`, el cliente recibe directamente el error aunque una segunda llamada habría tenido éxito. El desarrollador necesita configurar la política de reintentos en Gateway para manejar fallos transitorios de los servicios backend sin propagar errores al cliente. El filtro `Retry` es complementario al Circuit Breaker: actúa antes (intenta recuperarse del fallo en la misma petición), mientras que el Circuit Breaker actúa cuando el fallo es sistemático y prolongado.

> [ADVERTENCIA] El reintento de peticiones no idempotentes (POST, PATCH) puede producir duplicados en el backend. Por defecto, `RetryGatewayFilterFactory` solo reintenta métodos seguros (GET, HEAD). Añadir POST/PATCH a la lista de métodos a reintentar requiere que el backend sea idempotente o que se implemente deduplicación.

## Diagrama: ciclo de reintento con backoff exponencial

El siguiente diagrama muestra la secuencia de reintentos y el cálculo del backoff.

```
Petición original
      │
      ▼
  Intento 1 → fallo (status en lista o excepción)
      │
      ├── ¿retries > 0 y method en lista y status en lista?
      │          │ sí
      │          ▼
      │    espera firstBackoff (ej: 50ms)
      │          │
      ▼
  Intento 2 → fallo
      │
      ├── ¿retries > 1?
      │          │ sí
      │          ▼
      │    espera firstBackoff * factor^1 (ej: 100ms)
      │          │ (acotado por maxBackoff si está configurado)
      │          ▼
  Intento 3 → OK  ─────────────────────────────► respuesta al cliente

      Si todos los intentos fallan:
      Intento N → fallo ────────────────────────► error al cliente
```

## Ejemplo central

El siguiente ejemplo muestra la configuración del filtro `Retry` con backoff exponencial, la única variante con suficiente flexibilidad para entornos de producción.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # Ruta con Retry conservador: solo GET, 3 intentos, backoff exponencial
        - id: catalog-retry-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - name: Retry
              args:
                retries: 3            # número de reintentos (no cuenta el intento original)
                statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE, GATEWAY_TIMEOUT
                # statuses acepta nombres HttpStatus o códigos: 502, 503, 504
                methods: GET, HEAD    # solo métodos idempotentes por defecto
                series: SERVER_ERROR  # serie de status codes (5xx) a reintentar
                backoff:
                  firstBackoff: 50ms     # espera inicial
                  maxBackoff: 500ms      # techo del backoff (sin límite si se omite)
                  factor: 2             # multiplicador entre reintentos
                  basedOnPreviousValue: false
                  # basedOnPreviousValue=true: usa el tiempo real del último backoff
                  # basedOnPreviousValue=false: calcula siempre desde firstBackoff

        # Ruta agresiva para servicios con alto rate de fallos transitorios
        - id: order-retry-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET
          filters:
            - name: Retry
              args:
                retries: 2
                statuses: SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 1s
                  factor: 3
```

### Variante programática con RouteLocatorBuilder

```java
package com.example.gateway;

import org.springframework.cloud.gateway.filter.factory.RetryGatewayFilterFactory;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;

import java.time.Duration;

@Configuration
public class RetryRoutesConfig {

    @Bean
    public RouteLocator retryRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("catalog-retry", r -> r
                .path("/api/catalog/**")
                .filters(f -> f
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.BAD_GATEWAY,
                                     HttpStatus.SERVICE_UNAVAILABLE,
                                     HttpStatus.GATEWAY_TIMEOUT)
                        .setMethods(HttpMethod.GET, HttpMethod.HEAD)
                        .setBackoff(
                            Duration.ofMillis(50),   // firstBackoff
                            Duration.ofMillis(500),  // maxBackoff
                            2,                       // factor
                            false                    // basedOnPreviousValue
                        )))
                .uri("lb://catalog-service"))
            .build();
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge los parámetros de `RetryGatewayFilterFactory`.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `retries` | `int` | `3` | Número máximo de reintentos (no incluye el intento original) |
| `statuses` | `HttpStatus[]` o código numérico | `[]` | Códigos HTTP que activan el reintento |
| `methods` | `HttpMethod[]` | `GET` | Métodos HTTP que se reintentan; nunca añadir POST sin idempotencia garantizada |
| `series` | `HttpStatus.Series[]` | `SERVER_ERROR` | Series de status (ej: 5xx = `SERVER_ERROR`) que activan el reintento |
| `backoff.firstBackoff` | `Duration` | `5ms` | Espera antes del primer reintento |
| `backoff.maxBackoff` | `Duration` | sin límite | Techo máximo del backoff; evita esperas interminables |
| `backoff.factor` | `int` | `2` | Multiplicador entre reintentos (backoff exponencial) |
| `backoff.basedOnPreviousValue` | `boolean` | `false` | Si `true`, usa el tiempo real del último backoff como base del siguiente |

> [EXAMEN] `statuses` y `series` no son excluyentes: se puede configurar ambos y el reintento se activa si el status pertenece a cualquiera de los dos. La diferencia es la granularidad: `series` aplica a toda una familia (5xx), mientras que `statuses` permite incluir o excluir códigos específicos dentro de esa familia.

## Buenas y malas prácticas

**Hacer:**
- Configurar siempre `maxBackoff` para limitar el tiempo total máximo de reintento: sin él, con `factor=2` y `retries=5`, el backoff puede crecer indefinidamente y mantener la conexión del cliente abierta demasiado tiempo.
- Combinar `Retry` con `CircuitBreaker` en la misma ruta: `Retry` intenta recuperarse de fallos transitorios individuales; `CircuitBreaker` corta el flujo cuando el servicio lleva fallando de forma sistemática. Son capas complementarias, no alternativas.
- Mantener `methods` restringido a `GET` y `HEAD` por defecto: son los únicos métodos para los que el reintento es siempre seguro sin precondiciones de idempotencia.

**Evitar:**
- Añadir `methods: POST` sin garantizar que el backend es idempotente (o sin implementar deduplicación por `Idempotency-Key`): reintentar un POST de creación de pedido puede crear duplicados en la base de datos del backend.
- Configurar `retries` muy alto (> 5) sin `maxBackoff`: en un escenario de degradación del servicio, el gateway mantendrá conexiones abiertas durante segundos o minutos en cada petición, agotando el pool de conexiones Netty para el resto de usuarios.
- Usar `Retry` sin `CircuitBreaker` en servicios con alta tasa de fallos: si el servicio está caído, cada petición se reintentará `retries` veces antes de fallar, multiplicando la carga sobre un servicio ya en problemas.

---

← [3.4.5 Circuit Breaker en Gateway con fallback](sc-gateway-circuitbreaker.md) | [Índice](README.md) | [3.4.7 Filtros de seguridad y sesión](sc-gateway-filtros-seguridad.md) →
