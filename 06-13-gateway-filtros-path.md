# 6.3.1 Transformación de path: StripPrefix, PrefixPath, RewritePath, SetPath

← [6.2.5 Canary, Weight y Forward](./06-12-gateway-canary-forward.md) | [Índice](./README.md) | [6.3.2 Headers de petición →](./06-14-gateway-filtros-headers-request.md)

---

En una arquitectura de microservicios, el Gateway actúa como fachada pública: los clientes externos conocen una URL semántica y estable (`/api/v1/productos/123`), mientras que cada microservicio tiene su propia convención de paths interna (`/productos/123`). Si el Gateway reenvía el path sin modificarlo, el microservicio recibe un path que no reconoce y devuelve un 404. Los filtros de transformación de path resuelven exactamente este problema: adaptan la URL entrante a la que espera el servicio destino, sin exponer detalles internos al exterior y sin obligar a los servicios a conocer el esquema de rutas del Gateway.

## Comparativa de filtros

Antes de entrar en el detalle de cada filtro, la tabla siguiente resume el efecto de cada uno y el caso de uso más habitual. Elegir el filtro correcto depende de si la transformación es aditiva (`PrefixPath`), sustractiva (`StripPrefix`), estructural mediante regex (`RewritePath`) o de reemplazo total del path (`SetPath`).

| Filtro | Efecto | Caso de uso típico |
|---|---|---|
| `StripPrefix=N` | Elimina los primeros N segmentos del path | Versioning en el Gateway (`/api/v1/...` → `/...`) |
| `PrefixPath=/x` | Añade un prefijo fijo al path | Rutas internas con namespace (`/productos` → `/internal/productos`) |
| `RewritePath=/regexp, /reemplazo` | Reescritura completa mediante regex Java | Transformaciones complejas o renombrado de segmentos |
| `SetPath=/template` | Reemplaza el path completo con una plantilla | Migración de URLs legadas a nuevos esquemas |

---

## StripPrefix

> [CONCEPTO] `StripPrefix=N` elimina los N primeros segmentos del path antes de reenviar la petición al servicio. Es el filtro de path más utilizado porque resuelve el caso más habitual: el Gateway añade un prefijo de versión o de contexto que el servicio interno no conoce.

El parámetro `N` indica cuántos segmentos separados por `/` se eliminan desde la izquierda del path. Un segmento es la cadena entre dos barras consecutivas. Por ejemplo, en el path `/api/v1/productos/123`:

- `StripPrefix=1` → elimina `/api` → resultado: `/v1/productos/123`
- `StripPrefix=2` → elimina `/api/v1` → resultado: `/productos/123`
- `StripPrefix=3` → elimina `/api/v1/productos` → resultado: `/123`

