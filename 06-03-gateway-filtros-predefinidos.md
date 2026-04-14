# 6.3 Filtros Predefinidos

← [6.2 Rutas y Predicates](./06-02-gateway-rutas.md) | [Índice](./README.md) | [6.4 Filtros Personalizados →](./06-04-gateway-filtros-custom.md)

---

Los filtros predefinidos son transformaciones listas para usar que Spring Cloud Gateway incluye de serie. Sin ellos, cada equipo tendría que reimplementar lógica repetitiva (eliminar prefijos de ruta, propagar headers, limitar el tamaño de peticiones, añadir headers de seguridad) en cada Gateway de cada proyecto. Conocer el catálogo completo evita escribir código Java innecesario: la mayoría de necesidades reales se cubren combinando filtros predefinidos en YAML.

> [PREREQUISITO] Los filtros se declaran dentro de `filters:` en la definición de la ruta o en `default-filters:` para aplicarlos a todas las rutas. Ver [6.2](./06-02-gateway-rutas.md).

---

## 6.3.1 Mapa de filtros por categoría

```
Filtros predefinidos
├── Transformación de path
│   ├── StripPrefix          elimina segmentos del inicio del path
│   ├── PrefixPath           añade un prefijo al path
│   ├── RewritePath          reescribe el path con regex
│   └── SetPath              reemplaza el path completo con variables del predicate
│
├── Manipulación de headers
│   ├── AddRequestHeader         añade header a la petición (siempre)
│   ├── SetRequestHeader         reemplaza header si existe, crea si no
│   ├── AddRequestHeadersIfNotPresent  añade solo si el header no existe
│   ├── MapRequestHeader         copia valor de un header a otro nombre
│   ├── RemoveRequestHeader      elimina header de la petición
│   ├── AddResponseHeader        añade header a la respuesta
│   ├── SetResponseHeader        reemplaza header en la respuesta
│   ├── RemoveResponseHeader     elimina header de la respuesta
│   ├── RewriteResponseHeader    modifica valor de header de respuesta con regex
│   ├── RewriteLocationResponseHeader  corrige header Location en redirects
│   └── DedupeResponseHeader     elimina valores duplicados de un header
│
├── Control de flujo
│   ├── CircuitBreaker       circuit breaker con fallback (Resilience4j)
│   ├── Retry                reintentos automáticos con backoff
│   └── RedirectTo           responde con redirección HTTP al cliente
│
├── Protección
│   ├── RequestSize          rechaza peticiones que superen un tamaño
│   └── SecureHeaders        añade headers de seguridad estándar (HSTS, CSP…)
│
└── Utilidades
    ├── TokenRelay           propaga token OAuth2 al microservicio downstream
    ├── CacheRequestBody     permite leer el body reactivo más de una vez
    └── SaveSession          persiste la sesión antes de enrutar
```

---

## 6.3.2 Transformación de path: StripPrefix, PrefixPath, RewritePath, SetPath

```yaml
spring:
  cloud:
    gateway:
      routes:

        # ── Transformación de path ─────────────────────────────────
        - id: demo-path
          uri: lb://demo-service
          predicates:
            - Path=/api/v1/pedidos/{id}
          filters:
            - StripPrefix=2
            # /api/v1/pedidos/42  →  /pedidos/42

            - PrefixPath=/internal
            # /pedidos/42  →  /internal/pedidos/42

            - RewritePath=/api/v1/(?<recurso>.*), /v2/${recurso}
            # /api/v1/pedidos/42  →  /v2/pedidos/42

            - SetPath=/recursos/{id}
            # usa la variable {id} capturada por el predicate Path

        # ── Manipulación de headers ────────────────────────────────
        - id: demo-headers
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Source, gateway           # añade siempre
            - SetRequestHeader=X-Env, produccion           # reemplaza si existe
            - AddRequestHeadersIfNotPresent=X-Request-Id:default-id  # solo si no existe
            - MapRequestHeader=X-Client-Token, Authorization          # copia valor a otro nombre
            - RemoveRequestHeader=Cookie                   # elimina cookies antes del microservicio
            - AddResponseHeader=X-Powered-By, spring-cloud-gateway
            - SetResponseHeader=Cache-Control, no-store
            - RemoveResponseHeader=X-Internal-Info         # oculta headers internos al cliente
            - RewriteResponseHeader=X-Debug, password=[^&]+, password=***  # enmascara datos sensibles
            - DedupeResponseHeader=Access-Control-Allow-Origin RETAIN_FIRST  # elimina duplicados CORS

        # ── Control de flujo ───────────────────────────────────────
        - id: demo-flujo
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: pedidosCB
                fallbackUri: forward:/fallback/pedidos
                statusCodes: 500,502,503
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET                               # no reintentar POST por defecto
                backoff:
                  firstBackoff: 50ms
                  maxBackoff: 500ms
                  factor: 2
                  basedOnPreviousValue: false
            - RedirectTo=301, https://nueva-url.com        # redirigir con código HTTP

        # ── Protección ─────────────────────────────────────────────
        - id: demo-proteccion
          uri: lb://upload-service
          predicates:
            - Path=/uploads/**
          filters:
            - name: RequestSize
              args:
                maxSize: 10MB                              # rechaza con 413 si supera el límite
            - SecureHeaders                                # añade HSTS, X-Frame-Options, CSP, etc.

        # ── Utilidades ─────────────────────────────────────────────
        - id: demo-utils
          uri: lb://oauth-service
          predicates:
            - Path=/api/secure/**
          filters:
            - TokenRelay=                                  # propaga Bearer token al microservicio
            - name: CacheRequestBody
              args:
                bodyClass: java.lang.String                # permite leer el body más de una vez
            - SaveSession                                  # persiste la sesión Spring antes de enrutar
```

