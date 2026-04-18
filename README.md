# Índice — Spring Cloud 2025.1.1

> Spring Boot 3.5.x · VMware Spring Professional (EDU-1202)

## Mapa de módulos

| # | Tema | Módulo | JSON |
|---|------|--------|------|
| 1 | Spring Cloud Config | sc-config | dominio-sc-config.json |
| 2 | Spring Cloud Netflix / Eureka | sc-eureka | dominio-sc-eureka.json |
| 3 | Spring Cloud OpenFeign | sc-feign | dominio-sc-feign.json |
| 4 | Spring Cloud Circuit Breaker / Resilience4j | sc-resilience4j | dominio-sc-resilience4j.json |
| 5 | Spring Cloud Gateway | sc-gateway | dominio-sc-gateway.json |
| 6 | Spring Cloud Stream | sc-stream | dominio-sc-stream.json |
| 7 | Spring Cloud Bus | sc-bus | dominio-sc-bus.json |
| 8 | Spring Cloud Security | sc-security | dominio-sc-security.json |
| 9 | Spring Cloud Kubernetes | sc-kubernetes | dominio-sc-kubernetes.json |
| 10 | Spring Cloud Contract | sc-contract | dominio-sc-contract.json |
| 11 | Spring Cloud Task | sc-task | dominio-sc-task.json |
| 12 | Spring Cloud Function | sc-function | dominio-sc-function.json |
| 13 | Patrones de microservicios | sc-patrones | dominio-sc-patrones.json |

---

## Verificación de cobertura

1. **¿Están todos los módulos del alcance representados?**
   Sí — los 13 módulos definidos en meta.md tienen su JSON correspondiente y aparecen en el índice.

2. **¿La ordenación respeta la estrategia de meta.md?**
   Sí — se sigue exactamente: Config → Eureka → Feign → Resilience4j → Gateway → Stream → Bus → Security → Kubernetes → Contract → Task → Function → Patrones.

3. **¿Hay nodos intermedios sin fichero propio correctamente representados?**
   Sí — sc-config (X.3 Backends), sc-feign (X.2 Configuración, X.4 Logging/Errores, X.9 Herencia/Compresión) aparecen como nodos de agrupación sin enlace, con sus hijos enlazados.

4. **¿Existen módulos sin ficheros hoja (solo [intermedio])?**
   No — todos los módulos tienen al menos sus ficheros hoja resueltos en los JSON. Los nodos intermedios agrupan subnodos hoja que sí tienen fichero.

5. **¿Hay duplicados de ficheros?**
   No — cada fichero .md aparece una única vez en el índice (verificado en el paso de VERIFICACIÓN FINAL).

---

## Índice completo

- 1 Spring Cloud Config
  - [1.1 Concepto, arquitectura y configuración base del Config Server](sc-config-arquitectura.md)
  - [1.2 Config Client — bootstrap context y propiedades clave](sc-config-client.md)
  - 1.3 Backends de configuración
    - [1.3.1 Backend Git — autenticación, search-paths y etiquetas](sc-config-git-backend.md)
    - [1.3.2 Backends alternativos — native, Vault, JDBC y composite](sc-config-backends.md)
  - [1.4 Resolución de configuración — perfiles y overrides](sc-config-resolucion.md)
  - [1.5 Refresh de configuración en caliente — manual, Bus y webhooks](sc-config-refresh.md)
  - [1.6 Seguridad del Config Server — autenticación HTTP Basic y cifrado](sc-config-security.md)
  - [1.7 Operación y alta disponibilidad — fail-fast, retry y Eureka discovery](sc-config-operacion.md)
  - [1.8 Testing / Verificación de Config Server](sc-config-testing.md)

