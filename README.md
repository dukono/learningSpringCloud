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

### BLOQUE 2 — CONFIGURACIÓN CENTRALIZADA
→ [03-01-config-concepto.md](./03-01-config-concepto.md) | [03-02-config-backends.md](./03-02-config-backends.md) | [03-03-config-server.md](./03-03-config-server.md) | [03-04-config-client.md](./03-04-config-client.md) | [03-05-config-refresh.md](./03-05-config-refresh.md) | [03-06-config-avanzado.md](./03-06-config-avanzado.md)

> **Extensión programática:** [03-02-extension-programatica.md](./03-02-extension-programatica.md) | [03-03-extension-programatica.md](./03-03-extension-programatica.md) | [03-04-extension-programatica.md](./03-04-extension-programatica.md) | [03-05-extension-programatica.md](./03-05-extension-programatica.md) | [03-06-extension-programatica.md](./03-06-extension-programatica.md)

**03-02-extension-programatica.md**
- `EnvironmentRepository` — interfaz central para crear backends custom
- `JGitEnvironmentRepository` como `@Bean` con repos dinámicos desde BD
- `MultiTenantGitRepositoryLocator` — un repo por tenant con cache
- `CompositeEnvironmentRepository` con lógica condicional por perfil
- Backend completamente custom desde REST API interna

**03-03-extension-programatica.md**
- `EnvironmentEncryptor` — cifrado con Vault Transit o HSM (reemplaza AES/RSA)
- `EnvironmentController` con `@Primary` — auditoría de accesos a la API
- `@ControllerAdvice` para filtrar propiedades sensibles por rol
- `EnvironmentPostProcessor` — resolver credenciales del propio Config Server antes del arranque

**03-04-extension-programatica.md**
- Diagrama del ciclo de vida Bootstrap: cuándo y cómo se invoca `PropertySourceLocator`
- `PropertySourceLocator` — fuente de propiedades custom (API interna, BD, sistema legado)
- Extender `ConfigServicePropertySourceLocator` — headers de autenticación propietarios
- `ConfigClientProperties` — diagnóstico y branch switching programático
- `BootstrapConfiguration` — resolver secretos antes que cualquier `@Value`

**03-05-extension-programatica.md**
- Diagrama de secuencia: qué ocurre internamente en `ContextRefresher.refresh()`
- `ContextRefresher` — disparar el refresco sin llamada HTTP (job, evento de negocio)
- Diferencia entre `refresh()` y `refreshEnvironment()`
- `RefreshScope.refresh("beanName")` — refrescar un bean específico
- `ApplicationEventPublisher` — publicar `RefreshRemoteApplicationEvent` sin HTTP
- Escuchar `RefreshRemoteApplicationEvent`, `EnvironmentChangeEvent`, `ScopeRefreshedEvent`

**03-06-extension-programatica.md**
- `TextEncryptor` custom con Google Tink (AES256-GCM)
- `VaultTransitTextEncryptor` — cifrado delegado a Vault Transit sin clave local
- `TextEncryptor` multiversión — rotación sin ventana de mantenimiento
- `FailoverConfigServiceLocator` — failover con balanceo aleatorio entre instancias

**03-01-config-concepto.md**
- Qué es Spring Cloud Config y para qué sirve
- Arquitectura: Config Server y Config Client
- Concepto de Config Client para desarrolladores Spring Boot
- Regla de fusión de configuración con ejemplo concreto
- Boot order: `depends_on` + healthcheck en Docker Compose, initContainer en Kubernetes

**03-02-config-backends.md**
- Backends: Git (HTTPS + SSH), Filesystem, Vault, JDBC, S3/GCS, Composite
- Parámetros Git: `default-label`, `search-paths`, `clone-on-start`, `refresh-rate`, `force-pull`
- Autenticación SSH al repositorio
- Esquema SQL para backend JDBC

**03-03-config-server.md**
- Setup del Config Server (`@EnableConfigServer`)
- Diagrama del flujo de petición: cliente → Config Server → Git → respuesta fusionada
- API REST y patrón de URLs `/{application}/{profile}/{label}`
- Autenticación básica con Spring Security
- `POST /actuator/refresh` en el servidor (re-sincroniza caché con Git) vs en el cliente
- Actuator: `/actuator/health`, `/actuator/env`
- `server.overrides` — propiedades que ningún cliente puede sobreescribir (§3.4.4)
- Endpoint de recursos `/{app}/{profile}/{label}/{path}` — servir ficheros completos (§3.4.5)
- Health check y `health.repositories` (§3.4.6)
- Endpoint `/monitor` — refresco selectivo desde webhooks Git (§3.4.7)