---

## Ejemplo completo integrado — Gateway en producción con todos los filtros

Una ruta de producción combina varios filtros. Este ejemplo integra transformación de path, headers de trazabilidad, protección por tamaño, circuit breaker con fallback y headers de seguridad en una única ruta:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin RETAIN_FIRST
        - AddResponseHeader=X-Gateway-Version, 2.0
        - SecureHeaders

      routes:
        - id: pedidos-completo
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
            - Method=GET,POST,PUT,DELETE
          filters:
            # Transformación de path
            - StripPrefix=1

            # Trazabilidad
            - AddRequestHeader=X-Correlation-Id, ${spring.application.name}
            - AddRequestHeadersIfNotPresent=X-Request-Id:default-id

            # Limpieza de headers internos
            - RemoveRequestHeader=Cookie
            - RemoveResponseHeader=X-Internal-Debug

            # Protección
            - name: RequestSize
              args:
                maxSize: 2MB

            # Reintentos solo para GET
            - name: Retry
              args:
                retries: 2
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 1000ms
                  factor: 2
                  basedOnPreviousValue: false

            # Circuit Breaker con fallback al propio Gateway
            - name: CircuitBreaker
              args:
                name: pedidosCB
                fallbackUri: forward:/fallback/pedidos
                statusCodes: 500,502,503,504
          order: 10
```

---

## Parámetros de los filtros más configurables

| Filtro | Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|---|
| `StripPrefix` | `parts` | int | — | Número de segmentos a eliminar del inicio del path |
| `PrefixPath` | `prefix` | String | — | Prefijo a añadir al path |
| `RewritePath` | `regexp`, `replacement` | String | — | Regex y reemplazo para reescribir el path |
| `RequestSize` | `maxSize` | DataSize | 5MB | Tamaño máximo del body; rechaza con 413 si se supera |
| `Retry` | `retries` | int | 3 | Número de reintentos (sin contar el intento original) |
| `Retry` | `statuses` | List\<HttpStatus\> | SERVER_ERROR | Códigos HTTP que disparan el reintento |
| `Retry` | `methods` | List\<HttpMethod\> | GET | Métodos HTTP que se reintentarán |
| `Retry` | `backoff.firstBackoff` | Duration | 5ms | Espera antes del primer reintento |
| `Retry` | `backoff.maxBackoff` | Duration | no limit | Espera máxima entre reintentos |
| `Retry` | `backoff.factor` | int | 2 | Factor multiplicador del backoff exponencial |
| `CircuitBreaker` | `name` | String | — | Nombre del CircuitBreaker (referencia a config Resilience4j) |
| `CircuitBreaker` | `fallbackUri` | String | — | URI de fallback cuando el CB está abierto |
| `CircuitBreaker` | `statusCodes` | Set\<String\> | `[]` | Códigos HTTP que cuentan como fallo |
| `DedupeResponseHeader` | `name` | String | — | Nombre del header a deduplicar |
| `DedupeResponseHeader` | `strategy` | String | RETAIN_FIRST | `RETAIN_FIRST`, `RETAIN_LAST`, `RETAIN_UNIQUE` |
| `CacheRequestBody` | `bodyClass` | Class | — | Tipo al que deserializar el body cacheado |

---

## 6.3.5 Buenas y malas prácticas

**Hacer:**
- Usar `DedupeResponseHeader=Access-Control-Allow-Origin` en `default-filters` cuando hay CORS tanto en el Gateway como en los microservicios — evita el error de CORS por header duplicado que aparece en producción.
- Configurar `Retry` solo para métodos idempotentes (`GET`, `HEAD`, `OPTIONS`) — reintentar un `POST` puede crear recursos duplicados si el servicio destino no es idempotente.
- Poner `CacheRequestBody` como **primer** filtro de la lista cuando otro filtro posterior necesita leer el body — si se coloca después del filtro que ya consumió el stream, ese filtro recibirá un stream vacío.
- Usar `RemoveRequestHeader=Cookie` en rutas a microservicios internos — las cookies son estado del cliente y los servicios de backend no deben procesarlas.

**Evitar:**
- Usar `AddRequestHeader` cuando se quiere garantizar un valor concreto — si el cliente ya envía ese header, `AddRequestHeader` crea un duplicado; usar `SetRequestHeader` en su lugar.
- Usar `Retry` con `statuses: SERVER_ERROR` sin filtrar métodos — un `POST` con error 500 reintentado crea efectos secundarios si el servicio tuvo éxito parcial antes del fallo.
- Omitir `RewriteLocationResponseHeader` cuando los microservicios devuelven redirects (3xx) — sin este filtro el header `Location` expone URLs internas al cliente.

---

## 6.3.3 Headers, control de flujo, protección y utilidades

Una ruta real combina transformación de path, trazabilidad, protección y resiliencia en una única declaración coherente:

```yaml
spring:
  cloud:
    gateway:
      # default-filters aplican a TODAS las rutas sin repetirlos en cada una
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin RETAIN_FIRST
        - AddResponseHeader=X-Gateway-Version, 2.0
        - SecureHeaders

      routes:
        - id: pedidos-produccion
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
            - Method=GET,POST,PUT,DELETE
          filters:
            # 1. Transformación de path
            - StripPrefix=1                        # /api/pedidos/42  →  /pedidos/42

            # 2. Trazabilidad
            - AddRequestHeadersIfNotPresent=X-Request-Id:auto-generated
            - AddRequestHeader=X-Gateway-Source, gateway-produccion
            - RemoveRequestHeader=Cookie           # limpiar cookies del cliente

            # 3. Limitar body
            - name: RequestSize
              args:
                maxSize: 2MB

            # 4. Retry solo para GET (idempotente)
            - name: Retry
              args:
                retries: 2
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 1000ms
                  factor: 2
                  basedOnPreviousValue: false

            # 5. Circuit Breaker — último filtro pre
            - name: CircuitBreaker
              args:
                name: pedidosCB
                fallbackUri: forward:/fallback/pedidos
                statusCodes: 500,502,503,504
          order: 10
