# Índice — Spring Cloud 2025.1.1 (Oakwood)

## Mapa de módulos

| Tema | Módulo | JSON |
|------|--------|------|
| 1 | Spring Cloud Config | dominio-sc-config.json |
| 2 | Spring Cloud Netflix / Eureka | dominio-sc-eureka.json |
| 3 | Spring Cloud Gateway | dominio-sc-gateway.json |
| 4 | Spring Cloud OpenFeign | dominio-sc-feign.json |
| 5 | Spring Cloud Circuit Breaker / Resilience4j | dominio-sc-circuitbreaker.json |
| 6 | Spring Cloud Stream | dominio-sc-stream.json |
| 7 | Spring Cloud Bus | dominio-sc-bus.json |
| 8 | Spring Cloud Security | dominio-sc-security.json |
| 9 | Spring Cloud Kubernetes | dominio-sc-kubernetes.json |
| 10 | Spring Cloud Contract | dominio-sc-contract.json |
| 11 | Spring Cloud Task | dominio-sc-task.json |
| 12 | Spring Cloud Function | dominio-sc-function.json |
| 13 | Patrones de microservicios distribuidos | dominio-sc-patrones.json |

---

## Verificación de cobertura

1. **Subdominios representados**: Los 13 módulos del alcance de meta.md están representados:
   Config (9 hojas), Eureka (13), Gateway (21), OpenFeign (13), Circuit Breaker (14),
   Stream (13), Bus (9), Security (15), Kubernetes (12), Contract (15), Task (9),
   Function (11), Patrones (9). Total: 163 nodos hoja.
   Nota: el protocolo fija un límite orientativo de 150 nodos hoja. En este caso los nodos
   provienen íntegramente de los ficheros dominio-*.json ya aprobados; el índice los integra
   sin generar nodos propios. El alcance está correctamente definido en meta.md.

2. **Temas estándar de la fuente autoritativa** (docs.spring.io/spring-cloud):
   Todos los módulos declarados en "Dentro del alcance" de meta.md tienen representación.
   Spring Data, Spring Batch y plataformas cloud específicas están fuera del alcance por
   decisión explícita de meta.md. No se detectan ausencias.

3. **Orden didáctico**: Config (prerequisito operativo) → Eureka (service discovery base)
   → Gateway + OpenFeign + Circuit Breaker (comunicación síncrona) → Stream + Bus
   (comunicación asíncrona) → Security + Kubernetes + Contract + Task + Function
   (módulos especializados) → Patrones (integrador transversal). Progresión válida:
   cada tema puede comprenderse con los conocimientos aportados por los anteriores.

4. **Cohesión de nodos**: cada nodo hoja proviene de dominio-*.json con tarea única
   verificada (un verbo, un objeto). Los nodos intermedios (Gateway X.4, Eureka
   "Ciclo de vida del registro") agrupan hijos semánticamente relacionados sin fichero propio.

5. **Nombres de fichero repetidos**: ninguno. Cada prefijo de módulo (sc-config-, sc-eureka-,
   sc-gateway-, sc-feign-, sc-circuitbreaker-, sc-stream-, sc-bus-, sc-security-,
   sc-kubernetes-, sc-contract-, sc-task-, sc-function-, sc-patrones-) es exclusivo.

Cobertura verificada: no se detectan lagunas.


## README.md

- 1 Spring Cloud Config
    - [1.1 Arquitectura y concepto del Config Server](sc-config-arquitectura.md)
    - [1.2 Config Client — configuración y arranque](sc-config-client.md)
    - [1.3 Resolución de configuración por perfiles y labels](sc-config-resolucion.md)
    - [1.4 Git backend — configuración y autenticación](sc-config-git-backend.md)
    - [1.5 Backends alternativos al Git backend](sc-config-backends.md)
    - [1.6 Actualización dinámica de configuración (Refresh)](sc-config-refresh.md)
    - [1.7 Seguridad del Config Server](sc-config-security.md)
    - [1.8 Operación del Config Server en producción (HA, Webhooks y Observabilidad)](sc-config-operacion.md)
    - [1.9 Testing / Verificación de Spring Cloud Config](sc-config-testing.md)
