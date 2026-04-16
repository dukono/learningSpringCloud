# 3.4.4 Rate Limiting con RedisRateLimiter

← [3.4.3 Filtros de response (cabeceras, status y body)](sc-gateway-filtros-response.md) | [Índice](README.md) | [3.4.5 Circuit Breaker en Gateway con fallback](sc-gateway-circuitbreaker.md) →

---

## Introducción

Sin rate limiting, un único cliente mal comportado (o un ataque de fuerza bruta) puede saturar un servicio backend y provocar degradación para todos los usuarios. El rate limiting en Gateway resuelve este problema en el punto de entrada: cada petición se evalúa contra un contador por cliente antes de llegar al backend, y si se supera el límite se rechaza con HTTP 429 sin consumir recursos del servicio. Spring Cloud Gateway implementa rate limiting mediante `RequestRateLimiterGatewayFilterFactory`, que utiliza `RedisRateLimiter` como implementación por defecto basada en el algoritmo token bucket sobre Redis reactivo.

> [PREREQUISITO] `RedisRateLimiter` requiere un servidor Redis accesible. Añadir `spring-boot-starter-data-redis-reactive` y configurar `spring.data.redis.host` / `spring.data.redis.port`.

> [ADVERTENCIA] Spring Cloud 2025.1.1 (Oakwood) / Spring Boot 4.0.x. Verificar en las Release Notes si `RedisRateLimiter` tiene deprecaciones o cambios de API respecto a versiones 2024.0.x antes de publicar en producción.

## Diagrama: flujo del token bucket con RedisRateLimiter

El siguiente diagrama muestra cómo funciona el algoritmo token bucket y la interacción entre Gateway, Redis y el backend.

```
Cliente
  │
  ▼
RequestRateLimiterGatewayFilter
  │
  ├─► KeyResolver.resolve(exchange) → clave del cliente (IP, userId, header...)
  │
  ├─► RedisRateLimiter.isAllowed(routeId, key, requestedTokens)
  │     │
  │     ├─► Redis: SCRIPT LUA atómico
  │     │    ├── tokens_remaining = tokens_actuales + tokens_reponidos_desde_último_check
  │     │    ├── tokens_remaining = min(tokens_remaining, burstCapacity)
  │     │    ├── allowed = tokens_remaining >= requestedTokens
  │     │    └── si allowed: tokens_remaining -= requestedTokens
  │     │
  │     └─► Response: { allowed: true/false, remainingTokens: N }
  │
  ├─── allowed=true  ──► continúa hacia el backend
  │                       Headers en response:
  │                       X-RateLimit-Remaining: N
  │                       X-RateLimit-Replenish-Rate: R
  │                       X-RateLimit-Burst-Capacity: B
  │
  └─── allowed=false ──► 429 Too Many Requests (sin llamar al backend)
```

## Ejemplo central

El siguiente ejemplo configura rate limiting con tres estrategias de `KeyResolver` distintas, que es el punto de decisión más crítico del sistema.

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

### Configuración de beans: KeyResolver y RedisRateLimiter

```java
package com.example.gateway;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import reactor.core.publisher.Mono;

@Configuration
public class RateLimitConfig {

    // ── KeyResolver 1: por IP remota (default para APIs públicas) ────────
    @Bean
    @Primary
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            String ip = exchange.getRequest().getRemoteAddress() != null
                ? exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                : "unknown";
            return Mono.just(ip);
        };
    }

    // ── KeyResolver 2: por usuario autenticado (OAuth2 / JWT) ────────────
    @Bean
    public KeyResolver principalKeyResolver() {
        // PrincipalNameKeyResolver es el bean built-in equivalente
        return exchange -> exchange.getPrincipal()
            .map(java.security.Principal::getName)
            .defaultIfEmpty("anonymous");
    }

    // ── KeyResolver 3: por header personalizado (API key) ────────────────
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> {
            String apiKey = exchange.getRequest()
                .getHeaders().getFirst("X-API-Key");
            if (apiKey == null || apiKey.isBlank()) {
                // denyEmptyKey=true en YAML rechazará esta petición con 403
                return Mono.just("EMPTY");
            }
            return Mono.just(apiKey);
        };
    }

    // ── RedisRateLimiter personalizado ────────────────────────────────────
    // replenishRate: tokens por segundo que se recargan
    // burstCapacity:  máximo de tokens acumulables (pico tolerable)
    // requestedTokens: tokens consumidos por cada petición (default 1)
    @Bean
    public RedisRateLimiter defaultRateLimiter() {
        return new RedisRateLimiter(10, 20, 1);
        // 10 req/s sostenidas, hasta 20 en pico, 1 token por petición
    }
}
```

