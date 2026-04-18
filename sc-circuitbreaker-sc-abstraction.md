# 4.8 Spring Cloud Circuit Breaker — Abstraction Layer

← [4.7 AOP — Orden de aspectos y fallbackMethod](sc-circuitbreaker-aop.md) | [Índice](README.md) | [4.9 Eventos y Métricas — Observabilidad completa](sc-circuitbreaker-eventos.md) →

---

## Introducción

Spring Cloud Circuit Breaker es una capa de abstracción sobre las implementaciones concretas de circuit breaker (Resilience4j, Spring Retry, Sentinel). Existe para permitir que el código de negocio sea independiente del proveedor de circuit breaker: si un proyecto migra de Resilience4j a otra implementación, el código que usa `CircuitBreakerFactory` no necesita cambiar, solo la dependencia y la configuración del factory. Se necesita cuando un equipo quiere mantener portabilidad o cuando se trabaja en un proyecto multi-módulo donde distintos módulos pueden usar distintas implementaciones.

> [CONCEPTO] La abstracción Spring Cloud Circuit Breaker añade una API portable pero no reemplaza la API nativa de Resilience4j. La API nativa (`CircuitBreakerRegistry`, `@CircuitBreaker`, etc.) ofrece más funcionalidad y configuración. La abstracción SC es la elección cuando la portabilidad es prioritaria.

## Arquitectura de la abstracción

La abstracción define dos interfaces principales y sus implementaciones para Resilience4j:

```
CircuitBreakerFactory (interface)
    └─► Resilience4JCircuitBreakerFactory (implementación para Resilience4j sync)

ReactiveCircuitBreakerFactory (interface)
    └─► ReactiveResilience4JCircuitBreakerFactory (implementación para flujos reactivos)

CircuitBreaker (interface SC — no es la de Resilience4j)
    └─► run(Supplier<T>) : T
    └─► run(Supplier<T>, Function<Throwable, T>) : T  ← con fallback

ReactiveCircuitBreaker (interface SC)
    └─► run(Mono<T>) : Mono<T>
    └─► run(Mono<T>, Function<Throwable, Mono<T>>) : Mono<T>
    └─► run(Flux<T>) : Flux<T>
    └─► run(Flux<T>, Function<Throwable, Flux<T>>) : Flux<T>
```

> [ADVERTENCIA] `org.springframework.cloud.client.circuitbreaker.CircuitBreaker` (la interfaz SC) y `io.github.resilience4j.circuitbreaker.CircuitBreaker` (la clase nativa de Resilience4j) son tipos distintos con el mismo nombre simple. Importar el tipo equivocado es un error frecuente.

## Ejemplo central

El ejemplo muestra el uso completo de `CircuitBreakerFactory` con configuración global y por instancia mediante `Customizer`, y la versión reactiva con `ReactiveCircuitBreakerFactory`:

```java
package com.example.abstraction;

import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.timelimiter.TimeLimiterConfig;
import org.springframework.cloud.circuitbreaker.resilience4j.Resilience4JCircuitBreakerFactory;
import org.springframework.cloud.circuitbreaker.resilience4j.Resilience4JConfigBuilder;
import org.springframework.cloud.client.circuitbreaker.Customizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.time.Duration;

@Configuration
public class CircuitBreakerFactoryConfig {

    // Configuración global (default) — afecta a todos los CircuitBreakers
    // creados por el factory si no tienen configuración específica
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id ->
            new Resilience4JConfigBuilder(id)
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(10)
                    .minimumNumberOfCalls(5)
                    .failureRateThreshold(50f)
                    .waitDurationInOpenState(Duration.ofSeconds(20))
                    .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(3))
                    .build())
                .build());
    }

    // Configuración específica para una instancia — sobreescribe el default
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> paymentCustomizer() {
        return factory -> factory.configure(builder ->
            builder.circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(5)
                    .failureRateThreshold(60f)
                    .waitDurationInOpenState(Duration.ofSeconds(60))
                    .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(5))
                    .build()),
            "paymentService"); // ID de la instancia a configurar
    }
}
```

Uso en un servicio con la abstracción SC (código portable, no depende de Resilience4j):

```java
package com.example.abstraction;

import org.springframework.cloud.client.circuitbreaker.CircuitBreaker;
import org.springframework.cloud.client.circuitbreaker.CircuitBreakerFactory;
import org.springframework.stereotype.Service;

@Service
public class PortablePaymentService {

    private final CircuitBreakerFactory circuitBreakerFactory;
    private final ExternalPaymentClient paymentClient;

    public PortablePaymentService(CircuitBreakerFactory circuitBreakerFactory,
                                   ExternalPaymentClient paymentClient) {
        this.circuitBreakerFactory = circuitBreakerFactory;
        this.paymentClient = paymentClient;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        // create() obtiene o crea la instancia con la config de "paymentService"
        CircuitBreaker cb = circuitBreakerFactory.create("paymentService");

        // run() aplica el circuit breaker sobre el Supplier
        // Si falla (o está abierto), invoca la función de fallback
        return cb.run(
            () -> paymentClient.process(request),
            throwable -> PaymentResult.failed("Payment unavailable: " + throwable.getMessage())
        );
    }
}
```

