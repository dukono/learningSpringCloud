# 4.4 Configuración por propiedades YAML — timeouts, compresión y orden de precedencia

<- [4.3 Configuración por clase Java](sc-feign-configuracion-java.md) | [Índice](README.md) | [4.5 Codecs](sc-feign-codecs.md) ->

---

## Introducción

La configuración de Feign por propiedades YAML permite ajustar el comportamiento de cada cliente sin recompilar. A diferencia de la configuración por clase Java, las propiedades pueden cambiarse en un Config Server, en un ConfigMap de Kubernetes o mediante una variable de entorno, lo que hace posible afinar timeouts o niveles de log en producción sin un redeploy.

El namespace raíz es `feign.client.config`. Bajo él, la clave `default` se aplica a todos los clientes; cualquier otra clave se aplica al cliente cuyo `contextId` o `name` coincida. Esta granularidad permite que el cliente de pagos tenga un `readTimeout` de 10 segundos mientras que el cliente de catálogo usa 2 segundos, con una sola fuente de configuración centralizada.

> [ADVERTENCIA] Desde Spring Cloud 2021.0 (`Hoxton.SR12+`), las propiedades YAML tienen mayor precedencia que la configuración por clase `@Configuration` para los campos que ambas cubren. Este comportamiento está controlado por `feign.client.default-to-properties=true` (valor por defecto). Si la aplicación depende de que Java tenga precedencia, hay que establecer explícitamente `feign.client.default-to-properties=false`.

---

## Diagrama: orden de precedencia y resolución de configuración

El siguiente diagrama muestra en qué orden Feign resuelve la configuración para un cliente concreto cuando hay múltiples fuentes activas.

```
 Resolución de configuración para el cliente "payment-service"
 ──────────────────────────────────────────────────────────────

 1. FeignClientsConfiguration (defaults del framework)
       ↓ sobreescrito por
 2. defaultConfiguration de @EnableFeignClients (global Java)
       ↓ sobreescrito por
 3. configuration = PaymentFeignConfig.class (local Java del cliente)
       ↓ sobreescrito por (cuando default-to-properties=true)
 4. feign.client.config.default (global YAML)
       ↓ sobreescrito por
 5. feign.client.config.payment-service (local YAML del cliente)  ← máxima precedencia
```

---

## Ejemplo central

El siguiente `application.yml` muestra la configuración completa del namespace `feign.client.config`, incluyendo configuración global, configuración por cliente, timeouts y compresión.

```yaml
feign:
  client:
    # true = YAML tiene mayor precedencia que @Configuration Java (default desde 2021.0)
    default-to-properties: true

    config:
      # Configuración global: aplica a todos los clientes que no tengan su propia sección
      default:
        connect-timeout: 2000       # ms; tiempo máximo para establecer conexión TCP
        read-timeout: 5000          # ms; tiempo máximo esperando el primer byte de respuesta
        logger-level: BASIC         # NONE | BASIC | HEADERS | FULL
        error-decoder: com.example.orderservice.feign.GlobalErrorDecoder

      # Configuración para el cliente con name/contextId = "payment-service"
      payment-service:
        connect-timeout: 3000
        read-timeout: 10000         # Pagos pueden tardar más; timeout independiente
        logger-level: HEADERS       # Más detalle en el cliente crítico

      # Configuración para el cliente con contextId = "catalogRead"
      catalogRead:
        connect-timeout: 1000
        read-timeout: 2000          # Lectura de catálogo: SLA estricto
        logger-level: NONE          # Sin log en lectura de alto volumen

  # Compresión de peticiones y respuestas (afecta a todos los clientes)
  compression:
    request:
      enabled: true
      mime-types:
        - application/json
        - application/xml
        - text/xml
      min-request-size: 2048        # bytes; no comprimir payloads pequeños

    response:
      enabled: true                 # descompresión de respuestas gzip del servidor
```

Para verificar que la configuración se aplica correctamente, añadir en `application.yml` de test:

```yaml
# Test: forzar la URL del cliente para no depender de Eureka
feign:
  client:
    config:
      payment-service:
        url: http://localhost:${wiremock.server.port}
```

> [CONCEPTO] La clave bajo `feign.client.config` debe coincidir con el **`contextId`** del cliente si está declarado, o con el **`name`** si no hay `contextId`. Si ambos están declarados y la clave YAML usa el `name` en lugar del `contextId`, la configuración por cliente no se aplica.

---

## Tabla de elementos clave

