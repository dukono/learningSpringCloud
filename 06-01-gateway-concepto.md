# Parte 6.1 — API Gateway: Concepto y Arquitectura Reactiva

← [Parte 5 — Load Balancing](./05-load-balancing.md) | [Volver al índice](./README.md) | Siguiente: [Rutas →](./06-02-gateway-rutas.md)

---

## 6.1 Qué es un API Gateway y sus responsabilidades

Un **API Gateway** es el **punto de entrada único** para todos los clientes externos (browsers, apps móviles, servicios de terceros) a la arquitectura de microservicios.

Sin Gateway, los clientes tendrían que conocer la URL interna de cada microservicio:
```
# SIN GATEWAY — el cliente conoce todo
app móvil → http://10.0.0.1:8081/usuarios/...
app móvil → http://10.0.0.2:8082/productos/...
app móvil → http://10.0.0.3:8083/pedidos/...
```

Con Gateway, hay una única fachada:
```
# CON GATEWAY — el cliente solo conoce una URL
app móvil → https://api.miempresa.com/usuarios/...
app móvil → https://api.miempresa.com/productos/...
app móvil → https://api.miempresa.com/pedidos/...
```

### Responsabilidades del Gateway

| Responsabilidad | Descripción |
|---|---|
| **Routing** | Enruta peticiones al microservicio correcto según la ruta |
| **Autenticación** | Valida tokens JWT antes de dejar pasar la petición |
| **Rate Limiting** | Limita el número de peticiones por cliente/IP |
| **SSL Termination** | Gestiona HTTPS; los servicios internos usan HTTP |
| **CORS** | Centraliza la gestión de Cross-Origin |
| **Transformación** | Modifica headers, rutas, body de peticiones/respuestas |
| **Logging** | Registra todas las peticiones de entrada |
| **Circuit Breaker** | Protege los servicios de sobrecarga desde el Gateway |
| **Load Balancing** | Distribuye carga entre instancias del mismo servicio |

---

## 6.2 Spring Cloud Gateway (arquitectura reactiva)

Spring Cloud Gateway está construido sobre **Spring WebFlux** (reactivo, no-bloqueante), lo que le permite manejar gran concurrencia con pocos hilos.

### Ciclo completo de una petición

```
Cliente HTTP/HTTPS
        │
        ▼
┌──────────────────────────────────────────────┐
│            Spring Cloud Gateway              │
│                                              │
│  ① Evaluación de Predicates                  │
│     ¿Path coincide? ¿Method correcto?        │
│     ¿Header presente? ¿IP permitida?...      │
│     → Si ninguna ruta coincide: 404          │
│                                              │
│  ② GlobalFilters (pre, orden ascendente)     │
│     JwtAuthFilter (orden -200)               │
│     LoggingGlobalFilter (orden -100)         │
│     → 401/403 si falla la autenticación      │
│                                              │
│  ③ GatewayFilters de la ruta (pre)           │
│     RateLimiter → 429 si excede el límite    │
│     CircuitBreaker → fallback si CB abierto  │
│     StripPrefix, RewritePath, AddHeader...   │
└───────────────────────┬──────────────────────┘
                        │ petición transformada
                        ▼
           Microservicio destino
           lb://pedidos-service
           (selección de instancia por LB)
                        │ respuesta
                        ▼
┌──────────────────────────────────────────────┐
│            Spring Cloud Gateway              │
│                                              │
│  ④ GatewayFilters de la ruta (post)          │
│     AddResponseHeader, SetStatus...          │
│                                              │
│  ⑤ GlobalFilters (post, orden descendente)   │
│     LoggingGlobalFilter (loguea respuesta)   │
└───────────────────────┬──────────────────────┘
                        │ respuesta transformada
                        ▼
               Cliente (respuesta final)
```

> Los pasos ② y ③ son **pre-filters** (se ejecutan antes de llamar al microservicio). Los pasos ④ y ⑤ son **post-filters** (se ejecutan después de recibir la respuesta). Un filtro puede actuar en ambas fases en el mismo bean (ver `06-03-gateway-filtros.md`).

### Diferencia con Zuul

