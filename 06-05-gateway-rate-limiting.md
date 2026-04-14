# 6.5 Rate Limiting con Redis

← [6.4 Filtros Personalizados](./06-04-gateway-filtros-custom.md) | [Índice](./README.md) | [6.6 Seguridad: JWT y OAuth2 →](./06-06-gateway-seguridad.md)

---

Sin rate limiting, cualquier cliente puede enviar miles de peticiones por segundo al Gateway. Las consecuencias son tres: abuso deliberado (scraping masivo, ataques de fuerza bruta), errores de cliente con bucles infinitos que generan tráfico no intencionado, y picos de uso legítimo en eventos especiales que saturan los microservicios. Centralizar el rate limiting en el Gateway protege toda la arquitectura en un único punto sin que cada microservicio implemente su propia lógica de throttling.

> [PREREQUISITO] La implementación estándar requiere **Redis en ejecución** como almacén distribuido del estado de los contadores. Sin Redis, el filtro lanza excepción al arrancar. Para desarrollo sin Redis, ver la sección de variante in-memory al final.

---

## 6.5.1 Algoritmo Token Bucket

```
replenishRate = 10 tokens/segundo
burstCapacity = 20 tokens (capacidad máxima del cubo)

Estado inicial:
  cubo: [████████████████████]  20 tokens

Ráfaga de 20 peticiones simultáneas:
  cubo: [                    ]  0 tokens — se permite la ráfaga completa

Intento de petición 21:
  cubo: [                    ]  0 tokens — RECHAZADA → HTTP 429

Después de 1 segundo (se regeneran 10 tokens):
  cubo: [██████████          ]  10 tokens — se permiten 10 peticiones más

Regla:
  velocidad sostenida máxima = replenishRate (10/s)
  ráfaga máxima puntual      = burstCapacity (20)
```

> [CONCEPTO] El Token Bucket permite absorber ráfagas cortas de tráfico legítimo (`burstCapacity > replenishRate`) mientras limita la velocidad media sostenida. No es un límite de ventana fija: el cubo se rellena continuamente, no en intervalos discretos.

---

## 6.5.2 Configuración con `RequestRateLimiter` y Redis reactivo

```xml
<!-- pom.xml — añadir junto a spring-cloud-starter-gateway -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    <!-- reactive es obligatorio: Gateway usa WebFlux y no puede usar el cliente Redis síncrono -->
</dependency>
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      password: ${REDIS_PASSWORD:}          # vacío si Redis sin autenticación
      # Para Redis Cluster:
      # cluster:
      #   nodes: redis-1:6379,redis-2:6379,redis-3:6379

  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10      # tokens generados por segundo
                redis-rate-limiter.burstCapacity: 20      # capacidad máxima del cubo
                redis-rate-limiter.requestedTokens: 1     # tokens que consume cada petición
                key-resolver: "#{@ipKeyResolver}"         # bean que identifica al "cliente"

        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 5
                redis-rate-limiter.burstCapacity: 10
                key-resolver: "#{@userKeyResolver}"       # límite por usuario autenticado
```

```java
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

@Configuration
public class RateLimiterConfig {

    // Por IP del cliente — el más simple, no requiere autenticación
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }

    // Por usuario autenticado — requiere que JwtAuthFilter ya haya propagado X-User-Id
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        ).defaultIfEmpty("anonymous");
        // defaultIfEmpty evita Mono.empty(), que causaría 403 por deny-empty-key=true
    }

    // Por API key en header — para APIs públicas con clientes identificados por key
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-API-Key")
        ).defaultIfEmpty("sin-api-key");
    }

    // Por ruta + usuario — granularidad máxima (cada ruta tiene su propio límite por usuario)
    @Bean
    public KeyResolver routeUserKeyResolver() {
        return exchange -> {
            String path = exchange.getRequest().getURI().getPath();
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            String clave = path + ":" + (userId != null ? userId : "anonymous");
            return Mono.just(clave);
        };
    }

    // Clave compuesta: plan del usuario + ruta
    // Permite que un usuario "premium" no consuma el bucket del usuario "free"
    @Bean
    public KeyResolver planRouteKeyResolver() {
        return exchange -> {
            String plan = exchange.getRequest().getHeaders().getFirst("X-Plan-Id");
            String path = exchange.getRequest().getURI().getPath();
            // Normalizar la ruta a nivel de recurso (sin IDs): /api/pedidos/42 → /api/pedidos
            String recurso = path.replaceAll("/[0-9]+", "");
            String planNorm = (plan != null) ? plan : "free";
            return Mono.just(planNorm + ":" + recurso);
        };
    }
}
```

---

## 6.5.4 Headers de respuesta `X-RateLimit-*` y back-pressure

El filtro añade automáticamente estos headers en cada respuesta (no solo en las bloqueadas):

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 10
X-RateLimit-Burst-Capacity: 20
X-RateLimit-Remaining: 7
X-RateLimit-Requested-Tokens: 1

