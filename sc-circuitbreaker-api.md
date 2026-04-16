# 5.3 API programática de Spring Cloud Circuit Breaker y Customizer

← [5.2 Setup: dependencias y autoconfiguración de Resilience4j](sc-circuitbreaker-setup.md) | [Índice](README.md) | [5.4 Configuración YAML avanzada del Circuit Breaker](sc-circuitbreaker-config.md) →

## Introducción

La API programática de Spring Cloud Circuit Breaker proporciona una abstracción sobre Resilience4j (y otras implementaciones) mediante las interfaces `CircuitBreakerFactory` y `CircuitBreaker`. Esta abstracción tiene un objetivo concreto: el código de negocio que protege llamadas remotas no depende de las clases de Resilience4j directamente, lo que permite cambiar de proveedor de Circuit Breaker sin modificar los servicios que lo usan.

El mecanismo `Customizer` es la pieza que conecta la abstracción con la configuración específica de Resilience4j: permite definir configuraciones por nombre de instancia o una configuración por defecto que se aplica a todas las instancias no configuradas explícitamente. Este patrón es especialmente útil cuando se configuran muchos servicios con comportamientos similares pero con pequeñas variaciones.

> [PREREQUISITO] Los conceptos de estados y transiciones se explican en 5.1. Las propiedades YAML se explican en 5.4; los Customizers de este fichero son la alternativa programática a esas propiedades YAML.

## Representación visual

El flujo de creación y uso de una instancia de Circuit Breaker a través de la API de Spring Cloud es el siguiente:

```
Startup de la aplicación:
──────────────────────────────────────────────────────────
  @Bean Customizer<Resilience4JCircuitBreakerFactory>
  ─────────────────────────────────────────────────────
  | configure("paymentService", CircuitBreakerConfig)  |
  | configure("default", CircuitBreakerConfig)         |
  └─────────────────────┬───────────────────────────────┘
                        │ registra configuraciones
                        ▼
              Resilience4JCircuitBreakerFactory
              (bean autoconfigursd en contexto Spring)

En tiempo de ejecución (llamada de negocio):
──────────────────────────────────────────────────────────
  factory.create("paymentService")
         │
         ▼
  CircuitBreaker (Spring Cloud abstraction)
         │
         ▼
  cb.run(
    () -> restClient.get().uri(...).body(Response.class),  ← supplier
    ex -> fallbackResponse(ex)                              ← fallback
  )
         │
         ├─ CLOSED: ejecuta supplier normalmente
         ├─ OPEN:   lanza CallNotPermittedException → ejecuta fallback
         └─ HALF_OPEN: ejecuta supplier como prueba
```

| Método API                          | Descripción                                                                 |
|-------------------------------------|-----------------------------------------------------------------------------|
| `factory.create(id)`                | Obtiene o crea una instancia de CB por nombre                               |
| `cb.run(supplier)`                  | Ejecuta el supplier; lanza excepción si circuito OPEN                       |
| `cb.run(supplier, fallback)`        | Ejecuta supplier; invoca fallback con la excepción si falla o OPEN          |
| `Customizer.forBean(type, fn)`      | Aplica una función de configuración al bean de la fábrica                   |
| `factory.configureDefault(fn)`      | Registra configuración aplicada a instancias sin configuración específica   |
| `factory.configure(fn, id...)`      | Registra configuración para instancias específicas por nombre               |

## Ejemplo central

Un servicio de pagos que protege dos llamadas remotas con configuraciones distintas: una para el servicio de tarjetas (alta criticidad, umbral bajo) y otra para el servicio de notificaciones (baja criticidad, umbral alto). Ambas usan la API programática de Spring Cloud con `CircuitBreakerFactory` y `Customizer`.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```yaml
# application.yml — mínimo necesario (Actuator)
spring:
  application:
    name: payment-service

management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

```java
// CircuitBreakerCustomizerConfig.java — configuración de las instancias
package com.example.payment.config;

import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig.SlidingWindowType;
import org.springframework.cloud.circuitbreaker.resilience4j.Resilience4JCircuitBreakerFactory;
import org.springframework.cloud.circuitbreaker.resilience4j.Resilience4JConfigBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.cloud.client.circuitbreaker.Customizer;
import java.time.Duration;