**03-04-config-client.md**
- Setup del cliente: `spring.config.import`
- Opciones: `configserver:`, `optional:configserver:`, `enabled: false`
- `spring.cloud.config.name` y `label` en el cliente
- Perfiles y entornos por fichero separado
- Configuración de reintentos (`fail-fast` + `retry`)
- `spring.cloud.config.watch`: polling automático de cambios sin Bus ni llamada manual
- Debug logging para diagnosticar qué propiedades se cargaron y desde qué fichero
- Múltiples fuentes en `spring.config.import` con orden de prioridad
- `server.overrides` — por qué `override-none` no las sobreescribe
- Bootstrap context y `bootstrap.yml` [LEGACY] — compatibilidad con Spring Boot 2.x
- Diagrama de prioridad de fuentes de configuración
- Testing del Config Client: 4 estrategias (§3.6.1)

**03-05-config-refresh.md**
- Refresco manual: `@RefreshScope` + `/actuator/refresh`
- Comportamiento cuando Git no está disponible durante el refresco
- Refresco masivo: Spring Cloud Bus (Kafka o RabbitMQ)
- Sintaxis `:**` en `busrefresh` para filtrar por instancia
- Refresco automático: Webhook Git → `/actuator/busrefresh`
- `EnvironmentChangeEvent` para lógica personalizada post-refresco
- `spring.cloud.bus.destination` — cambiar topic/exchange del Bus
- Tipos de eventos: `RefreshRemoteApplicationEvent` vs `EnvironmentChangeEvent`
- Refresco selectivo con `/monitor` — solo los servicios con ficheros modificados
- Comparativa de las cinco opciones de refresco
- Testing: `@RefreshScope` con `ContextRefresher`, Bus sin broker con `enabled: false` (§3.7.1)

**03-06-config-avanzado.md**
- Cifrado simétrico (AES) y asimétrico (RSA)
- Vault Transit como backend de cifrado (sin clave local, rotación automática)
- Rotación de clave: manual, Vault Transit, `TextEncryptor` multiversión — comparativa
- Descifrado en servidor vs en cliente
- Alta disponibilidad: load balancer, Eureka, múltiples URIs
- Testing: 5 estrategias (disabled, optional, embedded server, @RefreshScope, cifrado)

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

### BLOQUE 7 — API GATEWAY
→ [06-01-gateway-concepto.md](./06-01-gateway-concepto.md) | [06-02-gateway-rutas.md](./06-02-gateway-rutas.md) | [06-03-gateway-filtros.md](./06-03-gateway-filtros.md) | [06-04-gateway-avanzado.md](./06-04-gateway-avanzado.md)

**06-01-gateway-concepto.md**
- Qué es un API Gateway y sus responsabilidades
- Diagrama del ciclo completo de una petición (predicates → GlobalFilters → GatewayFilters → microservicio → respuesta)
- Spring Cloud Gateway: arquitectura reactiva vs Zuul
- Propiedades globales: `default-filters`, `httpclient`, `globalcors`, `metrics`
- SSL Termination: configurar HTTPS (`server.ssl.*`, PKCS12, mTLS)
- WebSocket proxying (`ws://`, `wss://`)
- Discovery Locator: auto-rutas desde Eureka/Consul
- Actuator: endpoints `/actuator/gateway/routes`, `/actuator/gateway/refresh`

**06-02-gateway-rutas.md**
- Conceptos: Route, Predicate, Filter
- Todos los predicates: Path, Method, Header, Query, Host, After/Before/Between, Cookie, RemoteAddr, Weight
- Configuración en YAML y Java DSL
- Rutas dinámicas con `RouteDefinitionLocator`

**06-03-gateway-filtros.md**
- Filtros predefinidos: StripPrefix, PrefixPath, RewritePath, SetPath, AddRequestHeader, SetRequestHeader, AddRequestHeadersIfNotPresent, MapRequestHeader, RemoveRequestHeader, RemoveResponseHeader, RewriteResponseHeader, RewriteLocationResponseHeader, DedupeResponseHeader, RequestSize, SecureHeaders, Retry, RedirectTo, CircuitBreaker, SaveSession
- `TokenRelay`: propagación automática de token OAuth2 al microservicio
- `CacheRequestBody`: leer el body reactivo más de una vez
- `GatewayFilter` personalizado con `AbstractGatewayFilterFactory`
- `GlobalFilter`, orden de ejecución y exclusión de rutas públicas

**06-04-gateway-avanzado.md**
- Rate Limiting con Redis (Token Bucket, KeyResolver por IP/usuario/API key)
- Headers de respuesta del rate limiter (`X-RateLimit-*`)
- Rate Limiting sin Redis (implementación personalizada in-memory)
- JWT: validar en Gateway y propagar identidad a microservicios
- Spring Security OAuth2 en el Gateway con `TokenRelay`
- Ejemplo integrado completo (JWT + rate limiting + circuit breaker + CORS)
- Testing: `WebTestClient`, WireMock para downstream, `@AutoConfigureWireMock`
- Comparativa Gateway vs Zuul y guía de migración

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
| [06-02-gateway-rutas.md](./06-02-gateway-rutas.md) | Bloque 7 | Completo |
| [06-03-gateway-filtros.md](./06-03-gateway-filtros.md) | Bloque 7 | Completo |
| [06-04-gateway-avanzado.md](./06-04-gateway-avanzado.md) | Bloque 7 | Completo |
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
