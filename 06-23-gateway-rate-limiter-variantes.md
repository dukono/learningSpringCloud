# 6.5.3 Clúster con Redis compartido y variante in-memory para desarrollo

← [6.5.2 KeyResolver](./06-22-gateway-key-resolver.md) | [Índice](./README.md) | [6.6.1 Flujo de validación JWT →](./06-24-gateway-seguridad-flujo.md)

---

En producción, el API Gateway se despliega con múltiples instancias para alta disponibilidad. Sin un almacén compartido, cada instancia mantiene sus propios contadores de rate limiting de forma independiente: un cliente puede enviar N peticiones a cada instancia del Gateway sin que ninguna de ellas supere el límite individualmente, aunque el total supere con creces la cuota permitida. Redis como almacén centralizado resuelve este problema porque todas las instancias del Gateway consultan y actualizan los mismos contadores. En entornos de desarrollo, donde Redis puede ser una dependencia no deseada, es posible implementar un `RateLimiter` alternativo en memoria.

## Diagrama: el problema de múltiples instancias sin estado compartido

Sin un almacén compartido, el balanceador de carga distribuye las peticiones entre instancias y cada una aplica su propio contador local, de modo que el límite efectivo se multiplica por el número de instancias del Gateway. El siguiente diagrama contrasta ambos escenarios con un límite configurado de 10 peticiones por segundo.

```
SIN Redis compartido (incorrecto):

  Cliente: 30 peticiones en 1 segundo
       │
       ├──► Gateway instancia A: 10/10 → 10 aceptadas, 0 rechazadas
       ├──► Gateway instancia B: 10/10 → 10 aceptadas, 0 rechazadas
       └──► Gateway instancia C: 10/10 → 10 aceptadas, 0 rechazadas
  Resultado: 30 peticiones aceptadas (límite efectivo 3×10 = 30, no 10)

CON Redis compartido (correcto):

  Cliente: 30 peticiones en 1 segundo
       │
       ├──► Gateway A consulta Redis → bucket: 10→7 tokens
       ├──► Gateway B consulta Redis → bucket: 7→4 tokens
       ├──► Gateway C consulta Redis → bucket: 4→0 tokens
       └──► ... peticiones siguientes → 0 tokens → 429
  Resultado: 10 peticiones aceptadas, 20 rechazadas (límite global: 10)
```

## Configuración de Redis en distintos modos

### Redis Standalone (desarrollo y staging)

La configuración mínima conecta al Redis local por defecto en el puerto 6379 sin contraseña. Los timeouts de conexión y de operación se ajustan para evitar que un Redis lento bloquee el pipeline reactivo del Gateway durante más de unos pocos segundos.

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      # password: opcional en desarrollo
      connect-timeout: 2s
      timeout: 1s
```

### Redis con autenticación y TLS (producción)

En producción, la contraseña se inyecta desde una variable de entorno para no exponerla en el repositorio, y TLS se activa para cifrar el tráfico entre el Gateway y Redis. El pool de conexiones Lettuce se dimensiona según la concurrencia esperada: `max-active` debe superar el número de peticiones simultáneas que el Gateway puede procesar.

```yaml
spring:
  data:
    redis:
      host: redis.internal.miempresa.com
      port: 6380
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 2
          max-wait: 500ms
```

### Redis Sentinel (alta disponibilidad con failover automático)

Redis Sentinel monitoriza un Redis master y sus réplicas. En caso de caída del master, Sentinel promueve automáticamente una réplica, y Lettuce reconecta sin intervención manual.

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - sentinel1.internal:26379
          - sentinel2.internal:26379
          - sentinel3.internal:26379
        password: ${REDIS_SENTINEL_PASSWORD}
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 20
```

### Redis Cluster (sharding horizontal para alta escala)

Redis Cluster distribuye los datos entre múltiples nodos mediante hash slots, lo que permite escalar horizontalmente la capacidad de escritura. La opción `adaptive: true` en Lettuce hace que el cliente actualice automáticamente el mapa de topología del cluster cuando un nodo se añade o se elimina, sin necesidad de reiniciar el Gateway.

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - redis-node1.internal:6379
          - redis-node2.internal:6379
          - redis-node3.internal:6379
          - redis-node4.internal:6379
          - redis-node5.internal:6379
          - redis-node6.internal:6379
        max-redirects: 3
      password: ${REDIS_PASSWORD}
      lettuce:
        cluster:
          refresh:
            adaptive: true           # actualiza la topología del cluster automáticamente
            period: 30s