- 2 Spring Cloud Netflix / Eureka
  - [2.1 Arquitectura de Eureka: Server y Client](sc-eureka-arquitectura.md)
  - [2.2 Configuración del servidor Eureka (standalone)](sc-eureka-server.md)
  - [2.3 Registro y configuración del cliente Eureka](sc-eureka-cliente.md)
  - [2.4 Ciclo de vida de instancias: heartbeat y evicción](sc-eureka-lease.md)
  - [2.5 Self-preservation mode y umbral de renovación](sc-eureka-self-preservation.md)
  - [2.6 Alta disponibilidad: clúster Eureka peer-aware](sc-eureka-cluster.md)
  - [2.7 Estados, salud y metadata de instancias](sc-eureka-instancias.md)
  - [2.8 Discovery Client API: DiscoveryClient y EurekaClient](sc-eureka-descubrimiento.md)
  - [2.9 REST API de Eureka: endpoints de administración](sc-eureka-api-rest.md)
  - [2.10 Seguridad en Eureka Server: autenticación básica](sc-eureka-seguridad.md)
  - [2.11 Integración con Config Server: Discovery First vs Config First](sc-eureka-integracion-config.md)
  - [2.12 Zonas, regiones y configuración avanzada de Eureka](sc-eureka-zonas.md)
  - [2.13 Troubleshooting: instancias ghost, registro lento y diagnóstico](sc-eureka-troubleshooting.md)
  - [2.14 Testing de aplicaciones con Eureka](sc-eureka-testing.md)

- 3 Spring Cloud OpenFeign
  - [3.1 Cliente declarativo básico con OpenFeign](sc-feign-cliente-declarativo.md)
  - 3.2 Configuración del cliente Feign — Java y propiedades
    - [3.2.1 Configuración por clase Java (FeignClientConfiguration)](sc-feign-configuracion-java.md)
    - [3.2.2 Configuración por propiedades (spring.cloud.openfeign.client.config.*)](sc-feign-configuracion-propiedades.md)
  - [3.3 Codecs — Encoder y Decoder en Feign](sc-feign-codecs.md)
  - 3.4 Logging y manejo de errores en Feign
    - [3.4.1 Logger.Level y configuración dual de logging](sc-feign-logging.md)
    - [3.4.2 ErrorDecoder — manejo de errores HTTP](sc-feign-errores.md)
  - [3.5 Interceptores de petición — RequestInterceptor](sc-feign-interceptores.md)
  - [3.6 Integración con Eureka y Spring Cloud LoadBalancer](sc-feign-eureka-lb.md)
  - [3.7 Integración con Resilience4j — Fallback y FallbackFactory](sc-feign-fallback.md)
  - [3.8 Timeouts y cliente HTTP subyacente](sc-feign-http-client.md)
  - 3.9 Herencia de interfaces y compresión
    - [3.9.1 Herencia de interfaces API compartida](sc-feign-herencia.md)
    - [3.9.2 Compresión de peticiones y respuestas](sc-feign-compresion.md)
  - [3.10 Testing / Verificación de OpenFeign](sc-feign-testing.md)

- 4 Spring Cloud Circuit Breaker / Resilience4j
  - [4.1 Circuit Breaker — Estados y Transiciones](sc-circuitbreaker-estados.md)
  - [4.2 Circuit Breaker — Configuración completa](sc-circuitbreaker-configuracion.md)
  - [4.3 Circuit Breaker — Uso con anotaciones y API programática](sc-circuitbreaker-api.md)
  - [4.4 Retry — Configuración y uso](sc-circuitbreaker-retry.md)
  - [4.5 Bulkhead — Semaphore y ThreadPool](sc-circuitbreaker-bulkhead.md)
  - [4.6 RateLimiter y TimeLimiter](sc-circuitbreaker-ratelimiter.md)
  - [4.7 AOP — Orden de aspectos y fallbackMethod](sc-circuitbreaker-aop.md)
  - [4.8 Spring Cloud Circuit Breaker — Abstraction Layer](sc-circuitbreaker-sc-abstraction.md)
  - [4.9 Eventos y Métricas — Observabilidad completa](sc-circuitbreaker-eventos.md)
  - [4.10 Integración con Feign](sc-circuitbreaker-feign.md)
  - [4.11 Integración con Spring Cloud Gateway](sc-circuitbreaker-gateway.md)
  - [4.12 Métricas avanzadas y Health Indicators](sc-circuitbreaker-metricas.md)
  - [4.13 Testing con Resilience4j](sc-circuitbreaker-testing.md)