- 2 Spring Cloud Netflix Eureka
    - [2.1 Arquitectura y rol de Eureka en el ecosistema de microservicios](sc-eureka-arquitectura.md)
    - [2.2 Configuración y arranque del Eureka Server](sc-eureka-server.md)
    - [2.3 Configuración y arranque del Eureka Client](sc-eureka-cliente.md)
    - [2.4 Registro de instancias: metadata y estado](sc-eureka-instancias.md)
    - 2.5 Ciclo de vida del registro: heartbeat y lease
        - [2.5.1 Heartbeat y lease del registro](sc-eureka-lease.md)
        - [2.5.2 Self-preservation mode](sc-eureka-self-preservation.md)
    - [2.6 Descubrimiento de servicios desde el cliente](sc-eureka-descubrimiento.md)
    - [2.7 Dashboard y API REST del Eureka Server](sc-eureka-api-rest.md)
    - [2.8 Alta disponibilidad: Eureka Server en cluster](sc-eureka-cluster.md)
    - [2.9 Seguridad del Eureka Server](sc-eureka-seguridad.md)
    - [2.10 Integración con Spring Cloud Config](sc-eureka-integracion-config.md)
    - [2.11 Operación y troubleshooting de Eureka](sc-eureka-troubleshooting.md)
    - [2.12 Testing / Verificación de Spring Cloud Netflix Eureka](sc-eureka-testing.md)
- 3 Spring Cloud Gateway
    - [3.1 Arquitectura reactiva y modelo de ejecución](sc-gateway-arquitectura.md)
    - [3.2 Definición y configuración de rutas](sc-gateway-rutas.md)
    - [3.3 Predicados de enrutamiento built-in](sc-gateway-predicados.md)
    - 3.4 GatewayFilters por ruta
        - [3.4.1 Filtros de cabeceras del request](sc-gateway-filtros-request-headers.md)
        - [3.4.2 Filtros de path y parámetros](sc-gateway-filtros-path.md)
        - [3.4.3 Filtros de response (cabeceras, status y body)](sc-gateway-filtros-response.md)
        - [3.4.4 Rate Limiting con RedisRateLimiter](sc-gateway-ratelimiting.md)
        - [3.4.5 Circuit Breaker en Gateway con fallback](sc-gateway-circuitbreaker.md)
        - [3.4.6 Retry: política de reintentos automáticos](sc-gateway-retry.md)
        - [3.4.7 Filtros de seguridad y sesión (SecureHeaders, SaveSession, TokenRelay)](sc-gateway-filtros-seguridad.md)
    - [3.5 GlobalFilters: filtros transversales y personalización](sc-gateway-globalfilters.md)
    - [3.6 Integración con Service Discovery y Load Balancer](sc-gateway-discovery.md)
    - [3.7 Timeouts y configuración del cliente HTTP Netty](sc-gateway-timeouts.md)
    - [3.8 CORS y TLS/HTTPS en Gateway](sc-gateway-cors-tls.md)
    - [3.9 Rutas dinámicas y recarga en caliente](sc-gateway-rutas-dinamicas.md)
    - [3.10 WebSockets y Server-Sent Events](sc-gateway-websockets.md)
    - [3.11 Filtros y predicados personalizados (extension points)](sc-gateway-extension-points.md)
    - [3.12 Spring Cloud Gateway MVC](sc-gateway-mvc.md)
    - [3.13 Actuator, administración y observabilidad](sc-gateway-actuator.md)
    - [3.14 Manejo de errores y respuestas de error](sc-gateway-error-handling.md)
    - [3.15 Testing / Verificación de Gateway](sc-gateway-testing.md)
