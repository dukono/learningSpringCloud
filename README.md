# Spring Cloud — Documentación Completa Profesional

> Documentación exhaustiva desde cero hasta nivel experto.
> Apta para pruebas técnicas empresariales y certificaciones.
> Cubre **Spring Boot 3.x / Spring Cloud 2023.x** + **Java 17/21**

---

## ÍNDICE GENERAL

---

### BLOQUE 1 — FUNDAMENTOS Y ARQUITECTURA
→ [01-fundamentos.md](./01-fundamentos.md) | [02-arquitectura.md](./02-arquitectura.md)

**01-fundamentos.md**
- 1.1 ¿Qué es Spring Cloud?
- 1.2 Historia y evolución
- 1.3 Spring Cloud vs Spring Boot vs Spring Framework
- 1.4 El problema que resuelve: arquitectura monolítica vs microservicios
- 1.5 Los 8 problemas clásicos de los microservicios
- 1.6 Cómo Spring Cloud resuelve esos problemas
- 1.7 Versiones, BOM y compatibilidad con Spring Boot
- 1.8 Las 8 falacias de la computación distribuida
- 1.9 CAP Theorem
- 1.10 Spring Cloud vs soluciones nativas de Kubernetes

**02-arquitectura.md**
- 2.1 Visión general de la arquitectura
- 2.2 Patrones de diseño en microservicios que implementa Spring Cloud
- 2.3 Mapa de componentes y sus relaciones
- 2.4 Flujo de una petición en un ecosistema Spring Cloud

---

### 3. Spring Cloud Config

- [3.1 Concepto y Arquitectura](./03-01-config-concepto.md)
  - 3.1.1 Qué es y para qué sirve
  - 3.1.2 Arquitectura: Config Server y Config Client
    - 3.1.2.1 El concepto de Config Client
    - 3.1.2.2 Orden de arranque (Docker Compose / Kubernetes)
  - 3.1.3 Regla de fusión de configuración

- [3.2 Backends de Configuración](./03-02-config-backends.md)
  - 3.2.1 Git (HTTPS y SSH)
  - 3.2.2 Filesystem (desarrollo local)
  - 3.2.3 HashiCorp Vault
  - 3.2.4 JDBC
  - 3.2.5 AWS S3 / GCS
  - 3.2.6 AWS SSM Parameter Store
  - 3.2.7 Azure App Configuration
  - 3.2.8 Composite (combinar backends)
  - 3.2.9 Múltiples repositorios Git por patrón
  - 3.2.10 Semántica del parámetro label
  - [3.2.11 Extensión Programática: EnvironmentRepository](./03-02-extension-programatica.md)
    - 3.2.11.1 EnvironmentRepository — interfaz central
    - 3.2.11.2 JGitEnvironmentRepository como bean
    - 3.2.11.3 Multi-tenant repository locator
    - 3.2.11.4 CompositeEnvironmentRepository programático
    - 3.2.11.5 Backend completamente custom
    - 3.2.11.6 Antipatrones

- [3.3 Config Server](./03-03-config-server.md)
  - 3.3.1 Directorio de caché local (basedir)
  - 3.3.2 Seguridad: autenticación básica
  - 3.3.3 Actuator del Config Server
  - 3.3.4 server.overrides
  - 3.3.5 Endpoint de recursos (/{app}/{profile}/{label}/{path})
  - 3.3.6 Health check y health.repositories
  - 3.3.7 Endpoint /monitor — push notifications
  - [3.3.8 Extensión Programática: EnvironmentEncryptor, EnvironmentController](./03-03-extension-programatica.md)
    - 3.3.8.1 EnvironmentEncryptor custom (Vault Transit, HSM)
    - 3.3.8.2 EnvironmentController con @Primary
    - 3.3.8.3 @ControllerAdvice para filtrar propiedades
    - 3.3.8.4 EnvironmentPostProcessor
    - 3.3.8.5 Antipatrones

