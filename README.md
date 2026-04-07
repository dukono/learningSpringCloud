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
→ [03-config.md](./03-config.md)

- 3.1 Qué es y para qué sirve
- 3.2 Arquitectura: Config Server y Config Client
- 3.3 Backends soportados: Git, filesystem, Vault, JDBC
- 3.4 Configuración del Config Server (`@EnableConfigServer`)
- 3.5 Configuración del Config Client (`bootstrap.yml` / `spring.config.import`)
- 3.6 Perfiles y entornos (`application-{profile}.yml`)
- 3.7 Refresco de configuración: `@RefreshScope` y Spring Cloud Bus
- 3.8 Cifrado y descifrado de propiedades sensibles
- 3.9 Alta disponibilidad del Config Server

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
→ [06-api-gateway.md](./06-api-gateway.md)

- 6.1 Qué es un API Gateway y sus responsabilidades
- 6.2 Spring Cloud Gateway (arquitectura reactiva)
- 6.3 Conceptos: Route, Predicate, Filter
- 6.4 Configuración de rutas (YAML y Java DSL)
- 6.5 Filtros predefinidos más importantes
- 6.6 Filtros personalizados (`GatewayFilter` y `GlobalFilter`)
- 6.7 Rate Limiting con Redis
- 6.8 Autenticación y autorización en el Gateway
- 6.9 Gateway vs Zuul (comparativa)

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
| [03-config.md](./03-config.md) | Bloque 2 | Completo |
| [04-service-discovery.md](./04-service-discovery.md) | Bloque 3 | Completo |
| [05-load-balancing.md](./05-load-balancing.md) | Bloque 4 | Completo |
| [08-comunicacion.md](./08-comunicacion.md) | Bloque 5 | Completo |
| [07-circuit-breaker.md](./07-circuit-breaker.md) | Bloque 6 | Completo |
| [06-api-gateway.md](./06-api-gateway.md) | Bloque 7 | Completo |
| [09-observabilidad.md](./09-observabilidad.md) | Bloque 8 | Completo |
| [10-seguridad.md](./10-seguridad.md) | Bloque 9 | Completo |
| [11-kubernetes.md](./11-kubernetes.md) | Bloque 10 | Completo |
| [12-proyecto-practico.md](./12-proyecto-practico.md) | Bloque 11 | Completo |
| [13-referencia-rapida.md](./13-referencia-rapida.md) | Bloque 12 | Completo |

---

## Convenciones

| Símbolo | Significado |
|---------|-------------|
| `[CONCEPTO]` | Definición teórica importante |
| `[CÓDIGO]` | Fragmento de código práctico |
| `[ADVERTENCIA]` | Error común o trampa frecuente |
| `[EXAMEN]` | Punto frecuente en pruebas técnicas |
| `[LEGACY]` | Componente deprecado — conocer para exámenes, no usar en proyectos nuevos |

---

> Versión objetivo: **Spring Cloud 2023.0.x (Leyton)** + **Spring Boot 3.2/3.3** + **Java 17/21**
