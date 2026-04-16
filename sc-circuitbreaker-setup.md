# 5.2 Setup: dependencias y autoconfiguración de Resilience4j

← [5.1 Circuit Breaker: estados, transiciones y modelo de evaluación](sc-circuitbreaker-estados.md) | [Índice](README.md) | [5.3 API programática de Spring Cloud Circuit Breaker y Customizer](sc-circuitbreaker-api.md) →

## Introducción

Incorporar Resilience4j a un proyecto Spring Boot puede hacerse de dos caminos: usando el starter de Spring Cloud o usando directamente el starter de Resilience4j para Spring Boot. La elección no es trivial: el starter de Spring Cloud añade la abstracción `CircuitBreakerFactory` y `ReactiveCircuitBreakerFactory`, que desacopla el código de aplicación de Resilience4j concreto y permite cambiar la implementación sin tocar código de negocio. El starter directo de Resilience4j expone la API nativa completa con más opciones de configuración avanzada, pero acopla el código al proveedor.

Entender qué beans se crean automáticamente y cuáles requieren configuración explícita es la base para evitar instancias duplicadas, configuraciones ignoradas o beans de fábrica que se solapan. Este fichero cubre ese setup inicial antes de que el lector entre en la configuración funcional de cada componente.

> [PREREQUISITO] Los conceptos de estado del Circuit Breaker (CLOSED, OPEN, HALF_OPEN) se explican en 5.1. Las propiedades YAML avanzadas de cada instancia se desarrollan en 5.4.

## Representación visual

El diagrama muestra qué artefactos Maven resuelven qué beans de Spring, y la relación entre los dos caminos de inclusión.

```
Camino A — Spring Cloud starter (recomendado)
─────────────────────────────────────────────
spring-cloud-starter-circuitbreaker-resilience4j
    └── spring-cloud-circuitbreaker-resilience4j
            └── resilience4j-spring-boot3
                    └── resilience4j-core

Beans autoconfigurads (Resilience4jAutoConfiguration):
  ┌─────────────────────────────────────┐
  │ CircuitBreakerRegistry              │  registro central
  │ Resilience4JCircuitBreakerFactory   │  fábrica imperativa
  │ RetryRegistry, BulkheadRegistry     │  registros auxiliares
  │ RateLimiterRegistry                 │  idem
  │ TimeLimiterRegistry                 │  idem
  └─────────────────────────────────────┘

Camino B — starter reactivo (para WebFlux)
──────────────────────────────────────────
spring-cloud-starter-circuitbreaker-reactor-resilience4j
    └── añade ReactiveResilience4JCircuitBreakerFactory
            (adicionalmente al bean imperativo)

Camino C — starter directo Resilience4j (sin Spring Cloud)
───────────────────────────────────────────────────────────
io.github.resilience4j:resilience4j-spring-boot3
    → Expone la misma autoconfiguración PERO
    → NO registra CircuitBreakerFactory de Spring Cloud
    → El código de aplicación usa io.github.resilience4j.* directamente
```

| Característica                  | spring-cloud-starter (A)                | resilience4j-spring-boot3 (C)          |
|---------------------------------|------------------------------------------|----------------------------------------|
| API abstrata Spring Cloud       | Sí (`CircuitBreakerFactory`)             | No                                     |
| Cambio de proveedor sin código  | Sí                                       | No                                     |
| AOP con anotaciones             | Sí (requiere starter de Resilience4j)    | Sí                                     |
| Actuator endpoints              | Sí (con actuator en classpath)           | Sí                                     |
| Configuración YAML              | `resilience4j.*` (mismo formato)         | `resilience4j.*` (mismo formato)       |
| Doble dependencia               | Peligro si se añaden ambas               | —                                      |

## Ejemplo central

Proyecto Spring Boot 4.0.x con el starter de Spring Cloud Circuit Breaker, habilitación de las anotaciones AOP y configuración mínima de Actuator.

```xml
<!-- pom.xml — dependencias para el starter de Spring Cloud -->
<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2025.1.1</spring-cloud.version>
</properties>

<dependencies>
    <!-- Starter Spring Cloud (imperative) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>

    <!-- Starter Spring Cloud (reactive WebFlux) — solo si se usa WebClient -->
    <!--
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
    </dependency>
    -->

    <!-- Actuator para endpoints /actuator/circuitbreakers -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- AOP — necesario para las anotaciones @CircuitBreaker, @Retry, etc. -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```yaml
# application.yml — configuración mínima de arranque
spring:
  application:
    name: order-service

# Habilitar la autoconfiguración (por defecto ya está en true)
spring:
  cloud:
    circuitbreaker:
      resilience4j:
        enabled: true

# Configuración de la primera instancia de Circuit Breaker
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true