- [3.4 Config Client](./03-04-config-client.md)
  - 3.4.1 Setup: dependencia y spring.config.import
  - 3.4.2 Opciones: optional, disabled
  - 3.4.3 Múltiples fuentes y prioridad
  - 3.4.4 Timeouts de conexión
  - 3.4.5 Precedencia: remoto vs local
  - 3.4.6 server.overrides en el cliente
  - 3.4.7 Autenticación al Config Server
  - 3.4.8 spring.cloud.config.watch
  - 3.4.9 Debug logging
  - 3.4.10 Configuración de reintentos
  - 3.4.11 Bootstrap context [LEGACY]
  - 3.4.12 Perfiles y entornos
  - 3.4.13 @ConfigurationProperties y refresco
  - 3.4.14 Testing del Config Client
  - [3.4.15 Extensión Programática: PropertySourceLocator](./03-04-extension-programatica.md)
    - 3.4.15.1 Ciclo de vida Bootstrap con PropertySourceLocator
    - 3.4.15.2 PropertySourceLocator completamente custom
    - 3.4.15.3 ConfigServicePropertySourceLocator extendido
    - 3.4.15.4 ConfigClientProperties programático
    - 3.4.15.5 BootstrapConfiguration
    - 3.4.15.6 Antipatrones

- [3.5 Refresco de Configuración](./03-05-config-refresh.md)
  - 3.5.1 Refresco manual con @RefreshScope
  - 3.5.2 Spring Cloud Bus (refresco masivo)
  - 3.5.3 spring.cloud.bus.destination
  - 3.5.4 Tipos de eventos del Bus
  - 3.5.5 Refresco automático por Webhook
  - 3.5.6 Refresco selectivo con /monitor
  - 3.5.7 Comparativa de opciones de refresco
  - 3.5.8 Testing del mecanismo de refresco
  - [3.5.9 Extensión Programática: ContextRefresher, RefreshScope](./03-05-extension-programatica.md)
    - 3.5.9.1 Qué ocurre en ContextRefresher.refresh()
    - 3.5.9.2 ContextRefresher — disparar el refresco sin HTTP
    - 3.5.9.3 RefreshScope — refrescar beans individuales
    - 3.5.9.4 Publicar RefreshRemoteApplicationEvent sin HTTP
    - 3.5.9.5 Listeners de eventos de refresco
    - 3.5.9.6 Antipatrones

- [3.6 Cifrado y Alta Disponibilidad](./03-06-config-avanzado.md)
  - 3.6.1 Clave simétrica (AES)
  - 3.6.2 Clave asimétrica RSA
  - 3.6.3 Vault Transit como backend de cifrado
  - 3.6.4 Verificar cifrado: endpoint /decrypt
  - 3.6.5 Rotación de la clave de cifrado
  - 3.6.6 Descifrado en el cliente
  - 3.6.7 Alta disponibilidad del Config Server
  - 3.6.8 Testing con Spring Cloud Config
  - [3.6.9 Extensión Programática: TextEncryptor, failover](./03-06-extension-programatica.md)
    - 3.6.9.1 TextEncryptor custom
    - 3.6.9.2 TextEncryptor con Vault Transit
    - 3.6.9.3 TextEncryptor multiversión (rotación sin ventana)
    - 3.6.9.4 Alta disponibilidad programática — failover
    - 3.6.9.5 Antipatrones

---

### BLOQUE 3 — DESCUBRIMIENTO DE SERVICIOS (SERVICE DISCOVERY)
→ [04-service-discovery.md](./04-service-discovery.md)

- 4.1 Qué es el Service Discovery y por qué es necesario
- 4.2 Modelos: Client-Side vs Server-Side Discovery
- 4.3 Spring Cloud Netflix Eureka — Servidor (`@EnableEurekaServer`)
- 4.4 Spring Cloud Netflix Eureka — Cliente (`@EnableDiscoveryClient`)
- 4.5 Heartbeat, lease y evicción de instancias
- 4.6 Self-preservation mode
- 4.7 Eureka en alta disponibilidad (clúster)
- 4.8 Alternativas a Eureka: Consul y Zookeeper

