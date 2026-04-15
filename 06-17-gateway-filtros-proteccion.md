# 6.3.5 Protección y utilidades: RequestSize, SecureHeaders, CacheRequestBody, SaveSession

← [6.3.4 Control de flujo](./06-16-gateway-filtros-flujo.md) | [Índice](./README.md) | [6.4.1 AbstractGatewayFilterFactory →](./06-18-gateway-filter-factory.md)

---

Estos filtros protegen el sistema y resuelven casos edge que los filtros de headers o path no cubren. No modifican rutas ni cabeceras de negocio: actúan sobre la integridad del request, la seguridad de la respuesta, el ciclo de vida del body reactivo y la sincronización de sesión.

## Tabla comparativa

Los cuatro filtros de este subtema actúan en capas distintas del ciclo de vida de la petición. La tabla siguiente los compara en función de su efecto y el escenario que justifica su uso, para decidir cuál o cuáles combinar en una ruta concreta.

| Filtro | Efecto | Cuándo usarlo |
|---|---|---|
| `RequestSize` | Rechaza peticiones que superan el tamaño máximo, devolviendo 413 | APIs que reciben uploads o payloads grandes |
| `SecureHeaders` | Añade headers de seguridad OWASP a todas las respuestas | Cualquier API pública; se recomienda en `default-filters` |
| `CacheRequestBody` | Lee el body en memoria y lo hace reutilizable múltiples veces | Cuando más de un filtro necesita leer el body de la petición |
| `SaveSession` | Persiste la sesión de Spring antes de reenviar la petición | Spring Session + Redis + filtros que escriben en sesión |

---

## RequestSize

