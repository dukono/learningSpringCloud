# 5.1 Circuit Breaker: estados, transiciones y modelo de evaluación

← [4.13 Testing](sc-feign-testing.md) | [Índice](README.md) | [5.2 Setup: dependencias y autoconfiguración de Resilience4j](sc-circuitbreaker-setup.md) →

## Introducción

En una arquitectura de microservicios, una llamada a un servicio externo que empieza a fallar de forma sostenida no solo afecta al cliente que la origina: si cada thread del pool queda bloqueado esperando respuesta de un servicio caído, toda la aplicación acaba sin threads disponibles y el fallo se propaga en cascada hacia los servicios que dependen de ella. Este patrón se denomina **cascading failure** y es uno de los modos de fallo más destructivos en sistemas distribuidos.

El Circuit Breaker resuelve este problema exactamente igual que el fusible eléctrico que le da nombre: cuando detecta que un circuito falla repetidamente, lo abre para interrumpir el flujo, proteger el sistema y dar tiempo al servicio remoto a recuperarse. Sin este patrón, la única alternativa es confiar en timeouts TCP (a menudo demasiado largos para ser útiles) o en reintentos que agravan la sobrecarga del servicio ya saturado.

> [PREREQUISITO] Este fichero describe el modelo conceptual del Circuit Breaker. La configuración de propiedades YAML se desarrolla en 5.4, y la API programática en 5.3.

## Representación visual

El Circuit Breaker de Resilience4j funciona como una máquina de estados con tres estados canónicos. Cada transición está gobernada por métricas que se evalúan sobre una ventana deslizante de llamadas recientes.

```
                         failureRate >= threshold
                    ┌──────────────────────────────┐
                    │      o slowCallRate >= threshold
                    ▼                              │
 ┌─────────────┐  OPEN  ┌──────────────────────────┘
 │             │◄───────┤
 │   CLOSED    │        │  waitDurationInOpenState vence
 │             │        │  (o automaticTransition=true)
 │  Evalúa     │        │
 │  todas las  │        ▼
 │  llamadas   │   HALF_OPEN
 │             │        │
 │             │        │  permittedNumberOfCalls
 │             │        │  llamadas de prueba
 └─────────────┘        │
        ▲               │  Si pasan → CLOSED
        └───────────────┘  Si fallan → OPEN

Estado CLOSED:  todas las llamadas pasan; la ventana acumula resultados
Estado OPEN:    todas las llamadas son rechazadas inmediatamente (CallNotPermittedException)
Estado HALF_OPEN: solo N llamadas de prueba pasan; el resto son rechazadas
```

El siguiente diagrama muestra la ventana deslizante y los dos modos de evaluación:

```
COUNT_BASED (slidingWindowSize = 10):
  Llamada: [1][2][3][4][5][6][7][8][9][10][11]...
                              └────────────────┘
                              ventana fija de 10 llamadas

TIME_BASED (slidingWindowSize = 10, unidad = segundos):
  t=0s ─────────────────────────────────────── t=10s
  └──────────── ventana de 10 segundos ────────┘
  (cada segundo descarta las llamadas del segundo más antiguo)
```

| Parámetro de evaluación       | COUNT_BASED                 | TIME_BASED                         |
|-------------------------------|-----------------------------|------------------------------------|
| Unidad de la ventana          | número de llamadas          | segundos                           |
| Cuándo empieza a evaluar      | tras `minimumNumberOfCalls` | tras `minimumNumberOfCalls`        |
| Precisión temporal            | baja (depende del tráfico)  | alta (descarta por tiempo)         |
| Caso de uso                   | alto throughput uniforme    | tráfico variable o bajo volumen    |

## Ejemplo central

El siguiente fragmento configura un Circuit Breaker con ventana basada en tiempo para proteger la llamada a un servicio de inventario. La aplicación es un microservicio Spring Boot 4.0.x que llama a un servicio externo vía `RestClient`.

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
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        sliding-window-type: TIME_BASED
        sliding-window-size: 10            # 10 segundos
        minimum-number-of-calls: 5
        failure-rate-threshold: 50         # 50% de fallos → OPEN
        slow-call-rate-threshold: 80       # 80% de llamadas lentas → OPEN
        slow-call-duration-threshold: 2s
        wait-duration-in-open-state: 30s
        automatic-transition-from-open-to-half-open-enabled: true
        permitted-number-of-calls-in-half-open-state: 3
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.shop.exception.ProductNotFoundException

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
// InventoryService.java
package com.example.shop.inventory;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import java.util.List;

@Service
public class InventoryService {

    private final RestClient restClient;

    public InventoryService(RestClient.Builder builder) {
        this.restClient = builder
                .baseUrl("http://inventory-service")
                .build();
    }

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackStock")
    public List<StockItem> getStockByProduct(String productId) {
        return restClient.get()
                .uri("/api/stock/{id}", productId)
                .retrieve()
                .body(new ParameterizedTypeReference<List<StockItem>>() {});
    }

    // El método fallback debe tener la misma firma más un parámetro Throwable
    private List<StockItem> fallbackStock(String productId, Throwable ex) {
        // Respuesta degradada: lista vacía indica "sin stock conocido"
        return List.of();
    }
}
```

```java
// StockItem.java (record para la respuesta deserializada)
package com.example.shop.inventory;

