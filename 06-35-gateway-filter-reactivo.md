# 6.8.3 GlobalFilter con repositorio reactivo y lógica de negocio por petición

← [6.8.2 RouteDefinitionWriter](./06-34-gateway-route-writer.md) | [Índice](./README.md) | [6.8.4 Migración Zuul →](./06-36-gateway-migracion-zuul.md)

---

Los filtros predefinidos de Spring Cloud Gateway (sección 6.3) operan con datos estáticos definidos en YAML. Cuando la lógica de un filtro depende de datos que cambian en tiempo de ejecución —planes de suscripción por cliente, listas de bloqueo, permisos por API key almacenados en base de datos o Redis—, necesitas un `GlobalFilter` que consulte un repositorio reactivo por cada petición. El reto técnico es hacer esa consulta sin bloquear el hilo del event loop: la cadena completa debe permanecer no bloqueante usando operadores de Project Reactor.

> [PREREQUISITO] Este documento requiere familiaridad con `GlobalFilter` y `Ordered` (sección 6.4.2), con `RouteDefinitionLocator` (sección 6.8.1) y con los fundamentos de Project Reactor (`Mono`, `flatMap`, `switchIfEmpty`).

## Flujo de un GlobalFilter con acceso reactivo

El filtro recibe la petición, extrae un identificador (API key, user-id del JWT, IP), consulta el repositorio reactivo de forma no bloqueante y, según el resultado, muta la petición y continúa la cadena o devuelve directamente una respuesta de error al cliente.

```
Petición entrante
    │
    ▼
GlobalFilter.filter(exchange, chain)
    │  extrae identificador de la petición (API key, user-id del JWT, IP...)
    │
    ▼
repositorioReactivo.findBy(identificador)   ← operación no bloqueante
    │  devuelve Mono<DatoNegocio>
    │
    ├── [dato encontrado]
    │       │  aplica lógica (enriquecer headers, verificar límite, etc.)
    │       ▼
    │   chain.filter(mutatedExchange)        ← continúa la cadena
    │
    └── [dato no encontrado / acceso denegado]
            ▼
        exchange.getResponse().setStatusCode(HttpStatus.XXX)
        exchange.getResponse().setComplete()  ← corta la cadena
```

## Implementación: filtro de cuota por plan de suscripción

El siguiente filtro consulta el plan de suscripción asociado a una API key, añade el límite de cuota como header interno y rechaza peticiones de claves desconocidas con `401 Unauthorized`.

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

// El repositorio devuelve Mono<SubscriptionPlan> y accede a Redis o R2DBC
@Component
public class SubscriptionPlanFilter implements GlobalFilter, Ordered {

    private static final String API_KEY_HEADER = "X-Api-Key";
    private static final String PLAN_HEADER    = "X-Subscription-Plan";
    private static final String QUOTA_HEADER   = "X-Daily-Quota";

    private final SubscriptionPlanRepository planRepository;

    public SubscriptionPlanFilter(SubscriptionPlanRepository planRepository) {
        this.planRepository = planRepository;
    }

    @Override
    public int getOrder() {
        // Ejecutar después del JwtAuthFilter (orden -200) pero antes del rate limiter
        return -100;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders().getFirst(API_KEY_HEADER);

        if (apiKey == null || apiKey.isBlank()) {
            return unauthorized(exchange, "Header X-Api-Key ausente");
        }

        return planRepository.findByApiKey(apiKey)
            .flatMap(plan -> {
                // Mutar la petición para añadir headers internos antes de reenviarla
                ServerWebExchange mutated = exchange.mutate()
                    .request(r -> r
                        .header(PLAN_HEADER, plan.getName())
                        .header(QUOTA_HEADER, String.valueOf(plan.getDailyQuota())))
                    .build();
                return chain.filter(mutated);
            })
            .switchIfEmpty(unauthorized(exchange, "API key desconocida: " + apiKey));
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String reason) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        // Añadir razón en header para que el cliente sepa qué falló
        exchange.getResponse().getHeaders().add("X-Auth-Failure", reason);
        return exchange.getResponse().setComplete();
    }
}
```

## Repositorio reactivo: acceso a Redis

Para datos de alta frecuencia de lectura (plan de suscripción, lista de bloqueo) Redis reactivo ofrece latencia sub-milisegundo sin bloquear el event loop:

```java
import org.springframework.data.redis.core.ReactiveRedisTemplate;
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Mono;

@Repository
public class SubscriptionPlanRepository {

    // Clave en Redis: "sub:plan:{apiKey}" → JSON del plan
    private static final String KEY_PREFIX = "sub:plan:";

    private final ReactiveRedisTemplate<String, SubscriptionPlan> redisTemplate;

    public SubscriptionPlanRepository(
            ReactiveRedisTemplate<String, SubscriptionPlan> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public Mono<SubscriptionPlan> findByApiKey(String apiKey) {
        return redisTemplate.opsForValue().get(KEY_PREFIX + apiKey);
    }

    public Mono<Boolean> save(String apiKey, SubscriptionPlan plan,
                              java.time.Duration ttl) {
        return redisTemplate.opsForValue().set(KEY_PREFIX + apiKey, plan, ttl);
    }
}
```

## Modelo de dominio

`SubscriptionPlan` almacena el nombre del plan, la cuota diaria de peticiones y el flag de activación. Debe implementar `Serializable` para que `Jackson2JsonRedisSerializer` pueda serializarlo y deserializarlo al leer y escribir en Redis.

```java
import java.io.Serializable;

// Debe ser serializable para Redis
public class SubscriptionPlan implements Serializable {

    private String name;       // "FREE", "PRO", "ENTERPRISE"
    private int dailyQuota;    // peticiones permitidas por día
    private boolean active;