# Cuando se supera el límite:
HTTP/1.1 429 Too Many Requests
X-RateLimit-Remaining: 0
X-RateLimit-Limit: 10
```

Los clientes pueden leer `X-RateLimit-Remaining` para implementar back-pressure proactivo antes de recibir un 429.

---

## 6.5.5 Parámetros del filtro RequestRateLimiter

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `redis-rate-limiter.replenishRate` | int | — (obligatorio) | Tokens generados por segundo (velocidad sostenida máxima) |
| `redis-rate-limiter.burstCapacity` | int | — (obligatorio) | Capacidad máxima del cubo (permite ráfagas) |
| `redis-rate-limiter.requestedTokens` | int | 1 | Tokens consumidos por cada petición |
| `key-resolver` | SpEL bean ref | `PrincipalNameKeyResolver` | Bean que identifica al cliente |
| `deny-empty-key` | boolean | true | Si `true`, devuelve 403 cuando el `KeyResolver` retorna `Mono.empty()` |
| `empty-key-status-code` | int | 403 | Código HTTP cuando `deny-empty-key=true` y la clave está vacía |

---

## 6.5.6 Buenas y malas prácticas

**Hacer:**
- Usar `defaultIfEmpty("anonymous")` en el `KeyResolver` para que usuarios sin header identificador compartan un bucket común en vez de ser rechazados con 403.
- Configurar límites diferentes por ruta según el coste del endpoint — un endpoint que hace búsquedas en BD merece un límite más restrictivo que uno de solo lectura de caché.
- Leer `X-RateLimit-Remaining` en los clientes para implementar back-pressure antes de llegar a 429 — reduce los errores visibles al usuario final.

**Evitar:**
- Usar `PrincipalNameKeyResolver` (el `KeyResolver` por defecto) sin Spring Security configurado — lanza excepción si no hay `Principal` en el exchange y el Gateway arranca pero falla en cada petición.
- Desplegar múltiples instancias del Gateway con un `RateLimiter` in-memory — cada instancia tiene su propio contador y el límite efectivo se multiplica por el número de instancias; usar Redis para clústeres.
- Poner `burstCapacity` igual a `replenishRate` — anula la capacidad de absorber ráfagas legítimas y genera falsos positivos en picos de tráfico normal.

---

## 6.5.7 Variante in-memory sin Redis (instancia única, desarrollo)

Para entornos de una sola instancia (desarrollo, staging sin Redis) se puede implementar un `RateLimiter` personalizado:

La implementación estándar de Spring necesita Redis porque el estado del cubo debe ser compartido entre todas las instancias del Gateway. En una sola instancia (desarrollo local), se puede usar un `ConcurrentHashMap` en memoria. Esta variante **no escala** a múltiples instancias.

```java
import org.springframework.cloud.gateway.filter.ratelimit.AbstractRateLimiter;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class InMemoryRateLimiter extends AbstractRateLimiter<InMemoryRateLimiter.Config> {

    private final Map<String, AtomicLong> contadores = new ConcurrentHashMap<>();
    private final Map<String, Long> ventanaInicio = new ConcurrentHashMap<>();

    public InMemoryRateLimiter() {
        super(Config.class, "in-memory-rate-limiter", null);
    }

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        long ahora = System.currentTimeMillis();
        long ventanaMs = 1000L;
        int maxPeticiones = 10;

        ventanaInicio.putIfAbsent(id, ahora);
        contadores.putIfAbsent(id, new AtomicLong(0));

        if (ahora - ventanaInicio.get(id) > ventanaMs) {
            ventanaInicio.put(id, ahora);
            contadores.get(id).set(0);
        }

        long cuenta = contadores.get(id).incrementAndGet();
        boolean permitida = cuenta <= maxPeticiones;

        return Mono.just(new Response(permitida, Map.of(
            "X-RateLimit-Remaining", String.valueOf(Math.max(0, maxPeticiones - cuenta)),
            "X-RateLimit-Limit", String.valueOf(maxPeticiones)
        )));
    }

    public static class Config {}
}
```

```yaml
# Usar el RateLimiter in-memory (solo instancia única)
filters:
  - name: RequestRateLimiter
    args:
      rate-limiter: "#{@inMemoryRateLimiter}"
      key-resolver: "#{@ipKeyResolver}"
```

> [ADVERTENCIA] El `RateLimiter` in-memory no funciona en despliegues con múltiples instancias del Gateway: cada instancia mantiene su propio contador. El límite efectivo total es `maxPeticiones × N_instancias`. Usar exclusivamente en desarrollo o staging con instancia única.

---

## 6.5.8 Comportamiento en clúster de Gateway con Redis compartido

Cuando el Gateway corre con múltiples instancias (alta disponibilidad), cada instancia comparte el estado del rate limiter a través de Redis. La clave en Redis tiene el formato:

```
request_rate_limiter.{keyResolver_value}.tokens
request_rate_limiter.{keyResolver_value}.timestamp
```

Redis usa operaciones atómicas (scripts Lua) para garantizar que el incremento y la comprobación del contador sean atómicos, evitando condiciones de carrera entre instancias. Esto significa que el límite total es respetado incluso con 10 instancias del Gateway respondiendo peticiones del mismo usuario simultáneamente.

```bash
# Inspeccionar el estado de los buckets en Redis
redis-cli keys "request_rate_limiter.*"
redis-cli get "request_rate_limiter.user-123.tokens"
```

---

← [6.4 Filtros Personalizados](./06-04-gateway-filtros-custom.md) | [Índice](./README.md) | [6.6 Seguridad: JWT y OAuth2 →](./06-06-gateway-seguridad.md)
