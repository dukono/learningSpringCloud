# 6.5.2 KeyResolver: estrategias por IP, usuario, API key y plan de suscripción

← [6.5.1 Token Bucket y RequestRateLimiter](./06-21-gateway-rate-limiter-config.md) | [Índice](./README.md) | [6.5.3 Clúster Redis e in-memory →](./06-23-gateway-rate-limiter-variantes.md)

---

El `KeyResolver` es la pieza que determina la identidad del cliente para el rate limiting. La clave que devuelve define el ámbito de la restricción: si devuelve la IP del cliente, todos los usuarios de esa IP comparten el mismo bucket; si devuelve el identificador de usuario, cada usuario tiene su propio bucket independientemente de la IP desde la que accede. Elegir la estrategia incorrecta tiene consecuencias directas: limitar por IP en una red corporativa penaliza a todos los empleados de la empresa como si fueran un único cliente; no limitar por usuario autenticado permite que un único usuario malintencionado consuma toda la cuota.

## Interfaz KeyResolver

`KeyResolver` es la interfaz de Spring Cloud Gateway que el desarrollador implementa para determinar qué clave identifica a cada cliente a efectos de rate limiting. La implementación recibe el `ServerWebExchange` completo, lo que da acceso a todos los datos de la petición: IP, headers, path, cookies y claims de seguridad. La cadena devuelta por `resolve()` es la clave con la que se busca o actualiza el bucket en Redis.

```java
package org.springframework.cloud.gateway.filter.ratelimit;

import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

public interface KeyResolver {
    Mono<String> resolve(ServerWebExchange exchange);
}
```

La interfaz tiene un único método que recibe el exchange completo y devuelve un `Mono<String>`. La cadena devuelta es la clave con la que se busca o crea el bucket en Redis. Si el `Mono` completa vacío (`Mono.empty()`) o con error (`Mono.error()`), el comportamiento depende del parámetro `deny-empty-key` de la ruta.

## KeyResolver por IP

La implementación más sencilla usa la dirección IP del cliente como clave. Es adecuada cuando el cliente no está autenticado o cuando los usuarios acceden desde IPs individuales conocidas. Cuando el Gateway está detrás de un proxy o load balancer, `getRemoteAddress()` devuelve siempre la IP del proxy, no la del cliente real; en ese caso se debe leer el header `X-Forwarded-For` como muestra la segunda implementación.

```java
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

@Configuration
public class KeyResolverConfig {

    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            var remoteAddress = exchange.getRequest().getRemoteAddress();
            if (remoteAddress == null) {
                return Mono.empty();  // deny-empty-key=true rechazará esta petición
            }
            return Mono.just(remoteAddress.getAddress().getHostAddress());
        };
    }
}
```

**Problema con proxies reversos**: cuando el Gateway está detrás de un load balancer o proxy, `getRemoteAddress()` devuelve siempre la IP del proxy, no la del cliente real. Para obtener la IP real:

```java
@Bean
public KeyResolver xForwardedIpKeyResolver() {
    return exchange -> {
        // X-Forwarded-For puede contener múltiples IPs: "IP-cliente, IP-proxy1, IP-proxy2"
        // La IP del cliente real es siempre la primera
        String xForwardedFor = exchange.getRequest().getHeaders()
            .getFirst("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isBlank()) {
            String clientIp = xForwardedFor.split(",")[0].trim();
            return Mono.just(clientIp);
        }
        var remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null
            ? Mono.just(remoteAddress.getAddress().getHostAddress())
            : Mono.empty();
    };
}
```

> [ADVERTENCIA] Confiar en `X-Forwarded-For` para rate limiting tiene riesgos de seguridad: un cliente malintencionado puede falsificar el header añadiendo IPs arbitrarias. Solo usar este header si el Gateway está detrás de un proxy que controlas y que garantiza la integridad del header. El predicado `XForwardedRemoteAddr` tiene las mismas consideraciones (ver 6.2.2).

## KeyResolver por usuario autenticado

