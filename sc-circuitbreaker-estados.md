# 4.1 Circuit Breaker — Estados y Transiciones

← [3.10 Testing de OpenFeign](sc-feign-testing.md) | [Índice](README.md) | [4.2 Configuración completa](sc-circuitbreaker-configuracion.md) →

---

## Introducción

El patrón Circuit Breaker resuelve el problema de la propagación de fallos en cascada entre microservicios: cuando un servicio downstream deja de responder o responde con errores, todas las llamadas entrantes siguen bloqueándose esperando timeouts, consumiendo threads y degradando el sistema completo. El Circuit Breaker existe para detectar ese fallo de forma rápida y cortarlo, devolviendo una respuesta de fallback inmediata mientras el downstream se recupera. Se necesita cada vez que un microservicio llama a otro servicio externo o a una base de datos que puede fallar.

> [CONCEPTO] Resilience4j es la implementación de referencia para Spring Cloud Circuit Breaker desde Spring Boot 2.4+. Reemplaza Hystrix [LEGACY] que quedó en modo mantenimiento.

## Diagrama de estados y transiciones

El Circuit Breaker es una máquina de estados finita con tres estados principales. CLOSED es el estado normal en que todas las llamadas pasan; el CB evalúa las llamadas en una sliding window y si la tasa de fallos supera `failureRateThreshold` transiciona a OPEN. En OPEN, todas las llamadas se rechazan inmediatamente con `CallNotPermittedException` sin llegar al downstream. Tras `waitDurationInOpenState`, transiciona a HALF_OPEN donde permite exactamente `permittedNumberOfCallsInHalfOpenState` llamadas de prueba; si esas llamadas tienen tasa de fallos aceptable vuelve a CLOSED, si no regresa a OPEN.

```
                    failure rate >= threshold
  CLOSED ──────────────────────────────────────────► OPEN
    ▲                                                   │
    │                                               waitDuration
    │                                               expires
    │                                                   │
    │   all probe calls succeed                         ▼
  CLOSED ◄─────────────────────────────────────── HALF_OPEN
                                                        │
                                                        │ probe calls fail
                                                        ▼
                                                      OPEN
```

Existe un cuarto estado, DISABLED (métricas y circuit breaker desactivados) y FORCED_OPEN (siempre abierto), accesibles solo de forma programática mediante la API del registro. La transición automática de OPEN a HALF_OPEN también puede configurarse con `automaticTransitionFromOpenToHalfOpenEnabled=true` sin necesidad de que llegue una llamada.

## Sliding Window: COUNT_BASED vs TIME_BASED

La sliding window determina qué conjunto de llamadas se usa para calcular la tasa de fallos. La elección entre COUNT_BASED y TIME_BASED tiene implicaciones importantes en entornos de baja y alta carga.

| Aspecto | COUNT_BASED | TIME_BASED |
|---------|-------------|------------|
| Unidad de window | Últimas N llamadas | Últimos N segundos |
| `slidingWindowSize` | Número de llamadas (ej: 100) | Duración en segundos (ej: 10) |
| Comportamiento bajo carga baja | Evalúa siempre N llamadas fijas | Puede evaluar pocas llamadas en períodos tranquilos |
| `minimumNumberOfCalls` | Mínimo necesario para evaluar | Mínimo necesario para evaluar |
| Uso recomendado | Servicios con carga constante | Servicios con carga variable o picos |

> [CONCEPTO] `minimumNumberOfCalls` es independiente del tipo de window. El CB nunca evalúa la tasa de fallos hasta que se hayan registrado al menos `minimumNumberOfCalls` llamadas dentro de la window. Valor por defecto: 100.

## Ejemplo central

El ejemplo muestra la configuración mínima de un Circuit Breaker con TIME_BASED window y su uso con anotación, incluyendo el fallback:

```java
package com.example.resilience;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ProductService {

    private final RestTemplate restTemplate;

    public ProductService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    // El nombre "productService" debe coincidir con la clave en application.yml
    // El fallbackMethod recibe los mismos parámetros + Throwable al final
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
    public String getProduct(Long id) {
        return restTemplate.getForObject(
            "http://product-service/products/{id}", String.class, id);
    }

    // Firma obligatoria: mismos parámetros que el método protegido + Throwable
    public String getProductFallback(Long id, Throwable ex) {
        return "Product " + id + " temporarily unavailable: " + ex.getMessage();
    }
}
```

