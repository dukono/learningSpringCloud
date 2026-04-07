# Parte 6.3 — API Gateway: Filtros Predefinidos y Personalizados

← [Rutas y Predicates](./06-02-gateway-rutas.md) | [Volver al índice](./README.md) | Siguiente: [Avanzado →](./06-04-gateway-avanzado.md)

---

## 6.5 Filtros predefinidos más importantes

### StripPrefix — eliminar segmentos del path

El problema que resuelve `StripPrefix` es una convención de diseño frecuente en APIs con Gateway: el Gateway agrupa todos los endpoints bajo prefijos por dominio (`/api/pedidos/**`, `/api/productos/**`) para que el cliente tenga URLs uniformes. Pero los microservicios no saben nada de ese prefijo — `pedidos-service` espera peticiones en `/pedidos/**`, no en `/api/pedidos/**`. El prefijo `/api` es una decisión del Gateway, no del servicio. `StripPrefix` elimina ese prefijo antes de enrutar, manteniendo los microservicios agnósticos a cómo está estructurado el Gateway.

```yaml
filters:
  - StripPrefix=1
# /api/pedidos/42 → /pedidos/42  (elimina el primer segmento)

  - StripPrefix=2
# /api/v1/pedidos/42 → /pedidos/42  (elimina los dos primeros segmentos)
```

### RewritePath — reescribir la URL con regex

`RewritePath` y `StripPrefix` resuelven problemas parecidos pero en casos distintos. `StripPrefix` solo elimina segmentos del inicio del path. `RewritePath` usa expresiones regulares y permite transformaciones arbitrarias: migrar versiones de URL (`/api/v1/` → sin versión), adaptar rutas de sistemas legacy, o reestructurar paths que no consisten en eliminar segmentos. Elegir `StripPrefix` cuando la transformación es simple; `RewritePath` cuando el path destino tiene una forma que no puede obtenerse solo eliminando segmentos del principio.

```yaml
filters:
  - RewritePath=/api/v1/(?<segmento>.*), /${segmento}
# /api/v1/pedidos/42 → /pedidos/42

  - RewritePath=/legacy/(?<recurso>.*), /v2/${recurso}
# /legacy/usuarios → /v2/usuarios
```

### SetPath — reemplazar el path completo

`SetPath` complementa a `RewritePath` y `StripPrefix` para casos donde la transformación no consiste en manipular segmentos del path existente sino en construir el path destino desde cero usando variables capturadas por el predicate. Es útil cuando el microservicio destino tiene una estructura de URLs completamente diferente a la que expone el Gateway al exterior: el predicate `Path` captura las variables de la URL entrante y `SetPath` las compone en la URL interna.

```yaml
filters:
  - SetPath=/v2/api/{segment}
# Reemplaza el path entero; {segment} viene de la variable del predicate Path
```

### PrefixPath — añadir un prefijo al path

Complemento a `StripPrefix`: en lugar de eliminar segmentos, añade un prefijo delante del path:

```yaml
filters:
  - PrefixPath=/api
# /pedidos/42 → /api/pedidos/42
```

Útil cuando el microservicio expone sus rutas bajo un prefijo y el cliente no lo incluye:

```yaml
# El cliente llama a /pedidos/42
# El microservicio espera /v2/pedidos/42
filters:
  - PrefixPath=/v2
```

### AddRequestHeader / AddResponseHeader

Añadir headers en el Gateway permite propagar metadatos que los microservicios necesitan sin que el cliente externo tenga que enviarlos ni saber que existen. Casos de uso frecuentes: un identificador de correlación (`X-Correlation-Id`) para que cada microservicio lo incluya en sus logs y pueda trazarse toda la cadena de llamadas; un marcador del entorno activo (`X-Environment: production`) para que los servicios ajusten su comportamiento; o información de la instancia del Gateway que procesó la petición para depuración en producción.

```yaml
filters:
  - AddRequestHeader=X-Request-Source, gateway         # añade si no existe
  - AddResponseHeader=X-Powered-By, spring-cloud-gateway
```

### SetRequestHeader / SetResponseHeader