- 5 Spring Cloud Gateway
  - [5.1 Arquitectura y ciclo de vida de Spring Cloud Gateway](sc-gateway-arquitectura.md)
  - [5.2 Configuración de rutas: YAML, Java DSL y Timeouts](sc-gateway-configuracion-rutas.md)
  - [5.3 Predicate Factories built-in](sc-gateway-predicate-factories.md)
  - [5.4 GatewayFilter Factories built-in](sc-gateway-filter-factories-builtin.md)
  - [5.5 GatewayFilter Factories de resiliencia: CircuitBreaker, Retry y RequestRateLimiter](sc-gateway-filter-factories-resiliencia.md)
  - [5.6 GlobalFilter: interfaz, orden y filtros built-in](sc-gateway-global-filters.md)
  - [5.7 Integración con Service Discovery (DiscoveryClientRouteDefinitionLocator)](sc-gateway-discovery-integration.md)
  - [5.8 CORS en Spring Cloud Gateway](sc-gateway-cors.md)
  - [5.9 Seguridad y OAuth2: SecurityWebFilterChain y TokenRelay](sc-gateway-seguridad-oauth2.md)
  - [5.10 Actuator y observabilidad del Gateway](sc-gateway-actuator-observabilidad.md)
  - [5.11 Extensiones custom: Custom Predicate Factory y Custom GatewayFilter Factory](sc-gateway-custom-extensiones.md)
  - [5.12 Testing / Verificación de Spring Cloud Gateway](sc-gateway-testing.md)

- 6 Spring Cloud Stream
  - [6.1 Spring Cloud Stream — Setup y dependencias](sc-stream-setup.md)
  - [6.2 Spring Cloud Stream — Modelo de programación funcional](sc-stream-modelo-funcional.md)
  - [6.3 Spring Cloud Stream — Configuración de bindings](sc-stream-bindings-config.md)
  - [6.4 Spring Cloud Stream — Binder abstraction](sc-stream-binder-abstraction.md)
  - [6.5 Spring Cloud Stream — Kafka binder: configuración completa](sc-stream-kafka-binder.md)
  - [6.6 Spring Cloud Stream — RabbitMQ binder: configuración completa](sc-stream-rabbit-binder.md)
  - [6.7 Spring Cloud Stream — Consumer Groups y competing consumers](sc-stream-consumer-groups.md)
  - [6.8 Spring Cloud Stream — Particionado y afinidad de mensajes](sc-stream-particionado.md)
  - [6.9 Spring Cloud Stream — Error handling, reintentos y DLQ](sc-stream-error-handling.md)
  - [6.10 Spring Cloud Stream — Serialización y contentType](sc-stream-serializacion.md)
  - [6.11 Spring Cloud Stream — Integración interna con Spring Integration](sc-stream-integracion-spring-integration.md)
  - [6.12 Spring Cloud Stream — Actuator y operación de bindings](sc-stream-actuator.md)
  - [6.13 Spring Cloud Stream — Escenarios y preguntas de examen](sc-stream-escenarios-examen.md)
  - [6.14 Spring Cloud Stream — Testing con TestChannelBinder](sc-stream-testing.md)

- 7 Spring Cloud Bus
  - [7.1 Spring Cloud Bus — Arquitectura y propósito](sc-bus-arquitectura.md)
  - [7.2 Spring Cloud Bus — Setup y auto-configuración](sc-bus-setup.md)
  - [7.3 Spring Cloud Bus — Refresh distribuido de configuración](sc-bus-refresh-distribuido.md)
  - [7.4 Spring Cloud Bus — Eventos personalizados con RemoteApplicationEvent](sc-bus-eventos-personalizados.md)
  - [7.5 Spring Cloud Bus — Configuración de brokers RabbitMQ y Kafka](sc-bus-broker-config.md)
  - [7.6 Spring Cloud Bus — Trazabilidad y destination pattern](sc-bus-observabilidad.md)
  - [7.7 Spring Cloud Bus — Seguridad de endpoints del Bus](sc-bus-seguridad-endpoint.md)
  - [7.8 Spring Cloud Bus — Endpoint busenv y cambio en caliente](sc-bus-busenv.md)
  - [7.9 Spring Cloud Bus — Testing y Troubleshooting](sc-bus-testing.md)

