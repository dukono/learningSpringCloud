# 6.4 Filtros Personalizados: GatewayFilter y GlobalFilter

← [6.3 Filtros Predefinidos](./06-03-gateway-filtros-predefinidos.md) | [Índice](./README.md) | [6.5 Rate Limiting →](./06-05-gateway-rate-limiting.md)

---

Los filtros predefinidos cubren el 95 % de los casos de uso. Crear un filtro personalizado tiene sentido cuando la lógica es específica del dominio y no existe un filtro predefinido equivalente: verificar una firma HMAC en el body de la petición, transformar respuestas XML de un sistema legacy, implementar throttling basado en el plan de suscripción del usuario, o añadir headers de trazabilidad que requieren lógica de negocio para calcularse. Hay dos tipos: `GatewayFilter` (aplicado solo a rutas donde se registra explícitamente) y `GlobalFilter` (aplicado automáticamente a todas las rutas).

---

## 6.4.1 Diagrama de orden de ejecución

```
Petición entrante
        │
        ▼
┌───────────────────────────────────┐
│  GlobalFilter  orden -200         │  ← JwtAuthFilter (ejecuta primero)
│  GlobalFilter  orden -100         │  ← LoggingGlobalFilter
│  GlobalFilter  orden   0          │  ← CorrelationIdFilter
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  GatewayFilters de la ruta (pre)  │  ← filtros declarados en la ruta
│   RateLimiter → CircuitBreaker    │     en el orden declarado en YAML/DSL
│   → StripPrefix → AddHeader       │
└───────────────┬───────────────────┘
                │ petición transformada
                ▼
        Microservicio destino
                │ respuesta
                ▼
┌───────────────────────────────────┐
│  GatewayFilters de la ruta (post) │  ← en orden inverso al de pre
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  GlobalFilters (post)             │  ← en orden inverso al de pre
│  GlobalFilter  orden   0          │
│  GlobalFilter  orden -100         │
│  GlobalFilter  orden -200         │  ← JwtAuthFilter (ejecuta último en post)
└───────────────┬───────────────────┘
                │
                ▼
       Respuesta al cliente
```

> [CONCEPTO] El número de `getOrder()` en un `GlobalFilter` determina cuándo se ejecuta en la fase **pre** (menor número = antes) y cuándo en la fase **post** (menor número = después, es decir, más próximo al cliente en la vuelta). Un `GlobalFilter` con `order = -200` es el primero en la ida y el último en la vuelta.

---

## 6.4.2 `AbstractGatewayFilterFactory` — filtro por ruta con parámetros YAML

Ejemplo de un filtro que extiende `AbstractGatewayFilterFactory` para ser usado en YAML por nombre.

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import reactor.core.publisher.Mono;
import java.util.List;