- 4 Spring Cloud OpenFeign
    - [4.1 Setup y bootstrap](sc-feign-setup.md)
    - [4.2 Declaración de @FeignClient](sc-feign-cliente-declarativo.md)
    - [4.3 Configuración por clase Java](sc-feign-configuracion-java.md)
    - [4.4 Configuración por propiedades YAML](sc-feign-configuracion-propiedades.md)
    - [4.5 Codecs — Encoder, Decoder, FormEncoder, multipart y ErrorDecoder](sc-feign-codecs.md)
    - [4.6 Interceptores y observabilidad](sc-feign-interceptores.md)
    - [4.7 Logging](sc-feign-logging.md)
    - [4.8 Manejo de errores HTTP](sc-feign-errores.md)
    - [4.9 Fallback y Circuit Breaker](sc-feign-fallback.md)
    - [4.10 Retryer](sc-feign-retryer.md)
    - [4.11 Cliente HTTP subyacente](sc-feign-cliente-http.md)
    - [4.12 Herencia de interfaz y @SpringQueryMap](sc-feign-herencia.md)
    - [4.13 Testing](sc-feign-testing.md)
- 5 Spring Cloud Circuit Breaker / Resilience4j
    - [5.1 Circuit Breaker: estados, transiciones y modelo de evaluación](sc-circuitbreaker-estados.md)
    - [5.2 Setup: dependencias y autoconfiguración de Resilience4j](sc-circuitbreaker-setup.md)
    - [5.3 API programática de Spring Cloud Circuit Breaker y Customizer](sc-circuitbreaker-api.md)
    - [5.4 Configuración YAML avanzada del Circuit Breaker](sc-circuitbreaker-config.md)
    - [5.5 Fallback: estrategias de respuesta degradada](sc-circuitbreaker-fallback.md)
    - [5.6 Retry: política de reintentos, backoff y @Retry](sc-circuitbreaker-retry.md)
    - [5.7 Bulkhead: aislamiento de recursos](sc-circuitbreaker-bulkhead.md)
    - [5.8 RateLimiter: control de tasa de llamadas](sc-circuitbreaker-ratelimiter.md)
    - [5.9 TimeLimiter: acotación de tiempos de ejecución asíncronos](sc-circuitbreaker-timelimiter.md)
    - [5.10 Anotaciones AOP y orden de decoradores combinados](sc-circuitbreaker-aop.md)
    - [5.11 Métricas con Micrometer y endpoints Actuator](sc-circuitbreaker-metricas.md)
    - [5.12 Health check y eventos CircuitBreakerEvent en producción](sc-circuitbreaker-eventos.md)
    - [5.13 Integración con OpenFeign, RestClient y WebClient](sc-circuitbreaker-feign.md)
    - [5.14 Testing del Circuit Breaker, Retry y Bulkhead](sc-circuitbreaker-testing.md)
- 6 Spring Cloud Stream
    - [6.1 Modelo de programación funcional](sc-stream-modelo-funcional.md)
    - [6.2 Bindings y binders](sc-stream-bindings.md)
    - [6.3 Binder Kafka — configuración y propiedades](sc-stream-binder-kafka.md)
    - [6.4 Kafka Streams binder](sc-stream-binder-kafka-streams.md)
    - [6.5 Binder RabbitMQ — configuración y propiedades](sc-stream-binder-rabbit.md)
    - [6.6 Serialización y deserialización (SerDes)](sc-stream-serdes.md)
    - [6.7 Particionado de mensajes](sc-stream-particionado.md)
    - [6.8 Gestión de errores y DLQ](sc-stream-error-handling.md)
    - [6.9 Polling y batch consumers](sc-stream-polling-batch.md)
    - [6.10 StreamBridge — producción de mensajes programática](sc-stream-streambridge.md)
    - [6.11 Multi-binder](sc-stream-multi-binder.md)
    - [6.12 Actuator y monitorización de bindings](sc-stream-actuator.md)
    - [6.13 Testing / Verificación de Spring Cloud Stream](sc-stream-testing.md)
- 7 Spring Cloud Bus
    - [7.1 Arquitectura de Spring Cloud Bus](sc-bus-arquitectura.md)
    - [7.2 Setup y dependencias de Spring Cloud Bus](sc-bus-setup.md)
    - [7.3 Configuración del broker subyacente para Spring Cloud Bus](sc-bus-broker-config.md)
    - [7.4 Refresh de configuración distribuido con Spring Cloud Bus](sc-bus-refresh-distribuido.md)
    - [7.5 Eventos personalizados de Spring Cloud Bus](sc-bus-eventos-personalizados.md)
    - [7.6 Seguridad del endpoint actuator de Spring Cloud Bus](sc-bus-seguridad-endpoint.md)
    - [7.7 Tracing y observabilidad de Spring Cloud Bus](sc-bus-observabilidad.md)
    - [7.8 Patrones de fallo y troubleshooting de Spring Cloud Bus](sc-bus-troubleshooting.md)
    - [7.9 Testing de Spring Cloud Bus](sc-bus-testing.md)