Propiedades completas del namespace `feign.client.config.[clientName]` que un senior debe conocer.

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `connect-timeout` | `int` (ms) | 2000 | Tiempo máximo para establecer la conexión TCP |
| `read-timeout` | `int` (ms) | 5000 | Tiempo máximo esperando bytes de respuesta tras conectar |
| `logger-level` | `Logger.Level` | `NONE` | Nivel de log: `NONE`, `BASIC`, `HEADERS`, `FULL` |
| `error-decoder` | `Class` | `ErrorDecoder.Default` | Clase que implementa `ErrorDecoder` para este cliente |
| `request-interceptors` | `List<Class>` | `[]` | Interceptores adicionales al cliente (complementan, no reemplazan) |
| `decode404` | `boolean` | `false` | Si `true`, HTTP 404 va al Decoder en lugar de al ErrorDecoder |
| `encoder` | `Class` | `SpringEncoder` | Encoder personalizado para este cliente |
| `decoder` | `Class` | `SpringDecoder` | Decoder personalizado para este cliente |
| `retryer` | `Class` | `Retryer.NEVER_RETRY` | Política de reintentos para este cliente |
| `feign.client.default-to-properties` | `boolean` | `true` | YAML tiene mayor precedencia que `@Configuration` Java |
| `feign.compression.request.enabled` | `boolean` | `false` | Activa compresión gzip de peticiones salientes |
| `feign.compression.response.enabled` | `boolean` | `false` | Activa descompresión de respuestas gzip |
| `feign.compression.request.mime-types` | `List<String>` | `[text/xml, application/xml, application/json]` | MIME types a comprimir |
| `feign.compression.request.min-request-size` | `int` (bytes) | 2048 | Umbral mínimo para comprimir; payloads menores no se comprimen |

---

## Buenas y malas prácticas

**Hacer:**
- Definir siempre una sección `feign.client.config.default` con los valores base de `connect-timeout` y `read-timeout`. Sin ella, Feign usa los defaults del framework (2000ms / 5000ms) que pueden ser inadecuados para el SLA de la aplicación.
- Ajustar el `read-timeout` por cliente según el SLA del servicio remoto. Un timeout global único favorece al cliente más lento y perjudica a los rápidos: peticiones que deberían fallar en 1s esperan 5s innecesariamente, consumiendo threads del pool.
- Usar la clave `contextId` como nombre de sección YAML cuando el cliente tiene `contextId` declarado. Usar `name` en ese caso produce una sección que nunca se aplica, dando la falsa impresión de que el cliente tiene configuración específica.

**Evitar:**
- Activar `feign.compression.request.enabled=true` sin verificar que el servidor remoto acepta `Content-Encoding: gzip` en peticiones. Muchos servidores REST no soportan compresión de request y devuelven `415 Unsupported Media Type` o simplemente ignoran el cuerpo comprimido, produciendo errores difíciles de diagnosticar.
- Poner `logger-level: FULL` en `feign.client.config.default` en producción. Nivel `FULL` loggea el cuerpo completo de todas las peticiones y respuestas; en servicios de alto volumen puede saturar el sistema de logging y exponer datos sensibles (tokens, PII) en los logs.
- Confiar en que `read-timeout: 0` desactiva el timeout. El valor `0` en algunos clientes HTTP subyacentes (HttpURLConnection) significa timeout infinito, pero en Apache HttpClient 5 significa 0ms y causa fallos inmediatos. Usar un valor alto explícito si se quiere timeout permisivo.

---

## Comparación: timeouts en Feign vs timeouts en LoadBalancer vs timeouts en Resilience4j

Tres capas pueden limitar el tiempo de una llamada Feign. Un senior debe saber cuál aplica en cada caso.

| Capa | Propiedad | Qué mide | Quién gana si hay conflicto |
|---|---|---|---|
| Feign | `feign.client.config.[c].connect-timeout` | Tiempo para establecer TCP | Corta primero el que expire antes |
| Feign | `feign.client.config.[c].read-timeout` | Tiempo esperando respuesta | Corta primero el que expire antes |
| Resilience4j TimeLimiter | `resilience4j.timelimiter.instances.[c].timeout-duration` | Tiempo total de la llamada | Si es menor que `read-timeout` de Feign, Resilience4j corta antes |
| Spring Cloud LoadBalancer | No tiene timeout propio | — | Depende del cliente HTTP |

> [EXAMEN] En entrevista: "Tengo `read-timeout: 5000` en Feign y `timeout-duration: 2s` en Resilience4j TimeLimiter para el mismo cliente. La llamada tarda 3s. ¿Qué ocurre?" Respuesta: Resilience4j TimeLimiter expira a los 2s y lanza `TimeoutException`, que a su vez puede activar el fallback del Circuit Breaker. El timeout de Feign (5s) nunca llega a dispararse porque Resilience4j cancela la ejecución antes.

---

<- [4.3 Configuración por clase Java](sc-feign-configuracion-java.md) | [Índice](README.md) | [4.5 Codecs](sc-feign-codecs.md) ->