### Configuración YAML de rutas con rate limiting

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379

  cloud:
    gateway:
      routes:
        # Ruta 1: rate limiting por IP, 10 req/s sostenidas, 20 en pico
        - id: public-api-route
          uri: lb://api-service
          predicates:
            - Path=/api/public/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@ipKeyResolver}"
                deny-empty-key: true   # rechaza con 403 si key-resolver devuelve vacío

        # Ruta 2: rate limiting por API key, límites más altos para clientes premium
        - id: premium-api-route
          uri: lb://api-service
          predicates:
            - Path=/api/premium/**
            - Header=X-API-Key, .+
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@apiKeyResolver}"
                deny-empty-key: false  # si no hay API key, permite pero aplica rate limit vacío

        # Ruta 3: usando RedisRateLimiter bean personalizado
        - id: user-api-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                rate-limiter: "#{@defaultRateLimiter}"
                key-resolver: "#{@principalKeyResolver}"
```

## Tabla de elementos clave

La siguiente tabla recoge las propiedades de configuración del rate limiting que un profesional senior debe conocer de memoria.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `redis-rate-limiter.replenishRate` | `int` | requerido | Tokens recargados por segundo; define la tasa sostenible |
| `redis-rate-limiter.burstCapacity` | `int` | requerido | Máximo de tokens acumulables; define el pico tolerable |
| `redis-rate-limiter.requestedTokens` | `int` | `1` | Tokens consumidos por cada petición; usar > 1 para peticiones costosas |
| `key-resolver` | SpEL bean ref | `PrincipalNameKeyResolver` | Bean `KeyResolver` que identifica al cliente |
| `deny-empty-key` | `boolean` | `true` | Si `true`, rechaza con 403 cuando `KeyResolver` devuelve vacío |
| `spring.cloud.gateway.redis-rate-limiter.include-headers` | `boolean` | `true` | Añade cabeceras `X-RateLimit-*` a la response |
| `X-RateLimit-Remaining` | (response header) | — | Tokens restantes tras esta petición |
| `X-RateLimit-Replenish-Rate` | (response header) | — | Tasa de recarga configurada |
| `X-RateLimit-Burst-Capacity` | (response header) | — | Capacidad de pico configurada |

> [EXAMEN] La diferencia entre `replenishRate` y `burstCapacity`: `replenishRate` determina cuántas peticiones por segundo puede hacer un cliente de forma sostenida en el tiempo; `burstCapacity` determina cuántas puede hacer en un instante si los tokens estaban acumulados. Si `burstCapacity == replenishRate`, no hay posibilidad de picos. Si `burstCapacity > replenishRate`, se toleran ráfagas momentáneas.

## Buenas y malas prácticas

**Hacer:**
- Configurar `burstCapacity` mayor que `replenishRate` para tolerar ráfagas legítimas de tráfico: un usuario que hace 5 peticiones simultáneas en 100ms es comportamiento normal en aplicaciones web, no un ataque.
- Usar `deny-empty-key=true` en APIs públicas: si el `KeyResolver` no puede determinar el cliente (ej: petición sin IP resuelta), es más seguro rechazar que aplicar sin discriminación.
- Exponer las cabeceras `X-RateLimit-*` a los clientes de la API: les permite implementar backoff inteligente sin hacer polling al servidor.
- Usar `requestedTokens > 1` para endpoints costosos (ej: búsquedas complejas, exports): permite definir un único `replenishRate` global y ajustar el coste relativo de cada operación.

**Evitar:**
- Usar la misma clave de Redis para rate limiting en múltiples instancias del gateway con configuraciones distintas: si dos rutas usan la misma `key-resolver` y el mismo cliente, los contadores de Redis se comparten y la suma de ambas rutas puede superar el límite esperado.
- Configurar `burstCapacity` muy alto (ej: 10000) para "no tener problemas": un burst capacity muy alto hace que un atacante pueda enviar 10000 peticiones instantáneas antes de que el rate limiter responda.
- Usar `KeyResolver` basado en `X-Forwarded-For` sin verificar que el proxy upstream es de confianza: un cliente puede falsificar la cabecera `X-Forwarded-For` para eludir el rate limiting.

---

← [3.4.3 Filtros de response (cabeceras, status y body)](sc-gateway-filtros-response.md) | [Índice](README.md) | [3.4.5 Circuit Breaker en Gateway con fallback](sc-gateway-circuitbreaker.md) →
