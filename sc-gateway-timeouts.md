# 3.7 Timeouts y configuración del cliente HTTP Netty

← [3.6 Integración con Service Discovery y Load Balancer](sc-gateway-discovery.md) | [Índice](README.md) | [3.8 CORS y TLS/HTTPS en Gateway](sc-gateway-cors-tls.md) →

---

## Introducción

Un gateway sin timeouts correctamente configurados se convierte en un amplificador de latencia: una petición a un backend lento puede mantener una conexión Netty ocupada durante minutos, agotando el pool de conexiones y produciendo timeouts en cascada para el resto de usuarios. La configuración del cliente HTTP de Gateway controla tres niveles de latencia: el tiempo para establecer la conexión TCP (`connect-timeout`), el tiempo para recibir el primer byte de respuesta (`response-timeout`), y el pool de conexiones disponibles para el proxy. Adicionalmente, `codec.max-in-memory-size` es el parámetro crítico que determina si `ModifyRequestBody` y `ModifyResponseBody` funcionan sin errores en producción. El desarrollador necesita configurar timeouts y el pool de conexiones del cliente HTTP de Gateway para controlar la latencia y el uso de recursos en producción.

> [ADVERTENCIA] Spring Cloud 2025.1.1 (Oakwood) puede haber introducido nuevas propiedades de `httpclient` respecto a versiones anteriores. Verificar contra las Release Notes oficiales antes de publicar en producción.

## Diagrama: parámetros del cliente HTTP y su posición en el ciclo de vida de la conexión

El siguiente diagrama muestra en qué momento del ciclo TCP/HTTP actúa cada parámetro.

```
Cliente → Gateway → Backend

   Gateway (Reactor Netty HttpClient)
   │
   ├── connect-timeout (ms)
   │   └── Tiempo máximo para establecer conexión TCP con el backend.
   │       Expira → ConnectTimeoutException → 504 Gateway Timeout
   │
   ├── [TCP handshake completado]
   │
   ├── response-timeout (Duration: Xs / XmXs)
   │   └── Tiempo máximo desde que se envía el request hasta recibir
   │       el primer byte del response del backend.
   │       Expira → ReadTimeoutException → 504 Gateway Timeout
   │
   ├── [Response recibido]
   │
   └── codec.max-in-memory-size (bytes)
       └── Límite del buffer en memoria para leer el body completo.
           Aplica a: ModifyRequestBody, ModifyResponseBody.
           Supera el límite → DataBufferLimitException → 500

   Pool de conexiones:
   ├── pool.type: ELASTIC (ilimitado) / FIXED (acotado) / DISABLED (sin pool)
   ├── pool.max-connections: máximo de conexiones simultáneas (solo FIXED)
   └── pool.acquire-timeout: tiempo máximo esperando conexión libre del pool
       Expira → PoolAcquireTimeoutException → 503
```

## Ejemplo central

