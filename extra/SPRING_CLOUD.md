# Spring Cloud — Guía Completa Profesional

> Documentación exhaustiva desde cero hasta nivel experto.  
> Apta para pruebas técnicas empresariales y certificaciones.

---

## ÍNDICE GENERAL

### PARTE 1 — FUNDAMENTOS CONCEPTUALES

- [1.1 ¿Qué es Spring Cloud?](#11-qué-es-spring-cloud)
- [1.2 Historia y evolución](#12-historia-y-evolución)
- [1.3 Spring Cloud vs Spring Boot vs Spring Framework](#13-spring-cloud-vs-spring-boot-vs-spring-framework)
- [1.4 El problema que resuelve: arquitectura monolítica vs microservicios](#14-el-problema-que-resuelve-arquitectura-monolítica-vs-microservicios)
- [1.5 Los 8 problemas clásicos de los microservicios](#15-los-8-problemas-clásicos-de-los-microservicios)
- [1.6 Cómo Spring Cloud resuelve esos problemas](#16-cómo-spring-cloud-resuelve-esos-problemas)
- [1.7 Versiones, BOM y compatibilidad con Spring Boot](#17-versiones-bom-y-compatibilidad-con-spring-boot)

---

### PARTE 2 — ARQUITECTURA DE SPRING CLOUD

- [2.1 Visión general de la arquitectura](#21-visión-general-de-la-arquitectura)
- [2.2 Patrones de diseño en microservicios que implementa Spring Cloud](#22-patrones-de-diseño-en-microservicios-que-implementa-spring-cloud)
- [2.3 Mapa de componentes y sus relaciones](#23-mapa-de-componentes-y-sus-relaciones)
- [2.4 Flujo de una petición en un ecosistema Spring Cloud](#24-flujo-de-una-petición-en-un-ecosistema-spring-cloud)

---

### PARTE 3 — SPRING CLOUD CONFIG (Configuración centralizada)

- [3.1 Qué es y para qué sirve](#31-qué-es-y-para-qué-sirve)
- [3.2 Arquitectura: Config Server y Config Client](#32-arquitectura-config-server-y-config-client)
- [3.3 Backends soportados: Git, filesystem, Vault, JDBC](#33-backends-soportados-git-filesystem-vault-jdbc)
- [3.4 Configuración del Config Server](#34-configuración-del-config-server)
- [3.5 Configuración del Config Client](#35-configuración-del-config-client)
- [3.6 Perfiles y entornos (application-{profile}.yml)](#36-perfiles-y-entornos-application-profileyml)
- [3.7 Refresco de configuración: @RefreshScope y Spring Cloud Bus](#37-refresco-de-configuración-refreshscope-y-spring-cloud-bus)
- [3.8 Cifrado y descifrado de propiedades sensibles](#38-cifrado-y-descifrado-de-propiedades-sensibles)
- [3.9 Alta disponibilidad del Config Server](#39-alta-disponibilidad-del-config-server)

---

### PARTE 4 — SERVICE DISCOVERY (Descubrimiento de servicios)

- [4.1 Qué es el Service Discovery y por qué es necesario](#41-qué-es-el-service-discovery-y-por-qué-es-necesario)
- [4.2 Modelos: Client-Side vs Server-Side Discovery](#42-modelos-client-side-vs-server-side-discovery)
- [4.3 Spring Cloud Netflix Eureka — Servidor](#43-spring-cloud-netflix-eureka--servidor)
- [4.4 Spring Cloud Netflix Eureka — Cliente](#44-spring-cloud-netflix-eureka--cliente)
- [4.5 Heartbeat, lease y evicción de instancias](#45-heartbeat-lease-y-evicción-de-instancias)
- [4.6 Self-preservation mode](#46-self-preservation-mode)
- [4.7 Eureka en alta disponibilidad (clúster)](#47-eureka-en-alta-disponibilidad-clúster)
- [4.8 Alternativas a Eureka: Consul y Zookeeper](#48-alternativas-a-eureka-consul-y-zookeeper)

---

### PARTE 5 — LOAD BALANCING (Balanceo de carga)

- [5.1 Qué es el balanceo de carga en microservicios](#51-qué-es-el-balanceo-de-carga-en-microservicios)
- [5.2 Spring Cloud LoadBalancer (sucesor de Ribbon)](#52-spring-cloud-loadbalancer-sucesor-de-ribbon)
- [5.3 Algoritmos: Round Robin y Random](#53-algoritmos-round-robin-y-random)
- [5.4 Configuración y personalización del balanceador](#54-configuración-y-personalización-del-balanceador)
- [5.5 Integración con Eureka y OpenFeign](#55-integración-con-eureka-y-openfeign)

---

### PARTE 6 — API GATEWAY

- [6.1 Qué es un API Gateway y sus responsabilidades](#61-qué-es-un-api-gateway-y-sus-responsabilidades)
- [6.2 Spring Cloud Gateway (arquitectura reactiva)](#62-spring-cloud-gateway-arquitectura-reactiva)
- [6.3 Conceptos: Route, Predicate, Filter](#63-conceptos-route-predicate-filter)
- [6.4 Configuración de rutas (YAML y Java DSL)](#64-configuración-de-rutas-yaml-y-java-dsl)
- [6.5 Filtros predefinidos más importantes](#65-filtros-predefinidos-más-importantes)
- [6.6 Filtros personalizados (GatewayFilter y GlobalFilter)](#66-filtros-personalizados-gatewayfilter-y-globalfilter)
- [6.7 Rate Limiting con Redis](#67-rate-limiting-con-redis)
- [6.8 Autenticación y autorización en el Gateway](#68-autenticación-y-autorización-en-el-gateway)
- [6.9 Gateway vs Zuul (comparativa)](#69-gateway-vs-zuul-comparativa)

---

### PARTE 7 — CIRCUIT BREAKER (Tolerancia a fallos)

- [7.1 El problema: fallos en cascada](#71-el-problema-fallos-en-cascada)
- [7.2 Patrón Circuit Breaker explicado](#72-patrón-circuit-breaker-explicado)
- [7.3 Estados: CLOSED, OPEN, HALF-OPEN](#73-estados-closed-open-half-open)
- [7.4 Resilience4j en Spring Cloud](#74-resilience4j-en-spring-cloud)
- [7.5 Configuración de CircuitBreaker con Resilience4j](#75-configuración-de-circuitbreaker-con-resilience4j)
- [7.6 Fallback: respuestas de emergencia](#76-fallback-respuestas-de-emergencia)
- [7.7 Retry, TimeLimiter y Bulkhead](#77-retry-timelimiter-y-bulkhead)
- [7.8 Integración con Spring Cloud Gateway](#78-integración-con-spring-cloud-gateway)
- [7.9 Monitoreo con Actuator y métricas](#79-monitoreo-con-actuator-y-métricas)

---

### PARTE 8 — COMUNICACIÓN ENTRE SERVICIOS

- [8.1 Comunicación síncrona vs asíncrona](#81-comunicación-síncrona-vs-asíncrona)
- [8.2 Spring Cloud OpenFeign — cliente HTTP declarativo](#82-spring-cloud-openfeign--cliente-http-declarativo)
- [8.3 Configuración de Feign: timeouts, interceptors, logging](#83-configuración-de-feign-timeouts-interceptors-logging)
- [8.4 Feign + LoadBalancer + Circuit Breaker (integración completa)](#84-feign--loadbalancer--circuit-breaker-integración-completa)
- [8.5 Spring Cloud Stream — mensajería con Kafka y RabbitMQ](#85-spring-cloud-stream--mensajería-con-kafka-y-rabbitmq)
- [8.6 Binders, Bindings y canales en Spring Cloud Stream](#86-binders-bindings-y-canales-en-spring-cloud-stream)
- [8.7 Spring Cloud Bus — propagación de eventos](#87-spring-cloud-bus--propagación-de-eventos)

---

### PARTE 9 — OBSERVABILIDAD (Trazabilidad distribuida)

- [9.1 El problema: rastrear peticiones entre microservicios](#91-el-problema-rastrear-peticiones-entre-microservicios)
- [9.2 Conceptos: Trace ID, Span ID, correlación](#92-conceptos-trace-id-span-id-correlación)
- [9.3 Spring Cloud Sleuth (inyección automática de trazas)](#93-spring-cloud-sleuth-inyección-automática-de-trazas)
- [9.4 Micrometer Tracing (sucesor de Sleuth en Spring Boot 3)](#94-micrometer-tracing-sucesor-de-sleuth-en-spring-boot-3)
- [9.5 Zipkin: servidor de trazas distribuidas](#95-zipkin-servidor-de-trazas-distribuidas)
- [9.6 Integración con Grafana y Prometheus](#96-integración-con-grafana-y-prometheus)

---

### PARTE 10 — SEGURIDAD

- [10.1 Seguridad en arquitecturas de microservicios](#101-seguridad-en-arquitecturas-de-microservicios)
- [10.2 OAuth2 y JWT en Spring Cloud](#102-oauth2-y-jwt-en-spring-cloud)
- [10.3 Spring Cloud Security con Authorization Server](#103-spring-cloud-security-con-authorization-server)
- [10.4 Propagación de tokens entre servicios](#104-propagación-de-tokens-entre-servicios)
- [10.5 Seguridad en el Config Server](#105-seguridad-en-el-config-server)
- [10.6 mTLS y comunicación segura entre microservicios](#106-mtls-y-comunicación-segura-entre-microservicios)

---

### PARTE 11 — SPRING CLOUD KUBERNETES

- [11.1 Spring Cloud en entornos Kubernetes](#111-spring-cloud-en-entornos-kubernetes)
- [11.2 Service Discovery nativo en Kubernetes](#112-service-discovery-nativo-en-kubernetes)
- [11.3 ConfigMaps y Secrets como fuente de configuración](#113-configmaps-y-secrets-como-fuente-de-configuración)
- [11.4 Eureka vs Kubernetes Service Discovery](#114-eureka-vs-kubernetes-service-discovery)

---

### PARTE 12 — PROYECTO PRÁCTICO COMPLETO

- [12.1 Diseño del sistema de ejemplo](#121-diseño-del-sistema-de-ejemplo)
- [12.2 Config Server con repositorio Git](#122-config-server-con-repositorio-git)
- [12.3 Eureka Server](#123-eureka-server)
- [12.4 Microservicio de productos](#124-microservicio-de-productos)
- [12.5 Microservicio de pedidos (con Feign + Circuit Breaker)](#125-microservicio-de-pedidos-con-feign--circuit-breaker)
- [12.6 API Gateway con rutas y filtros](#126-api-gateway-con-rutas-y-filtros)
- [12.7 Trazabilidad con Zipkin](#127-trazabilidad-con-zipkin)
- [12.8 Docker Compose del ecosistema completo](#128-docker-compose-del-ecosistema-completo)

---

### PARTE 13 — REFERENCIA RÁPIDA Y CERTIFICACIÓN

- [13.1 Tabla resumen de componentes](#131-tabla-resumen-de-componentes)
- [13.2 Anotaciones clave de Spring Cloud](#132-anotaciones-clave-de-spring-cloud)
- [13.3 Propiedades de configuración más importantes](#133-propiedades-de-configuración-más-importantes)
- [13.4 Preguntas frecuentes en entrevistas técnicas](#134-preguntas-frecuentes-en-entrevistas-técnicas)
- [13.5 Preguntas tipo certificación con respuestas razonadas](#135-preguntas-tipo-certificación-con-respuestas-razonadas)
- [13.6 Errores comunes y cómo resolverlos](#136-errores-comunes-y-cómo-resolverlos)
- [13.7 Checklist antes de ir a producción](#137-checklist-antes-de-ir-a-producción)

---

---

## PARTE 1 — FUNDAMENTOS CONCEPTUALES

---

### 1.1 ¿Qué es Spring Cloud?

Spring Cloud es un **conjunto de frameworks y librerías** que se construyen sobre Spring Boot y proporcionan soluciones listas para usar a los problemas más comunes que surgen al construir sistemas distribuidos y arquitecturas de microservicios.

No es un único producto, sino una **colección de proyectos** bajo el paraguas de Spring Cloud, cada uno resolviendo un problema concreto:

| Problema | Proyecto que lo resuelve |
|---|---|
| Configuración centralizada | Spring Cloud Config |
| Descubrimiento de servicios | Spring Cloud Netflix Eureka / Consul |
| Balanceo de carga | Spring Cloud LoadBalancer |
| Puerta de entrada única | Spring Cloud Gateway |
| Tolerancia a fallos | Spring Cloud Circuit Breaker (Resilience4j) |
| Cliente HTTP declarativo | Spring Cloud OpenFeign |
| Trazabilidad distribuida | Micrometer Tracing + Zipkin |
| Mensajería asíncrona | Spring Cloud Stream |

**Definición formal:** Spring Cloud provee herramientas para que los desarrolladores construyan rápidamente patrones comunes en sistemas distribuidos como: gestión de configuración, descubrimiento de servicios, circuit breakers, routing inteligente, micro-proxies, bus de control, tokens de acceso, sesión global y contratos de prueba.

**Punto clave:** Spring Cloud **no reinventa la rueda**. Integra y adapta soluciones ya probadas (Netflix OSS, HashiCorp Consul, Apache Kafka, etc.) bajo una API unificada y coherente con el ecosistema Spring.

---

### 1.2 Historia y evolución

```
2014 — Netflix publica su stack de microservicios como Open Source (Eureka, Ribbon, Hystrix, Zuul)
2015 — Spring Cloud 1.x integra el stack de Netflix (Spring Cloud Netflix)
2018 — Netflix pone en modo mantenimiento Hystrix, Ribbon y Zuul 1.x
2019 — Spring Cloud introduce alternativas propias: Gateway, LoadBalancer, Circuit Breaker SPI
2020 — Spring Cloud 2020.x (Ilford) consolida la migración fuera de Netflix OSS
2022 — Spring Boot 3 / Spring Cloud 2022.x migra a Jakarta EE y Java 17
2023 — Micrometer Tracing reemplaza a Spring Cloud Sleuth
2024 — Spring Cloud 2023.x (Leyton), soporte completo para Spring Boot 3.2+
```

**Por qué importa conocer la historia:**  
Muchos proyectos empresariales usan versiones antiguas con Hystrix o Ribbon. En entrevistas y pruebas técnicas aparecen ambas generaciones. Este documento cubre **la generación actual** pero señala las diferencias con la anterior donde es relevante.

---

### 1.3 Spring Cloud vs Spring Boot vs Spring Framework

Es una confusión muy frecuente en principiantes. La relación es jerárquica:

```
Spring Framework
    └── Spring Boot          (simplifica la configuración de Spring)
            └── Spring Cloud (añade capacidades para sistemas distribuidos)
```

| Aspecto | Spring Framework | Spring Boot | Spring Cloud |
|---|---|---|---|
| **Nivel** | Base | Sobre Spring Framework | Sobre Spring Boot |
| **Propósito** | IoC, DI, AOP, MVC | Auto-configuración, arranque rápido | Sistemas distribuidos |
| **Sin él** | Mucha configuración XML/Java | Posible pero tedioso | Habría que integrar todo manualmente |
| **Ejemplo** | `@Component`, `@Autowired` | `@SpringBootApplication`, starters | `@EnableEurekaServer`, `@FeignClient` |
| **Aplicación standalone** | Sí | Sí | No (siempre necesita Spring Boot) |

**Regla de oro:** Todo proyecto Spring Cloud **es** un proyecto Spring Boot. No todo proyecto Spring Boot necesita Spring Cloud (solo si tiene múltiples servicios que necesitan coordinarse).

---

### 1.4 El problema que resuelve: arquitectura monolítica vs microservicios

#### Arquitectura Monolítica

En una aplicación monolítica, **todo el código vive en un único desplegable** (un WAR o JAR):

```
[ Monolito ]
  ├── Módulo Usuarios
  ├── Módulo Productos  
  ├── Módulo Pedidos
  ├── Módulo Pagos
  └── Módulo Notificaciones
        ↕
   [ Base de datos única ]
```

**Ventajas del monolito:**
- Desarrollo inicial simple
- Una sola unidad a desplegar
- Llamadas entre módulos son llamadas de método (rápidas, sin red)
- Transacciones ACID triviales

**Problemas del monolito a escala:**
- Un bug en el módulo de notificaciones puede tumbar todo el sistema
- Para escalar el módulo de productos hay que escalar **todo** el monolito
- El equipo crece → conflictos en el repositorio, deploys coordinados obligatorios
- Tecnología única: si el módulo de IA necesita Python, no puede usarlo
- Tiempo de compilación y arranque crece con el sistema

#### Arquitectura de Microservicios

Cada módulo se convierte en un **servicio independiente**, con su propia base de datos, desplegable por separado:

```
[API Gateway]
     |
     ├──→ [Servicio Usuarios   :8081]  ←→ [DB Usuarios]
     ├──→ [Servicio Productos  :8082]  ←→ [DB Productos]
     ├──→ [Servicio Pedidos    :8083]  ←→ [DB Pedidos]
     │         ↓ (llama a)
     │    [Servicio Productos  :8082]
     ├──→ [Servicio Pagos      :8084]  ←→ [DB Pagos]
     └──→ [Servicio Notif.     :8085]  ←→ [Cola Mensajes]
```

**Ventajas:**
- Escalar solo el servicio que lo necesita
- Equipos autónomos por servicio
- Distintas tecnologías por servicio
- Fallo aislado: si cae Notificaciones, los pedidos siguen funcionando
- Despliegues independientes

**Nuevos problemas que aparecen** (y que Spring Cloud resuelve — ver 1.5):

---

### 1.5 Los 8 problemas clásicos de los microservicios

Al pasar a microservicios, aparecen problemas que no existían en el monolito:

#### Problema 1: ¿Dónde está cada servicio?
En un monolito, un módulo llama a otro con una llamada de método. En microservicios, el Servicio A necesita conocer la IP y puerto del Servicio B. Con auto-escalado, esas IPs **cambian dinámicamente**.  
→ **Solución: Service Discovery (Eureka)**

#### Problema 2: ¿Cómo gestiono la configuración de 50 servicios?
Cada microservicio tiene su `application.yml`. En producción hay que actualizar URLs, credenciales, flags de feature… en 50 repositorios distintos.  
→ **Solución: Config Server**

#### Problema 3: ¿Cómo enrutan las peticiones del cliente?
El cliente (app móvil, SPA) no puede conocer las URLs internas de 50 servicios. Tampoco puede manejar autenticación, rate limiting y CORS en cada uno.  
→ **Solución: API Gateway**

#### Problema 4: ¿Qué pasa si un servicio se cae o está lento?
Si el Servicio A llama al Servicio B y B tarda 30 segundos, A acumula hilos esperando → A se queda sin recursos → C que llama a A también falla → **fallo en cascada** que tumba todo el sistema.  
→ **Solución: Circuit Breaker (Resilience4j)**

#### Problema 5: ¿Cómo distribuyo la carga entre instancias?
Si hay 3 instancias del Servicio Productos, ¿quién decide a cuál de las tres enviar cada petición?  
→ **Solución: Load Balancer**

#### Problema 6: ¿Cómo sigo una petición entre 10 servicios?
Un error en producción: la petición pasó por Gateway → Auth → Pedidos → Productos → Inventario. ¿En cuál falló? Los logs están en 5 máquinas distintas.  
→ **Solución: Distributed Tracing (Zipkin + Micrometer)**

#### Problema 7: ¿Cómo comunico servicios sin acoplamiento temporal?
Algunos eventos no necesitan respuesta inmediata. El Servicio de Pagos confirma un pago y el de Notificaciones debe enviar un email, pero no necesita bloquear al de Pagos.  
→ **Solución: Spring Cloud Stream (Kafka/RabbitMQ)**

#### Problema 8: ¿Cómo autentico/autorizo en cada servicio?
¿Valido el JWT en cada microservicio? ¿Quién emite los tokens? ¿Cómo propago la identidad del usuario entre llamadas internas?  
→ **Solución: OAuth2 + JWT centralizado en Gateway**

---

### 1.6 Cómo Spring Cloud resuelve esos problemas

```
PROBLEMA                     COMPONENTE SPRING CLOUD          TECNOLOGÍA SUBYACENTE
─────────────────────────    ──────────────────────────────   ─────────────────────
Descubrimiento de servicios  Spring Cloud Netflix Eureka      Netflix Eureka
Configuración centralizada   Spring Cloud Config              Git, Vault, JDBC
Puerta de entrada            Spring Cloud Gateway             Project Reactor (WebFlux)
Fallos en cascada            Spring Cloud Circuit Breaker     Resilience4j
Balanceo de carga            Spring Cloud LoadBalancer        Integrado en Spring
Cliente HTTP declarativo     Spring Cloud OpenFeign           OpenFeign + OkHttp
Trazabilidad distribuida     Micrometer Tracing               Zipkin, Jaeger
Mensajería asíncrona         Spring Cloud Stream              Kafka, RabbitMQ
Propagación de eventos       Spring Cloud Bus                 Kafka, RabbitMQ
```

---

### 1.7 Versiones, BOM y compatibilidad con Spring Boot

Spring Cloud usa un esquema de versiones por **nombre de ciudad** (hasta 2020) y luego por **año** (desde 2020):

| Spring Cloud | Nombre código | Spring Boot compatible |
|---|---|---|
| 2023.0.x | Leyton | 3.2.x, 3.3.x |
| 2022.0.x | Kilburn | 3.0.x, 3.1.x |
| 2021.0.x | Jubilee | 2.6.x, 2.7.x |
| 2020.0.x | Ilford | 2.4.x, 2.5.x |
| Hoxton | — | 2.2.x, 2.3.x |
| Greenwich | — | 2.1.x |

**CRÍTICO:** Mezclar versiones incompatibles de Spring Boot y Spring Cloud es la causa número uno de problemas en proyectos nuevos.

#### Uso del BOM (Bill of Materials)

En lugar de gestionar versiones individuales de cada dependencia Spring Cloud, se usa el BOM:

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Con el BOM importado, las dependencias individuales **no necesitan versión**:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <!-- sin <version> — la gestiona el BOM -->
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

#### Con Gradle:

```groovy
// build.gradle
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.3"
    }
}
```

**Regla práctica:** Siempre consultar la [matriz de compatibilidad oficial](https://spring.io/projects/spring-cloud) antes de crear un proyecto nuevo.

---

> **FIN PARTE 1** — Continúa en Parte 2: Arquitectura de Spring Cloud

---

## PARTE 2 — ARQUITECTURA DE SPRING CLOUD

---

### 2.1 Visión general de la arquitectura

Un ecosistema Spring Cloud completo se compone de capas que trabajan juntas. Cada capa tiene una responsabilidad clara:

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTES                             │
│            (Browser, App móvil, Servicio externo)           │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTP/HTTPS
┌─────────────────────────▼───────────────────────────────────┐
│                    API GATEWAY                              │
│         (Spring Cloud Gateway — puerto único de entrada)    │
│    Routing · Auth · Rate Limiting · SSL Termination         │
└──────┬──────────────────┬──────────────────┬────────────────┘
       │                  │                  │
┌──────▼──────┐   ┌───────▼──────┐   ┌──────▼──────┐
│  Servicio A │   │  Servicio B  │   │  Servicio C │
│  (Spring    │   │  (Spring     │   │  (Spring    │
│   Boot)     │   │   Boot)      │   │   Boot)     │
└──────┬──────┘   └───────┬──────┘   └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │ todos se registran en
┌─────────────────────────▼───────────────────────────────────┐
│              SERVICE REGISTRY (Eureka Server)               │
│         "Directorio telefónico" de microservicios           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│               CONFIG SERVER                                 │
│    (centraliza application.yml de todos los servicios)      │
│    Backend: repositorio Git con archivos de configuración   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              OBSERVABILIDAD                                 │
│   Zipkin (trazas) · Prometheus (métricas) · ELK (logs)      │
└─────────────────────────────────────────────────────────────┘
```

**Flujo de arranque de un microservicio:**

1. El microservicio arranca y lee su configuración del **Config Server**
2. Se registra en **Eureka Server** (anuncia su IP y puerto)
3. Comienza a atender peticiones
4. Para llamar a otro servicio, consulta Eureka para descubrir sus instancias
5. Usa **LoadBalancer** para elegir una instancia y **Feign** para hacer la llamada HTTP
6. Si el servicio destino falla, el **Circuit Breaker** corta el flujo y devuelve un fallback

---

### 2.2 Patrones de diseño en microservicios que implementa Spring Cloud

Spring Cloud no es arbitrario: implementa **patrones de diseño catalogados** por la industria.

#### Patrón: Service Registry and Discovery
- **Problema:** Las IPs de los servicios son dinámicas
- **Solución:** Un registro central donde cada servicio publica su localización
- **Implementación:** Eureka, Consul

#### Patrón: Client-Side Load Balancing
- **Problema:** Hay múltiples instancias del mismo servicio, ¿a cuál llamar?
- **Solución:** El cliente (no un balanceador externo) elige la instancia
- **Implementación:** Spring Cloud LoadBalancer

#### Patrón: API Gateway / Edge Server
- **Problema:** Los clientes externos no deben conocer la topología interna
- **Solución:** Un único punto de entrada que enruta, autentica y filtra
- **Implementación:** Spring Cloud Gateway

#### Patrón: Circuit Breaker
- **Problema:** Un servicio lento bloquea recursos y produce fallos en cascada
- **Solución:** Cortar el circuito cuando el servicio falla y usar un fallback
- **Implementación:** Resilience4j

#### Patrón: Externalized Configuration
- **Problema:** Configuración hardcodeada o dispersa en múltiples repos
- **Solución:** Repositorio central de configuración, accesible en tiempo de arranque
- **Implementación:** Spring Cloud Config

#### Patrón: Distributed Tracing
- **Problema:** Seguir una petición a través de N servicios es imposible sin correlación
- **Solución:** Inyectar IDs de traza y span en cada petición HTTP y log
- **Implementación:** Micrometer Tracing + Zipkin

#### Patrón: Sidecar
- **Problema:** Integrar servicios escritos en otros lenguajes al ecosistema Spring Cloud
- **Solución:** Un proceso Java secundario (sidecar) gestiona la comunicación
- **Implementación:** Spring Cloud Netflix Sidecar

#### Patrón: Bulkhead
- **Problema:** Un servicio sobreuso todos los recursos compartidos
- **Solución:** Aislar recursos por servicio (pools de hilos separados)
- **Implementación:** Resilience4j Bulkhead

---

### 2.3 Mapa de componentes y sus relaciones

```
                    ┌─────────────────────────────────────┐
                    │         spring-cloud-config          │
                    │    Config Server ←── Git Repo        │
                    │         ↑                            │
                    │    (todos los servicios leen aquí)   │
                    └─────────────────┬───────────────────┘
                                      │ bootstrap / startup
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
┌─────────▼──────────┐   ┌───────────▼──────────┐  ┌────────────▼────────────┐
│   Gateway Service  │   │  Eureka Server        │  │  Microservicio A/B/C    │
│ spring-cloud-      │   │  spring-cloud-netflix │  │  spring-boot            │
│ gateway            │   │  -eureka-server       │  │  + eureka-client        │
│                    │   │                       │  │  + config-client        │
│ Routes ──────────────→ │  Registry             │ ←─ registers itself        │
│ Predicates         │   │  (IP:port catalog)  ←────→ heartbeat               │
│ Filters            │   └───────────────────────┘  │                         │
│                    │             ↑                 │  Feign Client ──────────┼──→ calls B
│ LoadBalancer ←─────┼─────────────┘                 │  Circuit Breaker        │
│ Circuit Breaker    │                               │  LoadBalancer           │
└────────────────────┘                               └─────────────────────────┘
          │
          │ propagates traceId
          ▼
┌─────────────────────┐
│  Zipkin / Tempo     │   ←── todos los servicios envían spans
│  (distributed       │
│   tracing UI)       │
└─────────────────────┘
```

---

### 2.4 Flujo de una petición en un ecosistema Spring Cloud

Traza completa de: **"Cliente externo pide el detalle de un pedido"**

```
1. CLIENTE
   GET https://api.miempresa.com/pedidos/42
   
2. API GATEWAY (Spring Cloud Gateway)
   - Recibe la petición en el puerto 443
   - Valida el JWT (filtro de autenticación)
   - Identifica la ruta: /pedidos/** → servicio "pedidos-service"
   - Consulta al LoadBalancer: ¿qué instancia de "pedidos-service" uso?
   
3. LOAD BALANCER (Spring Cloud LoadBalancer)
   - Consulta Eureka: "dame las instancias registradas de pedidos-service"
   - Eureka responde: [192.168.1.10:8083, 192.168.1.11:8083]
   - Round Robin → elige 192.168.1.10:8083
   
4. PEDIDOS-SERVICE (192.168.1.10:8083)
   - Recibe GET /pedidos/42
   - Necesita los datos del producto → llama a "productos-service"
   - Usa Feign Client: @FeignClient("productos-service")
   - Feign + LoadBalancer → Eureka → instancia de productos-service
   - Circuit Breaker: ¿está productos-service saludable?
     - SI → hace la llamada HTTP
     - NO → devuelve fallback (producto genérico o error controlado)
   
5. PRODUCTOS-SERVICE
   - Responde con los datos del producto

6. PEDIDOS-SERVICE
   - Combina datos del pedido + producto
   - Responde al Gateway

7. API GATEWAY
   - Añade headers de respuesta (CORS, cache-control)
   - Devuelve al cliente

EN PARALELO (todo el flujo):
   - Cada servicio inyecta traceId/spanId en sus logs
   - Cada servicio envía spans a Zipkin
   - Zipkin muestra el árbol completo de la petición
```

**Puntos clave de este flujo:**
- El cliente solo conoce la URL del Gateway
- Ningún servicio conoce las IPs hardcodeadas de los demás
- La configuración de todos los servicios viene del Config Server
- Si cualquier servicio falla, el Circuit Breaker evita el fallo en cascada
- Todo el flujo es trazable desde Zipkin con un único `traceId`

---

> **FIN PARTE 2** — Continúa en Parte 3: Spring Cloud Config