```

---

## 6.3.4 Parámetros de los filtros más configurables

| Filtro | Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|---|
| `StripPrefix` | `parts` | int | — | Segmentos a eliminar del inicio del path |
| `PrefixPath` | `prefix` | String | — | Prefijo a añadir delante del path |
| `RewritePath` | `regexp`, `replacement` | String | — | Regex y texto de reemplazo |
| `RequestSize` | `maxSize` | DataSize | 5MB | Rechaza con 413 si el body supera este límite |
| `Retry` | `retries` | int | 3 | Número de reintentos (sin el intento original) |
| `Retry` | `statuses` | List | SERVER_ERROR | Códigos HTTP que disparan el reintento |
| `Retry` | `methods` | List | GET | Métodos HTTP que se reintentarán |
| `Retry` | `backoff.firstBackoff` | Duration | 5ms | Espera antes del primer reintento |
| `Retry` | `backoff.factor` | int | 2 | Multiplicador del backoff exponencial |
| `CircuitBreaker` | `name` | String | — | Nombre del CB (referencia a config Resilience4j) |
| `CircuitBreaker` | `fallbackUri` | String | — | URI del fallback cuando el CB está abierto |
| `CircuitBreaker` | `statusCodes` | Set | `[]` | Códigos HTTP que cuentan como fallo del CB |
| `DedupeResponseHeader` | `name` | String | — | Nombre del header a deduplicar |
| `DedupeResponseHeader` | `strategy` | String | RETAIN_FIRST | `RETAIN_FIRST`, `RETAIN_LAST`, `RETAIN_UNIQUE` |
| `CacheRequestBody` | `bodyClass` | Class | — | Tipo al que deserializar el body para caché |

---

## 6.3.6 Comparación: filtros de path

| Necesidad | Filtro recomendado | Cuándo NO usarlo |
|---|---|---|
| Eliminar `/api` del inicio | `StripPrefix=1` | Si el número de segmentos a eliminar varía por petición |
| Añadir `/v2` delante | `PrefixPath=/v2` | Si el prefijo depende de la petición concreta |
| Cambiar `/api/v1/X` a `/v2/X` | `RewritePath` con grupo nombrado | Si la transformación es solo eliminar segmentos del inicio |
| Reemplazar el path entero con variables del predicate | `SetPath` | Si el path destino no puede formarse con variables `{var}` capturadas |

---

← [6.2 Rutas y Predicates](./06-02-gateway-rutas.md) | [Índice](./README.md) | [6.4 Filtros Personalizados →](./06-04-gateway-filtros-custom.md)