@Component
public class AuditGatewayFilterFactory
        extends AbstractGatewayFilterFactory<AuditGatewayFilterFactory.Config> {

    private static final Logger log = LoggerFactory.getLogger(AuditGatewayFilterFactory.class);

    public AuditGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("sistema");   // orden de parámetros en la forma compacta de YAML
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // PRE: registra la petición antes de enrutar
            log.info("[AUDIT][{}] {} {} - IP: {}",
                config.getSistema(),
                request.getMethod(),
                request.getURI().getPath(),
                request.getRemoteAddress().getAddress().getHostAddress());

            long inicio = System.currentTimeMillis();

            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                // POST: registra la respuesta cuando el microservicio ya respondió
                ServerHttpResponse response = exchange.getResponse();
                long latencia = System.currentTimeMillis() - inicio;
                log.info("[AUDIT][{}] respuesta {} en {}ms",
                    config.getSistema(),
                    response.getStatusCode(),
                    latencia);
            }));
        };
    }

    public static class Config {
        private String sistema = "GATEWAY";

        public String getSistema() { return sistema; }
        public void setSistema(String sistema) { this.sistema = sistema; }
    }
}
```

```yaml
# Uso en YAML — nombre = clase sin "GatewayFilterFactory"
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - Audit=PEDIDOS-SERVICE   # pasa "PEDIDOS-SERVICE" como config.sistema
```

---

## 6.4.3 `GlobalFilter` + `Ordered` — filtro para todas las rutas

Ejemplo de un `GlobalFilter` que implementa `Ordered` para definir su orden de ejecución.

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;
import java.util.List;
import java.util.UUID;

@Component
public class CorrelationIdGlobalFilter implements GlobalFilter, Ordered {

    private static final String CORRELATION_HEADER = "X-Correlation-Id";

    // Rutas que no requieren procesamiento (health, actuator, login)
    private static final List<String> RUTAS_EXCLUIDAS = List.of(
        "/actuator/health", "/actuator/info", "/auth/login", "/auth/register"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();

        // Excluir rutas públicas sin modificar la petición
        if (RUTAS_EXCLUIDAS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        // Reutilizar el correlation ID si el cliente ya lo envía; generar uno nuevo si no
        String correlationId = request.getHeaders().getFirst(CORRELATION_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        final String finalCorrelationId = correlationId;

        // Propagar el correlation ID hacia el microservicio downstream
        ServerHttpRequest mutatedRequest = request.mutate()
            .header(CORRELATION_HEADER, finalCorrelationId)
            .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build())
            .then(Mono.fromRunnable(() -> {
                // Devolver el mismo correlation ID al cliente en la respuesta
                exchange.getResponse().getHeaders()
                    .add(CORRELATION_HEADER, finalCorrelationId);
            }));
    }

    @Override
    public int getOrder() {
        return -100;   // ejecuta después del JwtAuthFilter (-200) y antes de los GatewayFilters
    }
}
```

```yaml
# No requiere ninguna configuración YAML adicional.
# El GlobalFilter se aplica automáticamente a todas las rutas por ser un @Component.
```

---

## 6.4.4 Exclusión de rutas públicas en GlobalFilter

A veces es necesario excluir ciertas rutas de la lógica de un `GlobalFilter`, como las rutas públicas de login o las de health check. Esto se puede hacer verificando el path de la petición.

```java
// Ejemplo dentro del método filter() de un GlobalFilter
private static final List<String> RUTAS_EXCLUIDAS = List.of(
    "/actuator/health", "/actuator/info", "/auth/login", "/auth/register"
);

@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    ServerHttpRequest request = exchange.getRequest();
    String path = request.getURI().getPath();

    // Excluir rutas públicas sin modificar la petición
    if (RUTAS_EXCLUIDAS.stream().anyMatch(path::startsWith)) {
        return chain.filter(exchange);
    }

    // ... lógica del filtro ...
}
```

---

## 6.4.5 Filtro con acceso a beans de Spring: validación HMAC

Un caso de uso frecuente que no cubre ningún filtro predefinido es la validación de firma HMAC en el body de la petición: sistemas de pago o webhooks de terceros (GitHub, Stripe) envían un header `X-Signature` calculado como HMAC-SHA256 del body. Si el body no coincide con la firma, la petición debe rechazarse con 401. Este filtro combina `CacheRequestBody` (para leer el body más de una vez), inyección de un bean `HmacKeyProvider` (que puede leer la clave desde Vault o una BD), y lógica reactiva sin bloqueos.

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import reactor.core.publisher.Mono;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;
import java.util.List;