| Aspecto | Zuul 1.x (antiguo) | Spring Cloud Gateway |
|---|---|---|
| **Modelo** | Bloqueante (Servlet) | No-bloqueante (Reactor/WebFlux) |
| **Rendimiento** | Limitado por hilos | Alto, maneja miles de conexiones concurrentes |
| **Estado** | Deprecado | Activo, mantenido |
| **Filtros** | ZuulFilter | GatewayFilter / GlobalFilter |

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- IMPORTANTE: NO incluir spring-boot-starter-web junto con gateway -->
<!-- Gateway usa WebFlux; son incompatibles entre sí -->
```

### Clase principal

```java
@SpringBootApplication
public class GatewayApplication {
    // No necesita ninguna anotación especial
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

---

### Propiedades globales del Gateway

Conforme el Gateway crece y gestiona decenas de rutas, hay tres problemas que aparecen repetidamente si no se configura nada a nivel global: CORS se declara en cada ruta por separado y acaban siendo incoherentes; el cliente HTTP reactivo (Reactor Netty) usa sus valores por defecto de pool y timeouts, que son inadecuados para producción; y hay filtros que deberían aplicarse a todas las rutas (como `DedupeResponseHeader` para CORS) pero se olvidan en alguna. Las propiedades globales del Gateway resuelven estos tres problemas en un único bloque de configuración.

```yaml
spring:
  cloud:
    gateway:
      # Filtros aplicados a TODAS las rutas (sin necesidad de declararlos en cada ruta)
      default-filters:
        - AddResponseHeader=X-Gateway, spring-cloud-gateway
        - DedupeResponseHeader=Access-Control-Allow-Origin

      # Configuración del cliente HTTP reactivo (Reactor Netty)
      httpclient:
        connect-timeout: 2000          # ms para establecer conexión TCP con el microservicio
        response-timeout: 10s          # tiempo máximo para recibir la respuesta completa
        pool:
          max-connections: 500         # máximo de conexiones en el pool
          acquire-timeout: 5000        # ms esperando una conexión libre del pool

      # CORS global para todas las rutas
      globalcors:
        cors-configurations:
          '[/**]':                              # aplica a todas las rutas
            allowed-origins: "https://miapp.com"
            allowed-methods: GET,POST,PUT,DELETE
            allowed-headers: "*"
            allow-credentials: true
            max-age: 3600

      # Endpoint de Actuator para inspeccionar rutas en runtime
      # GET /actuator/gateway/routes — lista todas las rutas configuradas
      # GET /actuator/gateway/routes/{id} — detalle de una ruta concreta
      # POST /actuator/gateway/refresh — recarga rutas dinámicas sin reiniciar

      # Métricas del Gateway para Micrometer/Prometheus
      metrics:
        enabled: true   # defecto: false; expone métricas por ruta (latencia, errores, etc.)
                        # añade tags: routeId, routeUri, outcome (SUCCESSFUL/CLIENT_ERROR/SERVER_ERROR)
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, gateway, prometheus
```

---

### Auto-registro de rutas desde el Service Discovery (Discovery Locator)

En lugar de declarar cada ruta manualmente, se puede activar la creación automática de rutas a partir del registro de servicios (Eureka, Consul):

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true           # crea automáticamente una ruta por cada servicio registrado en Eureka
                                  # defecto: false
          lower-case-service-id: true   # convierte el nombre del servicio a minúsculas en la URL
                                        # sin esto: /PEDIDOS-SERVICE/** ; con esto: /pedidos-service/**
```

Con esto activo, si `pedidos-service` está en Eureka, el Gateway crea automáticamente:
```
GET /pedidos-service/**  →  lb://pedidos-service/**
```

> **Ventaja:** sin configuración de rutas en YAML para cada servicio nuevo.
> **Inconveniente:** las rutas generadas no tienen filtros personalizados ni control fino sobre predicates. Para producción, combinar: Discovery Locator para servicios nuevos + rutas explícitas para los que necesitan filtros específicos.

---

### SSL Termination — configurar HTTPS en el Gateway

El Gateway gestiona HTTPS con los clientes externos; los microservicios internos usan HTTP plano.

```xml
<!-- Necesario para generar el keystore si no tienes uno ya -->
<!-- keytool -genkeypair -alias gateway -keyalg RSA -keysize 2048
     -storetype PKCS12 -keystore gateway-keystore.p12 -validity 365 -->
```

```yaml
server:
  port: 443                              # puerto HTTPS estándar
  ssl:
    enabled: true
    key-store: classpath:gateway-keystore.p12   # ruta al keystore (classpath o file:)
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12               # PKCS12 (recomendado) o JKS (legacy)
    key-alias: gateway                   # alias del certificado dentro del keystore
    # Para TLS mutuo (mTLS) — el cliente también presenta certificado:
    # client-auth: need                  # need = obligatorio, want = opcional
    # trust-store: classpath:trusted-clients.p12
    # trust-store-password: ${TRUST_STORE_PASSWORD}
```

```yaml
# Los microservicios internos siguen usando HTTP
# El Gateway termina el SSL y redirige internamente:
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: http://pedidos-service:8080   # HTTP interno, no HTTPS
          predicates:
            - Path=/api/pedidos/**
```

> Los certificados autofirmados son válidos para desarrollo pero los browsers los rechazan. En producción usar Let's Encrypt o un certificado firmado por una CA corporativa.

---

### WebSocket proxying

Spring Cloud Gateway soporta proxying de WebSocket de forma nativa. Solo hay que usar `ws://` o `wss://` como esquema de la URI destino:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ws-chat
          uri: ws://chat-service:8080      # ws:// para WebSocket sin TLS
          predicates:
            - Path=/ws/chat/**

        - id: wss-notificaciones
          uri: wss://notif-service:8443    # wss:// para WebSocket con TLS
          predicates:
            - Path=/ws/notificaciones/**
```

El Gateway eleva automáticamente la conexión HTTP a WebSocket cuando el cliente envía el header `Upgrade: websocket`. No requiere configuración adicional.

> Con Zuul 1.x era necesario un módulo separado y el soporte era limitado. Con Spring Cloud Gateway, WebSocket funciona igual que cualquier otra ruta.

---

> **[ADVERTENCIA]** Si se incluye `spring-boot-starter-web` junto con `spring-cloud-starter-gateway`, la aplicación fallará al arrancar. Gateway usa WebFlux y es incompatible con el stack Servlet (MVC/Tomcat). Elegir uno de los dos.

---

← [Parte 5 — Load Balancing](./05-load-balancing.md) | [Volver al índice](./README.md) | Siguiente: [Rutas →](./06-02-gateway-rutas.md)