```

> [ADVERTENCIA] Con Redis Cluster, los comandos de Lua que Spring Cloud Gateway usa para el rate limiting (operaciones atómicas sobre múltiples claves) deben ejecutarse sobre claves que están en el mismo slot del cluster. Spring Cloud Gateway usa hash tags en las claves de rate limiting para garantizar esto, pero si se implementa un `RateLimiter` personalizado, esta restricción debe tenerse en cuenta.

> [EXAMEN] La diferencia clave entre Sentinel y Cluster: Sentinel proporciona alta disponibilidad (failover) para un único nodo master con réplicas. Cluster proporciona sharding horizontal y alta disponibilidad simultáneamente. Sentinel es suficiente para la mayoría de los casos de rate limiting; Cluster se necesita cuando el volumen de peticiones genera una carga que un único nodo Redis no puede absorber.

## Variante in-memory para desarrollo local

Para entornos de desarrollo donde Redis no está disponible, se puede implementar un `RateLimiter` alternativo basado en memoria:

```java
import org.springframework.cloud.gateway.filter.ratelimit.AbstractRateLimiter;
import org.springframework.cloud.gateway.filter.ratelimit.RateLimiter;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

// Solo para desarrollo — NO usar en producción con múltiples instancias
@Component
@Profile("dev")
public class InMemoryRateLimiter implements RateLimiter<InMemoryRateLimiter.Config> {

    private final Map<String, AtomicLong> counters = new ConcurrentHashMap<>();
    private final Map<String, Long> windowStart = new ConcurrentHashMap<>();

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        return Mono.fromCallable(() -> {
            long now = System.currentTimeMillis();
            long windowMs = 1000L; // ventana de 1 segundo

            windowStart.putIfAbsent(id, now);
            counters.putIfAbsent(id, new AtomicLong(0));

            // Reinicia el contador si la ventana expiró
            if (now - windowStart.get(id) > windowMs) {
                windowStart.put(id, now);
                counters.get(id).set(0);
            }

            long count = counters.get(id).incrementAndGet();
            int limit = 100; // límite fijo en dev
            boolean allowed = count <= limit;

            return new Response(allowed, Map.of(
                "X-RateLimit-Remaining", String.valueOf(Math.max(0, limit - count)),
                "X-RateLimit-Burst-Capacity", String.valueOf(limit),
                "X-RateLimit-Replenish-Rate", String.valueOf(limit)
            ));
        });
    }

    @Override
    public Map<String, Config> getConfig() { return Map.of(); }

    @Override
    public Class<Config> getConfigClass() { return Config.class; }

    @Override
    public Config newConfig() { return new Config(); }

    public static class Config {}
}
```

```yaml
# application-dev.yml
spring:
  cloud:
    gateway:
      routes:
        - id: api-dev
          uri: lb://productos-service
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                rate-limiter: "#{@inMemoryRateLimiter}"
                key-resolver: "#{@ipKeyResolver}"
```

> [ADVERTENCIA] El `InMemoryRateLimiter` mostrado es una implementación simplificada para desarrollo. No usa el algoritmo Token Bucket sino una ventana fija, lo que produce el fenómeno de "double burst" en los límites de ventana. Para producción, usar siempre `RedisRateLimiter`.

## Fallback cuando Redis no está disponible

Cuando el Gateway no puede conectar con Redis, el comportamiento por defecto de `RedisRateLimiter` es **permitir todas las peticiones** (fail-open). Esto evita que un fallo de Redis provoque una interrupción del servicio, pero significa que el rate limiting deja de funcionar durante la caída.

> [CONCEPTO] El comportamiento fail-open de `RedisRateLimiter` es una decisión de diseño deliberada: la disponibilidad del servicio se prioriza sobre la aplicación del rate limiting. En sistemas donde el rate limiting es crítico para la seguridad (protección contra DDoS), se puede sobrescribir este comportamiento implementando un `RateLimiter` personalizado que falle cerrado.

Para monitorizar si Redis está disponible y alertar en caso de caída:

```yaml
management:
  health:
    redis:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health, gateway
```

## Claves que el Gateway crea en Redis

Spring Cloud Gateway crea dos claves por cada combinación de `(routeId, clientKey)`:

```
request_rate_limiter.{routeId}.{clientKey}.tokens
request_rate_limiter.{routeId}.{clientKey}.timestamp
```

Para inspeccionar en Redis y depurar en producción:

```bash
# Ver todas las claves de rate limiting
redis-cli KEYS "request_rate_limiter.*"