- 8 Spring Cloud Security
    - [8.1 Fundamentos OAuth2 y JWT para microservicios](sc-security-oauth2-fundamentals.md)
    - [8.2 Resource Server JWT — configuración y personalización](sc-security-resource-server-jwt.md)
    - [8.3 Resource Server avanzado — claims, opaque tokens y multi-tenancy](sc-security-resource-server-advanced.md)
    - [8.4 OAuth2 Client servlet — registro, gestión de tokens y Client Credentials](sc-security-oauth2-client.md)
    - [8.5 OAuth2 Client avanzado — WebClient, ReactiveManager y logout OIDC](sc-security-oauth2-client-advanced.md)
    - [8.6 Token Relay en Spring Cloud Gateway](sc-security-token-relay.md)
    - [8.7 Autorización por URL y métodos con Spring Security](sc-security-authorization.md)
    - [8.8 Propagación de identidad entre microservicios síncronos](sc-security-propagation.md)
    - [8.9 Propagación de identidad en contextos asíncronos y mensajería](sc-security-propagation-async.md)
    - [8.10 Spring Authorization Server — configuración base](sc-security-authorization-server.md)
    - [8.11 Spring Authorization Server — personalización avanzada y OIDC](sc-security-authorization-server-advanced.md)
    - [8.12 Seguridad en microservicios reactivos con WebFlux](sc-security-reactive.md)
    - [8.13 CORS, CSRF y cabeceras de seguridad HTTP](sc-security-cors-csrf.md)
    - [8.14 Configuración avanzada de seguridad y producción](sc-security-production.md)
    - [8.15 Testing / Verificación de Spring Cloud Security](sc-security-testing.md)
- 9 Spring Cloud Kubernetes
    - [9.1 Setup y starters de Spring Cloud Kubernetes](sc-kubernetes-setup.md)
    - [9.2 ConfigMap PropertySource — lectura y fuentes](sc-kubernetes-configmap.md)
    - [9.3 ConfigMap PropertySource — recarga dinámica](sc-kubernetes-configmap-reload.md)
    - [9.4 Secrets PropertySource](sc-kubernetes-secrets.md)
    - [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md)
    - [9.6 KubernetesDiscoveryClient — descubrimiento de servicios](sc-kubernetes-discovery.md)
    - [9.7 Reactive DiscoveryClient y variantes de implementación](sc-kubernetes-discovery-reactive.md)
    - [9.8 Load Balancer con Kubernetes](sc-kubernetes-loadbalancer.md)
    - [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md)
    - [9.10 Health Indicators de Spring Cloud Kubernetes](sc-kubernetes-health.md)
    - [9.11 Leader Election con Spring Cloud Kubernetes](sc-kubernetes-leader.md)
    - [9.12 Testing de Spring Cloud Kubernetes](sc-kubernetes-testing.md)
- 10 Spring Cloud Contract
    - [10.1 Arquitectura CDC y modelo Consumer-Driven Contracts](sc-contract-fundamentos.md)
    - [10.2 Ciclo de vida CDC y workflow CI/CD](sc-contract-workflow.md)
    - 10.3 Plugin de build
        - [10.3.1 Plugin Maven/Gradle — goals, tasks y testMode](sc-contract-plugin.md)
        - [10.3.2 Plugin Maven/Gradle — propiedades de personalización](sc-contract-plugin-config.md)
    - 10.4 DSL de contratos
        - [10.4.1 DSL Groovy/YAML — estructura base de contratos HTTP](sc-contract-dsl.md)
        - [10.4.2 Matchers estándar en contratos](sc-contract-matchers.md)
        - [10.4.3 Custom matchers y extensión del DSL](sc-contract-custom-matchers.md)
    - [10.5 Clase base de tests del producer](sc-contract-producer-tests.md)
    - [10.6 Stub JAR y modos de StubRunner](sc-contract-stubs.md)
    - [10.7 Stub Runner en el consumer](sc-contract-stub-runner.md)
    - [10.8 Contract Messaging — contratos de eventos](sc-contract-messaging.md)
    - [10.9 Repositorio de contratos y publicación de stubs](sc-contract-stubs-repo.md)
    - [10.10 Compatibilidad, versionado y troubleshooting de contratos](sc-contract-troubleshooting.md)
    - [10.11 Soporte GraphQL en Spring Cloud Contract](sc-contract-graphql.md)
    - [10.12 Testing / Verificación de Spring Cloud Contract](sc-contract-testing.md)
