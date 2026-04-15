# 6.5.1 Token Bucket, RequestRateLimiter y configuración con Redis

← [6.4.3 Filtros con lógica de negocio](./06-20-gateway-filter-negocio.md) | [Índice](./README.md) | [6.5.2 KeyResolver →](./06-22-gateway-key-resolver.md)

---

El rate limiting en un API Gateway es la defensa de primera línea contra el abuso de APIs: protege los microservicios downstream de tráfico excesivo, aplica cuotas por cliente y garantiza que los SLAs sean cumplibles incluso bajo carga elevada. Implementarlo en el Gateway, y no en cada microservicio individualmente, evita duplicar lógica y centraliza la política en un único punto de control. Spring Cloud Gateway implementa rate limiting mediante el filtro `RequestRateLimiter` con el algoritmo Token Bucket y Redis como almacén distribuido del estado del contador.

## Algoritmo Token Bucket

El Token Bucket es el algoritmo estándar para rate limiting porque permite ráfagas controladas mientras impone un límite sostenido a lo largo del tiempo.

```
┌──────────────────────────────────────────────┐
│  Token Bucket (capacidad = burstCapacity)    │
│                                              │
│  ████████████████████░░░░  18 tokens         │
│                                              │
│  ← recarga: replenishRate tokens/segundo     │
│  → consumo: requestedTokens por petición     │
└──────────────────────────────────────────────┘

replenishRate = 10 tokens/s  →  límite sostenido: 10 req/s
burstCapacity = 20 tokens    →  ráfaga máxima: 20 req instantáneas
requestedTokens = 1          →  coste por petición: 1 token

Ejemplo:
- Si llegan 20 peticiones en el primer segundo → todas aceptadas (bucket lleno)
- Si llegan 10 más en el segundo siguiente → 10 aceptadas (recarga de 10)
- Si llegan 15 más sin pausa → 10 aceptadas, 5 rechazadas con 429
```

> [CONCEPTO] `burstCapacity` es la capacidad máxima del bucket. `replenishRate` es cuántos tokens se añaden por segundo. Si `burstCapacity > replenishRate`, el sistema permite ráfagas cortas por encima de la tasa sostenida. Si `burstCapacity == replenishRate`, no hay posibilidad de ráfaga: el límite es estrictamente `replenishRate` peticiones por segundo.

## Dependencia requerida

El filtro `RequestRateLimiter` con Redis requiere el cliente reactivo Lettuce, que se incluye automáticamente con la siguiente dependencia. Sin ella, el contexto de Spring falla al arrancar indicando que no encuentra la clase `ReactiveRedisTemplate`, que es la que el filtro usa internamente para ejecutar los scripts Lua de actualización atómica del bucket.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

El cliente reactivo Lettuce (incluido en la dependencia) es el que Spring Cloud Gateway usa para comunicarse con Redis de forma no bloqueante.

## Configuración YAML completa

El bloque siguiente muestra una ruta con `RequestRateLimiter` completamente configurado: los tres parámetros del Token Bucket, la referencia SpEL al `KeyResolver` y el comportamiento ante claves vacías. La configuración de Redis se declara en la misma estructura `spring.data.redis` que usaría cualquier otro componente reactivo de la aplicación.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api-publica
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - name: RequestRateLimiter
              args:
                # Tokens que se añaden al bucket por segundo (tasa sostenida)
                redis-rate-limiter.replenishRate: 10
                # Capacidad máxima del bucket (permite ráfagas cortas)
                redis-rate-limiter.burstCapacity: 20
                # Tokens que consume cada petición (normalmente 1)
                redis-rate-limiter.requestedTokens: 1
                # Bean KeyResolver que determina la clave de particionado del límite
                key-resolver: "#{@ipKeyResolver}"
                # Si el KeyResolver devuelve vacío: true = denegar, false = permitir
                deny-empty-key: true

  data:
    redis:
      host: localhost
      port: 6379
```

> [EXAMEN] `key-resolver` acepta una expresión SpEL que referencia un bean del contexto de Spring. La sintaxis `"#{@nombreDelBean}"` es obligatoria, incluyendo las comillas y el símbolo `#`. Sin el `@`, Spring interpreta el valor como una cadena literal y no como una referencia a un bean.