Si el filtro `JwtValidationFilter` (ver 6.6.2) propaga el identificador del usuario en el header `X-User-Id`, el `KeyResolver` de usuario puede leerlo directamente sin necesidad de decodificar el token de nuevo:

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        if (userId != null && !userId.isBlank()) {
            return Mono.just("user:" + userId);
        }
        // Fallback a IP para peticiones no autenticadas en rutas mixtas
        var remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null
            ? Mono.just("anon:" + remoteAddress.getAddress().getHostAddress())
            : Mono.empty();
    };
}
```

El prefijo `"user:"` o `"anon:"` en la clave garantiza que los buckets de usuarios autenticados y anónimos no colisionen en Redis aunque casualmente tengan el mismo identificador.

> [CONCEPTO] El `KeyResolver` no valida el token JWT, solo lee el header `X-User-Id` que ya fue propagado por el filtro de autenticación. Si el filtro de autenticación no se ejecutó (ruta pública) o el usuario no está autenticado, `X-User-Id` no existe y el resolver hace fallback a IP. Esta separación de responsabilidades es correcta: el `KeyResolver` solo extrae la clave, no valida identidades.

## KeyResolver por API key

Cuando la API requiere que los clientes se identifiquen con una clave de acceso en el header `X-Api-Key`, el `KeyResolver` por API key es la opción natural. La clave de particionado en Redis será el valor del header, por lo que cada cliente con una API key distinta tiene su propio bucket independiente. Si el header está ausente, el resolver devuelve `Mono.empty()` para que `deny-empty-key` decida si se bloquea o se permite la petición.

```java
@Bean
public KeyResolver apiKeyResolver() {
    return exchange -> {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-Api-Key");
        if (apiKey == null || apiKey.isBlank()) {
            // Sin API key: denegar la petición (cuando deny-empty-key=true)
            return Mono.empty();
        }
        return Mono.just("apikey:" + apiKey);
    };
}
```

> [EXAMEN] Cuando `KeyResolver` devuelve `Mono.empty()` y la ruta tiene `deny-empty-key: true` (el valor por defecto), el Gateway devuelve `403 Forbidden` sin llegar al servicio. Si `deny-empty-key: false`, el Gateway permite la petición sin aplicar rate limiting. La combinación `deny-empty-key: false` con un `KeyResolver` que puede devolver vacío efectivamente desactiva el rate limiting para esas peticiones.

## KeyResolver por plan de suscripción

Este caso avanzado combina el plan de suscripción del cliente (obtenido de un filtro previo que validó el token y propagó el plan) con límites distintos por plan:

```java
@Bean
public KeyResolver subscriptionPlanKeyResolver() {
    return exchange -> {
        // El filtro de autenticación propagó el plan en el header interno
        String plan = exchange.getRequest().getHeaders()
            .getFirst("X-Subscription-Plan");
        String userId = exchange.getRequest().getHeaders()
            .getFirst("X-User-Id");

        if (userId == null) {
            return Mono.empty();
        }

        // La clave incluye el plan para que los límites por plan sean independientes
        String planPrefix = (plan != null) ? plan.toLowerCase() : "free";
        return Mono.just(planPrefix + ":" + userId);
    };
}
```

Para aplicar límites distintos por plan, se necesitan rutas separadas con diferentes `RedisRateLimiter` o una implementación de `RateLimiter` que consulte el plan desde la clave:

```yaml
routes:
  # Ruta para usuarios FREE: 5 req/s
  - id: api-free
    uri: lb://productos-service
    predicates:
      - Path=/api/productos/**
      - Header=X-Subscription-Plan, free
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 5
          redis-rate-limiter.burstCapacity: 10
          key-resolver: "#{@subscriptionPlanKeyResolver}"

  # Ruta para usuarios PRO: 50 req/s
  - id: api-pro
    uri: lb://productos-service
    predicates:
      - Path=/api/productos/**
      - Header=X-Subscription-Plan, pro
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 50
          redis-rate-limiter.burstCapacity: 100
          key-resolver: "#{@subscriptionPlanKeyResolver}"
```

## Múltiples KeyResolvers y selección por ruta

Se pueden declarar varios beans `KeyResolver` en el mismo `@Configuration` y referenciar uno distinto en cada ruta:

```java
@Configuration
public class KeyResolverConfig {

    @Bean
    public KeyResolver ipKeyResolver() { /* ... */ }

    @Bean
    public KeyResolver userKeyResolver() { /* ... */ }

    @Bean
    public KeyResolver apiKeyResolver() { /* ... */ }
}
```

```yaml
routes:
  - id: ruta-publica
    filters:
      - name: RequestRateLimiter
        args:
          key-resolver: "#{@ipKeyResolver}"     # ← referencia por nombre de bean

  - id: ruta-autenticada
    filters:
      - name: RequestRateLimiter
        args:
          key-resolver: "#{@userKeyResolver}"