---

### BLOQUE 4 — BALANCEO DE CARGA (LOAD BALANCING)
→ [05-load-balancing.md](./05-load-balancing.md)

- 5.1 Qué es el balanceo de carga en microservicios
- 5.2 Spring Cloud LoadBalancer (sucesor de Ribbon)
- 5.3 Algoritmos: Round Robin y Random
- 5.4 Configuración y personalización del balanceador
- 5.5 Integración con Eureka y OpenFeign

---

### BLOQUE 5 — COMUNICACIÓN ENTRE SERVICIOS
→ [08-comunicacion.md](./08-comunicacion.md)

- 8.1 Comunicación síncrona vs asíncrona
- 8.2 Spring Cloud OpenFeign — cliente HTTP declarativo
- 8.3 Configuración de Feign: timeouts, interceptors, logging
- 8.4 Feign + LoadBalancer + Circuit Breaker (integración completa)
- 8.5 Spring Cloud Stream — mensajería con Kafka y RabbitMQ
- 8.6 Binders, Bindings y canales en Spring Cloud Stream
- 8.7 Spring Cloud Bus — propagación de eventos

---

### BLOQUE 6 — RESILIENCIA Y TOLERANCIA A FALLOS
→ [07-circuit-breaker.md](./07-circuit-breaker.md)

- 7.1 El problema: fallos en cascada
- 7.2 Patrón Circuit Breaker explicado
- 7.3 Estados: CLOSED, OPEN, HALF-OPEN
- 7.4 Resilience4j en Spring Cloud
- 7.5 Configuración de CircuitBreaker con Resilience4j
- 7.6 Fallback: respuestas de emergencia
- 7.7 Retry, TimeLimiter y Bulkhead
- 7.8 Integración con Spring Cloud Gateway
- 7.9 Monitoreo con Actuator y métricas

---

### 6. API Gateway

- 6.1 Concepto y Arquitectura (agrupa 7 hojas — concepto, ciclo reactivo, comparación, configuración base, protocolos y servicios de infraestructura son sub-dominios distintos del setup inicial)
  - [6.1.1 Qué es un API Gateway: responsabilidades y casos de uso](./06-01-gateway-concepto.md)
  - [6.1.2 Arquitectura reactiva: ciclo completo de una petición](./06-02-gateway-arquitectura.md)
  - [6.1.3 Spring Cloud Gateway vs Zuul](./06-03-gateway-vs-zuul.md)
  - [6.1.4 Propiedades globales: `default-filters`, `httpclient`, `globalcors`, `metrics`](./06-04-gateway-propiedades-globales.md)
  - [6.1.5 SSL Termination: HTTPS, PKCS12 y mTLS](./06-05-gateway-ssl.md)
  - [6.1.6 WebSocket proxying: `ws://` y `wss://`](./06-06-gateway-websocket.md)
  - [6.1.7 Discovery Locator y Actuator](./06-07-gateway-discovery-actuator.md)

- 6.2 Rutas: Predicates y Configuración (agrupa 5 hojas — modelo de datos, catálogo de predicates, dos estilos de configuración y casos especiales de ruta son sub-dominios distintos)
  - [6.2.1 Modelo de Route: id, uri, order, predicates, filters y orden de evaluación](./06-08-gateway-route-modelo.md)
  - [6.2.2 Catálogo de Predicates: Path, Method, Header, Query, Host, After/Before/Between, Cookie, RemoteAddr, XForwardedRemoteAddr](./06-09-gateway-predicates.md)
  - [6.2.3 Configuración en YAML: ejemplos completos con casos reales](./06-10-gateway-rutas-yaml.md)
  - [6.2.4 Configuración en Java DSL: lógica OR y casos complejos](./06-11-gateway-rutas-dsl.md)
  - [6.2.5 Canary deployment con `Weight` y rutas `forward:`](./06-12-gateway-canary-forward.md)