public record StockItem(String sku, int quantity, String warehouseId) {}
```

## Tabla de elementos clave

Los parámetros que gobiernan el comportamiento del Circuit Breaker en Resilience4j son los siguientes. Todos se declaran bajo el prefijo `resilience4j.circuitbreaker.instances.[name]`.

| Parámetro                                  | Tipo / Valores                  | Default       | Descripción                                                                 |
|--------------------------------------------|---------------------------------|---------------|-----------------------------------------------------------------------------|
| `sliding-window-type`                      | `COUNT_BASED`, `TIME_BASED`     | `COUNT_BASED` | Tipo de ventana para evaluar métricas                                       |
| `sliding-window-size`                      | entero positivo                 | `100`         | Tamaño de la ventana (llamadas o segundos según tipo)                       |
| `minimum-number-of-calls`                  | entero positivo                 | `100`         | Llamadas mínimas antes de evaluar umbrales                                  |
| `failure-rate-threshold`                   | 0.0 – 100.0 (%)                 | `50`          | Porcentaje de fallos que abre el circuito                                   |
| `slow-call-rate-threshold`                 | 0.0 – 100.0 (%)                 | `100`         | Porcentaje de llamadas lentas que abre el circuito                          |
| `slow-call-duration-threshold`             | duración (p.ej. `2s`)           | `60s`         | Duración máxima antes de considerar una llamada lenta                       |
| `wait-duration-in-open-state`              | duración (p.ej. `30s`)          | `60s`         | Tiempo que el circuito permanece OPEN antes de pasar a HALF_OPEN            |
| `automatic-transition-from-open-to-half-open-enabled` | boolean          | `false`       | Si `true`, transición automática OPEN → HALF_OPEN sin necesidad de llamada  |
| `permitted-number-of-calls-in-half-open-state` | entero positivo             | `10`          | Llamadas de prueba permitidas en estado HALF_OPEN                           |
| `max-wait-duration-in-half-open-state`     | duración (p.ej. `0`)            | `0`           | Tiempo máximo en HALF_OPEN; `0` = esperar indefinidamente                  |
| `record-exceptions`                        | lista de clases                 | `[]` (todas)  | Excepciones que cuentan como fallo                                          |
| `ignore-exceptions`                        | lista de clases                 | `[]`          | Excepciones que no cuentan (ni como fallo ni como éxito)                    |
| `base-config`                              | nombre de config base           | —             | Hereda propiedades de una configuración compartida en `configs.[name]`      |

## Buenas y malas prácticas

**Hacer:**

- Ajustar `sliding-window-size` al volumen real de tráfico. Con `COUNT_BASED` y tráfico bajo (p.ej. 2 llamadas/min), un valor de 100 hace que el circuito tarde 50 minutos en evaluar, inútil en producción. Usar `TIME_BASED` con una ventana de 30–60 segundos para tráfico intermitente.
- Configurar `minimum-number-of-calls` entre 5 y 20 según el throughput. Si no se configura, el default de 100 hace que el circuito nunca abra en servicios de bajo volumen.
- Diferenciar `record-exceptions` e `ignore-exceptions`. Las excepciones de negocio (404, validación) no deben abrir el circuito; solo los fallos de infraestructura (5xx, timeout, `IOException`) deben registrarse como fallos.
- Activar `automatic-transition-from-open-to-half-open-enabled: true` en producción para evitar que un circuito permanezca OPEN indefinidamente si el tráfico es bajo y ninguna llamada lo "despierta".
- Exponer `/actuator/circuitbreakers` y configurar `management.health.circuitbreakers.enabled: true` para visibilidad en dashboards y alertas de Kubernetes readiness probe.

**Evitar:**

- No usar el valor default de `failure-rate-threshold: 50` sin revisión. En algunos servicios un 20% de fallos ya es crítico; en otros un 60% puede ser tolerable durante despliegues. Calibrar sobre datos reales.
- No mezclar `record-exceptions` con `ignore-exceptions` para la misma excepción; si una excepción aparece en ambas listas, `ignore-exceptions` tiene precedencia y la excepción se ignora silenciosamente.
- Evitar `slow-call-duration-threshold` muy bajo (p.ej. `100ms`) en servicios con latencia variable de base. El circuito se abrirá por llamadas legítimas lentas y no por fallos reales, generando falsos positivos.
- No asumir que `waitDurationInOpenState` es el único mecanismo de recuperación. Si el servicio remoto tarda más que este valor en recuperarse y las llamadas de HALF_OPEN siguen fallando, el circuito volverá a OPEN indefinidamente; diseñar el fallback para resistir este ciclo.

## Comparación: COUNT_BASED vs TIME_BASED

La elección del tipo de ventana tiene consecuencias operativas importantes. Esta tabla ayuda a tomar la decisión correcta según el perfil de tráfico del servicio.

| Criterio                          | COUNT_BASED                              | TIME_BASED                                      |
|-----------------------------------|------------------------------------------|-------------------------------------------------|
| Sensibilidad temporal             | Baja: 100 fallos puede significar horas  | Alta: refleja lo ocurrido en los últimos N seg  |
| Uso de memoria                    | Fijo (N slots circulares)                | Variable (depende del tráfico)                  |
| Tráfico bajo                      | Inadecuado: umbral nunca se alcanza      | Correcto: evalúa lo que hubo en la ventana      |
| Tráfico alto y uniforme           | Correcto: evaluación rápida              | Puede consumir más memoria                      |
| Recomendación para microservicios | Solo si el throughput supera 10 req/s    | Preferido en general                            |

---

← [4.13 Testing](sc-feign-testing.md) | [Índice](README.md) | [5.2 Setup: dependencias y autoconfiguración de Resilience4j](sc-circuitbreaker-setup.md) →