Versión reactiva con `ReactiveCircuitBreakerFactory`:

```java
package com.example.abstraction;

import org.springframework.cloud.client.circuitbreaker.ReactiveCircuitBreaker;
import org.springframework.cloud.client.circuitbreaker.ReactiveCircuitBreakerFactory;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Service
public class ReactivePaymentService {

    private final ReactiveCircuitBreakerFactory reactiveFactory;
    private final ReactivePaymentClient paymentClient;

    public ReactivePaymentService(ReactiveCircuitBreakerFactory reactiveFactory,
                                   ReactivePaymentClient paymentClient) {
        this.reactiveFactory = reactiveFactory;
        this.paymentClient = paymentClient;
    }

    public Mono<PaymentResult> processPayment(PaymentRequest request) {
        ReactiveCircuitBreaker rcb = reactiveFactory.create("paymentService");

        // run(Mono, fallback) — aplica el CB sobre un Mono
        return rcb.run(
            paymentClient.processReactive(request),
            throwable -> Mono.just(PaymentResult.failed("Circuit open"))
        );
    }
}
```

## Tabla: SC Abstraction vs API nativa Resilience4j

| Aspecto | SC Abstraction | API Nativa Resilience4j |
|---------|---------------|------------------------|
| Portabilidad de código | Total — independiente del proveedor | Solo Resilience4j |
| Configuración | Via `Customizer<Resilience4JCircuitBreakerFactory>` | Via YAML `resilience4j.*` o builder |
| Acceso a eventos | No directo | `circuitBreaker.getEventPublisher()` |
| Métricas Actuator | Disponibles si resilience4j-spring-boot3 está en classpath | Completas |
| Sintaxis | `factory.create(id).run(supplier, fallback)` | `@CircuitBreaker` o `registry.circuitBreaker(id)` |
| Reactive support | `ReactiveCircuitBreakerFactory` | `CircuitBreaker.decorateMono()` etc. |
| Bulkhead, Retry, RateLimiter | No (solo CircuitBreaker) | Sí — todas las anotaciones |

> [CONCEPTO] La abstracción SC solo cubre CircuitBreaker (y TimeLimiter implícito). Para Retry, Bulkhead, RateLimiter siempre se usa la API nativa de Resilience4j.

## Dependencias necesarias

```xml
<!-- Para la abstracción SC con Resilience4j síncrono -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

<!-- Para la abstracción SC con Resilience4j reactivo (WebFlux) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar la abstracción SC en librerías internas compartidas entre equipos que pueden usar diferentes implementaciones de CB.
- Definir siempre un `Customizer` con `configureDefault` para establecer valores base sensatos (sin él, se usan los defaults de Resilience4j que son conservadores para producción pesada).
- Combinar la abstracción SC para CircuitBreaker con las anotaciones nativas para Retry y Bulkhead.

**Malas prácticas:**
- Usar la abstracción SC cuando se necesita acceso a `CircuitBreaker.getEventPublisher()` o configuración avanzada de Retry/Bulkhead: la abstracción no los expone.
- Mezclar `Customizer<Resilience4JCircuitBreakerFactory>` con propiedades YAML `resilience4j.circuitbreaker.*` sin entender la precedencia.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia entre `factory.configureDefault(...)` y `factory.configure(..., "instanceId")`?

> [EXAMEN] 2. ¿Qué tipo devuelve `CircuitBreakerFactory.create("myService")`? ¿Es el mismo tipo que `CircuitBreakerRegistry.circuitBreaker("myService")`?

> [EXAMEN] 3. ¿Puede la abstracción SC configurar Retry y Bulkhead además del CircuitBreaker?

> [EXAMEN] 4. ¿Qué dependencia se necesita para usar `ReactiveCircuitBreakerFactory` con Resilience4j?

> [EXAMEN] 5. En `ReactiveCircuitBreaker.run(Mono<T>, Function<Throwable, Mono<T>>)`, ¿cuándo se invoca la función de fallback?

---

← [4.7 AOP — Orden de aspectos y fallbackMethod](sc-circuitbreaker-aop.md) | [Índice](README.md) | [4.9 Eventos y Métricas — Observabilidad completa](sc-circuitbreaker-eventos.md) →