```

## La propiedad `deny-empty-key`

Cuando el `KeyResolver` devuelve `Mono.empty()` —porque el header buscado no existe, la IP no es resoluble o el usuario no está autenticado— la propiedad `deny-empty-key` determina si la petición se bloquea o se deja pasar sin aplicar rate limiting. La configuración siguiente muestra los dos comportamientos y el código de respuesta alternativo al `403` por defecto.

```yaml
filters:
  - name: RequestRateLimiter
    args:
      key-resolver: "#{@apiKeyResolver}"
      deny-empty-key: true          # 403 si el KeyResolver devuelve vacío (defecto)
      empty-key-status: UNAUTHORIZED # código de respuesta alternativo al 403 por defecto
```

| Valor de `deny-empty-key` | `KeyResolver` devuelve `Mono.empty()` | Resultado |
|---|---|---|
| `true` (defecto) | Sin clave | `403 Forbidden` (o el `empty-key-status` configurado) |
| `false` | Sin clave | La petición se permite sin aplicar rate limiting |

> [CONCEPTO] `deny-empty-key: false` es útil en rutas donde el rate limiting es opcional: si el usuario está autenticado, se aplica el límite por usuario; si no lo está, se permite la petición sin restricción. Combinado con un `KeyResolver` que devuelve `Mono.empty()` para usuarios anónimos, actúa como rate limiting condicional.

## Tabla de estrategias de KeyResolver

Cada estrategia de `KeyResolver` implica compromisos distintos entre precisión de identificación, seguridad y complejidad de implementación. La siguiente tabla resume las cinco estrategias descritas en las secciones anteriores para facilitar la elección según el contexto de la ruta.

| Estrategia | Clave | Pros | Contras |
|---|---|---|---|
| IP directa | `remoteAddress.getHostAddress()` | Sin dependencias de autenticación | Falla con proxies; penaliza redes NAT compartidas |
| IP via X-Forwarded-For | Primera IP del header | Funciona detrás de proxies | Falsificable si el header no está protegido |
| Usuario autenticado | `X-User-Id` propagado por filtro JWT | Límite justo por identidad real | Requiere autenticación previa; rutas públicas necesitan fallback |
| API key | Header `X-Api-Key` | Trazable al cliente; independiente de sesión | La clave puede robarse si no se usa HTTPS |
| Plan de suscripción | `plan:userId` | Diferenciación de tier | Requiere propagación del plan desde el filtro de autenticación |

## Buenas y malas prácticas

Hacer:
- Usar siempre un prefijo en la clave del `KeyResolver` (`"user:"`, `"ip:"`, `"apikey:"`) para evitar colisiones entre diferentes tipos de claves en el mismo Redis.
- Implementar un fallback en el `KeyResolver` para los casos donde el dato principal no está disponible: si no hay `X-User-Id`, usar la IP como fallback en lugar de devolver `Mono.empty()`.
- Testear el `KeyResolver` de forma unitaria independientemente del filtro: es una función pura `exchange → Mono<String>` que se puede testear sin levantar el Gateway.

Evitar:
- No prefijear las claves de distintos `KeyResolver` que puedan coexistir en el mismo Redis. Un userId `"12345"` y una apiKey `"12345"` colisionarían en el mismo bucket.
- Hacer llamadas bloqueantes dentro del `KeyResolver` (consultas JDBC, REST síncronas). El `KeyResolver` se ejecuta en el pipeline reactivo del Gateway. Para lookups remotos, usar `WebClient` o `ReactiveRedisTemplate`.
- Ignorar el caso `remoteAddress == null`. Aunque raro en producción, es posible en tests o conexiones locales, y devolver `Mono.empty()` o `Mono.just("unknown")` es más robusto que lanzar una excepción.

---

← [6.5.1 Token Bucket y RequestRateLimiter](./06-21-gateway-rate-limiter-config.md) | [Índice](./README.md) | [6.5.3 Clúster Redis e in-memory →](./06-23-gateway-rate-limiter-variantes.md)