## Configuración de Redis para producción

En producción, Redis debe configurarse con autenticación, TLS y un pool de conexiones dimensionado para el volumen de tráfico esperado. Cada petición al Gateway genera una llamada a Redis para ejecutar el script Lua del Token Bucket, por lo que la latencia y la disponibilidad de Redis impactan directamente en el rendimiento del Gateway.

```yaml
spring:
  data:
    redis:
      # Redis con autenticación y TLS (producción)
      host: redis.miempresa.internal
      port: 6380
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
      connect-timeout: 2s
      timeout: 1s
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5
          max-wait: 500ms
```

## Respuesta al cliente cuando se supera el límite

Cuando el bucket no tiene suficientes tokens, el Gateway devuelve `429 Too Many Requests` con los siguientes headers informativos:

| Header | Descripción | Ejemplo |
|---|---|---|
| `X-RateLimit-Remaining` | Tokens restantes en el bucket | `0` |
| `X-RateLimit-Burst-Capacity` | Capacidad máxima del bucket | `20` |
| `X-RateLimit-Replenish-Rate` | Tasa de recarga en tokens/segundo | `10` |
| `X-RateLimit-Requested-Tokens` | Tokens consumidos por la petición | `1` |

Los clientes bien comportados usan estos headers para implementar backoff adaptativo: si `X-RateLimit-Remaining` es bajo, reducen la frecuencia de peticiones antes de recibir el 429.

## RedisRateLimiter como bean programático

En lugar de configurar los parámetros en cada ruta del YAML, se puede declarar un bean `RedisRateLimiter` con valores predefinidos y referenciarlo desde las rutas:

```java
import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RateLimiterConfig {

    // Límite estándar: 10 req/s sostenidos, ráfagas de hasta 20
    @Bean
    public RedisRateLimiter standardRateLimiter() {
        return new RedisRateLimiter(10, 20, 1);
    }

    // Límite reducido para rutas de escritura
    @Bean
    public RedisRateLimiter writeRateLimiter() {
        return new RedisRateLimiter(2, 5, 1);
    }
}
```

```yaml
filters:
  # Referencia al bean en lugar de configurar los parámetros inline
  - name: RequestRateLimiter
    args:
      rate-limiter: "#{@standardRateLimiter}"
      key-resolver: "#{@ipKeyResolver}"
```

> [CONCEPTO] Usar beans `RedisRateLimiter` nombrados permite reutilizar la misma configuración en múltiples rutas y cambiar los límites en un único punto. Es especialmente útil cuando hay varias rutas con el mismo perfil de tráfico esperado.

## Rate limiting diferenciado por ruta

Cada ruta puede tener límites independientes ajustados a su perfil de carga y coste computacional. Las rutas públicas admiten tasas altas, las rutas de administración se restringen por usuario para prevenir abusos, y las operaciones costosas como exportaciones se penalizan con un coste por token mayor usando `requestedTokens`.

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Ruta pública: límite permisivo
        - id: api-publica
          uri: lb://productos-service
          predicates:
            - Path=/api/public/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@ipKeyResolver}"

        # Ruta admin: límite restrictivo
        - id: api-admin
          uri: lb://admin-service
          predicates:
            - Path=/api/admin/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 5
                redis-rate-limiter.burstCapacity: 10
                key-resolver: "#{@userKeyResolver}"

        # Ruta de escritura costosa: cada petición cuesta 5 tokens
        - id: api-export
          uri: lb://export-service
          predicates:
            - Path=/api/export/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 10
                redis-rate-limiter.requestedTokens: 5
                key-resolver: "#{@userKeyResolver}"