`AddRequestHeader` añade un header incluso si ya existe (puede crear duplicados), y `AddRequestHeadersIfNotPresent` solo actúa si el header no existe. `SetRequestHeader` es el término medio: si el header ya existe, lo reemplaza; si no existe, lo crea. Es el filtro correcto cuando el Gateway necesita garantizar que un header tiene un valor específico, independientemente de lo que el cliente externo haya enviado — por ejemplo, forzar una política de caché específica en todas las respuestas de una ruta.

```yaml
filters:
  - SetRequestHeader=X-Request-Id, mi-valor-fijo       # sobreescribe el header si ya existe
  - SetResponseHeader=Cache-Control, no-store
```

### AddRequestHeadersIfNotPresent — añadir solo si no existe

A diferencia de `AddRequestHeader` (que siempre añade el header, pudiendo duplicarlo), este filtro solo actúa si el header **no está ya presente** en la petición:

```yaml
filters:
  - AddRequestHeadersIfNotPresent=X-Request-Id:default-id, X-Source:gateway
  # Si la petición ya trae X-Request-Id, se respeta su valor
  # Si no lo trae, se añade con "default-id"
```

Útil para establecer valores por defecto que el cliente puede sobreescribir.

### MapRequestHeader — renombrar un header

Copia el valor de un header a otro nombre. El header original se mantiene:

```yaml
filters:
  - MapRequestHeader=X-Client-Token, Authorization
  # Copia el valor de X-Client-Token al header Authorization
  # Caso de uso: el cliente envía un header propio y el microservicio espera Authorization
```

### RemoveRequestHeader / RemoveResponseHeader

Los navegadores envían automáticamente cookies, headers de sesión y otros datos que son válidos en la comunicación cliente-Gateway pero no deben llegar a los microservicios internos. Las cookies contienen estado del cliente (sesiones, preferencias) que los servicios de backend no deben ver ni procesar: hacerlo crearía un acoplamiento indebido entre el estado del cliente y la lógica del servicio. En el otro sentido, los microservicios pueden incluir en sus respuestas headers internos con información de diagnóstico (versión del servicio, nombre del pod, IP interna) que no deben exponerse al cliente externo por razones de seguridad.

```yaml
filters:
  - RemoveRequestHeader=Cookie             # elimina cookies antes de pasar al microservicio
  - RemoveResponseHeader=X-Internal-Info   # oculta headers internos al cliente
```

### RewriteResponseHeader — modificar valor de header de respuesta

Modifica el valor de un header de respuesta usando regex, sin eliminarlo:

```yaml
filters:
  - RewriteResponseHeader=X-Response-Red, password=[^&]+, password=***
  # Oculta contraseñas en headers de diagnóstico antes de enviarlos al cliente
  # Formato: nombre-header, regex-a-buscar, reemplazo
```

### RewriteLocationResponseHeader — corregir redirects en producción

Cuando el Gateway está detrás de un load balancer o proxy externo, las respuestas de redirección del microservicio incluyen la URL interna (ej. `http://pedidos-service:8080/nuevo-id`). Este filtro reescribe el header `Location` para que apunte a la URL pública:

```yaml
filters:
  - RewriteLocationResponseHeader=AS_IN_REQUEST, Location, ,
  # Parámetros: stripVersionMode, locationHeaderName, hostValue, protocolsRegex
  # AS_IN_REQUEST: usa el mismo host y protocolo que la petición original
  # Resultado: http://pedidos-service:8080/nuevo-id → https://api.miempresa.com/nuevo-id
```

> **[ADVERTENCIA]** Sin este filtro, las respuestas de redirección (301/302) de los microservicios exponen URLs internas al cliente. Es un problema de seguridad y también rompe la navegación del cliente.

### DedupeResponseHeader — eliminar duplicados

El problema de headers duplicados ocurre cuando dos capas independientes añaden el mismo header de respuesta sin saber que la otra también lo hace. El caso más frecuente es CORS: el microservicio puede incluir `Access-Control-Allow-Origin` en su respuesta, y el Gateway también lo añade como parte de su gestión centralizada de CORS. El resultado es un header con dos valores (`Access-Control-Allow-Origin: *\nAccess-Control-Allow-Origin: https://miapp.com`), que los navegadores rechazan con un error de CORS que parece un problema de configuración cuando en realidad es un problema de duplicación.

```yaml
filters:
  # Útil cuando el Gateway y el microservicio añaden el mismo header CORS
  - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
  # El segundo parámetro es la estrategia: RETAIN_FIRST (defecto), RETAIN_LAST, RETAIN_UNIQUE
```