El siguiente ejemplo muestra la configuración completa del cliente HTTP con timeouts globales y overrides por ruta.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      # ── Timeouts globales del cliente HTTP ─────────────────────────
      httpclient:
        connect-timeout: 1000           # ms; tiempo para establecer conexión TCP
        response-timeout: 5s            # Duration; tiempo hasta primer byte de response
                                        # Acepta: 5s, PT5S, 5000ms

        # ── Pool de conexiones Reactor Netty ────────────────────────
        pool:
          type: FIXED                   # ELASTIC (default), FIXED, DISABLED
          max-connections: 500          # solo FIXED; conexiones máximas al mismo backend
          acquire-timeout: 2000         # ms; tiempo esperando conexión libre del pool
          max-idle-time: 30s            # tiempo máximo que una conexión permanece ociosa
          max-life-time: 120s           # tiempo de vida máximo de una conexión

        # ── Límite de buffer en memoria ─────────────────────────────
        codec:
          max-in-memory-size: 10MB      # default: 256KB — CRÍTICO para ModifyRequestBody/ModifyResponseBody

      routes:
        # Ruta con timeout estándar (usa los globales)
        - id: catalog-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - StripPrefix=1

        # Ruta con timeout override por metadatos (override de los globales)
        - id: report-route
          uri: lb://report-service
          predicates:
            - Path=/api/reports/**
          filters:
            - StripPrefix=1
          metadata:
            # Override de connect-timeout para esta ruta (en ms)
            connect-timeout: 2000
            # Override de response-timeout para esta ruta (en ms)
            response-timeout: 30000     # reports pueden tardar hasta 30s

        # Ruta con modificación de body (requiere codec.max-in-memory-size adecuado)
        - id: transform-route
          uri: lb://transform-service
          predicates:
            - Path=/api/transform/**
          filters:
            - name: ModifyRequestBody
              # ModifyRequestBody con payloads de hasta 10MB necesita
              # que codec.max-in-memory-size >= 10MB
```

### Configuración programática para casos avanzados

```java
package com.example.gateway;

import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import org.springframework.cloud.gateway.config.HttpClientCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.netty.http.client.HttpClient;
import reactor.netty.resources.ConnectionProvider;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Configuration
public class NettyHttpClientConfig {

    /**
     * HttpClientCustomizer permite personalizar el HttpClient de Reactor Netty
     * más allá de lo que ofrecen las propiedades spring.cloud.gateway.httpclient.*.
     *
     * Útil para: compresión, proxy HTTP, configuración avanzada de TCP keep-alive,
     * o pools con nombres específicos para diferentes backends.
     */
    @Bean
    public HttpClientCustomizer customHttpClientCustomizer() {
        return httpClient -> httpClient
            // Timeout de conexión TCP a nivel Netty (alternativo a connect-timeout de YAML)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000)
            // Handlers de timeout a nivel de canal
            .doOnConnected(connection ->
                connection
                    .addHandlerLast(new ReadTimeoutHandler(5, TimeUnit.SECONDS))
                    .addHandlerLast(new WriteTimeoutHandler(5, TimeUnit.SECONDS)))
            // Habilitar compresión HTTP (Accept-Encoding: gzip, deflate)
            .compress(true)
            // Seguir redirects del backend automáticamente (default: false)
            .followRedirect(false);
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge todos los parámetros del cliente HTTP con sus valores recomendados para producción.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `httpclient.connect-timeout` | `int` (ms) | `45000` | Tiempo máximo para establecer conexión TCP |
| `httpclient.response-timeout` | `Duration` | sin límite | Tiempo hasta primer byte de respuesta del backend |
| `httpclient.pool.type` | `ELASTIC\|FIXED\|DISABLED` | `ELASTIC` | Tipo de pool de conexiones |
| `httpclient.pool.max-connections` | `int` | 2 * CPUs | Máximo de conexiones simultáneas (solo FIXED) |
| `httpclient.pool.acquire-timeout` | `int` (ms) | `45000` | Tiempo esperando conexión libre del pool |
| `httpclient.pool.max-idle-time` | `Duration` | sin límite | Tiempo máximo de conexión ociosa |
| `httpclient.pool.max-life-time` | `Duration` | sin límite | Tiempo de vida máximo de una conexión |
| `httpclient.codec.max-in-memory-size` | `DataSize` | `256KB` | Límite de buffer en memoria para lectura de body completo |
| `routes[*].metadata.connect-timeout` | `int` (ms) | global | Override del connect-timeout para una ruta específica |
| `routes[*].metadata.response-timeout` | `long` (ms) | global | Override del response-timeout para una ruta específica |

> [EXAMEN] La diferencia entre `connect-timeout` y `response-timeout`: `connect-timeout` mide el tiempo del handshake TCP inicial (Gateway no ha enviado ningún byte al backend aún); `response-timeout` mide el tiempo desde que Gateway envía el request completo hasta que recibe el primer byte de la respuesta HTTP. Son errores distintos: `ConnectTimeoutException` vs `ReadTimeoutException`.

> [ADVERTENCIA] El default de `codec.max-in-memory-size` es 256KB. Si se usa `ModifyRequestBody` o `ModifyResponseBody` con payloads superiores a 256KB sin cambiar este valor, se obtiene `DataBufferLimitException: Exceeded limit on max bytes to buffer: 262144` en tiempo de ejecución, no en el arranque.

## Buenas y malas prácticas

**Hacer:**
- Configurar `response-timeout` siempre en producción: sin él, una petición a un backend que nunca responde mantiene la conexión Netty abierta indefinidamente, agotando el pool.
- Usar `pool.type=FIXED` con `max-connections` acotado en producción: `ELASTIC` puede crear miles de conexiones ante un pico de tráfico, saturando los file descriptors del SO.
- Ajustar `codec.max-in-memory-size` específicamente cuando se usan `ModifyRequestBody`/`ModifyResponseBody`: el valor debe ser mayor que el payload máximo esperado para no tener errores en producción.
- Usar `metadata.response-timeout` por ruta para endpoints con SLAs distintos: un endpoint de búsqueda compleja puede tener 10s de timeout; un health check debería tener 500ms.

**Evitar:**
- Dejar `pool.acquire-timeout` en el valor por defecto (45s) con `pool.type=FIXED`: si el pool se agota (todos los backends lentos), los clientes esperarán 45s antes de obtener un error, produciendo timeouts en cascada.
- Configurar `pool.max-idle-time` sin `pool.max-life-time`: las conexiones muy longevas pueden recibir RST del servidor backend (ej: nginx cierra conexiones idle después de 75s por defecto), produciendo `Connection reset` en peticiones aparentemente correctas.
- Subir `codec.max-in-memory-size` a valores muy altos (ej: 1GB) sin calcular la memoria heap requerida: si hay 100 peticiones concurrentes con `ModifyResponseBody` y 10MB cada una, se necesita 1GB de heap solo para esos buffers.

---

← [3.6 Integración con Service Discovery y Load Balancer](sc-gateway-discovery.md) | [Índice](README.md) | [3.8 CORS y TLS/HTTPS en Gateway](sc-gateway-cors-tls.md) →