El caso más frecuente es `StripPrefix=2` cuando el Gateway expone rutas con prefijo `/api/v1/` y el microservicio espera las rutas sin ese prefijo:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-service
          uri: lb://productos-service
          predicates:
            - Path=/api/v1/productos/**
          filters:
            - StripPrefix=2   # /api/v1/productos/123 → /productos/123
```

> [EXAMEN] El valor de `StripPrefix` debe coincidir exactamente con el número de segmentos del prefijo que añade el Gateway. Un valor incorrecto provoca que el servicio reciba un path inesperado y devuelva 404 sin ningún mensaje de error relacionado con el Gateway.

---

## PrefixPath

> [CONCEPTO] `PrefixPath=/prefijo` antepone un prefijo fijo al path de la petición antes de reenviarla. Es el complemento natural de `StripPrefix`: mientras que `StripPrefix` quita prefijos, `PrefixPath` los añade.

Este filtro es útil cuando el Gateway expone una ruta corta y limpia al exterior, pero el microservicio destino tiene sus endpoints bajo un subpath interno. Por ejemplo, el Gateway publica `/productos/**` pero el microservicio organiza sus endpoints bajo `/internal/productos/**`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-interno
          uri: lb://productos-service
          predicates:
            - Path=/productos/**
          filters:
            - PrefixPath=/internal   # /productos/123 → /internal/productos/123
```

El prefijo se concatena directamente al path entrante, por lo que el resultado es `/internal` + `/productos/123` = `/internal/productos/123`. El prefijo siempre debe comenzar con `/` para producir un path válido.

> [ADVERTENCIA] `PrefixPath` no es compatible con paths que ya contienen el prefijo. Si una petición llega con `/internal/productos/123` y el filtro añade `/internal`, el path resultante sería `/internal/internal/productos/123`. Asegúrate de que los predicates de la ruta filtren únicamente los paths que requieren el prefijo.

---

## RewritePath

> [CONCEPTO] `RewritePath=/regexp, /reemplazo` permite reescribir el path usando una expresión regular Java con grupos con nombre. Es el filtro más potente y flexible de los cuatro, ya que puede transformar la estructura del path de forma arbitraria, no solo añadir o quitar segmentos fijos.

La sintaxis para capturar grupos con nombre en regex Java es `(?<nombre>patrón)`. En el reemplazo, se referencian con `$\{nombre}`. La barra invertida antes de `{` es obligatoria en YAML: sin ella, Spring Boot interpreta `${nombre}` como una referencia a una propiedad de configuración y el contexto falla al arrancar con un error del tipo `Could not resolve placeholder`.

Ejemplo de renombrado de segmento: el Gateway expone `/users/{id}` pero el servicio interno usa la convención `/api/usuarios/{id}`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: usuarios-service
          uri: lb://usuarios-service
          predicates:
            - Path=/users/**
          filters:
            - RewritePath=/users/(?<id>.*), /api/usuarios/$\{id}
            # /users/42      → /api/usuarios/42
            # /users/42/perfil → /api/usuarios/42/perfil
```

Otro ejemplo con múltiples grupos capturados: transformar `/v1/tiendas/Madrid/productos/camisas` en `/stores/es/Madrid/items/camisas`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: tiendas-service
          uri: lb://tiendas-service
          predicates:
            - Path=/v1/tiendas/**
          filters:
            - RewritePath=/v1/tiendas/(?<ciudad>[^/]+)/productos/(?<item>.*), /stores/es/$\{ciudad}/items/$\{item}
            # /v1/tiendas/Madrid/productos/camisas → /stores/es/Madrid/items/camisas
```

> [ADVERTENCIA] En YAML, el par `regexp, reemplazo` es un único argumento de cadena separado por coma y espacio. Si el patrón o el reemplazo contienen caracteres especiales de YAML (como `:`, `#` o `*`), encierra el valor entre comillas simples para evitar errores de parseo.

> [EXAMEN] La barra invertida en `$\{id}` es uno de los errores más frecuentes en exámenes y en producción. Sin ella, Spring Boot falla al arrancar, no en tiempo de ejecución de la petición.

---

## SetPath

> [CONCEPTO] `SetPath=/template` reemplaza el path completo de la petición por una plantilla. Las variables `{var}` dentro de la plantilla se resuelven con los valores capturados por el predicate `Path` de la misma ruta. Es el filtro más adecuado cuando se conoce el path de destino de forma estructurada y no se necesita regex.

Las variables de la plantilla deben coincidir exactamente con los nombres de los segmentos variables definidos en el predicate `Path` con la sintaxis `{var}`. Si el predicate no captura una variable que `SetPath` necesita, la plantilla no se sustituye y el path resultante contiene el literal `{var}` sin reemplazar.

Ejemplo de migración de URLs legadas: el sistema antiguo usaba `/legacy/{service}/{resource}` y el nuevo usa `/api/{service}/v2/{resource}`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: legacy-migration
          uri: lb://backend-service
          predicates:
            - Path=/legacy/{service}/{resource}
          filters:
            - SetPath=/api/{service}/v2/{resource}
            # /legacy/productos/123 → /api/productos/v2/123
            # /legacy/pedidos/456   → /api/pedidos/v2/456
```

El predicate `Path=/legacy/{service}/{resource}` captura los valores de `service` y `resource` de cada petición entrante. `SetPath` accede a esos valores directamente por nombre para construir el path de destino. No se necesita sintaxis de regex ni grupos con nombre.

> [ADVERTENCIA] `SetPath` reemplaza el path completo, incluidos los segmentos que no son variables. Si el predicate captura `/legacy/productos/123` con el pattern `/legacy/{service}/{resource}`, el path resultante de `SetPath=/api/{service}/v2/{resource}` será exactamente `/api/productos/v2/123`, sin rastro del path original.

---

## Tabla de parámetros

Los cuatro filtros de transformación de path comparten una interfaz minimalista: cada uno acepta uno o dos parámetros tipados. La tabla siguiente recoge el nombre exacto de cada parámetro, su tipo y las restricciones que hay que tener en cuenta al configurarlos en YAML.

| Filtro | Parámetro | Tipo | Descripción |
|---|---|---|---|
| `StripPrefix` | `parts` | `int` | Número de segmentos a eliminar desde la izquierda del path. Debe ser mayor que 0. |
| `PrefixPath` | `prefix` | `String` | Prefijo a anteponer al path. Debe comenzar con `/`. |
| `RewritePath` | `regexp` | `String` (regex Java) | Expresión regular con grupos con nombre para capturar partes del path. |
| `RewritePath` | `replacement` | `String` | Patrón de reemplazo. Usa `$\{nombre}` para referenciar grupos capturados. |
| `SetPath` | `template` | `String` | Plantilla del path de destino. Usa `{var}` para insertar variables capturadas por el predicate `Path`. |

---

## Buenas y malas prácticas

Hacer:
- Usa `StripPrefix` para el caso habitual de versioning en el Gateway. Es el filtro más sencillo, más legible y el más fácil de mantener cuando el prefijo es fijo y homogéneo en todas las rutas.
- Documenta en los comentarios de la configuración el path de entrada y el path de salida para cada filtro. La transformación de paths es invisible para el observador externo y puede ser difícil de depurar sin esa referencia.
- Combina `RewritePath` con predicates restrictivos. Cuanto más específico sea el predicate, menos probable es que el patrón regex produzca transformaciones inesperadas en paths no previstos.
- Valida la configuración con tests de integración que comprueben el path real que recibe el servicio destino, no solo el código de respuesta HTTP.
- Cuando el path de destino tiene una estructura fija y predecible, prefiere `SetPath` sobre `RewritePath`. La plantilla de `SetPath` es más legible y menos propensa a errores de regex.

Evitar:
- Usar `StripPrefix=1` cuando la ruta tiene dos segmentos de prefijo a eliminar. El servicio recibirá un path con el segmento sobrante y devolverá 404. Cuenta siempre los segmentos del prefijo antes de asignar el valor.
- Escribir `${id}` sin barra invertida en `RewritePath` dentro de YAML. Spring Boot intenta resolver esa expresión como una propiedad de configuración al arrancar la aplicación y falla con un error de placeholder no resuelto. Usa siempre `$\{id}`.
- Usar `SetPath` con variables que no han sido capturadas por el predicate `Path` de la misma ruta. Si el predicate no incluye `{var}` en su pattern, `SetPath` no tiene ningún valor con el que sustituir `{var}` y el path resultante contiene el literal `{var}` tal cual, lo que provoca 404 en el servicio.
- Mezclar `StripPrefix` y `PrefixPath` en la misma ruta sin verificar el orden de aplicación de los filtros. Aunque técnicamente es posible, el resultado puede ser confuso y difícil de mantener. En ese caso es más claro usar `RewritePath` con un patrón explícito.

---

← [6.2.5 Canary, Weight y Forward](./06-12-gateway-canary-forward.md) | [Índice](./README.md) | [6.3.2 Headers de petición →](./06-14-gateway-filtros-headers-request.md)