### RequestSize — rechazar peticiones demasiado grandes

Sin un límite de tamaño, cualquier cliente puede enviar una petición con un body de varios gigabytes. El Gateway la aceptaría y la mantendría en memoria mientras la reenvía al microservicio, consumiendo recursos durante todo el tiempo que tarde en transmitirse. Un único cliente malicioso o con un bug podría agotar la memoria del Gateway o del microservicio destino, afectando a todos los demás clientes. `RequestSize` rechaza estas peticiones inmediatamente con 413, sin siquiera contactar al microservicio.

```yaml
filters:
  - name: RequestSize
    args:
      maxSize: 5MB   # rechaza con 413 Payload Too Large si el body supera este límite
                     # defecto: 5MB si no se configura
```

### SecureHeaders — headers de seguridad estándar

Cada header que añade `SecureHeaders` protege contra un vector de ataque específico del navegador. `Strict-Transport-Security` evita ataques de degradación SSL (que un proxy intermediario fuerce la conexión a HTTP inseguro). `X-Frame-Options: DENY` evita el clickjacking (que la aplicación sea embebida en un `<iframe>` de una página maliciosa). `X-Content-Type-Options: nosniff` evita que el navegador ejecute un fichero HTML disfrazado de imagen. `Content-Security-Policy` restringe desde qué orígenes puede cargar scripts el navegador, reduciendo el impacto de ataques XSS. Centralizarlos en el Gateway garantiza cobertura uniforme; sin Gateway, cada microservicio tendría que configurarlos individualmente.

```yaml
filters:
  - SecureHeaders
  # Añade automáticamente:
  # X-XSS-Protection: 1 ; mode=block
  # Strict-Transport-Security: max-age=631138519
  # X-Frame-Options: DENY
  # X-Content-Type-Options: nosniff
  # Content-Security-Policy: ...
  # Referrer-Policy: no-referrer
```

```yaml
# Para excluir headers específicos de SecureHeaders:
spring:
  cloud:
    gateway:
      filter:
        secure-headers:
          disable:
            - x-frame-options
            - content-security-policy
```

### Retry — reintentos automáticos

Los microservicios en entornos cloud tienen fallos transitorios: una instancia recibe un deploy justo cuando llega una petición, el garbage collector hace una pausa, una base de datos tiene un pico de latencia. Estos fallos son pasajeros y una segunda petición unos milisegundos después normalmente tiene éxito. Sin `Retry`, ese fallo transitorio llega al usuario como un error 502. Con `Retry`, el Gateway absorbe silenciosamente el reintento y el usuario recibe una respuesta correcta sin saber que hubo un problema.

```yaml
filters:
  - name: Retry
    args:
      retries: 3                              # número de reintentos (no incluye el intento original)
      statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE   # códigos HTTP que disparan el reintento
      methods: GET, POST                      # métodos HTTP que se reintentarán
      series: SERVER_ERROR                    # también por serie (5xx)
      exceptions: java.io.IOException         # excepciones que disparan el reintento
      backoff:
        firstBackoff: 10ms                    # espera antes del primer reintento
        maxBackoff: 50ms                      # espera máxima entre reintentos
        factor: 2                             # factor de multiplicación (backoff exponencial)
        basedOnPreviousValue: false           # si true, el factor se aplica al tiempo anterior
```

> **[ADVERTENCIA]** No usar `Retry` con métodos no idempotentes (`POST`, `PUT`) sin verificar que el microservicio destino es idempotente. Un POST reintentado puede crear recursos duplicados.

### RedirectTo — redirigir

Útil para tres casos concretos: migrar URLs antiguas a nuevas rutas sin romper bookmarks o enlaces externos (`301` permanente); forzar HTTPS cuando un cliente llega por HTTP (`302` o `301` a la versión HTTPS); y redirigir tráfico durante una transición de dominio. A diferencia de `RewritePath` que transforma la URL internamente, `RedirectTo` devuelve la redirección al cliente y el cliente hace una nueva petición a la URL destino.

```yaml
filters:
  - RedirectTo=302, https://nueva-url.com   # redirige con código HTTP dado
```

### CircuitBreaker — con fallback en YAML