- 8 Spring Cloud Security
  - [8.1 OAuth2 en microservicios — Roles y flujos de autorización](sc-security-oauth2-conceptos-flujos.md)
  - [8.2 Spring Authorization Server — Configuración y registro de clientes](sc-security-authorization-server.md)
  - [8.3 Resource Server — Validación de JWT en microservicios](sc-security-resource-server.md)
  - [8.4 OAuth2 Client — Gestión de tokens de salida entre servicios](sc-security-oauth2-client.md)
  - [8.5 JWT — Estructura, claims estándar y conversión a Spring Security](sc-security-jwt-claims.md)
  - [8.6 Token Relay en Spring Cloud Gateway — Propagación del token de usuario](sc-security-token-relay-gateway.md)
  - [8.7 Propagación de tokens entre microservicios — Feign, WebClient y RestTemplate](sc-security-propagacion-servicios.md)
  - [8.8 mTLS y seguridad avanzada en Spring Cloud Gateway](sc-security-mtls.md)
  - [8.9 Testing de seguridad OAuth2 — JWT mocks y Authorization Server simulado](sc-security-testing.md)

- 9 Spring Cloud Kubernetes
  - [9.1 Spring Cloud Kubernetes — Motivación y Arquitectura](sc-kubernetes-arquitectura.md)
  - [9.2 Spring Cloud Kubernetes — ConfigMap como PropertySource](sc-kubernetes-configmap.md)
  - [9.3 Spring Cloud Kubernetes — Secrets como PropertySource](sc-kubernetes-secrets.md)
  - [9.4 Spring Cloud Kubernetes — Service Discovery con KubernetesDiscoveryClient](sc-kubernetes-discovery.md)
  - [9.5 Spring Cloud Kubernetes — Starters: Fabric8 vs Cliente Oficial](sc-kubernetes-starters.md)
  - [9.6 Spring Cloud Kubernetes — Health Indicators y Actuator](sc-kubernetes-health.md)
  - [9.7 Spring Cloud Kubernetes — Reload de Configuración](sc-kubernetes-reload.md)
  - [9.8 Spring Cloud Kubernetes — Leader Election](sc-kubernetes-leader.md)
  - [9.9 Spring Cloud Kubernetes — Integración con Istio](sc-kubernetes-istio.md)
  - [9.10 Spring Cloud Kubernetes — Testing](sc-kubernetes-testing.md)

- 10 Spring Cloud Contract
  - [10.1 Spring Cloud Contract — Fundamentos CDC](sc-contract-fundamentos.md)
  - [10.2 Spring Cloud Contract — DSL Groovy y YAML](sc-contract-dsl.md)
  - [10.3 Spring Cloud Contract — Contratos HTTP](sc-contract-http.md)
  - [10.4 Spring Cloud Contract — Contratos Mensajería](sc-contract-mensajeria.md)
  - [10.5 Spring Cloud Contract — Plugin Maven y Gradle](sc-contract-plugin-config.md)
  - [10.6 Spring Cloud Contract — Clases Base Productor](sc-contract-base-class.md)
  - [10.7 Spring Cloud Contract — Stub Runner](sc-contract-stub-runner.md)
  - [10.8 Spring Cloud Contract — Repositorio de Stubs](sc-contract-stubs-repo.md)
  - [10.9 Spring Cloud Contract — Matchers Personalizados](sc-contract-matchers.md)
  - [10.10 Spring Cloud Contract — Contratos GraphQL](sc-contract-graphql.md)
  - [10.11 Spring Cloud Contract — Workflow CI/CD](sc-contract-workflow.md)
  - [10.12 Spring Cloud Contract — Troubleshooting](sc-contract-troubleshooting.md)

- 11 Spring Cloud Task
  - [11.1 Spring Cloud Task — Fundamentos y Ciclo de Vida](sc-task-fundamentos.md)
  - [11.2 Spring Cloud Task — TaskExecution y Persistencia en Base de Datos](sc-task-execution-bd.md)
  - [11.3 Spring Cloud Task — Configuración y Datasource](sc-task-configuracion.md)
  - [11.4 Spring Cloud Task — ApplicationRunner, CommandLineRunner y Argumentos](sc-task-runners.md)
  - [11.5 Spring Cloud Task — Integración con Spring Batch](sc-task-batch.md)
  - [11.6 Spring Cloud Task — Integración con Spring Cloud Data Flow](sc-task-data-flow.md)
  - [11.7 Spring Cloud Task — Task Events y TaskExecutionListener](sc-task-events.md)
  - [11.8 Spring Cloud Task — Composed Tasks](sc-task-composed.md)
  - [11.9 Spring Cloud Task — Testing](sc-task-testing.md)
  - [11.10 Spring Cloud Task — Troubleshooting](sc-task-troubleshooting.md)

