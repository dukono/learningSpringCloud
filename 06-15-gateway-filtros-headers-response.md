# 6.3.3 Headers de respuesta: AddResponseHeader, SetResponseHeader, RemoveResponseHeader, DedupeResponseHeader, SetStatus

← [6.3.2 Headers de petición](./06-14-gateway-filtros-headers-request.md) | [Índice](./README.md) | [6.3.4 Control de flujo →](./06-16-gateway-filtros-flujo.md)

---

Los filtros de headers de respuesta actúan sobre el mensaje HTTP que el servicio de destino devuelve al Gateway antes de que este lo reenvíe al cliente. Permiten añadir cabeceras de seguridad que los servicios internos no incluyen, eliminar cabeceras que exponen información sensible del backend, normalizar valores duplicados que algunos servicios emiten en varias instancias del mismo header, y sobreescribir el código de estado HTTP para adaptarlo a los contratos de la API pública. Estos filtros son especialmente relevantes cuando los servicios downstream no controlan directamente sus cabeceras o cuando el Gateway centraliza políticas de seguridad HTTP que aplican a toda la plataforma.

## Comparativa de los filtros de headers de respuesta

Los filtros de respuesta se diferencian principalmente en su comportamiento cuando el header que intentan modificar ya existe en la respuesta del servicio downstream. La distinción entre `AddResponseHeader` (acumula) y `SetResponseHeader` (sobreescribe) es especialmente relevante para headers como `Access-Control-Allow-Origin`, donde un valor duplicado puede provocar errores en el navegador.

| Filtro | Comportamiento si el header ya existe | Caso de uso principal |
|---|---|---|
| `AddResponseHeader` | Añade una nueva instancia (puede duplicar) | Añadir cabeceras de trazabilidad o seguridad |
| `SetResponseHeader` | Sobreescribe todos los valores existentes | Normalizar o forzar un valor único |
| `RemoveResponseHeader` | Elimina todas las instancias del header | Eliminar headers internos del backend |
| `RewriteResponseHeader` | Aplica regex al valor de un header y lo reemplaza | Transformar URLs internas en valores de headers |
| `RewriteLocationResponseHeader` | Normaliza el header `Location` para que apunte al Gateway | Redirecciones desde backends con URLs internas |
| `DedupeResponseHeader` | Elimina duplicados según la estrategia | Limpiar headers CORS duplicados |
| `SetStatus` | Reemplaza el código de estado HTTP | Adaptar el código de estado al contrato público |

## AddResponseHeader — añadir una cabecera a la respuesta

`AddResponseHeader` añade una cabecera al mensaje de respuesta que el Gateway devuelve al cliente. Si el header ya existe en la respuesta del servicio downstream, el filtro añade una instancia adicional sin eliminar las existentes, lo que puede generar valores duplicados. Cuando el comportamiento deseado es sobreescribir, se debe usar `SetResponseHeader` en su lugar.

La forma abreviada recibe el nombre del header y el valor como dos argumentos separados por coma:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - AddResponseHeader=X-Gateway-Version, 3.1
            - AddResponseHeader=X-Response-Time, ${fecha}
```

El valor del header soporta variables del contexto del exchange. La variable más útil es `{path}`, que se resuelve con el path de la petición original, y `{host}`, que contiene el host de la petición.

```yaml
filters:
  - AddResponseHeader=X-Original-Path, {path}
```

> [CONCEPTO] `AddResponseHeader` se ejecuta sobre la respuesta, no sobre la petición. Esto significa que actúa en la fase de salida del pipeline del Gateway: primero la petición llega al servicio, el servicio responde, y entonces el filtro añade el header antes de enviar esa respuesta al cliente.

## SetResponseHeader — sobreescribir una cabecera de respuesta

`SetResponseHeader` reemplaza todos los valores existentes del header especificado por el nuevo valor. Si el header no existía, lo crea. Es el filtro adecuado cuando se necesita garantizar que un header tenga exactamente un valor determinado, independientemente de lo que haya enviado el servicio de destino.

```yaml
filters:
  # Fuerza Content-Type a JSON aunque el servicio lo omita o lo declare incorrecto
  - SetResponseHeader=Content-Type, application/json;charset=UTF-8

  # Sobreescribe la política de caché del servicio con la política del Gateway
  - SetResponseHeader=Cache-Control, no-store, no-cache, must-revalidate
```

Un caso de uso habitual es garantizar las cabeceras de seguridad HTTP que el equipo de seguridad exige en toda la plataforma:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - SetResponseHeader=Strict-Transport-Security, max-age=31536000; includeSubDomains
        - SetResponseHeader=X-Frame-Options, DENY
        - SetResponseHeader=X-Content-Type-Options, nosniff
```

