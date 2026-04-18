# 13.7 Patrones de resiliencia como diseño: Circuit Breaker, Bulkhead, Retry, Rate Limiting

← [13.6 API Composition y Aggregator](sc-patrones-api-composition-aggregator.md) | [Índice](README.md) | [13.8 Observabilidad distribuida](sc-patrones-observabilidad-distribuida.md) →

---

## Introducción

Los patrones de resiliencia son principios de diseño arquitectural independientes de su implementación en una librería concreta. En microservicios, los fallos en cascada son el riesgo más crítico: un servicio lento o caído puede propagarse y tumbar todo el sistema si no se diseñan barreras de contención. Circuit Breaker, Bulkhead, Retry con backoff y Rate Limiting son las cuatro piezas fundamentales del diseño defensivo de microservicios.

> [PREREQUISITO] Este fichero describe los patrones como conceptos de diseño. La implementación concreta en Resilience4j está documentada en los ficheros del módulo 4 (sc-circuitbreaker-*).

## Circuit Breaker como patrón de diseño

> [CONCEPTO] **Circuit Breaker**: el Circuit Breaker envuelve una llamada remota y actúa como un interruptor de corriente eléctrica. Cuando los fallos superan un umbral, el circuito se **abre** y las llamadas subsiguientes fallan rápido sin llegar al servicio destino (fail-fast). Después de un tiempo de espera, el circuito pasa a **semi-abierto** y permite un número limitado de llamadas de prueba. Si las pruebas son exitosas, el circuito se **cierra** y el tráfico se reanuda normalmente.

Los tres estados y sus transiciones:

```
         umbral de fallos superado
CLOSED ────────────────────────────► OPEN
  ▲                                    │
  │ pruebas exitosas                   │ timeout de espera
  │                                    ▼
HALF-OPEN ◄──────────────────────── (espera)
              prueba fallida
HALF-OPEN ──────────────────────────► OPEN
```

El beneficio arquitectural del Circuit Breaker no es solo proteger el servicio destino — es proteger al **llamador**: en lugar de acumular threads/goroutines bloqueadas esperando un servicio que no responde, los recursos del llamador se liberan inmediatamente con un fallo rápido.

## Bulkhead como patrón de diseño

> [CONCEPTO] **Bulkhead**: el nombre viene de los mamparos de los barcos (bulkheads) que dividen el casco en compartimentos estancos. Si un compartimento se inunda, los demás quedan intactos. En microservicios, el Bulkhead aísla los recursos (threads, conexiones) por servicio destino, de modo que un servicio lento no consume todos los recursos del llamador y deja sin capacidad a otras llamadas independientes.

La diferencia con el Circuit Breaker es que el Circuit Breaker actúa cuando hay **fallos**, mientras que el Bulkhead actúa cuando hay **saturación de recursos**. Se complementan.

| Patrón | Protege contra | Mecanismo |
|---|---|---|
| Circuit Breaker | Fallos en cascada | Fail-fast cuando falla rate supera umbral |
| Bulkhead | Saturación de recursos | Pool separado de threads/conexiones por destino |
| Retry | Fallos transitorios | Re-intento con backoff exponencial |
| Rate Limiter | Sobrecarga del servicio propio | Limitar peticiones entrantes |

## Retry y Timeout como patrones

> [CONCEPTO] **Retry con backoff exponencial y jitter**: el Retry es adecuado para fallos transitorios (red momentáneamente saturada, servicio reiniciándose). Sin embargo, un Retry sin backoff puede agravar el fallo: si cientos de servicios reintentan simultáneamente al mismo tiempo, generan un pico de carga que impide la recuperación del servicio destino. El **backoff exponencial** aumenta el tiempo de espera entre reintentos (1s, 2s, 4s, 8s...). El **jitter** añade aleatoriedad al backoff para distribuir los reintentos en el tiempo y evitar la sincronización.

El **Timeout** es el complemento del Retry: limita cuánto tiempo espera el llamador antes de declarar el fallo. Sin timeout, un servicio lento puede bloquear al llamador indefinidamente.

> [ADVERTENCIA] No todos los errores son reintentables. Los errores de lógica de negocio (400 Bad Request, 422 Unprocessable Entity) no deben reintentarse — son errores deterministas. Solo los errores transitorios (503, 502, timeouts de red) justifican el retry.

## Rate Limiting como patrón

> [CONCEPTO] **Rate Limiting**: limita la tasa de peticiones aceptadas por un servicio en un período de tiempo para protegerlo de sobrecarga intencional (DDoS) o accidental (burst de tráfico). Se aplica en el API Gateway para clientes externos, o dentro del servicio para proteger recursos internos.

Los tres algoritmos principales de Rate Limiting:

| Algoritmo | Comportamiento | Uso típico |
|---|---|---|
| Token Bucket | Permite bursts hasta la capacidad del bucket | APIs con bursts permitidos |
| Leaky Bucket | Suaviza el tráfico a tasa constante | Protección de sistemas backend sensibles |
| Sliding Window | Ventana deslizante de peticiones | Balance entre precisión y eficiencia |