- 11 Spring Cloud Task
    - [11.1 Fundamentos de Spring Cloud Task](sc-task-fundamentos.md)
    - [11.2 TaskRepository y persistencia de ejecuciones](sc-task-repository.md)
    - [11.3 Configuración y propiedades spring.cloud.task.*](sc-task-configuracion.md)
    - [11.4 Argumentos y parámetros de lanzamiento](sc-task-arguments.md)
    - [11.5 Integración con Spring Cloud Data Flow](sc-task-scdf-integration.md)
    - [11.6 Composed Tasks y Composed Task Runner](sc-task-composed.md)
    - [11.7 Integración con Spring Batch](sc-task-batch-integration.md)
    - [11.8 Métricas, observabilidad y trazabilidad](sc-task-observability.md)
    - [11.9 Testing de Spring Cloud Task](sc-task-testing.md)
- 12 Spring Cloud Function
    - [12.1 Modelo de programación funcional: Function, Consumer, Supplier y FunctionCatalog](sc-function-modelo.md)
    - [12.2 Configuración de funciones y exposición HTTP](sc-function-config.md)
    - [12.3 Manejo de tipos, inferencia y serialización de payloads](sc-function-tipos.md)
    - [12.4 Routing dinámico con RoutingFunction y expresiones SpEL](sc-function-routing.md)
    - 12.5 Despliegue serverless
        - [12.5.1 Despliegue en AWS Lambda con SpringBootRequestHandler y FunctionInvoker](sc-function-aws.md)
        - [12.5.2 Despliegue en Azure Functions y Google Cloud Functions](sc-function-cloud.md)
    - [12.6 Compilación a imagen nativa con GraalVM AOT](sc-function-native.md)
    - [12.7 Integración con Spring Cloud Stream: binding de funciones a canales](sc-function-stream.md)
    - [12.8 Puntos de extensión avanzados: FunctionAroundWrapper y FunctionRegistration](sc-function-extension.md)
    - [12.9 Observabilidad y operación de funciones expuestas](sc-function-operations.md)
    - [12.10 Testing y verificación de Spring Cloud Function](sc-function-testing.md)
- 13 Patrones de microservicios distribuidos con Spring Cloud
    - [13.1 CQRS con Spring Cloud Stream y Gateway](sc-patrones-cqrs.md)
    - [13.2 Saga de coreografía con Spring Cloud Stream](sc-patrones-saga-coreografia.md)
    - [13.3 Saga de orquestación con Spring Cloud Stream](sc-patrones-saga-orquestacion.md)
    - [13.4 Outbox Pattern con Spring Cloud Stream](sc-patrones-outbox.md)
    - [13.5 Strangler Fig con Spring Cloud Gateway](sc-patrones-strangler.md)
    - [13.6 API Composition con Spring Cloud Gateway y OpenFeign](sc-patrones-api-composition.md)
    - [13.7 Sidecar Pattern con Spring Cloud Netflix Sidecar](sc-patrones-sidecar.md)
    - [13.8 Resiliencia transversal en patrones distribuidos](sc-patrones-resiliencia-transversal.md)
    - [13.9 Testing de patrones distribuidos con Spring Cloud](sc-patrones-testing.md)

## Tabla resumen