Configurados en `default-filters`, estos headers se añaden a todas las respuestas del Gateway independientemente de qué servicio las generó.

> [ADVERTENCIA] `SetResponseHeader` no se puede usar para sobreescribir el header `Set-Cookie`: la especificación HTTP permite múltiples instancias de `Set-Cookie` y sobreescribir solo el primer valor puede invalidar sesiones activas. Para manipular cookies de respuesta se deben usar otras estrategias como `AddResponseHeader` con nombre exacto de cookie o un filtro personalizado.

## RemoveResponseHeader — eliminar una cabecera de la respuesta

`RemoveResponseHeader` elimina completamente un header de la respuesta antes de enviarla al cliente. El caso de uso más habitual es ocultar información interna del backend: versiones de servidor, nombres de frameworks o headers propietarios que no deben exponerse al exterior.

```yaml
filters:
  # Elimina el header que revela el servidor web usado por el backend
  - RemoveResponseHeader=Server
  # Elimina cabeceras internas de trazabilidad que no deben llegar al cliente
  - RemoveResponseHeader=X-Powered-By
  - RemoveResponseHeader=X-Application-Context
```

Para aplicar la eliminación a todas las rutas, se puede configurar en `default-filters`:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - RemoveResponseHeader=Server
        - RemoveResponseHeader=X-Powered-By
        - RemoveResponseHeader=X-Application-Context
        - RemoveResponseHeader=X-AspNet-Version
```

> [EXAMEN] `RemoveResponseHeader` solo puede eliminar un header por declaración. Para eliminar varios headers es necesario declarar el filtro múltiples veces, una por cada header a eliminar. No existe una variante que acepte una lista de headers en una sola declaración.

## DedupeResponseHeader — eliminar valores duplicados

`DedupeResponseHeader` resuelve el problema de los headers duplicados que aparecen cuando el Gateway añade una cabecera que el servicio downstream también envía, o cuando varios middlewares intermedios la añaden de forma independiente. El caso más frecuente son las cabeceras CORS: si el servicio downstream incluye `Access-Control-Allow-Origin` y el Gateway también lo añade mediante `AddResponseHeader`, el cliente recibe dos instancias del mismo header, lo que algunos navegadores interpretan como un error.

La sintaxis acepta el nombre del header y, opcionalmente, una estrategia:

```yaml
filters:
  # Sintaxis abreviada: elimina duplicados con la estrategia RETAIN_FIRST (por defecto)
  - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin

  # Con estrategia explícita
  - DedupeResponseHeader=Access-Control-Allow-Origin, RETAIN_FIRST
```

Las tres estrategias disponibles son:

| Estrategia | Comportamiento |
|---|---|
| `RETAIN_FIRST` | Conserva el primer valor y elimina los siguientes. Es la estrategia por defecto. |
| `RETAIN_LAST` | Conserva el último valor y elimina los anteriores. |
| `RETAIN_UNIQUE` | Conserva solo los valores únicos, eliminando duplicados exactos pero manteniendo valores distintos. |

> [CONCEPTO] En la forma abreviada, múltiples headers se separan con espacio dentro del mismo argumento: `DedupeResponseHeader=Header1 Header2`. Esto es específico de este filtro: a diferencia de otros filtros, los múltiples headers se declaran en una sola línea separados por espacios, no con múltiples declaraciones del filtro.

El uso combinado con `AddResponseHeader` para CORS desde `default-filters` es el patrón canónico:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - AddResponseHeader=Access-Control-Allow-Origin, *
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

> [ADVERTENCIA] Si el servicio downstream envía `Access-Control-Allow-Origin: https://app.miempresa.com` y el Gateway añade `Access-Control-Allow-Origin: *`, `DedupeResponseHeader` con `RETAIN_FIRST` conservará el del servicio downstream. Si el comportamiento deseado es que el Gateway controle el valor, se debe usar `SetResponseHeader` (que sobreescribe) en lugar de `AddResponseHeader` (que añade).

## RewriteResponseHeader — reescribir el valor de una cabecera de respuesta con regex

`RewriteResponseHeader` aplica una expresión regular al valor de un header de respuesta y lo sustituye por el resultado del reemplazo. Es útil cuando el backend devuelve un header con un valor que contiene información interna que debe transformarse antes de enviarse al cliente, como una URL interna en el header `Location` o un dominio privado en `Content-Location`.

La sintaxis recibe tres argumentos: nombre del header, patrón regex y cadena de reemplazo.