## Ejemplo central: diseño de resiliencia en un microservicio de pedidos

El siguiente ejemplo configura todos los patrones de resiliencia en un microservicio de pedidos usando Resilience4j como implementación de referencia: Circuit Breaker en la llamada al servicio de inventario, Bulkhead separado, Retry con backoff exponencial y Rate Limiter en el endpoint de creación de pedidos.

```yaml
# application.yml — configuración de patrones de resiliencia con Resilience4j

resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50         # abre el circuito si >50% de llamadas fallan
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 80        # trata llamadas lentas como fallos

  bulkhead:
    instances:
      inventory-service:
        maxConcurrentCalls: 10           # máximo 10 llamadas concurrentes al servicio de inventario
        maxWaitDuration: 100ms           # espera máxima para obtener permiso del bulkhead

  retry:
    instances:
      inventory-service:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2  # 500ms, 1000ms, 2000ms
        randomizedWaitFactor: 0.5        # jitter: ±50% del tiempo de espera
        retryExceptions:
          - org.springframework.web.reactive.function.client.WebClientResponseException$ServiceUnavailable
          - java.net.ConnectException

  ratelimiter:
    instances:
      create-order:
        limitRefreshPeriod: 1s
        limitForPeriod: 100              # máximo 100 peticiones por segundo
        timeoutDuration: 0               # falla inmediatamente si se supera el límite
```

```java
package com.example.orders.service;

import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class OrderService {

    private final WebClient inventoryClient;

    public OrderService(WebClient.Builder webClientBuilder) {
        this.inventoryClient = webClientBuilder
            .baseUrl("http://inventory-service")
            .build();
    }

    // Orden de aspectos Resilience4j: RateLimiter → CircuitBreaker → Retry → Bulkhead → TimeLimiter
    // (de mayor a menor prioridad de protección)

    @RateLimiter(name = "create-order", fallbackMethod = "rateLimitFallback")
    @CircuitBreaker(name = "inventory-service", fallbackMethod = "circuitBreakerFallback")
    @Retry(name = "inventory-service")
    @Bulkhead(name = "inventory-service", type = Bulkhead.Type.SEMAPHORE)
    public Mono<Boolean> checkInventoryAvailability(String productId, int quantity) {
        return inventoryClient.get()
            .uri("/stock/{productId}/available?quantity={quantity}", productId, quantity)
            .retrieve()
            .bodyToMono(Boolean.class);
    }

    // Fallback cuando el CircuitBreaker está abierto
    public Mono<Boolean> circuitBreakerFallback(String productId, int quantity, Throwable t) {
        // Degradación: asumir disponibilidad (para no bloquear pedidos en fallo de inventario)
        // En producción: evaluar si este es el comportamiento correcto para el negocio
        return Mono.just(true);
    }

    // Fallback cuando el RateLimiter rechaza la petición
    public Mono<Boolean> rateLimitFallback(String productId, int quantity, Throwable t) {
        return Mono.error(new RateLimitExceededException("Too many requests. Please retry later."));
    }
}

class RateLimitExceededException extends RuntimeException {
    public RateLimitExceededException(String message) {
        super(message);
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Combinar Circuit Breaker + Retry + Timeout en todas las llamadas a servicios externos: el Retry maneja fallos transitorios, el Circuit Breaker protege de fallos persistentes, el Timeout previene bloqueos indefinidos.
- Configurar el Bulkhead con pools separados por servicio destino, no un pool global.
- Usar jitter en el backoff exponencial del Retry para evitar el problema del "thundering herd".
- Aplicar Rate Limiting en el API Gateway para clientes externos y dentro del servicio para APIs internas críticas.

**Malas prácticas:**
- Aplicar Retry a errores no transitorios (4xx son deterministas, nunca mejorarán con reintentos).
- Omitir el Timeout en Retry — si el servicio tarda 30 segundos, el Retry lo repite 3 veces = 90 segundos de bloqueo.
- Configurar un Circuit Breaker con ventana demasiado pequeña — se abre y cierra continuamente (flapping).

## Verificación y práctica

> [EXAMEN] 1. ¿Qué problema resuelve el Circuit Breaker a nivel arquitectural y cuáles son sus tres estados y transiciones?

> [EXAMEN] 2. ¿Cómo el Bulkhead aísla fallos entre servicios mediante pools de recursos separados, y en qué se diferencia del Circuit Breaker?

> [EXAMEN] 3. ¿Por qué un Retry sin backoff exponencial puede agravar un fallo en cascada, y cómo el jitter mejora la situación?

> [EXAMEN] 4. ¿Dónde se aplica Rate Limiting (gateway vs servicio) y qué diferencia hay entre los algoritmos Token Bucket y Sliding Window?

> [EXAMEN] 5. ¿Cuál es el orden correcto de los aspectos de Resilience4j cuando se aplican múltiples anotaciones sobre un mismo método?

---

← [13.6 API Composition y Aggregator](sc-patrones-api-composition-aggregator.md) | [Índice](README.md) | [13.8 Observabilidad distribuida](sc-patrones-observabilidad-distribuida.md) →
