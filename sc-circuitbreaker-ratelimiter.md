# 5.8 RateLimiter: control de tasa de llamadas con Resilience4j

← [5.7 Bulkhead: aislamiento de recursos con SemaphoreBulkhead y ThreadPoolBulkhead](sc-circuitbreaker-bulkhead.md) | [Índice](README.md) | [5.9 TimeLimiter: acotación de tiempos de ejecución asíncronos](sc-circuitbreaker-timelimiter.md) →

## Introducción

El RateLimiter de Resilience4j resuelve un problema diferente al del Circuit Breaker y el Bulkhead: mientras estos reaccionan a fallos que ya están ocurriendo, el RateLimiter actúa preventivamente limitando cuántas llamadas pueden hacerse en un período de tiempo dado. Su caso de uso principal es proteger servicios downstream de sobrecargas por volumen de peticiones, ya sea para cumplir un SLA contractual de un servicio de terceros (p.ej. APIs externas con límite de 100 req/s), o para proteger un servicio propio frente a un cliente que envía demasiadas peticiones.

Es importante diferenciar este componente del Rate Limiting en Spring Cloud Gateway. El Gateway limita tasa de llamadas entrantes al borde del sistema (protege los servicios propios de clientes externos); el RateLimiter de Resilience4j limita las llamadas salientes desde un microservicio hacia sus dependencias (protege servicios externos o internos de la sobrecarga generada por este servicio específico).

> [CONCEPTO] El algoritmo de Resilience4j es **AtomicRateLimiter**: un mecanismo de token bucket que rellena `limitForPeriod` permisos cada `limitRefreshPeriod`. Las llamadas que no pueden obtener permiso esperan hasta `timeoutDuration`; si el timeout vence, se lanza `RequestNotPermitted`.

## Representación visual

El modelo del token bucket del AtomicRateLimiter de Resilience4j es el siguiente:

```
Cada limitRefreshPeriod (p.ej. 1s):
────────────────────────────────────────────────────
  Bucket se rellena a limitForPeriod permisos (p.ej. 10)

  Llamada 1 ──→ obtiene permiso ──→ ejecuta
  Llamada 2 ──→ obtiene permiso ──→ ejecuta
  ...
  Llamada 10 ──→ obtiene permiso ──→ ejecuta
  Llamada 11 ──→ espera timeoutDuration
                  │
                  ├─ permiso disponible antes del timeout ──→ ejecuta
                  └─ timeout agotado ──→ RequestNotPermitted

  Al final de limitRefreshPeriod: bucket se rellena de nuevo
```

| Componente               | Resilience4j RateLimiter                       | Spring Cloud Gateway RateLimiter               |
|--------------------------|------------------------------------------------|------------------------------------------------|
| Dirección de control     | Llamadas salientes (cliente protege la API)    | Llamadas entrantes (gateway protege el servicio)|
| Granularidad             | Por instancia de servicio                      | Por ruta del gateway                            |
| Almacenamiento del estado| En memoria (por instancia JVM)                 | Redis (distribuido, compartido entre instancias)|
| Identificación de cliente| Por nombre de instancia CB                     | Por clave de resolución (IP, usuario, etc.)     |
| Uso típico               | Cumplir SLA de API externa / proteger interno  | Throttling de APIs públicas expuestas           |

## Ejemplo central

Servicio que llama a una API de geolocalización de terceros con límite contractual de 100 llamadas por segundo, protegido con RateLimiter para no superar ese límite desde ninguna instancia.

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
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
resilience4j:
  ratelimiter:
    instances:
      geoApi:
        # Permite 80 llamadas por segundo (margen bajo el límite de 100 del contrato)
        limit-for-period: 80
        limit-refresh-period: 1s
        timeout-duration: 500ms         # espera hasta 500ms antes de rechazar
      internalReportService:
        limit-for-period: 5
        limit-refresh-period: 1s
        timeout-duration: 0             # rechaza inmediatamente si no hay permiso
    
    # Habilitar métricas de RateLimiter en health
    configs:
      default:
        register-health-indicator: true

management:
  endpoints:
    web:
      exposure:
        include: health,ratelimiters,ratelimiterevents
  health:
    ratelimiters:
      enabled: true
```

```java
// GeoLocationService.java — llamada a API externa con límite de tasa
package com.example.shipping.service;