```yaml
filters:
  # El backend devuelve: Location: http://internal-service:8080/api/v1/productos/123
  # El cliente debe recibir: Location: https://api.miempresa.com/api/v1/productos/123
  - RewriteResponseHeader=Location, http://internal-service:8080, https://api.miempresa.com
```

Un caso de uso más complejo con regex y grupos de captura, para eliminar un puerto interno:

```yaml
filters:
  - RewriteResponseHeader=Location, ^http://[^/]+(?::\d+)?(/.*)$, https://api.miempresa.com$\{1}
```

> [ADVERTENCIA] `RewriteResponseHeader` opera sobre el valor completo del header como una cadena. Si el header contiene varias URLs separadas por coma (como algunos headers `Link`), el patrón debe considerar el formato completo. Un patrón demasiado greedy puede reemplazar partes no deseadas del valor.

## RewriteLocationResponseHeader — normalizar el header Location tras redirecciones

`RewriteLocationResponseHeader` es un filtro especializado en el header `Location` que aparece en respuestas de redirección (`3xx`) o creación de recursos (`201 Created`). Normaliza el host y el protocolo del valor del `Location` para que apunte al Gateway en lugar de al backend interno, evitando que el cliente sea redirigido a una URL interna inaccesible desde el exterior.

```yaml
filters:
  - name: RewriteLocationResponseHeader
    args:
      stripVersion: AS_IN_REQUEST   # mantiene o elimina la versión de la URL
      locationHeaderName: Location  # nombre del header a reescribir (por defecto: Location)
      hostValue: api.miempresa.com  # host de destino en el header reescrito
      protocolsRegex: https?|ftps?  # regex de protocolos que aplican
```

El parámetro `stripVersion` controla si se elimina el prefijo de versión del path:

| Valor | Comportamiento |
|---|---|
| `NEVER_STRIP` | No elimina nunca el prefijo de versión |
| `AS_IN_REQUEST` | Elimina el prefijo solo si la petición original tampoco lo incluía |
| `ALWAYS_STRIP` | Siempre elimina el prefijo de versión |

> [CONCEPTO] La diferencia entre `RewriteResponseHeader` y `RewriteLocationResponseHeader` es de propósito y nivel de abstracción. `RewriteResponseHeader` es genérico: aplica cualquier reemplazo regex a cualquier header. `RewriteLocationResponseHeader` es específico: está diseñado para normalizar el header `Location` teniendo en cuenta la semántica de versioning y el esquema del protocolo. Para el caso específico de `Location`, `RewriteLocationResponseHeader` es la opción recomendada porque gestiona correctamente los casos edge de versioning.

## SetStatus — modificar el código de estado HTTP de la respuesta

`SetStatus` reemplaza el código de estado HTTP de la respuesta antes de enviarla al cliente. Acepta tanto el valor numérico del código (`404`) como el nombre del enum de Spring (`NOT_FOUND`). Ambas formas son equivalentes.

```yaml
routes:
  # Ruta de mantenimiento: devuelve 503 a cualquier petición
  - id: maintenance-route
    uri: lb://static-service
    predicates:
      - Path=/api/**
    filters:
      - SetStatus=503

  # Ruta que normaliza el 201 del backend a 200 para el cliente
  - id: productos-create
    uri: lb://productos-service
    predicates:
      - Path=/api/productos
      - Method=POST
    filters:
      - SetStatus=OK
```

> [CONCEPTO] `SetStatus` modifica el código de estado de la respuesta que el Gateway devuelve al cliente, no el código que el servicio downstream devolvió al Gateway. El código original del backend sigue siendo accesible internamente en los filtros que se ejecutan antes de que la respuesta llegue al cliente.

Un caso de uso habitual es la ruta catch-all con código de error personalizado, que devuelve `404` en lugar del `200` por defecto que devolvería un servicio de páginas estáticas:

```yaml
- id: not-found-route
  uri: lb://error-service
  order: 9999
  predicates:
    - Path=/**
  filters:
    - SetStatus=404
```

> [EXAMEN] Cuando se necesita devolver el código de estado original del backend a la vez que se captura su valor para procesarlo, se puede acceder al código original mediante el atributo `ServerWebExchangeUtils.ORIGINAL_RESPONSE_CONTENT_TYPE_ATTR` en un filtro personalizado. `SetStatus` sobreescribe el código antes de enviar la respuesta, pero no modifica los atributos internos del exchange.

## Tabla de propiedades

Los siete filtros de headers de respuesta presentan interfaces heterogéneas: algunos aceptan solo nombre y valor, mientras que `DedupeResponseHeader` espera una lista de nombres separada por espacios y `RewriteLocationResponseHeader` usa un conjunto de parámetros con nombre propio. La tabla siguiente consolida todos los parámetros, sus tipos y el comportamiento de cada uno para facilitar la consulta rápida.