En un Gateway que sirve múltiples servicios, un microservicio lento es especialmente peligroso. Cada petición a ese servicio ocupa una conexión del pool del Gateway mientras espera respuesta. Si el servicio tarda 30 segundos en responder y llegan 50 peticiones por segundo, el Gateway acumula 1500 peticiones pendientes en pocos minutos, agotando el pool y haciendo que el Gateway deje de responder también para los servicios que funcionan correctamente — un fallo local se convierte en un fallo global. El Circuit Breaker corta este ciclo: detecta que el servicio está fallando y devuelve el fallback inmediatamente, sin consumir recursos del pool ni esperar.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: pedidosCircuitBreaker        # nombre del circuit breaker (referencia a la config de Resilience4j)
      fallbackUri: forward:/fallback/pedidos   # endpoint del Gateway al que redirigir si el CB está abierto
      # statusCodes: 500,502,503         # códigos HTTP que cuentan como fallo (además de excepciones)
```

```yaml
# Configuración del Circuit Breaker en el application.yml del Gateway:
resilience4j:
  circuitbreaker:
    instances:
      pedidosCircuitBreaker:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
```

> La guía completa de Circuit Breaker (estados, métricas, Bulkhead, TimeLimiter) está en [07-circuit-breaker.md](./07-circuit-breaker.md).

### TokenRelay — propagar token OAuth2 al microservicio

Cuando el Gateway usa **Spring Security OAuth2**, este filtro propaga automáticamente el Bearer token del cliente entrante a todos los microservicios downstream, sin necesidad de extraerlo y añadirlo manualmente:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-security</artifactId>
</dependency>
```

```yaml
# En la ruta que necesita propagar el token:
filters:
  - TokenRelay=
  # Lee el token del SecurityContext (Spring Security) y lo añade como
  # Authorization: Bearer <token> en la petición al microservicio
```

```java
// Equivalente en Java DSL:
.filters(f -> f.tokenRelay())
```

```yaml
# Configuración completa para usar TokenRelay con OAuth2 Resource Server:
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server.miempresa.com
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - TokenRelay=      # propaga el token al microservicio
            - StripPrefix=1
```

> `TokenRelay` requiere Spring Security configurado en el Gateway. A diferencia del `JwtAuthFilter` manual (que usa JJWT directamente), `TokenRelay` delega la validación a Spring Security y solo se ocupa de la propagación. Elegir uno u otro según si ya se usa Spring Security o no.

### CacheRequestBody — leer el body más de una vez

En programación reactiva, el body de una petición es un stream que solo se puede leer una vez. Si dos filtros necesitan leer el body (p.ej. un filtro de firma HMAC y un filtro de logging), el segundo obtiene un stream vacío:

```yaml
filters:
  - name: CacheRequestBody
    args:
      bodyClass: java.lang.String   # tipo al que deserializar el body almacenado en caché
  # A partir de aquí, todos los filtros pueden leer el body desde:
  # exchange.getAttribute(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR)
```

```java
// Leer el body cacheado en un filtro posterior:
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    String body = exchange.getAttribute(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
    log.info("Body de la petición: {}", body);
    return chain.filter(exchange);
}
```

> `CacheRequestBody` debe ser el **primer filtro** de la lista para que los siguientes filtros ya encuentren el body almacenado. Si se pone después de un filtro que ya intentó leer el body, ese filtro ya habrá consumido el stream.

### SaveSession — guardar sesión antes de redirigir

En arquitecturas reactivas, la sesión HTTP se guarda de forma lazy (solo cuando hay un flush explícito o cuando el ciclo de respuesta termina). Si el Gateway reenvía la petición a un microservicio que necesita leer los atributos de sesión (como el estado OAuth2 guardado durante el flujo de autorización), puede leer una sesión vacía porque el Gateway aún no la ha persistido. `SaveSession` fuerza la escritura de la sesión antes de enrutar, garantizando que el servicio downstream siempre lee el estado completo.

```yaml
filters:
  - SaveSession
  # Fuerza que Spring Session persista la sesión antes de reenviar la petición
  # Necesario cuando se usa Spring Session + Spring Security y el Gateway reenvía a servicios
  # que leen la sesión (por ejemplo, servicios OAuth2 con estado de sesión)
```

---

## 6.6 Filtros personalizados (GatewayFilter y GlobalFilter)