| Nro | Tema                                              | Ficheros hoja | Líneas est./fichero |
|-----|---------------------------------------------------|:-------------:|:-------------------:|
|  1  | Spring Cloud Config                               |       9       |       ~250–400      |
|  2  | Spring Cloud Netflix Eureka                       |      13       |       ~250–380      |
|  3  | Spring Cloud Gateway                              |      21       |       ~130–490      |
|  4  | Spring Cloud OpenFeign                            |      13       |       ~250–380      |
|  5  | Spring Cloud Circuit Breaker / Resilience4j       |      14       |       ~250–420      |
|  6  | Spring Cloud Stream                               |      13       |       ~250–400      |
|  7  | Spring Cloud Bus                                  |       9       |       ~250–410      |
|  8  | Spring Cloud Security                             |      15       |       ~250–420      |
|  9  | Spring Cloud Kubernetes                           |      12       |       ~250–380      |
| 10  | Spring Cloud Contract                             |      15       |       ~250–400      |
| 11  | Spring Cloud Task                                 |       9       |       ~250–380      |
| 12  | Spring Cloud Function                             |      11       |       ~250–380      |
| 13  | Patrones de microservicios distribuidos           |       9       |       ~250–420      |
| **Total** |                                             |   **163**     |   **~41 000 total** |

Nota: los nodos intermedios (sin fichero) no se contabilizan. Total de ficheros .md = 163 nodos hoja.

---

### Desglose por fichero

#### Tema 1 — Spring Cloud Config (9 ficheros)

| Fichero                        | Líneas est. |
|--------------------------------|:-----------:|
| sc-config-arquitectura.md      |    ~350     |
| sc-config-client.md            |    ~300     |
| sc-config-resolucion.md        |    ~280     |
| sc-config-git-backend.md       |    ~400     |
| sc-config-backends.md          |    ~300     |
| sc-config-refresh.md           |    ~320     |
| sc-config-security.md          |    ~280     |
| sc-config-operacion.md         |    ~370     |
| sc-config-testing.md           |    ~250     |

#### Tema 2 — Spring Cloud Netflix Eureka (13 ficheros)

| Fichero                           | Líneas est. |
|-----------------------------------|:-----------:|
| sc-eureka-arquitectura.md         |    ~350     |
| sc-eureka-server.md               |    ~300     |
| sc-eureka-cliente.md              |    ~300     |
| sc-eureka-instancias.md           |    ~280     |
| sc-eureka-lease.md                |    ~300     |
| sc-eureka-self-preservation.md    |    ~250     |
| sc-eureka-descubrimiento.md       |    ~280     |
| sc-eureka-api-rest.md             |    ~280     |
| sc-eureka-cluster.md              |    ~380     |
| sc-eureka-seguridad.md            |    ~250     |
| sc-eureka-integracion-config.md   |    ~300     |
| sc-eureka-troubleshooting.md      |    ~350     |
| sc-eureka-testing.md              |    ~250     |

#### Tema 3 — Spring Cloud Gateway (21 ficheros)

| Fichero                               | Líneas est. |
|---------------------------------------|:-----------:|
| sc-gateway-arquitectura.md            |    ~370     |
| sc-gateway-rutas.md                   |    ~330     |
| sc-gateway-predicados.md              |    ~490     |
| sc-gateway-filtros-request-headers.md |    ~330     |
| sc-gateway-filtros-path.md            |    ~250     |
| sc-gateway-filtros-response.md        |    ~420     |
| sc-gateway-ratelimiting.md            |    ~300     |
| sc-gateway-circuitbreaker.md          |    ~340     |
| sc-gateway-retry.md                   |    ~130     |
| sc-gateway-filtros-seguridad.md       |    ~210     |
| sc-gateway-globalfilters.md           |    ~400     |
| sc-gateway-discovery.md               |    ~280     |
| sc-gateway-timeouts.md                |    ~250     |
| sc-gateway-cors-tls.md                |    ~350     |
| sc-gateway-rutas-dinamicas.md         |    ~290     |
| sc-gateway-websockets.md              |    ~180     |
| sc-gateway-extension-points.md        |    ~210     |
| sc-gateway-mvc.md                     |    ~210     |
| sc-gateway-actuator.md                |    ~300     |
| sc-gateway-error-handling.md          |    ~180     |
| sc-gateway-testing.md                 |    ~250     |

#### Tema 4 — Spring Cloud OpenFeign (13 ficheros)