@Configuration
public class CircuitBreakerCustomizerConfig {

    /**
     * Configuración por defecto: aplica a todas las instancias no configuradas
     * explícitamente. Umbral conservador para servicios de baja prioridad.
     */
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id ->
                new Resilience4JConfigBuilder(id)
                        .circuitBreakerConfig(CircuitBreakerConfig.custom()
                                .slidingWindowType(SlidingWindowType.COUNT_BASED)
                                .slidingWindowSize(20)
                                .minimumNumberOfCalls(5)
                                .failureRateThreshold(60)
                                .waitDurationInOpenState(Duration.ofSeconds(30))
                                .permittedNumberOfCallsInHalfOpenState(3)
                                .automaticTransitionFromOpenToHalfOpenEnabled(true)
                                .build())
                        .build()
        );
    }

    /**
     * Configuración específica para el servicio de tarjetas:
     * umbral de fallo bajo (30%) porque cualquier error en pagos es crítico.
     */
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> cardServiceCustomizer() {
        return factory -> factory.configure(
                builder -> builder.circuitBreakerConfig(
                        CircuitBreakerConfig.custom()
                                .slidingWindowType(SlidingWindowType.TIME_BASED)
                                .slidingWindowSize(10)
                                .minimumNumberOfCalls(3)
                                .failureRateThreshold(30)
                                .slowCallRateThreshold(50)
                                .slowCallDurationThreshold(Duration.ofSeconds(1))
                                .waitDurationInOpenState(Duration.ofSeconds(60))
                                .permittedNumberOfCallsInHalfOpenState(2)
                                .automaticTransitionFromOpenToHalfOpenEnabled(true)
                                .recordExceptions(
                                        java.io.IOException.class,
                                        org.springframework.web.client.HttpServerErrorException.class
                                )
                                .build()
                ).build(),
                "cardService"   // nombre de la instancia
        );
    }
}
```

```java
// PaymentService.java — uso de la API programática
package com.example.payment.service;

import org.springframework.cloud.client.circuitbreaker.CircuitBreaker;
import org.springframework.cloud.client.circuitbreaker.CircuitBreakerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import com.example.payment.model.CardAuthResponse;
import com.example.payment.model.NotificationResult;

@Service
public class PaymentService {

    private final CircuitBreakerFactory<?, ?> circuitBreakerFactory;
    private final RestClient restClient;

    public PaymentService(
            CircuitBreakerFactory<?, ?> circuitBreakerFactory,
            RestClient.Builder builder) {
        this.circuitBreakerFactory = circuitBreakerFactory;
        this.restClient = builder.build();
    }

    public CardAuthResponse authorizeCard(String cardToken, long amountCents) {
        // La instancia "cardService" usa la configuración específica del Customizer
        CircuitBreaker cardCb = circuitBreakerFactory.create("cardService");

        return cardCb.run(
                () -> restClient.post()
                        .uri("http://card-service/api/authorize")
                        .body(new CardAuthRequest(cardToken, amountCents))
                        .retrieve()
                        .body(CardAuthResponse.class),
                ex -> CardAuthResponse.declined("Circuit breaker abierto: " + ex.getClass().getSimpleName())
        );
    }

    public NotificationResult sendPaymentNotification(String userId, String message) {
        // La instancia "notificationService" usa la configuración por defecto
        CircuitBreaker notifCb = circuitBreakerFactory.create("notificationService");

        return notifCb.run(
                () -> restClient.post()
                        .uri("http://notification-service/api/notify")
                        .body(new NotificationRequest(userId, message))
                        .retrieve()
                        .body(NotificationResult.class),
                ex -> NotificationResult.skipped()
        );
    }
}
```

```java
// Modelos auxiliares necesarios para compilar el ejemplo
package com.example.payment.model;

public record CardAuthRequest(String cardToken, long amountCents) {}

public record CardAuthResponse(String status, String reason) {
    public static CardAuthResponse declined(String reason) {
        return new CardAuthResponse("DECLINED", reason);
    }
}