```

> [CONCEPTO] `requestedTokens` permite modelar el coste relativo de cada endpoint. Una operación de exportación que consume 5 tokens por petición reduce efectivamente la tasa máxima a `replenishRate / requestedTokens = 10/5 = 2 exportaciones por segundo`, sin necesidad de cambiar `replenishRate`.

## Tabla de parámetros del filtro RequestRateLimiter

El filtro `RequestRateLimiter` acepta tanto los parámetros del algoritmo Token Bucket como opciones de comportamiento ante casos límite. Los tres parámetros de `redis-rate-limiter` son propios de la implementación Redis; `key-resolver` y `deny-empty-key` son genéricos para cualquier implementación de `RateLimiter`.

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `redis-rate-limiter.replenishRate` | int | — (obligatorio) | Tokens añadidos al bucket por segundo. Define el límite sostenido |
| `redis-rate-limiter.burstCapacity` | int | — (obligatorio) | Capacidad máxima del bucket. Define el límite de ráfaga |
| `redis-rate-limiter.requestedTokens` | int | `1` | Tokens que consume cada petición. Valores >1 reducen el throughput efectivo |
| `key-resolver` | SpEL bean ref | — (obligatorio) | Expresión SpEL que referencia el bean `KeyResolver` |
| `rate-limiter` | SpEL bean ref | `RedisRateLimiter` | Bean `RateLimiter` a usar. Permite sustituir Redis por otra implementación |
| `deny-empty-key` | boolean | `true` | Si `true`, rechaza la petición cuando `KeyResolver` devuelve vacío |
| `empty-key-status` | HttpStatus | `FORBIDDEN` | Código de respuesta cuando `deny-empty-key=true` y la clave está vacía |

## Buenas y malas prácticas

Hacer:
- Usar `burstCapacity = 2 × replenishRate` como punto de partida. Esta regla de thumb permite absorber ráfagas breves sin abrir la puerta a abusos sostenidos. Ajustar en función de los patrones de tráfico reales observados en métricas.
- No configurar `RequestRateLimiter` en rutas de health check (`/actuator/health`) ni de Eureka. Estas rutas son llamadas por la infraestructura de forma frecuente y su bloqueo puede causar problemas de registro y monitorización.
- Monitorizar el header `X-RateLimit-Remaining` en los dashboards de producción. Un valor persistentemente cercano a 0 indica que los límites están demasiado ajustados o que el tráfico ha crecido más de lo previsto.
- Documentar en el YAML de cada ruta el criterio que justifica los valores de `replenishRate` y `burstCapacity`, para que los cambios futuros sean razonados.

Evitar:
- No configurar `key-resolver`. Sin un `KeyResolver` explícito, el filtro usa la dirección IP remota como clave por defecto, lo que puede penalizar a todos los usuarios de una misma red corporativa o NAT como si fueran un único cliente.
- Usar Redis standalone en producción con múltiples instancias del Gateway sin configurar Redis Sentinel o Cluster. Si Redis falla, el comportamiento de fallback (permitir o bloquear todo) puede no ser el esperado.
- Poner el mismo `replenishRate` y `burstCapacity` en todas las rutas sin considerar el coste relativo de cada operación. Un endpoint que genera un informe tardará más en el backend y debería tener un límite más bajo que un endpoint de lectura simple.

## Comparación: RequestRateLimiter con Redis vs in-memory

La elección entre Redis y una solución in-memory determina si el límite se aplica de forma consistente entre todas las instancias del Gateway o de forma independiente por instancia. En despliegues con múltiples réplicas, un contador local por instancia multiplica el límite efectivo por el número de pods, anulando la protección.

| Aspecto | Redis (RequestRateLimiter) | In-memory (Resilience4j / local) |
|---|---|---|
| Consistencia con múltiples instancias | Sí (estado compartido en Redis) | No (cada instancia tiene su propio contador) |
| Dependencias adicionales | `spring-boot-starter-data-redis-reactive` | Solo Resilience4j (ya en el classpath si se usa CircuitBreaker) |
| Alta disponibilidad | Requiere Redis Cluster o Sentinel | No aplica (stateless por instancia) |
| Adecuado para producción multi-instancia | Sí | No |
| Adecuado para desarrollo local | Sí (con Redis en Docker) | Sí (sin dependencias externas) |
| Latencia adicional por petición | ~1ms (llamada a Redis local) | ~0ms (operación en memoria) |

---

← [6.4.3 Filtros con lógica de negocio](./06-20-gateway-filter-negocio.md) | [Índice](./README.md) | [6.5.2 KeyResolver →](./06-22-gateway-key-resolver.md)