| Fichero                                  | Líneas est. |
|------------------------------------------|:-----------:|
| sc-feign-setup.md                        |    ~300     |
| sc-feign-cliente-declarativo.md          |    ~320     |
| sc-feign-configuracion-java.md           |    ~380     |
| sc-feign-configuracion-propiedades.md    |    ~300     |
| sc-feign-codecs.md                       |    ~350     |
| sc-feign-interceptores.md                |    ~300     |
| sc-feign-logging.md                      |    ~250     |
| sc-feign-errores.md                      |    ~300     |
| sc-feign-fallback.md                     |    ~320     |
| sc-feign-retryer.md                      |    ~280     |
| sc-feign-cliente-http.md                 |    ~350     |
| sc-feign-herencia.md                     |    ~260     |
| sc-feign-testing.md                      |    ~280     |

#### Tema 5 — Spring Cloud Circuit Breaker / Resilience4j (14 ficheros)

| Fichero                            | Líneas est. |
|------------------------------------|:-----------:|
| sc-circuitbreaker-estados.md       |    ~350     |
| sc-circuitbreaker-setup.md         |    ~300     |
| sc-circuitbreaker-api.md           |    ~350     |
| sc-circuitbreaker-config.md        |    ~420     |
| sc-circuitbreaker-fallback.md      |    ~320     |
| sc-circuitbreaker-retry.md         |    ~350     |
| sc-circuitbreaker-bulkhead.md      |    ~380     |
| sc-circuitbreaker-ratelimiter.md   |    ~330     |
| sc-circuitbreaker-timelimiter.md   |    ~280     |
| sc-circuitbreaker-aop.md           |    ~400     |
| sc-circuitbreaker-metricas.md      |    ~320     |
| sc-circuitbreaker-eventos.md       |    ~280     |
| sc-circuitbreaker-feign.md         |    ~350     |
| sc-circuitbreaker-testing.md       |    ~280     |

#### Tema 6 — Spring Cloud Stream (13 ficheros)

| Fichero                             | Líneas est. |
|-------------------------------------|:-----------:|
| sc-stream-modelo-funcional.md       |    ~380     |
| sc-stream-bindings.md               |    ~350     |
| sc-stream-binder-kafka.md           |    ~400     |
| sc-stream-binder-kafka-streams.md   |    ~400     |
| sc-stream-binder-rabbit.md          |    ~350     |
| sc-stream-serdes.md                 |    ~300     |
| sc-stream-particionado.md           |    ~280     |
| sc-stream-error-handling.md         |    ~350     |
| sc-stream-polling-batch.md          |    ~280     |
| sc-stream-streambridge.md           |    ~300     |
| sc-stream-multi-binder.md           |    ~280     |
| sc-stream-actuator.md               |    ~250     |
| sc-stream-testing.md                |    ~280     |

#### Tema 7 — Spring Cloud Bus (9 ficheros)

| Fichero                            | Líneas est. |
|------------------------------------|:-----------:|
| sc-bus-arquitectura.md             |    ~300     |
| sc-bus-setup.md                    |    ~280     |
| sc-bus-broker-config.md            |    ~300     |
| sc-bus-refresh-distribuido.md      |    ~410     |
| sc-bus-eventos-personalizados.md   |    ~350     |
| sc-bus-seguridad-endpoint.md       |    ~250     |
| sc-bus-observabilidad.md           |    ~280     |
| sc-bus-troubleshooting.md          |    ~350     |
| sc-bus-testing.md                  |    ~280     |

#### Tema 8 — Spring Cloud Security (15 ficheros)

| Fichero                                      | Líneas est. |
|----------------------------------------------|:-----------:|
| sc-security-oauth2-fundamentals.md           |    ~380     |
| sc-security-resource-server-jwt.md           |    ~400     |
| sc-security-resource-server-advanced.md      |    ~420     |
| sc-security-oauth2-client.md                 |    ~380     |
| sc-security-oauth2-client-advanced.md        |    ~380     |
| sc-security-token-relay.md                   |    ~300     |
| sc-security-authorization.md                 |    ~350     |
| sc-security-propagation.md                   |    ~320     |
| sc-security-propagation-async.md             |    ~320     |
| sc-security-authorization-server.md          |    ~380     |
| sc-security-authorization-server-advanced.md |    ~400     |
| sc-security-reactive.md                      |    ~350     |
| sc-security-cors-csrf.md                     |    ~300     |
| sc-security-production.md                    |    ~350     |
| sc-security-testing.md                       |    ~280     |