| Filtro | Parámetro | Tipo | Descripción |
|---|---|---|---|
| `AddResponseHeader` | `name` | String | Nombre del header que se añade a la respuesta |
| `AddResponseHeader` | `value` | String | Valor del header. Soporta variables `{path}`, `{host}` |
| `SetResponseHeader` | `name` | String | Nombre del header cuyo valor se sobreescribe |
| `SetResponseHeader` | `value` | String | Nuevo valor único del header |
| `RemoveResponseHeader` | `name` | String | Nombre del header que se elimina completamente |
| `RewriteResponseHeader` | `name` | String | Nombre del header cuyo valor se transforma |
| `RewriteResponseHeader` | `regexp` | String (regex Java) | Patrón regex aplicado al valor del header |
| `RewriteResponseHeader` | `replacement` | String | Cadena de reemplazo; soporta `$\{1}` para grupos capturados |
| `RewriteLocationResponseHeader` | `stripVersion` | Enum | `NEVER_STRIP`, `AS_IN_REQUEST`, `ALWAYS_STRIP` |
| `RewriteLocationResponseHeader` | `locationHeaderName` | String | Nombre del header a reescribir. Por defecto: `Location` |
| `RewriteLocationResponseHeader` | `hostValue` | String | Host con el que se reemplaza el host interno |
| `RewriteLocationResponseHeader` | `protocolsRegex` | String | Regex de protocolos a los que aplica. Por defecto: `https?|ftps?` |
| `DedupeResponseHeader` | `name` | String | Uno o más nombres de headers separados por espacio |
| `DedupeResponseHeader` | `strategy` | Enum | `RETAIN_FIRST` (defecto), `RETAIN_LAST`, `RETAIN_UNIQUE` |
| `SetStatus` | `status` | String/int | Código numérico (ej. `404`) o nombre del enum Spring (`NOT_FOUND`) |

## Buenas y malas prácticas

Centralizar los headers de seguridad HTTP (`Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options`) en `default-filters` con `SetResponseHeader` garantiza que aplican a todas las respuestas del Gateway independientemente del servicio que las genera. Esto elimina la necesidad de que cada microservicio gestione sus propias cabeceras de seguridad.

Combinar `AddResponseHeader=Access-Control-Allow-Origin` con `DedupeResponseHeader` en `default-filters` es el patrón recomendado para gestionar CORS de forma centralizada cuando los servicios downstream también pueden enviar ese header.

Usar `RemoveResponseHeader` en `default-filters` para eliminar `Server`, `X-Powered-By` y otros headers que revelan información del stack tecnológico. Esta es una medida de hardening básica que se puede aplicar sin modificar ningún servicio individual.

No usar `AddResponseHeader` cuando el objetivo es garantizar un valor único: si el servicio ya envía el header, el resultado serán dos instancias. Usar `SetResponseHeader` en ese caso.

No usar `SetStatus` para ocultar errores reales del backend de los sistemas de monitorización. Si el código de estado se modifica a nivel del Gateway, los dashboards y alertas basados en códigos HTTP pueden dejar de detectar errores reales en los servicios. Si se modifica el código, asegurarse de que los logs del Gateway registran el código original.

No acumular múltiples `RemoveResponseHeader` en `default-filters` sin documentar por qué se elimina cada header. En auditorías de seguridad, la ausencia de ciertos headers puede interpretarse erróneamente como un problema si no hay documentación que explique que se eliminan intencionalmente en el Gateway.

## Comparación: filtros de respuesta vs filtros de petición

Aunque son simétricos en nombre, los filtros de respuesta y los de petición tienen diferencias importantes en cuanto a cuándo actúan en el pipeline y qué información está disponible en cada momento.

| Aspecto | Filtros de petición (request) | Filtros de respuesta (response) |
|---|---|---|
| Fase de ejecución | Antes de enviar la petición al backend | Después de recibir la respuesta del backend |
| Acceso al código de estado | No (aún no existe) | Sí (`SetStatus`, lógica condicional) |
| Impacto en el backend | Directo (el backend ve los headers modificados) | Ninguno (el backend ya respondió) |
| Caso de uso principal | Propagación de contexto, autenticación, routing | Seguridad HTTP, normalización, sanitización |
| Visibilidad para el cliente | Solo si el backend los reenvía en la respuesta | Directa (los ve el cliente) |

---

← [6.3.2 Headers de petición](./06-14-gateway-filtros-headers-request.md) | [Índice](./README.md) | [6.3.4 Control de flujo →](./06-16-gateway-filtros-flujo.md)