- 6.3 Filtros Predefinidos (agrupa 5 hojas — cada grupo opera sobre una parte distinta de la petición/respuesta: path, headers de entrada, headers de salida, flujo y protección)
  - [6.3.1 Transformación de path: StripPrefix, PrefixPath, RewritePath, SetPath](./06-13-gateway-filtros-path.md)
  - [6.3.2 Headers de petición: AddRequestHeader, SetRequestHeader, AddRequestHeadersIfNotPresent, MapRequestHeader, RemoveRequestHeader](./06-14-gateway-filtros-headers-request.md)
  - [6.3.3 Headers de respuesta: AddResponseHeader, SetResponseHeader, RemoveResponseHeader, RewriteResponseHeader, RewriteLocationResponseHeader, DedupeResponseHeader](./06-15-gateway-filtros-headers-response.md)
  - [6.3.4 Control de flujo: CircuitBreaker, Retry, RedirectTo](./06-16-gateway-filtros-flujo.md)
  - [6.3.5 Protección y utilidades: RequestSize, SecureHeaders, CacheRequestBody, SaveSession](./06-17-gateway-filtros-proteccion.md)

- 6.4 Filtros Personalizados (agrupa 3 hojas — fábrica por ruta, filtro global y filtro con lógica de negocio son mecanismos de extensión con ciclos de vida distintos)
  - [6.4.1 `AbstractGatewayFilterFactory`: filtro por ruta con parámetros YAML](./06-18-gateway-filter-factory.md)
  - [6.4.2 `GlobalFilter` y `Ordered`: filtro global y exclusión de rutas públicas](./06-19-gateway-global-filter.md)
  - [6.4.3 Filtros con lógica de negocio: validación HMAC](./06-20-gateway-filter-negocio.md)

- 6.5 Rate Limiting (agrupa 3 hojas — algoritmo y config, estrategias de clave y variantes de despliegue son aspectos independientes del mismo mecanismo)
  - [6.5.1 Token Bucket, `RequestRateLimiter` y configuración con Redis](./06-21-gateway-rate-limiter-config.md)
  - [6.5.2 `KeyResolver`: estrategias por IP, usuario, API key y plan de suscripción](./06-22-gateway-key-resolver.md)
  - [6.5.3 Clúster con Redis compartido y variante in-memory para desarrollo](./06-23-gateway-rate-limiter-variantes.md)

- 6.6 Seguridad: JWT y OAuth2 (agrupa 4 hojas — flujo conceptual, implementación JWT manual, integración OAuth2 y relay de tokens son mecanismos de seguridad con responsabilidades distintas)
  - [6.6.1 Flujo de validación JWT centralizada y arquitectura de seguridad](./06-24-gateway-seguridad-flujo.md)
  - [6.6.2 `JwtAuthFilter` con JJWT y propagación de identidad: `X-User-Id`, `X-User-Roles`](./06-25-gateway-jwt-filter.md)
  - [6.6.3 Spring Security OAuth2 Resource Server con `issuer-uri` y JWKS](./06-26-gateway-oauth2.md)
  - [6.6.4 `TokenRelay`: propagación del Bearer token al microservicio](./06-27-gateway-token-relay.md)

- 6.7 Testing del Gateway (agrupa 5 hojas — estrategia general y cada estrategia cubre una capa de aislamiento distinta: rutas, filtros, rate limiting y circuit breaker)
  - [6.7.1 Estrategia de testing: pirámide y configuración `application-test.yml`](./06-28-gateway-testing-estrategia.md)
  - [6.7.2 Tests de rutas: `@WebFluxTest` + `RouteLocator`](./06-29-gateway-testing-rutas.md)
  - [6.7.3 Tests de filtros y seguridad: `@SpringBootTest` + WireMock](./06-30-gateway-testing-filtros.md)
  - [6.7.4 Tests de rate limiting: Testcontainers Redis](./06-31-gateway-testing-rate-limiting.md)
  - [6.7.5 Tests de Circuit Breaker con WireMock](./06-32-gateway-testing-circuit-breaker.md)

