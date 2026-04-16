# 4.11 Cliente HTTP subyacente — Apache HttpClient 5, OkHttp, pool de conexiones y soporte reactivo

<- [4.10 Retryer](sc-feign-retryer.md) | [Índice](README.md) | [4.12 Herencia de interfaz y @SpringQueryMap](sc-feign-herencia.md) ->

---

## Introducción

Por defecto, Feign usa `HttpURLConnection` de Java como cliente HTTP subyacente. Esta implementación no tiene pool de conexiones: cada llamada abre una nueva conexión TCP, completa la petición y cierra la conexión. En un servicio que recibe 500 req/s y llama a tres microservicios externos, esto significa 1500 conexiones TCP por segundo, con el coste de handshake y establecimiento de cada una. En producción, este comportamiento produce latencias elevadas, saturación de puertos efímeros y timeouts inexplicables bajo carga.

Reemplazar `HttpURLConnection` por Apache HttpClient 5 (con pool de conexiones) u OkHttp es la optimización operativa más directa disponible en Feign. Basta con añadir la dependencia correspondiente: Spring Cloud autoconfigura el bean de cliente automáticamente cuando detecta la librería en el classpath.

El soporte reactivo (Feign con `WebClient` como transporte) es una opción experimental para escenarios donde el servicio consumidor es 100% reactivo y se quiere mantener el estilo declarativo de Feign sin bloquear threads del pool de Reactor. Su madurez en producción es limitada; para proyectos reactivos maduros, `WebClient` directamente es la elección más consolidada.

> [PREREQUISITO] Apache HttpClient 5 requiere la dependencia `feign-hc5`; OkHttp requiere `feign-okhttp`. Sin estas dependencias, Feign usa `HttpURLConnection` aunque se configure en YAML. La autoconfiguración solo aplica si la librería está en el classpath.

---

## Diagrama: comparativa de modelos de conexión

El siguiente diagrama muestra la diferencia entre `HttpURLConnection` y un cliente con pool de conexiones.

```
 HttpURLConnection (default — sin pool):
 ─────────────────────────────────────────
 petición 1 → [TCP connect] → [HTTP req/resp] → [TCP close]
 petición 2 → [TCP connect] → [HTTP req/resp] → [TCP close]
 petición 3 → [TCP connect] → [HTTP req/resp] → [TCP close]
 Cada petición: ~50-150ms de overhead de handshake TCP+TLS

 Apache HttpClient 5 / OkHttp (con pool de conexiones):
 ─────────────────────────────────────────────────────
 arranque → [TCP connect × N] → pool de N conexiones keep-alive
 petición 1 → toma conexión del pool → HTTP req/resp → devuelve al pool
 petición 2 → toma conexión del pool → HTTP req/resp → devuelve al pool
 petición 3 → toma conexión del pool → HTTP req/resp → devuelve al pool
 Overhead de handshake: ~0ms (conexión ya establecida)
```

---

## Ejemplo central

El siguiente ejemplo muestra la configuración de Apache HttpClient 5 y OkHttp con pool de conexiones personalizado.

**Dependencias Maven para Apache HttpClient 5:**

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-hc5</artifactId>
    <!-- versión gestionada por spring-cloud-dependencies BOM -->
</dependency>
```

**application.yml — configuración del pool de Apache HttpClient 5:**

```yaml
feign:
  httpclient:
    hc5:
      enabled: true                         # activa Apache HttpClient 5 (confirmación explícita)
    # Propiedades del pool de conexiones
    max-connections: 200                    # máximo de conexiones totales en el pool
    max-connections-per-route: 50           # máximo de conexiones por host:puerto
    connection-timer-repeat: 3000           # ms; intervalo de limpieza de conexiones expiradas
    time-to-live: 900                       # segundos; vida máxima de una conexión keep-alive
    time-to-live-unit: seconds
    follow-redirects: false                 # no seguir redirects automáticamente en Feign
    disable-ssl-validation: false           # nunca deshabilitar en producción
    ok-http:
      read-timeout: 5000                    # ms; afecta solo si se usa OkHttp
```

**Dependencias Maven para OkHttp:**

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

**application.yml — configuración del pool de OkHttp:**

```yaml
feign:
  okhttp:
    enabled: true                           # activa OkHttp
  httpclient:
    max-connections: 200
    max-connections-per-route: 50
```

**Configuración programática de OkHttp para control avanzado del pool:**

```java
package com.example.orderservice.config;