Rechaza peticiones cuyo body supera el tamaño máximo configurado, devolviendo `413 Payload Too Large` antes de intentar reenviar al servicio. Protege los servicios de uploads accidentales o ataques de peticiones masivas.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: upload-service
          uri: http://upload-service:8080
          predicates:
            - Path=/uploads/**
          filters:
            - name: RequestSize
              args:
                maxSize: 5MB
```

También se puede configurar globalmente como `default-filter` para que aplique a todas las rutas:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - name: RequestSize
          args:
            maxSize: 10MB
```

### Parámetros

`RequestSize` acepta un único parámetro obligatorio que define el umbral de tamaño a partir del cual el Gateway rechaza la petición con `413 Payload Too Large` antes de intentar el enrutamiento.

| Parámetro | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `maxSize` | DataSize | Sí | Tamaño máximo del body. Formatos aceptados: `512B`, `5KB`, `5MB`, `1GB` |

### Buenas prácticas

- Definir `RequestSize` como `default-filter` y sobreescribirlo por ruta solo donde se necesite un límite mayor.
- Combinar siempre con `CacheRequestBody` cuando otros filtros necesitan leer el body: primero `RequestSize` rechaza lo que sea demasiado grande, luego `CacheRequestBody` almacena lo que pase.

### Malas prácticas

- Configurar un `maxSize` grande sin ajustar `spring.codec.max-in-memory-size`.

> [ADVERTENCIA] Netty tiene su propio límite de tamaño de petición en `spring.codec.max-in-memory-size` (por defecto 256 KB). `RequestSize` no sustituye ese límite: si el codec de Netty rechaza la petición primero, `RequestSize` nunca llega a evaluarse. Para peticiones grandes, aumentar ambos valores de forma coherente:
>
> ```yaml
> spring:
>   codec:
>     max-in-memory-size: 10MB
>   cloud:
>     gateway:
>       default-filters:
>         - name: RequestSize
>           args:
>             maxSize: 10MB
> ```

---

## SecureHeaders

Añade automáticamente un conjunto de headers de seguridad HTTP recomendados por OWASP a todas las respuestas. No requiere parámetros para su uso básico.

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - SecureHeaders
```

Headers que añade por defecto:

| Header | Valor por defecto |
|---|---|
| `X-Xss-Protection` | `1; mode=block` |
| `Strict-Transport-Security` | `max-age=631138519` |
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `no-referrer` |
| `Content-Security-Policy` | `default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline'` |
| `X-Download-Options` | `noopen` |
| `X-Permitted-Cross-Domain-Policies` | `none` |

Para desactivar o sobreescribir headers individuales sin eliminar el filtro:

```yaml
spring:
  cloud:
    gateway:
      filter:
        secure-headers:
          disable:
            - x-frame-options
          content-security-policy: "default-src 'self'"
```

### Parámetros

`SecureHeaders` no requiere parámetros en la declaración del filtro: su ajuste fino se realiza a través de las propiedades del namespace `spring.cloud.gateway.filter.secure-headers`, donde se puede desactivar headers individuales o sobreescribir sus valores sin tocar la definición de la ruta.

| Parámetro | Tipo | Descripción |
|---|---|---|
| Sin parámetros requeridos | — | La configuración se gestiona en `spring.cloud.gateway.filter.secure-headers.*` |
| `disable` | Lista de strings | Nombres de headers a desactivar (en minúsculas con guiones) |
| `content-security-policy` | String | Sobreescribe el valor del header `Content-Security-Policy` |

### Buenas prácticas

- Activar `SecureHeaders` en `default-filters` para que todas las rutas lo hereden.
- Sobreescribir solo los headers que la aplicación necesita cambiar; no desactivar todo el filtro.

### Malas prácticas

- Desactivar `x-frame-options` globalmente; hacerlo solo en las rutas concretas que requieren iframes.
- Dejar el `Content-Security-Policy` por defecto en APIs que sirven contenido desde dominios externos sin actualizar la política.

---

## CacheRequestBody

En WebFlux (reactivo), el body de la petición es un stream que solo se puede leer una vez. Si dos filtros intentan leer el body —por ejemplo, un filtro de autenticación que lee el JSON para validar una firma, y luego el propio servicio que también necesita el body— el segundo filtro recibe un stream ya consumido y vacío. `CacheRequestBody` lee el body completo en memoria y lo hace accesible múltiples veces.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: secure-api
          uri: http://api-service:8080
          predicates:
            - Path=/api/**
          filters:
            - name: CacheRequestBody
              args:
                bodyClass: java.lang.String
            - name: RequestHeaderToRequestUri
              args:
                name: X-Forwarded-Prefix
```

Después de aplicar este filtro, el body cacheado se puede acceder en filtros posteriores desde código Java:

```java
String body = exchange.getAttribute(ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);
```

### Parámetros

`CacheRequestBody` requiere declarar el tipo Java al que deserializar el body: esta información determina cómo el codec de WebFlux lee el stream reactivo y lo almacena en el atributo de intercambio para que los filtros siguientes puedan recuperarlo repetidamente.

| Parámetro | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `bodyClass` | Nombre de clase Java | Sí | Tipo al que deserializar el body. Usar `java.lang.String` para JSON o texto plano |

### Buenas prácticas

- Declarar `CacheRequestBody` antes de cualquier filtro que necesite leer el body; el orden en la lista de filtros es el orden de ejecución.
- Combinar con `RequestSize` para limitar el tamaño antes de cachear.

### Malas prácticas

- Usar `CacheRequestBody` en rutas de subida de ficheros grandes sin limitar el tamaño con `RequestSize`.

> [ADVERTENCIA] `CacheRequestBody` carga el body completo en memoria del Gateway. Sin un límite de tamaño previo, un body de varios GB puede agotar la memoria del proceso. Declarar siempre `RequestSize` antes de `CacheRequestBody` en la lista de filtros de la ruta.

> [EXAMEN] `CacheRequestBody` debe declararse ANTES del filtro que necesita leer el body en la lista de filtros de la ruta. El orden de declaración es el orden de ejecución en los filtros GatewayFilter.

---

## SaveSession

Fuerza la persistencia de la sesión de Spring antes de reenviar la petición al servicio. Solo es relevante en proyectos que usan Spring Session WebFlux con un backend de sesión externo (Redis). Garantiza que los datos de sesión escritos por un filtro anterior estén disponibles en el backend antes de que el servicio los lea.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: stateful-service
          uri: http://stateful-service:8080
          predicates:
            - Path=/app/**
          filters:
            - SaveSession
```

Caso de uso típico: un filtro de autenticación escribe el usuario autenticado en la sesión → `SaveSession` persiste la sesión en Redis → el servicio downstream lee el usuario de Redis con la garantía de que el dato ya existe.

### Parámetros

`SaveSession` es un filtro sin configuración: su comportamiento es fijo y se activa únicamente por el hecho de estar declarado en la lista de filtros de la ruta, sin aceptar ningún argumento adicional.

| Parámetro | Tipo | Descripción |
|---|---|---|
| Sin parámetros | — | El filtro no acepta configuración adicional |

### Buenas prácticas

- Declarar `SaveSession` al final de la cadena de filtros, después de cualquier filtro que escriba datos en la sesión.
- Usarlo solo en rutas donde el servicio downstream necesita leer la sesión.

### Malas prácticas

- Añadir `SaveSession` en `default-filters` si la mayoría de las rutas son stateless: genera escrituras innecesarias en Redis en cada petición.

> Si el proyecto no usa Spring Session con almacenamiento externo —es decir, no tiene `spring-session-data-redis` o equivalente en el classpath— este filtro no tiene efecto visible y su presencia en la configuración es inocua pero innecesaria.

---

← [6.3.4 Control de flujo](./06-16-gateway-filtros-flujo.md) | [Índice](./README.md) | [6.4.1 AbstractGatewayFilterFactory →](./06-18-gateway-filter-factory.md)