- 6.8 Extensión Avanzada: Gateway Programático (agrupa 5 hojas — cada mecanismo opera sobre una capa distinta del Gateway: localización de rutas, escritura, filtrado reactivo, migración y antipatrones)
  - [6.8.1 `RouteDefinitionLocator`: rutas dinámicas desde base de datos con caché reactivo](./06-33-gateway-route-locator.md)
  - [6.8.2 `RouteDefinitionWriter`: CRUD de rutas en runtime sin reiniciar](./06-34-gateway-route-writer.md)
  - [6.8.3 `GlobalFilter` con repositorio reactivo y lógica de negocio por petición](./06-35-gateway-filter-reactivo.md)
  - [6.8.4 Migración Zuul [LEGACY] → Gateway: equivalencias de extensión](./06-36-gateway-migracion-zuul.md)
  - [6.8.5 Antipatrones](./06-37-gateway-antipatrones.md)

---

### BLOQUE 8 — TRAZABILIDAD Y OBSERVABILIDAD
→ [09-observabilidad.md](./09-observabilidad.md)

- 9.1 El problema: rastrear peticiones entre microservicios
- 9.2 Conceptos: Trace ID, Span ID, correlación
- 9.3 Spring Cloud Sleuth (inyección automática de trazas) `[LEGACY]`
- 9.4 Micrometer Tracing (sucesor de Sleuth en Spring Boot 3)
- 9.5 Zipkin: servidor de trazas distribuidas
- 9.6 Integración con Grafana y Prometheus

---

### BLOQUE 9 — SEGURIDAD
→ [10-seguridad.md](./10-seguridad.md)

- 10.1 Seguridad en arquitecturas de microservicios
- 10.2 OAuth2 y JWT en Spring Cloud
- 10.3 Spring Authorization Server
- 10.4 Propagación de tokens entre servicios
- 10.5 Seguridad en el Config Server
- 10.6 mTLS y comunicación segura entre microservicios

---

### BLOQUE 10 — DESPLIEGUE Y ECOSISTEMA CLOUD
→ [11-kubernetes.md](./11-kubernetes.md)

- 11.1 Spring Cloud en entornos Kubernetes
- 11.2 Service Discovery nativo en Kubernetes
- 11.3 ConfigMaps y Secrets como fuente de configuración
- 11.4 Eureka vs Kubernetes Service Discovery

---

### BLOQUE 11 — PROYECTO PRÁCTICO COMPLETO
→ [12-proyecto-practico.md](./12-proyecto-practico.md)

- 12.1 Diseño del sistema de ejemplo (e-commerce con microservicios)
- 12.2 Config Server con repositorio Git
- 12.3 Eureka Server
- 12.4 Microservicio de productos
- 12.5 Microservicio de pedidos (con Feign + Circuit Breaker)
- 12.6 API Gateway con rutas y filtros
- 12.7 Trazabilidad con Zipkin
- 12.8 Docker Compose del ecosistema completo

---

### BLOQUE 13 — CONTRACT TESTING
→ [14-contract-testing.md](./14-contract-testing.md)

- 14.1 El problema: compatibilidad entre servicios
- 14.2 Conceptos: contrato, productor, consumidor, stub
- 14.3 Arquitectura de Spring Cloud Contract
- 14.4 Dependencias Maven (productor y consumidor)
- 14.5 Escribir contratos en Groovy DSL
- 14.6 Contratos en YAML
- 14.7 Clase base y tests generados en el productor
- 14.8 Stub Runner en el consumidor
- 14.9 Stub Runner con repositorio remoto
- 14.10 Flujo completo de trabajo en equipo
- 14.11 Valores dinámicos en contratos
- 14.12 Errores comunes