#### Tema 9 — Spring Cloud Kubernetes (12 ficheros)

| Fichero                            | Líneas est. |
|------------------------------------|:-----------:|
| sc-kubernetes-setup.md             |    ~300     |
| sc-kubernetes-configmap.md         |    ~320     |
| sc-kubernetes-configmap-reload.md  |    ~300     |
| sc-kubernetes-secrets.md           |    ~280     |
| sc-kubernetes-namespace-rbac.md    |    ~350     |
| sc-kubernetes-discovery.md         |    ~320     |
| sc-kubernetes-discovery-reactive.md|    ~300     |
| sc-kubernetes-loadbalancer.md      |    ~300     |
| sc-kubernetes-client.md            |    ~380     |
| sc-kubernetes-health.md            |    ~260     |
| sc-kubernetes-leader.md            |    ~300     |
| sc-kubernetes-testing.md           |    ~280     |

#### Tema 10 — Spring Cloud Contract (15 ficheros)

| Fichero                          | Líneas est. |
|----------------------------------|:-----------:|
| sc-contract-fundamentos.md       |    ~350     |
| sc-contract-workflow.md          |    ~380     |
| sc-contract-plugin.md            |    ~320     |
| sc-contract-plugin-config.md     |    ~300     |
| sc-contract-dsl.md               |    ~400     |
| sc-contract-matchers.md          |    ~350     |
| sc-contract-custom-matchers.md   |    ~320     |
| sc-contract-producer-tests.md    |    ~350     |
| sc-contract-stubs.md             |    ~320     |
| sc-contract-stub-runner.md       |    ~360     |
| sc-contract-messaging.md         |    ~350     |
| sc-contract-stubs-repo.md        |    ~300     |
| sc-contract-troubleshooting.md   |    ~320     |
| sc-contract-graphql.md           |    ~280     |
| sc-contract-testing.md           |    ~280     |

#### Tema 11 — Spring Cloud Task (9 ficheros)

| Fichero                       | Líneas est. |
|-------------------------------|:-----------:|
| sc-task-fundamentos.md        |    ~320     |
| sc-task-repository.md         |    ~380     |
| sc-task-configuracion.md      |    ~300     |
| sc-task-arguments.md          |    ~280     |
| sc-task-scdf-integration.md   |    ~380     |
| sc-task-composed.md           |    ~350     |
| sc-task-batch-integration.md  |    ~350     |
| sc-task-observability.md      |    ~300     |
| sc-task-testing.md            |    ~250     |

#### Tema 12 — Spring Cloud Function (11 ficheros)

| Fichero                    | Líneas est. |
|----------------------------|:-----------:|
| sc-function-modelo.md      |    ~380     |
| sc-function-config.md      |    ~320     |
| sc-function-tipos.md       |    ~350     |
| sc-function-routing.md     |    ~300     |
| sc-function-aws.md         |    ~350     |
| sc-function-cloud.md       |    ~320     |
| sc-function-native.md      |    ~350     |
| sc-function-stream.md      |    ~350     |
| sc-function-extension.md   |    ~300     |
| sc-function-operations.md  |    ~280     |
| sc-function-testing.md     |    ~280     |

#### Tema 13 — Patrones de microservicios distribuidos (9 ficheros)

| Fichero                                 | Líneas est. |
|-----------------------------------------|:-----------:|
| sc-patrones-cqrs.md                     |    ~420     |
| sc-patrones-saga-coreografia.md         |    ~370     |
| sc-patrones-saga-orquestacion.md        |    ~350     |
| sc-patrones-outbox.md                   |    ~350     |
| sc-patrones-strangler.md                |    ~300     |
| sc-patrones-api-composition.md          |    ~380     |
| sc-patrones-sidecar.md                  |    ~300     |
| sc-patrones-resiliencia-transversal.md  |    ~380     |
| sc-patrones-testing.md                  |    ~320     |