public record NotificationRequest(String userId, String message) {}

public record NotificationResult(String status) {
    public static NotificationResult skipped() {
        return new NotificationResult("SKIPPED");
    }
}
```

## Tabla de elementos clave

Los métodos y tipos clave de la API programática de Spring Cloud Circuit Breaker con Resilience4j.

| Tipo / Método                                              | Paquete / Import                                                             | Descripción                                                                  |
|------------------------------------------------------------|------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| `CircuitBreakerFactory<C,B>`                               | `org.springframework.cloud.client.circuitbreaker`                           | Interfaz base; inyectar como `CircuitBreakerFactory<?,?>`                    |
| `Resilience4JCircuitBreakerFactory`                        | `org.springframework.cloud.circuitbreaker.resilience4j`                     | Implementación concreta autoconfigured                                       |
| `ReactiveResilience4JCircuitBreakerFactory`                | `org.springframework.cloud.circuitbreaker.resilience4j`                     | Fábrica reactiva para WebFlux                                                |
| `factory.create(id)`                                       | —                                                                            | Obtiene o crea una instancia de CB por nombre                                |
| `factory.configureDefault(Function<String,R>)`             | —                                                                            | Configura la instancia por defecto (aplica a IDs no configurados)            |
| `factory.configure(Consumer<B>, String...)`                | —                                                                            | Configura instancias específicas por nombre                                  |
| `CircuitBreaker.run(Supplier<T>)`                          | `org.springframework.cloud.client.circuitbreaker`                           | Ejecuta el supplier protegido; lanza excepción si OPEN                       |
| `CircuitBreaker.run(Supplier<T>, Function<Throwable,T>)`   | —                                                                            | Ejecuta con fallback; el fallback recibe la excepción                        |
| `Customizer<Resilience4JCircuitBreakerFactory>`            | `org.springframework.cloud.client.circuitbreaker`                           | Bean que modifica la fábrica antes de que sea usada                          |
| `Resilience4JConfigBuilder`                                | `org.springframework.cloud.circuitbreaker.resilience4j`                     | Builder para construir la configuración de una instancia                     |
| `CircuitBreakerConfig`                                     | `io.github.resilience4j.circuitbreaker`                                     | Configuración nativa de Resilience4j                                         |

## Buenas y malas prácticas

**Hacer:**

- Inyectar `CircuitBreakerFactory<?,?>` (con wildcards) en lugar de `Resilience4JCircuitBreakerFactory` directamente. El punto de la abstracción es que el código de negocio no depende de la implementación concreta.
- Usar `configureDefault` para establecer un baseline seguro para todos los Circuit Breakers y sobreescribir solo las instancias que requieren umbrales específicos con `configure(...)`.
- Reutilizar instancias llamando `factory.create(id)` con el mismo ID: Resilience4j devuelve la misma instancia del registry (no crea una nueva en cada llamada), lo que permite compartir el estado entre invocaciones.
- Documentar internamente el mapa `nombre → servicio remoto → justificación de umbrales`. Sin este mapeo, los valores de `failureRateThreshold` son arbitrarios y difíciles de auditar en producción.

**Evitar:**

- No crear múltiples `Customizer` beans con `configureDefault` en distintas clases de configuración. Solo se aplicará uno (el último en orden de beans de Spring) y el otro será silenciosamente ignorado, causando comportamientos inesperados.
- Evitar llamar a `factory.create(id)` dentro de bucles o en cada request sin reusar la instancia. Aunque Resilience4j hace cache internamente, el lookup tiene coste; preferir almacenar la instancia de `CircuitBreaker` en un campo de la clase.
- No mezclar configuración YAML (`resilience4j.circuitbreaker.instances.*`) con `Customizer` programático para la misma instancia sin entender la precedencia: los `Customizer` beans tienen precedencia sobre la configuración YAML cuando usan la API `Resilience4JCircuitBreakerFactory`.

---

← [5.2 Setup: dependencias y autoconfiguración de Resilience4j](sc-circuitbreaker-setup.md) | [Índice](README.md) | [5.4 Configuración YAML avanzada del Circuit Breaker](sc-circuitbreaker-config.md) →