---

### BLOQUE 14 — TESTING DE MICROSERVICIOS
→ [15-testing.md](./15-testing.md)

- 15.1 Pirámide de tests adaptada a microservicios
- 15.2 Niveles: unitario, slice (@WebMvcTest, @DataJpaTest), integración
- 15.3 Testcontainers: PostgreSQL, Kafka, @ServiceConnection
- 15.4 WireMock: simular servicios HTTP externos
- 15.5 Testear un cliente Feign
- 15.6 Testear filtros del API Gateway
- 15.7 Testear Circuit Breaker con CircuitBreakerRegistry
- 15.8 Test de integración completo
- 15.9 Tabla resumen: qué herramienta usar en cada caso

---

### BLOQUE 15 — TRANSACCIONES DISTRIBUIDAS Y SAGA
→ [16-saga-transactions.md](./16-saga-transactions.md)

- 16.1 El problema: ACID en microservicios
- 16.2 Por qué 2PC no funciona en microservicios
- 16.3 Patrón Saga: transacciones compensatorias
- 16.4 Coreografía vs Orquestación
- 16.5 Implementación con Spring + Kafka (Coreografía)
- 16.6 Patrón Outbox: garantizar publicación de eventos
- 16.7 Idempotencia en consumidores
- 16.8 Tabla resumen de patrones

---

### BLOQUE 16 — MIGRACIÓN LEGACY → MODERNO
→ [17-migracion-legacy.md](./17-migracion-legacy.md)

- 17.1 Mapa de deprecaciones (Hystrix, Ribbon, Zuul, Sleuth)
- 17.2 Hystrix → Resilience4j (dependencias, anotaciones, configuración)
- 17.3 Ribbon → Spring Cloud LoadBalancer
- 17.4 Zuul → Spring Cloud Gateway
- 17.5 Spring Cloud Sleuth → Micrometer Tracing
- 17.6 bootstrap.yml → spring.config.import
- 17.7 Checklist completo de migración

---

### BLOQUE 17 — SERVICE MESH Y KUBERNETES AVANZADO
→ [18-service-mesh.md](./18-service-mesh.md)

- 18.1 El problema: lógica de red en código vs en infraestructura
- 18.2 Arquitectura de Istio: plano de control y plano de datos
- 18.3 Spring Cloud vs Istio: tabla comparativa completa
- 18.4 Cuándo usar Spring Cloud vs cuándo usar Istio
- 18.5 Spring Cloud en Kubernetes: ConfigMap, Secrets, RBAC
- 18.6 Circuit Breaker con Istio sin tocar el código
- 18.7 mTLS automático y AuthorizationPolicy
- 18.8 Observabilidad con Istio sin micrometer
- 18.9 Árbol de decisión: qué usar según tu contexto

---

### BLOQUE 12 — REFERENCIA RÁPIDA Y CERTIFICACIÓN
→ [13-referencia-rapida.md](./13-referencia-rapida.md)

- 13.1 Cheatsheet de anotaciones clave
- 13.2 Cheatsheet de propiedades de configuración
- 13.3 Tabla de deprecaciones y sus reemplazos
- 13.4 Preguntas frecuentes en entrevistas técnicas
- 13.5 Preguntas tipo certificación con respuestas razonadas
- 13.6 Errores comunes y cómo resolverlos
- 13.7 Checklist antes de ir a producción

---

## Estado de la documentación

