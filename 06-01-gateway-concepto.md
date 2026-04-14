# 6.1 Concepto y Arquitectura Reactiva

← [5 — Load Balancing](./05-load-balancing.md) | [Índice](./README.md) | [6.2 Rutas y Predicates →](./06-02-gateway-rutas.md)

---

Sin Gateway, los clientes externos tienen que conocer la URL de cada microservicio: puerto, ruta base, versión de la API. Cualquier cambio en esa estructura (mover un servicio, cambiar un prefijo de ruta, escalar horizontalmente) rompe a los clientes. **Spring Cloud Gateway** es el punto de entrada único que oculta toda esa complejidad: los clientes solo conocen una URL pública y el Gateway se encarga del enrutamiento, seguridad, rate limiting, transformación de peticiones y observabilidad de forma centralizada.

> [PREREQUISITO] El proyecto Gateway debe excluir `spring-boot-starter-web`. El stack es WebFlux (reactivo); incluir ambos hace que la aplicación falle al arrancar.

---

## 6.1.1 Qué es un API Gateway y sus responsabilidades

```
Cliente HTTP/HTTPS
        │
        ▼
┌─────────────────────────────────────────────────────┐
│                Spring Cloud Gateway                  │
│                                                      │
│  ① Evaluación de Predicates                          │
│     Path, Method, Header, Query, Host, RemoteAddr…  │
│     → si ninguna ruta coincide: 404                  │
│                                                      │
│  ② GlobalFilters  (pre, orden ascendente)            │
│     JwtAuthFilter (orden -200)                       │
│     LoggingFilter (orden -100)                       │
│     → 401/403 si falla autenticación                 │
│                                                      │
│  ③ GatewayFilters de la ruta (pre)                   │
│     RateLimiter → 429 si excede límite               │
│     CircuitBreaker → fallback si CB abierto          │
│     StripPrefix, RewritePath, AddHeader…             │
└──────────────────────────┬──────────────────────────┘
                           │ petición transformada
                           ▼
              Microservicio destino
              lb://pedidos-service
              (instancia seleccionada por LB)
                           │ respuesta
                           ▼
┌─────────────────────────────────────────────────────┐
│                Spring Cloud Gateway                  │
│                                                      │
│  ④ GatewayFilters de la ruta (post)                  │
│     AddResponseHeader, SetStatus…                    │
│                                                      │
│  ⑤ GlobalFilters (post, orden descendente)           │
│     LoggingFilter (loguea respuesta + latencia)      │
└──────────────────────────┬──────────────────────────┘
                           │ respuesta transformada
                           ▼
                  Cliente (respuesta final)
```

> [CONCEPTO] Los pasos ② y ③ son **pre-filters** (antes del microservicio). Los pasos ④ y ⑤ son **post-filters** (tras recibir la respuesta). Un mismo bean puede actuar en ambas fases usando `.then(Mono.fromRunnable(...))`.

---

## 6.1.2 Arquitectura reactiva: ciclo completo de una petición

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!--
  NO incluir spring-boot-starter-web junto con gateway.
  Gateway usa WebFlux; los dos stacks son incompatibles.
-->
```

```java
// No requiere ninguna anotación especial en la clase principal
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

---

## 6.1.3 Spring Cloud Gateway vs Zuul

| Aspecto | Zuul 1.x | Spring Cloud Gateway |
|---|---|---|
| **Modelo de E/S** | Bloqueante — un hilo por conexión | No-bloqueante — Reactor/WebFlux |
| **Framework base** | Spring MVC (Servlet) | Spring WebFlux |
| **Estado oficial** | Deprecado | Activo y mantenido |
| **Rendimiento bajo carga** | Limitado por pool de hilos | Excelente, miles de conexiones concurrentes |
| **Filtros** | `ZuulFilter` | `GatewayFilter` / `GlobalFilter` |
| **WebSocket** | Soporte limitado (módulo separado) | Soporte nativo |
| **Predicates** | No existe el concepto | Potentes y extensibles |

> [EXAMEN] Pregunta frecuente: *¿por qué no se puede usar `spring-boot-starter-web` en un Gateway?* Porque Gateway depende de `spring-webflux` y el starter web registra un `DispatcherServlet` (Servlet API), lo que genera un conflicto de contexto en el arranque con un mensaje de error poco descriptivo.

---

## 6.1.4 Propiedades globales

Las propiedades globales resuelven tres problemas recurrentes cuando el Gateway crece: CORS declarado en cada ruta de forma inconsistente, timeouts del cliente HTTP con valores por defecto inadecuados para producción, y filtros que deberían aplicarse a todas las rutas pero se olvidan en alguna.

```yaml
spring:
  cloud:
    gateway:

      # Filtros aplicados a TODAS las rutas sin necesidad de declararlos en cada una
      default-filters:
        - AddResponseHeader=X-Gateway, spring-cloud-gateway
        - DedupeResponseHeader=Access-Control-Allow-Origin

      # Cliente HTTP reactivo (Reactor Netty)
      httpclient:
        connect-timeout: 2000           # ms para establecer conexión TCP
        response-timeout: 10s           # tiempo máximo para recibir la respuesta completa
        pool:
          max-connections: 500          # máximo de conexiones en el pool
          acquire-timeout: 5000         # ms esperando una conexión libre del pool

      # CORS centralizado para todas las rutas
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "https://miapp.com"
            allowed-methods: GET,POST,PUT,DELETE
            allowed-headers: "*"
            allow-credentials: true
            max-age: 3600

      # Métricas por ruta para Micrometer/Prometheus
      metrics:
        enabled: true    # defecto: false
                         # añade tags: routeId, routeUri, outcome (SUCCESSFUL/CLIENT_ERROR/SERVER_ERROR)
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, gateway, prometheus
# Endpoints útiles:
# GET  /actuator/gateway/routes           → lista todas las rutas configuradas
# GET  /actuator/gateway/routes/{id}      → detalle de una ruta
# POST /actuator/gateway/refresh          → recarga rutas dinámicas sin reiniciar
```