- 12 Spring Cloud Function
  - [12.1 Spring Cloud Function — Modelo de programación funcional](sc-function-modelo-programacion.md)
  - [12.2 Spring Cloud Function — FunctionCatalog](sc-function-catalog.md)
  - [12.3 Spring Cloud Function — Adaptador HTTP](sc-function-adaptador-http.md)
  - [12.4 Spring Cloud Function — Adaptadores cloud (AWS / Azure / GCP)](sc-function-adaptadores-cloud.md)
  - [12.5 Spring Cloud Function — Composición de funciones](sc-function-composicion.md)
  - [12.6 Spring Cloud Function — Message<T> y MessageHeaders](sc-function-message-headers.md)
  - [12.7 Spring Cloud Function — Routing dinámico](sc-function-routing.md)
  - [12.8 Spring Cloud Function — Integración con Spring Cloud Stream](sc-function-integracion-stream.md)
  - [12.9 Spring Cloud Function — Tipos reactivos (Flux/Mono)](sc-function-tipos-reactivos.md)
  - [12.10 Spring Cloud Function — Testing y packaging](sc-function-testing.md)

- 13 Patrones de microservicios
  - [13.1 Descomposición por capacidades de negocio y subdominios DDD](sc-patrones-descomposicion-dominio.md)
  - [13.2 Comunicación síncrona: API Gateway y Backend for Frontend](sc-patrones-comunicacion-sincrona.md)
  - [13.3 Comunicación asíncrona y eventos: EDA, Choreography vs Orchestration, Outbox Pattern](sc-patrones-comunicacion-asincrona-eventos.md)
  - [13.4 Patrón Saga: choreography, orchestration, compensación e idempotencia](sc-patrones-saga.md)
  - [13.5 CQRS y Event Sourcing: patrón ES independiente, separación de modelos, proyecciones y consistencia eventual](sc-patrones-cqrs-event-sourcing.md)
  - [13.6 API Composition y Aggregator: llamadas paralelas reactivas y fallos parciales](sc-patrones-api-composition-aggregator.md)
  - [13.7 Patrones de resiliencia como diseño: Circuit Breaker, Bulkhead, Retry, Rate Limiting](sc-patrones-resilience-design-patterns.md)
  - [13.8 Observabilidad distribuida: trazas, métricas, logs y health checks](sc-patrones-observabilidad-distribuida.md)
  - [13.9 Patrones de seguridad: Access Token, API Gateway Auth y service-to-service auth](sc-patrones-seguridad-patrones.md)
  - [13.10 Patrones de despliegue: Strangler Fig, Sidecar, ACL y Database per Service](sc-patrones-deployment-patterns.md)
  - [13.11 Antipatrones: Distributed Monolith, Chatty Services, Shared Database, Mega-Service y Two-Phase Commit](sc-patrones-antipatrones.md)
  - [13.12 Testing de microservicios: Contract Testing, Component Tests, Chaos Engineering y estrategia E2E](sc-patrones-testing.md)

---

## Tabla resumen

| Nro | Módulo | Ficheros hoja | Líneas est./fichero |
|-----|--------|:-------------:|:-------------------:|
| 1 | Spring Cloud Config | 9 | ~80 |
| 2 | Spring Cloud Netflix / Eureka | 14 | ~70 |
| 3 | Spring Cloud OpenFeign | 12 | ~70 |
| 4 | Spring Cloud Circuit Breaker / Resilience4j | 13 | ~80 |
| 5 | Spring Cloud Gateway | 12 | ~90 |
| 6 | Spring Cloud Stream | 14 | ~80 |
| 7 | Spring Cloud Bus | 9 | ~70 |
| 8 | Spring Cloud Security | 9 | ~80 |
| 9 | Spring Cloud Kubernetes | 10 | ~70 |
| 10 | Spring Cloud Contract | 12 | ~75 |
| 11 | Spring Cloud Task | 10 | ~70 |
| 12 | Spring Cloud Function | 10 | ~70 |
| 13 | Patrones de microservicios | 12 | ~90 |
| — | **TOTAL** | **147** | — |