Configuración YAML correspondiente:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        sliding-window-type: TIME_BASED
        sliding-window-size: 10            # 10 segundos
        minimum-number-of-calls: 5
        failure-rate-threshold: 50         # % de fallos para abrir
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
```

## Tabla de elementos clave

La siguiente tabla resume las métricas que el CB evalúa en cada estado para decidir las transiciones:

| Estado | Métrica evaluada | Acción si umbral superado |
|--------|-----------------|--------------------------|
| CLOSED | `failureRateThreshold` sobre sliding window | Transiciona a OPEN |
| CLOSED | `slowCallRateThreshold` (% de llamadas lentas) | Transiciona a OPEN |
| OPEN | — (todas rechazadas) | Espera `waitDurationInOpenState` |
| HALF_OPEN | `failureRateThreshold` sobre N pruebas | Si falla → OPEN; si pasa → CLOSED |

> [EXAMEN] Una llamada lenta (mayor que `slowCallDurationThreshold`) **cuenta como éxito** para `failureRateThreshold`, pero cuenta como fallo para `slowCallRateThreshold`. El CB puede transicionar a OPEN si **cualquiera** de los dos umbrales se supera.

## Buenas y malas prácticas

**Buenas prácticas:**
- Definir `minimumNumberOfCalls` bajo (5-10) en tests para acelerar las transiciones en entornos de prueba.
- Usar TIME_BASED en servicios con carga variable para evitar que la window tarde demasiado en llenarse.
- Siempre implementar un `fallbackMethod` que devuelva datos cacheados o un valor degradado.
- Registrar un listener en `onStateTransition` para alertar cuando el CB abre.

**Malas prácticas:**
- Confiar en valores por defecto (`minimumNumberOfCalls=100`) en producción sin ajustar al tráfico real.
- Usar `waitDurationInOpenState` muy corto (< 5s) en servicios con tiempos de recuperación lentos, causando que el CB oscile entre OPEN y HALF_OPEN.
- Lanzar excepciones en el `fallbackMethod` — esto causa una `FallbackExecutionException` no gestionada.

## Comparación: COUNT_BASED vs TIME_BASED en la práctica

COUNT_BASED garantiza que siempre se evalúan exactamente N llamadas, lo que es predecible pero puede significar que en servicios de muy baja carga el CB tarde minutos en detectar un problema (si `slidingWindowSize=100` y llegan 2 llamadas/minuto). TIME_BASED es más reactivo en baja carga pero en alta carga puede evaluar miles de llamadas, lo que consume más memoria. Para la mayoría de microservicios en producción con carga constante, COUNT_BASED con `slidingWindowSize=50` y `minimumNumberOfCalls=10` es un punto de partida sólido.

## Verificación y práctica

> [EXAMEN] 1. Un CircuitBreaker tiene `failureRateThreshold=50`, `minimumNumberOfCalls=10` y `slidingWindowSize=20` (COUNT_BASED). Se han ejecutado 8 llamadas, 6 de ellas han fallado. ¿Está el CB en estado OPEN?

> [EXAMEN] 2. ¿Qué excepción lanza Resilience4j cuando un CircuitBreaker está en estado OPEN y llega una nueva llamada?

> [EXAMEN] 3. Con `automaticTransitionFromOpenToHalfOpenEnabled=false` (valor por defecto), ¿cuándo transiciona el CB de OPEN a HALF_OPEN?

> [EXAMEN] 4. ¿Cuál es la diferencia entre `failureRateThreshold` y `slowCallRateThreshold`? ¿Puede el CB abrirse aunque no haya ningún error HTTP?

> [EXAMEN] 5. En estado HALF_OPEN con `permittedNumberOfCallsInHalfOpenState=3`, si la primera llamada falla y las dos siguientes tienen éxito, ¿a qué estado transiciona el CB?

---

← [3.10 Testing de OpenFeign](sc-feign-testing.md) | [Índice](README.md) | [4.2 Configuración completa](sc-circuitbreaker-configuracion.md) →