### Parámetros del cliente HTTP

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `httpclient.connect-timeout` | int (ms) | 45000 | Tiempo máximo para establecer conexión TCP |
| `httpclient.response-timeout` | Duration | 30s | Tiempo máximo esperando respuesta completa |
| `httpclient.pool.max-connections` | int | 500 | Máximo de conexiones en el pool de Netty |
| `httpclient.pool.acquire-timeout` | long (ms) | 45000 | Tiempo esperando una conexión libre del pool |
| `metrics.enabled` | boolean | false | Activa métricas Micrometer por ruta |
| `discovery.locator.enabled` | boolean | false | Crea rutas automáticas desde el registro de servicios |
| `discovery.locator.lower-case-service-id` | boolean | false | Convierte el serviceId a minúsculas en la URL |

---

## 6.1.5 SSL Termination — HTTPS en el Gateway

El Gateway gestiona HTTPS con los clientes externos; los microservicios internos siguen usando HTTP plano.

```bash
# Generar keystore PKCS12 para desarrollo
keytool -genkeypair -alias gateway -keyalg RSA -keysize 2048 \
  -storetype PKCS12 -keystore gateway-keystore.p12 -validity 365
```

```yaml
server:
  port: 443
  ssl:
    enabled: true
    key-store: classpath:gateway-keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: gateway
    # Para mTLS (el cliente también presenta certificado):
    # client-auth: need
    # trust-store: classpath:trusted-clients.p12
    # trust-store-password: ${TRUST_STORE_PASSWORD}
```

```yaml
# Los microservicios internos siguen usando HTTP
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: http://pedidos-service:8080   # HTTP interno, no HTTPS
          predicates:
            - Path=/api/pedidos/**
```

> [ADVERTENCIA] Los certificados autofirmados funcionan en desarrollo pero los navegadores los rechazan. En producción usar Let's Encrypt (`certbot`) o un certificado firmado por la CA corporativa.

---

## 6.1.6 WebSocket proxying

Spring Cloud Gateway soporta proxying de WebSocket de forma nativa usando `ws://` o `wss://` como esquema:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ws-chat
          uri: ws://chat-service:8080
          predicates:
            - Path=/ws/chat/**

        - id: wss-notificaciones
          uri: wss://notif-service:8443
          predicates:
            - Path=/ws/notificaciones/**
```

El Gateway eleva automáticamente la conexión HTTP a WebSocket cuando el cliente envía `Upgrade: websocket`. No requiere configuración adicional ni dependencias extra.

---

## 6.1.7 Discovery Locator — rutas automáticas desde Eureka/Consul

En lugar de declarar cada ruta manualmente, el Discovery Locator crea rutas automáticas para cada servicio registrado:

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true                  # defecto: false
          lower-case-service-id: true    # /PEDIDOS-SERVICE/** → /pedidos-service/**
```

Con esto activo, si `pedidos-service` está en Eureka, el Gateway crea automáticamente:
```
GET /pedidos-service/**  →  lb://pedidos-service/**
```

**Ventaja:** sin configuración de rutas en YAML para cada servicio nuevo.
**Inconveniente:** las rutas generadas no tienen filtros personalizados. Para producción: Discovery Locator para servicios nuevos + rutas explícitas para los que necesitan filtros específicos.

---

## 6.1.8 Actuator — gestión del Gateway en runtime

El Gateway expone endpoints de Actuator específicos para inspeccionar y administrar rutas sin reiniciar el proceso:

| Endpoint | Método | Descripción |
|---|---|---|
| `/actuator/gateway/routes` | GET | Lista todas las rutas activas con sus predicates y filtros |
| `/actuator/gateway/routes/{id}` | GET | Detalle de una ruta concreta por su ID |
| `/actuator/gateway/refresh` | POST | Recarga las rutas dinámicas (p. ej. desde `RouteDefinitionLocator`) |
| `/actuator/gateway/globalfilters` | GET | Lista todos los `GlobalFilter` activos y su orden |
| `/actuator/gateway/routefilters` | GET | Lista todos los `GatewayFilterFactory` disponibles |

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, gateway, prometheus
  endpoint:
    gateway:
      enabled: true
```

```bash
# Inspeccionar todas las rutas en producción sin reiniciar
curl http://localhost:8080/actuator/gateway/routes | jq .

# Forzar recarga de rutas dinámicas (después de un cambio en BD o Config Server)
curl -X POST http://localhost:8080/actuator/gateway/refresh
```

> [ADVERTENCIA] El endpoint `POST /actuator/gateway/refresh` recarga las rutas definidas por `RouteDefinitionLocator` pero **no** las rutas declaradas en YAML. Las rutas YAML solo cambian con un reinicio del proceso (o a través de Spring Cloud Config Bus si se usa `@RefreshScope`).

---


← [5 — Load Balancing](./05-load-balancing.md) | [Índice](./README.md) | [6.2 Rutas y Predicates →](./06-02-gateway-rutas.md)