    public SubscriptionPlan() {}

    public SubscriptionPlan(String name, int dailyQuota, boolean active) {
        this.name = name;
        this.dailyQuota = dailyQuota;
        this.active = active;
    }

    public String getName() { return name; }
    public int getDailyQuota() { return dailyQuota; }
    public boolean isActive() { return active; }

    public void setName(String name) { this.name = name; }
    public void setDailyQuota(int dailyQuota) { this.dailyQuota = dailyQuota; }
    public void setActive(boolean active) { this.active = active; }
}
```

## Configuración de RedisTemplate reactivo

Para que `ReactiveRedisTemplate` serialice `SubscriptionPlan` como JSON en lugar de usar la serialización Java por defecto, se declara un `@Bean` con `Jackson2JsonRedisSerializer` como serializador de valor y `StringRedisSerializer` para las claves, combinados en un `RedisSerializationContext`.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.ReactiveRedisConnectionFactory;
import org.springframework.data.redis.core.ReactiveRedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public ReactiveRedisTemplate<String, SubscriptionPlan> subscriptionPlanRedisTemplate(
            ReactiveRedisConnectionFactory factory) {

        Jackson2JsonRedisSerializer<SubscriptionPlan> valueSerializer =
            new Jackson2JsonRedisSerializer<>(SubscriptionPlan.class);

        RedisSerializationContext<String, SubscriptionPlan> context =
            RedisSerializationContext.<String, SubscriptionPlan>newSerializationContext(
                    new StringRedisSerializer())
                .value(valueSerializer)
                .build();

        return new ReactiveRedisTemplate<>(factory, context);
    }
}
```

## Patrón: caché local con tiempo de vida para reducir llamadas a Redis

Cuando el mismo API key se usa en cientos de peticiones por segundo, incluso Redis puede convertirse en un cuello de botella. Un caché en memoria con `Caffeine` reduce las llamadas externas manteniendo la naturaleza reactiva con `Mono.fromCallable`:

```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;

public class CachingSubscriptionPlanRepository {

    // Caché en memoria: máx 10.000 entradas, TTL 60s
    private final Cache<String, SubscriptionPlan> localCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofSeconds(60))
        .build();

    private final SubscriptionPlanRepository redisRepository;

    public CachingSubscriptionPlanRepository(SubscriptionPlanRepository redisRepository) {
        this.redisRepository = redisRepository;
    }

    public Mono<SubscriptionPlan> findByApiKey(String apiKey) {
        SubscriptionPlan cached = localCache.getIfPresent(apiKey);
        if (cached != null) {
            return Mono.just(cached);
        }
        // El acceso a Caffeine es síncrono pero muy rápido; el subscribeOn evita
        // que bloquee el thread del event loop si hay contención en el caché
        return redisRepository.findByApiKey(apiKey)
            .doOnNext(plan -> localCache.put(apiKey, plan));
    }
}
```

> [CONCEPTO] El patrón de dos niveles de caché (memoria local → Redis) es habitual en Gateways de alto tráfico. La caché local reduce la latencia a microsegundos para las claves más activas. El TTL corto (30–120 s) garantiza que los cambios de plan se propaguen sin necesidad de invalidación explícita.

## Parámetros clave del filtro

Los siguientes elementos de la API de Gateway son los que intervienen directamente en la implementación de un `GlobalFilter` con repositorio reactivo: permiten mutar la petición, continuar o cortar la cadena y gestionar la rama vacía del repositorio.

| Elemento | Descripción |
|---|---|
| `exchange.mutate().request(r -> r.header(...))` | Añade o modifica headers de la petición antes de reenviarla al microservicio |
| `chain.filter(mutated)` | Continúa la cadena con la petición mutada |
| `exchange.getResponse().setComplete()` | Corta la cadena y devuelve la respuesta al cliente sin llamar al microservicio |
| `switchIfEmpty(Mono<Void>)` | Rama ejecutada cuando el repositorio no devuelve resultado |
| `getOrder()` | Controla la posición del filtro en la cadena; valores más bajos se ejecutan antes |

## Buenas y malas prácticas

Hacer:
- Usar `switchIfEmpty` para manejar el caso en que el repositorio no encuentra el dato. Sin esta rama, una API key desconocida simplemente continuaría hacia el microservicio con los headers de plan ausentes, provocando errores 500 inesperados en el servicio destino.
- Acotar el TTL del caché local al tiempo máximo aceptable de propagación de un cambio de plan. Un TTL de 60 s significa que un downgrade de plan puede seguir permitiendo cuota alta durante ese tiempo: es una decisión de negocio, no un fallo técnico.
- Añadir métricas (`Counter`, `Timer`) en el filtro para registrar cuántas peticiones se autorizan, rechazan o tienen caché miss. Sin estas métricas es imposible detectar un ataque de fuerza bruta contra API keys inválidas.

Evitar:
- Nunca llamar a un repositorio bloqueante (JDBC síncrono, `RestTemplate`) directamente dentro del `filter()`. Hacerlo bloquea el event loop de Netty, degrada el throughput de todo el Gateway y puede provocar un interbloqueo si todos los threads del pool quedan bloqueados esperando la base de datos.
- No ignorar el `Mono` devuelto por `chain.filter()`. Si el `filter()` devuelve `Mono.empty()` en lugar de `chain.filter(exchange)`, la petición queda colgada sin respuesta hasta que se cierra la conexión por timeout del cliente.

---

← [6.8.2 RouteDefinitionWriter](./06-34-gateway-route-writer.md) | [Índice](./README.md) | [6.8.4 Migración Zuul →](./06-36-gateway-migracion-zuul.md)