import feign.okhttp.OkHttpClient;
import okhttp3.ConnectionPool;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class OkHttpFeignConfig {

    /**
     * Pool de conexiones OkHttp con configuración explícita.
     * maxIdleConnections: conexiones keep-alive inactivas que se mantienen abiertas.
     * keepAliveDuration: tiempo máximo que una conexión idle permanece en el pool.
     */
    @Bean
    public OkHttpClient okHttpFeignClient() {
        ConnectionPool pool = new ConnectionPool(
                50,               // maxIdleConnections
                30,               // keepAliveDuration
                TimeUnit.SECONDS
        );

        okhttp3.OkHttpClient httpClient = new okhttp3.OkHttpClient.Builder()
                .connectionPool(pool)
                .connectTimeout(2, TimeUnit.SECONDS)
                .readTimeout(5, TimeUnit.SECONDS)
                .writeTimeout(5, TimeUnit.SECONDS)
                .followRedirects(false)
                .build();

        return new OkHttpClient(httpClient);
    }
}
```

**Soporte reactivo experimental — Feign con WebClient (solo para contextos reactivos):**

```xml
<!-- Solo en proyectos WebFlux (reactive); NO mezclar con Spring MVC -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```yaml
# Activar soporte reactivo de Feign (experimental en Spring Cloud 2025.1.1)
feign:
  reactive:
    enabled: true
```

```java
// Con reactive enabled, los métodos de la interfaz pueden devolver Mono<T> o Flux<T>
@FeignClient(name = "catalog-service", contextId = "catalogReactive")
public interface ReactiveCatalogClient {

    @GetMapping("/api/products/{productId}")
    Mono<ProductDto> getProduct(@PathVariable("productId") String productId);

    @GetMapping("/api/products")
    Flux<ProductDto> streamProducts();
}
```

> [ADVERTENCIA] `feign-okhttp` y `feign-hc5` son mutuamente excluyentes. Si ambas dependencias están en el classpath, la que Spring Cloud autoconfigure depende del orden de la autoconfiguración. Para evitar ambigüedad, añadir solo la dependencia del cliente HTTP elegido.

---

## Tabla de elementos clave

| Elemento | Default | Apache HttpClient 5 | OkHttp |
|---|---|---|---|
| Dependencia | — | `feign-hc5` | `feign-okhttp` |
| Activación | `HttpURLConnection` | `feign.httpclient.hc5.enabled=true` | `feign.okhttp.enabled=true` |
| Pool de conexiones | No | Sí (`feign.httpclient.max-connections`) | Sí (`ConnectionPool` programático) |
| HTTP/2 | No | Sí (con configuración adicional) | Sí (automático) |
| Soporte mTLS | Básico | Avanzado (`SSLContext`, `TrustStore`) | Avanzado (`SSLSocketFactory`) |
| Madurez en Spring Cloud | Alta | Alta | Alta |
| `feign.httpclient.max-connections` | — | 200 (configurable) | 200 (configurable) |
| `feign.httpclient.max-connections-per-route` | — | 50 (configurable) | 50 (configurable) |

---

## Buenas y malas prácticas

**Hacer:**
- Usar Apache HttpClient 5 u OkHttp en cualquier servicio que realice más de 10 llamadas Feign por segundo. El pool de conexiones elimina el overhead de TCP handshake y reduce la latencia p99 de forma significativa.
- Dimensionar el pool (`max-connections` y `max-connections-per-route`) según la concurrencia real del servicio, no los valores por defecto. Un servicio que atiende 100 threads concurrentes y todos pueden hacer llamadas Feign necesita un pool de al menos 100 conexiones por ruta.
- Activar `connection-timer-repeat` para limpiar periódicamente conexiones expiradas del pool en Apache HttpClient 5. Sin este mecanismo, conexiones con `Idle Timeout` del servidor (p. ej. un balanceador que cierra conexiones idle a los 30s) permanecen en el pool y producen errores `Connection reset by peer` en la primera petición tras el timeout del servidor.

**Evitar:**
- Poner `disable-ssl-validation: true` en entornos no productivos y olvidarlo antes del despliegue a producción. La validación TLS no es un capricho de seguridad: protege contra ataques MITM en redes de producción. Usar certificados autofirmados con un `TrustStore` personalizado en lugar de desactivar la validación.
- Mezclar Apache HttpClient 5 y OkHttp en el mismo proyecto. Si ambas dependencias están en el classpath y ambas están habilitadas en YAML, el comportamiento depende del orden de evaluación de las `@ConditionalOnClass`. El resultado puede ser inconsistente entre versiones de Spring Cloud.
- Usar el soporte reactivo de Feign (`feign.reactive.enabled=true`) en proyectos con Spring MVC bloqueante. La integración es experimental y mezclar modelos bloqueante y reactivo en el mismo servicio produce comportamientos difíciles de predecir en el pool de threads de Reactor.

---

## Comparación: clientes HTTP subyacentes

| Criterio | `HttpURLConnection` | Apache HttpClient 5 | OkHttp |
|---|---|---|---|
| Pool de conexiones | No | Sí | Sí |
| HTTP/2 | No | Sí | Sí |
| Configuración YAML | — | `feign.httpclient.*` | `feign.okhttp.*` |
| Config programática | No | `HttpClientBuilder` | `OkHttpClient.Builder` |
| mTLS | Básico | Avanzado | Avanzado |
| Latencia bajo carga | Alta (sin pool) | Baja | Baja |
| Adecuado para | Prototipos, tests | Producción (Java-first) | Producción (ecosistema Android/JVM) |

---

<- [4.10 Retryer](sc-feign-retryer.md) | [Índice](README.md) | [4.12 Herencia de interfaz y @SpringQueryMap](sc-feign-herencia.md) ->