# Actuator — exponer endpoints de Circuit Breaker
management:
  endpoints:
    web:
      exposure:
        include: health,info,circuitbreakers,circuitbreakerevents
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
```

```java
// OrderServiceApplication.java — @SpringBootApplication estándar
package com.example.orders;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```java
// CircuitBreakerConfig.java — verificación de los beans autoconfigurads
package com.example.orders.config;

import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.boot.ApplicationRunner;
import org.springframework.cloud.circuitbreaker.resilience4j.Resilience4JCircuitBreakerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CircuitBreakerConfig {

    // Bean creado automáticamente por Resilience4jAutoConfiguration
    // Se puede inyectar directamente sin declararlo
    @Bean
    public ApplicationRunner printRegisteredCircuitBreakers(
            CircuitBreakerRegistry registry,
            Resilience4JCircuitBreakerFactory factory) {

        return args -> {
            // El registry está disponible al arranque; las instancias
            // de Resilience4j se crean lazily en el primer acceso
            System.out.println("CircuitBreakerFactory class: " + factory.getClass().getSimpleName());
            System.out.println("CircuitBreakerRegistry class: " + registry.getClass().getSimpleName());
        };
    }
}
```

## Tabla de elementos clave

Los beans y propiedades más relevantes durante el setup inicial de Resilience4j con Spring Cloud.

| Bean / Propiedad                                         | Tipo / Clase                                    | Default / Valor           | Descripción                                                                       |
|----------------------------------------------------------|-------------------------------------------------|---------------------------|-----------------------------------------------------------------------------------|
| `CircuitBreakerFactory`                                  | `Resilience4JCircuitBreakerFactory`             | Autoconfigured            | Fábrica imperative para crear instancias con `factory.create(id)`                 |
| `ReactiveCircuitBreakerFactory`                          | `ReactiveResilience4JCircuitBreakerFactory`     | Con starter reactor       | Fábrica reactiva para uso con WebFlux y WebClient                                 |
| `CircuitBreakerRegistry`                                 | `io.github.resilience4j.circuitbreaker.*`       | Autoconfigured            | Registro central de todas las instancias Resilience4j                             |
| `RetryRegistry`                                          | `io.github.resilience4j.retry.*`                | Autoconfigured            | Registro de instancias Retry                                                      |
| `BulkheadRegistry`                                      | `io.github.resilience4j.bulkhead.*`             | Autoconfigured            | Registro de instancias Bulkhead                                                   |
| `RateLimiterRegistry`                                   | `io.github.resilience4j.ratelimiter.*`          | Autoconfigured            | Registro de instancias RateLimiter                                                |
| `spring.cloud.circuitbreaker.resilience4j.enabled`      | `boolean`                                       | `true`                    | Activa/desactiva toda la autoconfiguración de Resilience4j Spring Cloud           |
| `resilience4j.circuitbreaker.instances.[name].*`        | Map de propiedades                              | —                         | Configuración YAML de una instancia específica de Circuit Breaker                 |
| `resilience4j.circuitbreaker.configs.[name].*`          | Map de propiedades                              | —                         | Configuración base compartida referenciada por `base-config`                      |
| `management.health.circuitbreakers.enabled`             | `boolean`                                       | `false`                   | Incluye el estado del CB en el endpoint `/actuator/health`                        |

## Buenas y malas prácticas

**Hacer:**

- Usar `spring-cloud-starter-circuitbreaker-resilience4j` como punto de entrada, no `resilience4j-spring-boot3` directamente, a menos que el proyecto no use la abstracción de Spring Cloud. El starter de Spring Cloud gestiona la versión compatible de Resilience4j automáticamente via el BOM.
- Incluir `spring-boot-starter-aop` explícitamente si se quieren usar las anotaciones `@CircuitBreaker`, `@Retry`, `@Bulkhead`. El starter de Spring Cloud no lo añade transitivamente en todos los escenarios.
- Añadir `spring-boot-starter-actuator` y configurar `management.endpoints.web.exposure.include=circuitbreakers` desde el principio. Los endpoints de Actuator son esenciales para diagnosticar el estado en desarrollo y en producción.
- Verificar el arranque inyectando `CircuitBreakerRegistry` en un `ApplicationRunner` para confirmar que la autoconfiguración ha funcionado correctamente antes de añadir lógica de negocio.

**Evitar:**

- No añadir simultáneamente `spring-cloud-starter-circuitbreaker-resilience4j` y `io.github.resilience4j:resilience4j-spring-boot3`. Crean beans duplicados y la autoconfiguración puede registrar configuraciones incompatibles. Si se necesitan las anotaciones AOP y la fábrica Spring Cloud, el primer starter es suficiente.
- Evitar declarar manualmente un `CircuitBreakerRegistry` como `@Bean` si ya está autoconfigurando. Sobreescribir el registry sin adaptar todos los Customizers produce instancias sin configuración aplicada.
- No confundir `spring-cloud-starter-circuitbreaker-resilience4j` con el starter reactivo. El starter imperativo no incluye `ReactiveCircuitBreakerFactory`; si el proyecto usa WebClient, se necesita el starter reactivo adicionalmente (o en sustitución si toda la aplicación es reactiva).

---

← [5.1 Circuit Breaker: estados, transiciones y modelo de evaluación](sc-circuitbreaker-estados.md) | [Índice](README.md) | [5.3 API programática de Spring Cloud Circuit Breaker y Customizer](sc-circuitbreaker-api.md) →