# Ver los tokens disponibles para una clave específica
redis-cli GET "request_rate_limiter.api-publica.user:12345.tokens"

# Ver todos los campos de rate limiting para un cliente
redis-cli SCAN 0 MATCH "request_rate_limiter.*user:12345*" COUNT 100
```

## Tests de integración con Testcontainers

Los tests de integración del rate limiter requieren un Redis real para verificar que los contadores se actualizan correctamente entre peticiones. Testcontainers levanta un contenedor Redis 7 durante la ejecución del test y `@DynamicPropertySource` inyecta las coordenadas del contenedor en el contexto de Spring antes de que arranque, de modo que el `RedisRateLimiter` del Gateway apunte al Redis del test en lugar de a uno externo.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RateLimiterIntegrationTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void dentro_del_limite_devuelve_200() {
        // Con replenishRate=10, las primeras 10 peticiones deben pasar
        for (int i = 0; i < 10; i++) {
            webTestClient.get().uri("/api/productos")
                .header("X-Forwarded-For", "192.168.1.1")
                .exchange()
                .expectStatus().isOk();
        }
    }

    @Test
    void superando_limite_devuelve_429() throws InterruptedException {
        // Llenamos el bucket
        for (int i = 0; i < 20; i++) {
            webTestClient.get().uri("/api/productos")
                .header("X-Forwarded-For", "10.0.0.99")
                .exchange();
        }
        // La siguiente petición debe ser rechazada
        webTestClient.get().uri("/api/productos")
            .header("X-Forwarded-For", "10.0.0.99")
            .exchange()
            .expectStatus().isEqualTo(429);
    }
}
```

## Tabla comparativa de modos de despliegue Redis

Los cuatro modos de despliegue cubiertos en las secciones anteriores difieren en su soporte de alta disponibilidad, consistencia entre instancias del Gateway y complejidad de configuración. La tabla siguiente condensa esas diferencias para orientar la elección según el entorno de despliegue.

| Aspecto | Standalone | Sentinel | Cluster | In-memory |
|---|---|---|---|---|
| Alta disponibilidad | No | Sí (failover automático) | Sí (sharding + HA) | No aplica |
| Consistencia multi-instancia Gateway | Sí | Sí | Sí | No |
| Escalabilidad horizontal de Redis | No | No (un master) | Sí (sharding) | No aplica |
| Complejidad de configuración | Baja | Media | Alta | Ninguna |
| Uso recomendado | Dev/staging | Producción estándar | Producción alta escala | Tests/dev local |
| Configuración Spring | `host` + `port` | `sentinel.master` + `sentinel.nodes` | `cluster.nodes` | `@Profile("dev")` bean |

## Buenas y malas prácticas

Hacer:
- Usar Redis Sentinel como mínimo en producción. Un Redis standalone sin réplica es un punto único de fallo: si cae, el rate limiting falla open y toda la protección desaparece.
- Configurar `lettuce.pool.max-active` según el número esperado de peticiones concurrentes al Gateway. El pool de conexiones de Lettuce se comparte entre todas las instancias del Gateway en el mismo proceso.
- Monitorizar la disponibilidad de Redis con el endpoint `/actuator/health` del Gateway en producción. Un `DOWN` en el componente `redis` debe generar una alerta aunque el servicio siga funcionando (fail-open).

Evitar:
- Usar el `InMemoryRateLimiter` en producción con múltiples instancias del Gateway. La consistencia entre instancias no está garantizada y los límites se multiplican por el número de instancias.
- No configurar TTL ni expiración en las claves de Redis usadas por el rate limiter. Spring Cloud Gateway gestiona la expiración automáticamente mediante los scripts Lua, pero si se crean claves manuales para depuración, recordar eliminarlas.
- Reducir `replenishRate` a valores muy bajos (1-2 req/s) sin testear el impacto en los clientes legítimos. Un límite demasiado restrictivo genera más `429` que tráfico legítimo y degrada la experiencia sin aportar protección real.

---

← [6.5.2 KeyResolver](./06-22-gateway-key-resolver.md) | [Índice](./README.md) | [6.6.1 Flujo de validación JWT →](./06-24-gateway-seguridad-flujo.md)