import com.example.shipping.exception.GeoApiOverloadedException;
import com.example.shipping.model.GeoLocation;
import io.github.resilience4j.ratelimiter.RequestNotPermitted;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class GeoLocationService {

    private final RestClient restClient;

    public GeoLocationService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("https://geo-api.example.com").build();
    }

    @RateLimiter(name = "geoApi", fallbackMethod = "fallbackGeoLocation")
    public GeoLocation getLocation(String address) {
        return restClient.get()
                .uri("/v1/geocode?address={addr}", address)
                .retrieve()
                .body(GeoLocation.class);
    }

    /**
     * Se invoca cuando RequestNotPermitted se lanza (límite de tasa agotado).
     * También cubre otras excepciones del servicio remoto.
     */
    private GeoLocation fallbackGeoLocation(String address, Throwable ex) {
        if (ex instanceof RequestNotPermitted) {
            // El rate limiter rechazó la llamada: respuesta degradada
            throw new GeoApiOverloadedException(
                    "Límite de tasa de la API geo alcanzado para: " + address);
        }
        // Otro error: devolver localización desconocida
        return new GeoLocation(address, 0.0, 0.0, false);
    }
}
```

```java
// ReportService.java — control de tasa para generación de informes intensivos
package com.example.shipping.service;

import com.example.shipping.model.ReportResult;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class ReportService {

    private final RestClient restClient;

    public ReportService(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://report-service").build();
    }

    @RateLimiter(name = "internalReportService", fallbackMethod = "fallbackReport")
    public ReportResult generateShippingReport(String period) {
        return restClient.post()
                .uri("/api/reports/shipping")
                .body(new ReportRequest(period))
                .retrieve()
                .body(ReportResult.class);
    }

    private ReportResult fallbackReport(String period, Throwable ex) {
        return new ReportResult(period, "REJECTED",
                "Generación de informe rechazada por límite de tasa", null);
    }
}
```

```java
// Modelos y excepciones
package com.example.shipping.model;

public record GeoLocation(String address, double latitude, double longitude, boolean found) {}
public record ReportRequest(String period) {}
public record ReportResult(String period, String status, String message, byte[] data) {}
```

```java
package com.example.shipping.exception;

public class GeoApiOverloadedException extends RuntimeException {
    public GeoApiOverloadedException(String message) { super(message); }
}
```

## Tabla de elementos clave

Propiedades del namespace `resilience4j.ratelimiter.instances.[name]`.

| Propiedad                       | Tipo     | Default | Descripción                                                                        |
|---------------------------------|----------|---------|------------------------------------------------------------------------------------|
| `limit-for-period`              | int      | `50`    | Número de permisos disponibles por período                                         |
| `limit-refresh-period`          | Duration | `500ns` | Período de refresco del bucket de permisos                                         |
| `timeout-duration`              | Duration | `5s`    | Tiempo máximo de espera para obtener permiso; `0` = rechazar inmediatamente        |
| `register-health-indicator`     | boolean  | `false` | Registra indicador en `/actuator/health`                                           |
| `subscribe-for-events`          | boolean  | `false` | Activa publicación de eventos de RateLimiter                                       |
| `allow-health-indicator-to-fail`| boolean  | `true`  | Si `false`, un RateLimiter exhausto pone el health en DOWN                         |

## Buenas y malas prácticas

**Hacer:**

- Dimensionar `limit-for-period` por debajo del límite real del contrato externo con un margen del 10-20%, especialmente si la aplicación corre en múltiples instancias. El RateLimiter de Resilience4j es por JVM, no distribuido; con 3 instancias y `limitForPeriod: 100`, se generan 300 llamadas reales por período.
- Combinar RateLimiter con Circuit Breaker para servicios externos: el RateLimiter evita sobrecargar el servicio cuando funciona bien, y el Circuit Breaker reacciona cuando empieza a fallar.
- Usar `timeout-duration: 0` para APIs donde la latencia es crítica y no es aceptable que el cliente espere permiso. En APIs de baja latencia, esperar hasta 5 segundos (el default) puede ser inaceptable.
- Exponer métricas de `resilience4j.ratelimiter.available.permissions` en Grafana para detectar cuándo el límite de tasa está siendo continuamente agotado (señal de que el `limit-for-period` es demasiado bajo para la carga real).

**Evitar:**

- No usar `RateLimiter` de Resilience4j como mecanismo de limitación de tasa para múltiples instancias distribuidas; el estado es local a la JVM. Para rate limiting distribuido, usar Spring Cloud Gateway con `RedisRateLimiter`.
- Evitar `limit-refresh-period` demasiado largo (p.ej. `60s` con `limitForPeriod: 100`). Esto permite 100 llamadas en el primer segundo del minuto y ninguna durante los 59 segundos restantes, creando picos de tráfico en lugar de distribuirlo.
- No confundir `RequestNotPermitted` (RateLimiter) con `CallNotPermittedException` (CircuitBreaker) en el código del fallback. Son excepciones distintas con significados distintos; un fallback que trata ambas igual puede enmascarar el estado real del sistema.

---

← [5.7 Bulkhead: aislamiento de recursos con SemaphoreBulkhead y ThreadPoolBulkhead](sc-circuitbreaker-bulkhead.md) | [Índice](README.md) | [5.9 TimeLimiter: acotación de tiempos de ejecución asíncronos](sc-circuitbreaker-timelimiter.md) →