@Component
public class HmacValidationGatewayFilterFactory
        extends AbstractGatewayFilterFactory<HmacValidationGatewayFilterFactory.Config> {

    @Autowired
    private HmacKeyProvider hmacKeyProvider;   // bean que devuelve la clave desde Vault o BD

    public HmacValidationGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("servicio");
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // CacheRequestBody debe estar antes de este filtro en la lista
            String body = exchange.getAttribute(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
            String signatureHeader = exchange.getRequest().getHeaders().getFirst("X-Signature");

            if (body == null || signatureHeader == null) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            return hmacKeyProvider.getKeyForService(config.getServicio())
                .flatMap(secretKey -> {
                    String computedSignature = computeHmac(body, secretKey);
                    if (!computedSignature.equals(signatureHeader)) {
                        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                        return exchange.getResponse().setComplete();
                    }
                    return chain.filter(exchange);
                });
        };
    }

    private String computeHmac(String data, String key) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(key.getBytes(), "HmacSHA256"));
            return Base64.getEncoder().encodeToString(mac.doFinal(data.getBytes()));
        } catch (Exception e) {
            throw new IllegalStateException("Error calculando HMAC", e);
        }
    }

    public static class Config {
        private String servicio;
        public String getServicio() { return servicio; }
        public void setServicio(String servicio) { this.servicio = servicio; }
    }
}
```

```yaml
# Uso en YAML: CacheRequestBody DEBE ir antes de HmacValidation
filters:
  - name: CacheRequestBody
    args:
      bodyClass: java.lang.String
  - HmacValidation=pagos-externos
```

---

## 6.4.6 Parámetros de extensión

| Elemento | Tipo | Descripción |
|---|---|---|
| `getOrder()` | int | Prioridad de ejecución; menor = antes en pre, después en post |
| `shortcutFieldOrder()` | List\<String\> | Orden de parámetros en la forma compacta de YAML (`- NombreFiltro=valor`) |
| `Config` (clase interna) | POJO | Parámetros del filtro; Spring los inyecta automáticamente desde YAML |
| `apply(Config)` | GatewayFilter | Retorna el filtro que se ejecutará; `chain.filter()` delega al siguiente |
| `exchange.mutate()` | ServerWebExchange | Crea una copia inmutable del exchange con headers/body modificados |

---

## 6.4.7 Buenas y malas prácticas

**Hacer:**
- Implementar `Ordered` en los `GlobalFilter` y asignar un `getOrder()` explícito — sin orden explícito, la ejecución entre varios `GlobalFilter` es no determinista.
- Usar `exchange.getRequest().mutate()` para modificar headers en lugar de intentar modificar el request directamente — `ServerHttpRequest` es inmutable en WebFlux.
- Excluir rutas de salud y login dentro del propio `GlobalFilter` chequeando el path — aplicar lógica de autenticación o auditoría a `/actuator/health` genera ruido y puede romper los health checks del orquestador.

**Evitar:**
- Bloquear dentro de un filtro reactivo con `.block()` — el modelo reactivo de WebFlux usa un pool de hilos muy pequeño; bloquear un hilo cancela el beneficio de rendimiento del Gateway y puede causar deadlocks.
- Guardar estado mutable en campos de instancia del filtro — los filtros son singletons de Spring; el estado concurrente compartido entre peticiones genera condiciones de carrera.
- Registrar el body completo de cada petición en el `GlobalFilter` sin usar `CacheRequestBody` primero — leer el body en el filtro consume el stream reactivo y el siguiente filtro o el microservicio recibirán un body vacío.

---

## 6.4.8 Comparación: GatewayFilter vs GlobalFilter

| Aspecto | GatewayFilter (`AbstractGatewayFilterFactory`) | GlobalFilter |
|---|---|---|
| **Alcance** | Solo rutas donde se declara explícitamente | Todas las rutas automáticamente |
| **Configuración** | Parámetros por ruta desde YAML | Sin parámetros por ruta (configuración fija o desde `@Value`) |
| **Caso de uso típico** | Auditoría por sistema, transformaciones específicas de dominio | JWT, correlation ID, logging global |
| **Exclusión de rutas** | No aplica — solo actúa donde se declara | Hay que implementar la exclusión manualmente en el código |
| **Declaración en YAML** | `- NombreFiltro=param1,param2` | No requiere declaración; activo por ser `@Component` |

---

← [6.3 Filtros Predefinidos](./06-03-gateway-filtros-predefinidos.md) | [Índice](./README.md) | [6.5 Rate Limiting →](./06-05-gateway-rate-limiting.md)