| Archivo | Bloque | Estado |
|---------|--------|--------|
| [01-fundamentos.md](./01-fundamentos.md) | Bloque 1 | Completo |
| [02-arquitectura.md](./02-arquitectura.md) | Bloque 1 | Completo |
| [03-01-config-concepto.md](./03-01-config-concepto.md) | Bloque 2 | Completo |
| [03-02-config-backends.md](./03-02-config-backends.md) | Bloque 2 | Completo |
| [03-02-extension-programatica.md](./03-02-extension-programatica.md) | Bloque 2 | Completo |
| [03-03-config-server.md](./03-03-config-server.md) | Bloque 2 | Completo |
| [03-03-extension-programatica.md](./03-03-extension-programatica.md) | Bloque 2 | Completo |
| [03-04-config-client.md](./03-04-config-client.md) | Bloque 2 | Completo |
| [03-04-extension-programatica.md](./03-04-extension-programatica.md) | Bloque 2 | Completo |
| [03-05-config-refresh.md](./03-05-config-refresh.md) | Bloque 2 | Completo |
| [03-05-extension-programatica.md](./03-05-extension-programatica.md) | Bloque 2 | Completo |
| [03-06-config-avanzado.md](./03-06-config-avanzado.md) | Bloque 2 | Completo |
| [03-06-extension-programatica.md](./03-06-extension-programatica.md) | Bloque 2 | Completo |
| [04-service-discovery.md](./04-service-discovery.md) | Bloque 3 | Completo |
| [05-load-balancing.md](./05-load-balancing.md) | Bloque 4 | Completo |
| [08-comunicacion.md](./08-comunicacion.md) | Bloque 5 | Completo |
| [07-circuit-breaker.md](./07-circuit-breaker.md) | Bloque 6 | Completo |
| [06-01-gateway-concepto.md](./06-01-gateway-concepto.md) | Bloque 7 | Completo |
| [06-02-gateway-arquitectura.md](./06-02-gateway-arquitectura.md) | Bloque 7 | Completo |
| [06-03-gateway-vs-zuul.md](./06-03-gateway-vs-zuul.md) | Bloque 7 | Completo |
| [06-04-gateway-propiedades-globales.md](./06-04-gateway-propiedades-globales.md) | Bloque 7 | Completo |
| [06-05-gateway-ssl.md](./06-05-gateway-ssl.md) | Bloque 7 | Completo |
| [06-06-gateway-websocket.md](./06-06-gateway-websocket.md) | Bloque 7 | Completo |
| [06-07-gateway-discovery-actuator.md](./06-07-gateway-discovery-actuator.md) | Bloque 7 | Completo |
| [06-08-gateway-route-modelo.md](./06-08-gateway-route-modelo.md) | Bloque 7 | Completo |
| [06-09-gateway-predicates.md](./06-09-gateway-predicates.md) | Bloque 7 | Completo |
| [06-10-gateway-rutas-yaml.md](./06-10-gateway-rutas-yaml.md) | Bloque 7 | Completo |
| [06-11-gateway-rutas-dsl.md](./06-11-gateway-rutas-dsl.md) | Bloque 7 | Completo |
| [06-12-gateway-canary-forward.md](./06-12-gateway-canary-forward.md) | Bloque 7 | Completo |
| [06-13-gateway-filtros-path.md](./06-13-gateway-filtros-path.md) | Bloque 7 | Completo |
| [06-14-gateway-filtros-headers-request.md](./06-14-gateway-filtros-headers-request.md) | Bloque 7 | Completo |
| [06-15-gateway-filtros-headers-response.md](./06-15-gateway-filtros-headers-response.md) | Bloque 7 | Completo |
| [06-16-gateway-filtros-flujo.md](./06-16-gateway-filtros-flujo.md) | Bloque 7 | Completo |
| [06-17-gateway-filtros-proteccion.md](./06-17-gateway-filtros-proteccion.md) | Bloque 7 | Completo |
| [06-18-gateway-filter-factory.md](./06-18-gateway-filter-factory.md) | Bloque 7 | Completo |
| [06-19-gateway-global-filter.md](./06-19-gateway-global-filter.md) | Bloque 7 | Completo |
| [06-20-gateway-filter-negocio.md](./06-20-gateway-filter-negocio.md) | Bloque 7 | Completo |
| [06-21-gateway-rate-limiter-config.md](./06-21-gateway-rate-limiter-config.md) | Bloque 7 | Completo |
| [06-22-gateway-key-resolver.md](./06-22-gateway-key-resolver.md) | Bloque 7 | Completo |
| [06-23-gateway-rate-limiter-variantes.md](./06-23-gateway-rate-limiter-variantes.md) | Bloque 7 | Completo |
| [06-24-gateway-seguridad-flujo.md](./06-24-gateway-seguridad-flujo.md) | Bloque 7 | Completo |
| [06-25-gateway-jwt-filter.md](./06-25-gateway-jwt-filter.md) | Bloque 7 | Completo |
| [06-26-gateway-oauth2.md](./06-26-gateway-oauth2.md) | Bloque 7 | Completo |
| [06-27-gateway-token-relay.md](./06-27-gateway-token-relay.md) | Bloque 7 | Completo |
| [06-28-gateway-testing-estrategia.md](./06-28-gateway-testing-estrategia.md) | Bloque 7 | Completo |
| [06-29-gateway-testing-rutas.md](./06-29-gateway-testing-rutas.md) | Bloque 7 | Completo |
| [06-30-gateway-testing-filtros.md](./06-30-gateway-testing-filtros.md) | Bloque 7 | Completo |
| [06-31-gateway-testing-rate-limiting.md](./06-31-gateway-testing-rate-limiting.md) | Bloque 7 | Completo |
| [06-32-gateway-testing-circuit-breaker.md](./06-32-gateway-testing-circuit-breaker.md) | Bloque 7 | Completo |
| [06-33-gateway-route-locator.md](./06-33-gateway-route-locator.md) | Bloque 7 | Completo |
| [06-34-gateway-route-writer.md](./06-34-gateway-route-writer.md) | Bloque 7 | Completo |
| [06-35-gateway-filter-reactivo.md](./06-35-gateway-filter-reactivo.md) | Bloque 7 | Completo |
| [06-36-gateway-migracion-zuul.md](./06-36-gateway-migracion-zuul.md) | Bloque 7 | Completo |
| [06-37-gateway-antipatrones.md](./06-37-gateway-antipatrones.md) | Bloque 7 | Completo |
| [09-observabilidad.md](./09-observabilidad.md) | Bloque 8 | Completo |
| [10-seguridad.md](./10-seguridad.md) | Bloque 9 | Completo |
| [11-kubernetes.md](./11-kubernetes.md) | Bloque 10 | Completo |
| [12-proyecto-practico.md](./12-proyecto-practico.md) | Bloque 11 | Completo |
| [14-contract-testing.md](./14-contract-testing.md) | Bloque 13 | Completo |
| [15-testing.md](./15-testing.md) | Bloque 14 | Completo |
| [16-saga-transactions.md](./16-saga-transactions.md) | Bloque 15 | Completo |
| [17-migracion-legacy.md](./17-migracion-legacy.md) | Bloque 16 | Completo |
| [18-service-mesh.md](./18-service-mesh.md) | Bloque 17 | Completo |
| [13-referencia-rapida.md](./13-referencia-rapida.md) | Bloque 12 | Pendiente |

---

## Convenciones

| Símbolo | Significado |
|---------|-------------|
| `[CONCEPTO]` | Definición teórica importante |
| `[CÓDIGO]` | Fragmento de código práctico |
| `[ADVERTENCIA]` | Error común o trampa frecuente |
| `[EXAMEN]` | Punto frecuente en pruebas técnicas |
| `[LEGACY]` | Componente deprecado — conocer para exámenes, no usar en proyectos nuevos |
| `[PREREQUISITO]` | Infraestructura o dependencia externa requerida antes de usar el componente |

---

> Versión objetivo: **Spring Cloud 2023.0.x (Leyton)** + **Spring Boot 3.2/3.3** + **Java 17/21**