Los filtros predefinidos cubren la mayoría de casos de uso. Crear un filtro personalizado tiene sentido cuando la lógica es específica del dominio o no existe un filtro predefinido equivalente: verificar una firma HMAC en el body de una petición, transformar el formato de una respuesta XML de un sistema legacy a JSON, implementar un algoritmo de throttling basado en el plan de suscripción del usuario, o añadir headers de trazabilidad que requieren lógica de negocio para calcularse.

Hay dos tipos de filtros personalizados:

| Tipo | Alcance | Implementa |
|---|---|---|
| `GatewayFilter` | Solo las rutas donde se registra explícitamente | `GatewayFilter` + `GatewayFilterFactory` |
| `GlobalFilter` | **Todas** las rutas automáticamente | `GlobalFilter` |

---

### GatewayFilter — aplicado a rutas específicas

Para crear un filtro que se pueda usar en YAML, hay que implementar `AbstractGatewayFilterFactory`:

```java
@Component
public class LoggingGatewayFilterFactory
        extends AbstractGatewayFilterFactory<LoggingGatewayFilterFactory.Config> {

    private static final Logger log = LoggerFactory.getLogger(LoggingGatewayFilterFactory.class);

    public LoggingGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            log.info("[{}] {} {}", config.getPrefix(),
                request.getMethod(), request.getURI());

            // Pre-filter: antes de enrutar al microservicio
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                // Post-filter: después de que el microservicio respondió
                ServerHttpResponse response = exchange.getResponse();
                log.info("[{}] Respuesta: {}", config.getPrefix(),
                    response.getStatusCode());
            }));
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("prefix");  // orden de parámetros en la forma compacta de YAML
    }

    public static class Config {
        private String prefix = "GATEWAY";
        // getter y setter
        public String getPrefix() { return prefix; }
        public void setPrefix(String prefix) { this.prefix = prefix; }
    }
}
```

```yaml
# Uso en YAML — la factory se llama "Logging" (nombre de la clase sin "GatewayFilterFactory")
filters:
  - Logging=MI-SERVICIO
```

---

### GlobalFilter — aplicado a TODAS las rutas

La conveniencia del `GlobalFilter` tiene un coste: cualquier error o bug afecta a todas las rutas sin excepción, incluidas las de health check y métricas. Antes de implementar un `GlobalFilter`, la pregunta que hay que responder es: ¿esta lógica aplica realmente a _todas_ las rutas, incluidas las públicas como `/auth/login` o `/actuator/health`? Si hay excepciones, hay que añadir la lógica de exclusión explícitamente en el propio filtro (como se muestra en la sección siguiente).

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Verificar que tiene el header de autorización
        if (!request.getHeaders().containsKey("Authorization")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Token válido — continúa la cadena de filtros
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;  // menor número = mayor prioridad de ejecución
    }
}
```

---

### Orden de ejecución de filtros

```
Petición entrante
      ↓
GlobalFilter (orden -200)      ← JwtAuthFilter
GlobalFilter (orden -100)      ← AuthGlobalFilter
GatewayFilter de la ruta       ← filtros declarados en la ruta
      ↓
Microservicio destino
      ↑
GatewayFilter de la ruta (post)
GlobalFilter (post, orden inverso)
      ↑
Respuesta al cliente
```

> Todos los `GlobalFilter` con `getOrder()` más negativo se ejecutan primero (pre) y últimos en la vuelta (post). El orden es importante cuando un filtro depende de que otro ya haya procesado la petición (por ejemplo, el filtro de logging necesita que el JWT ya esté validado para poder loguear el usuario).

---

### Desactivar un GlobalFilter para una ruta específica

Los `GlobalFilter` se aplican a todas las rutas. Para excluir una ruta concreta (por ejemplo, rutas públicas como `/health` o `/login`):

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    // Rutas que no requieren autenticación
    private static final List<String> PUBLIC_PATHS = List.of(
        "/actuator/health", "/auth/login", "/auth/register"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);  // ruta pública, no verificar token
        }

        // ... verificación del token
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() { return -100; }
}
```

---

← [Rutas y Predicates](./06-02-gateway-rutas.md) | [Volver al índice](./README.md) | Siguiente: [Avanzado →](./06-04-gateway-avanzado.md)
