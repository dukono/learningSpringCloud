# 6.2.1 Modelo de Route: id, uri, predicates, filters, order y metadata

← [6.1.7 Discovery Locator y Actuator](./06-07-gateway-discovery-actuator.md) | [Índice](./README.md) | [6.2.2 Catálogo de Predicates →](./06-09-gateway-predicates.md)

---

## Qué es una Route y por qué cada campo existe

Una Route es la unidad de configuración fundamental de Spring Cloud Gateway. Cada Route asocia un conjunto de condiciones de entrada (predicates) con un destino (uri) y, opcionalmente, un conjunto de transformaciones (filters) que se aplican a la petición antes de enviarla al destino y a la respuesta antes de devolverla al cliente.

Cuando llega una petición HTTP al Gateway, el motor de enrutamiento evalúa las rutas disponibles en orden ascendente de prioridad. Para cada ruta, evalúa todos sus predicates en forma de AND lógico: si todos se cumplen, la petición se envía al uri de esa ruta con los filters configurados aplicados. Si alguno no se cumple, el Gateway pasa a evaluar la siguiente ruta. Si ninguna ruta coincide, el Gateway devuelve 404.

Cada campo de una Route existe por una razón concreta: `id` permite identificar la ruta en logs y en los endpoints de Actuator; `uri` define el destino; `predicates` describe las condiciones de enrutamiento; `filters` permite manipular la petición y la respuesta; `order` controla la prioridad cuando varias rutas tienen predicates solapados; y `metadata` proporciona un mecanismo de extensión para configuración adicional por ruta, como timeouts individuales o contexto para filtros personalizados.

## Diagrama: anatomía de una Route

La siguiente representación muestra la estructura jerárquica de una Route con todos sus campos:

```
Route
├── id: "productos-route"             ← identificador en logs y Actuator
├── uri: "lb://productos-service"     ← destino (con o sin load balancer)
├── order: 1                          ← prioridad (menor = se evalúa antes)
├── predicates:                       ← condiciones AND de entrada
│   ├── Path=/api/productos/**
│   └── Method=GET,POST
├── filters:                          ← transformaciones request/response
│   ├── StripPrefix=1
│   └── AddResponseHeader=X-Service, productos
└── metadata:                         ← configuración adicional por ruta
    ├── connect-timeout: 2000
    └── response-timeout: 8000
```

El campo `predicates` es evaluado de arriba hacia abajo, pero todos deben cumplirse simultáneamente (AND). Si cualquiera de los predicates falla, toda la ruta se descarta y el Gateway evalúa la siguiente.

## Ejemplo completo con todos los campos

El siguiente ejemplo muestra dos rutas completamente definidas. La primera tiene todos los campos configurados explícitamente; la segunda actúa como catch-all para absorber cualquier petición que no haya coincidido con rutas anteriores.

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Ruta con todos los campos explícitos
        - id: api-productos-v2
          uri: lb://productos-service
          order: 1
          predicates:
            - Path=/api/v2/productos/**
            - Method=GET,POST,PUT,DELETE
            - Header=X-API-Version, 2
          filters:
            - StripPrefix=2
            - AddResponseHeader=X-API-Version, 2
            - AddRequestHeader=X-Internal-Source, gateway
          metadata:
            connect-timeout: 2000
            response-timeout: 8000

        # Ruta catch-all con order alto para no interferir con rutas específicas
        - id: fallback-route
          uri: lb://fallback-service
          order: 9999
          predicates:
            - Path=/**
          filters:
            - AddResponseHeader=X-Fallback, true
```

> [CONCEPTO] En la ruta `api-productos-v2`, los tres predicates (Path, Method y Header) se evalúan como AND: la petición debe cumplir los tres a la vez para ser enrutada a `productos-service`. Si falta cualquiera de los tres, el Gateway descarta esa ruta y evalúa `fallback-route`.

## Tabla de propiedades de RouteDefinition

La clase interna `RouteDefinition` es la que Spring Cloud Gateway usa para representar una ruta en memoria, tanto cuando se carga desde YAML como cuando se crea dinámicamente a través de la API de Actuator.

| Campo | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `id` | `String` | UUID generado automáticamente | Identificador único de la ruta. Aparece en logs y en `/actuator/gateway/routes`. |
| `uri` | `URI` | Obligatorio | URI del destino. Admite esquemas `lb://`, `http://`, `https://`, `ws://`, `wss://` y `forward:///`. |
| `order` | `int` | `0` | Prioridad de evaluación. Valor menor = se evalúa antes. |
| `predicates` | `List<PredicateDefinition>` | Lista vacía | Condiciones AND de entrada. Lista vacía significa que la ruta siempre coincide. |
| `filters` | `List<FilterDefinition>` | Lista vacía | Transformaciones aplicadas a la petición y/o respuesta. |
| `metadata` | `Map<String, Object>` | Mapa vacío | Metadatos de la ruta. Las claves `connect-timeout` y `response-timeout` (en ms) son interpretadas directamente por el cliente HTTP reactor-netty. |

## Detalle de los campos

### El campo `id`

El `id` es un identificador de cadena que debe ser único dentro de la configuración del Gateway. Si se omite en YAML, Spring Cloud Gateway genera automáticamente un UUID en tiempo de arranque, lo que hace que los logs y los endpoints de Actuator muestren valores como `_genkey_0`, difíciles de interpretar. Por esta razón, siempre se recomienda asignar un `id` explícito.

> [EXAMEN] Un id omitido no provoca error de arranque: el Gateway simplemente genera un UUID. Sin embargo, en entornos con múltiples instancias o con configuración dinámica vía Actuator, los ids autogenerados pueden variar entre instancias y causar inconsistencias.

La convención habitual es kebab-case con el patrón `nombre-servicio-v{version}` o `nombre-funcionalidad-route`:

```yaml
# Correcto: descriptivo y consistente
id: catalogo-productos-v1

# Evitar: sin contexto, difícil de encontrar en logs
id: route1
```

### El campo `uri`

El `uri` define el destino al que el Gateway reenvía la petición. Los formatos soportados son:

- `lb://nombre-servicio` — reenvío con load balancer usando el registro de Eureka o Consul. Es el formato más habitual en arquitecturas de microservicios.
- `http://host:puerto` — dirección fija sin TLS. Útil para servicios externos o en entornos de desarrollo.
- `https://host:puerto` — dirección fija con TLS.
- `ws://host:puerto` — WebSocket sin TLS.
- `wss://host:puerto` — WebSocket con TLS.
- `forward:///path` — reenvío interno al propio `DispatcherHandler` del Gateway. Se usa cuando el Gateway expone sus propios endpoints y quiere enrutarlos como si fueran rutas externas.

> [ADVERTENCIA] El esquema `lb://` requiere que `spring-cloud-starter-loadbalancer` esté en el classpath y que haya un servidor de descubrimiento (Eureka, Consul) configurado. Sin esas dependencias, el arranque fallará con un error indicando que el esquema `lb` no está soportado.

### Los campos `predicates` y `filters`

Los `predicates` son condiciones de entrada evaluadas en AND: todas deben cumplirse para que la ruta coincida. Una lista de `predicates` vacía equivale a un catch-all que coincide con cualquier petición.

Los `filters` son transformaciones que se aplican en cadena: primero a la petición antes de enviarla al destino, y luego a la respuesta antes de devolverla al cliente. El orden de declaración en la lista determina el orden de aplicación.

Los predicates se tratan en detalle en [6.2.2 Catálogo de Predicates](./06-09-gateway-predicates.md). Los filtros predefinidos se cubren en la sección 6.3.x y los filtros personalizados en 6.4.x.

### El campo `order`

El campo `order` controla en qué posición del proceso de evaluación se considera cada ruta. El Gateway evalúa las rutas de menor a mayor valor de `order`. Cuando dos rutas tienen el mismo valor de `order`, el orden de evaluación depende del orden de declaración en el fichero YAML: la primera declarada se evalúa primero.

> [EXAMEN] El valor por defecto de `order` es `0`. Si todas las rutas tienen `order: 0`, el orden de evaluación es el orden de declaración en YAML, de arriba hacia abajo.

Una ruta catch-all (con `Path=/**` o `predicates` vacío) debe tener el `order` más alto posible para garantizar que solo se activa cuando ninguna ruta más específica ha coincidido:

```yaml
# Incorrecto: la ruta catch-all con order 0 absorberá todas las peticiones
# antes de que lleguen a rutas más específicas con el mismo order
- id: catch-all
  uri: lb://fallback
  order: 0
  predicates:
    - Path=/**

- id: api-v2
  uri: lb://productos
  order: 0
  predicates:
    - Path=/api/v2/**
```

```yaml
# Correcto: la ruta específica tiene order menor y se evalúa primero
- id: api-v2
  uri: lb://productos
  order: 1
  predicates:
    - Path=/api/v2/**

- id: catch-all
  uri: lb://fallback
  order: 9999
  predicates:
    - Path=/**
```

### El campo `metadata`

El campo `metadata` es un mapa de pares clave-valor de tipo arbitrario (`Map<String, Object>`) que se asocia a la ruta. Su uso más habitual es sobreescribir los timeouts del cliente HTTP a nivel de ruta individual, sin modificar la configuración global del Gateway.

Las claves reconocidas por el cliente HTTP reactor-netty son:

- `connect-timeout` (entero, en milisegundos): tiempo máximo para establecer la conexión TCP con el destino.
- `response-timeout` (entero, en milisegundos): tiempo máximo para recibir la primera respuesta del destino una vez establecida la conexión.

```yaml
- id: servicio-lento
  uri: lb://batch-service
  order: 5
  predicates:
    - Path=/api/batch/**
  metadata:
    connect-timeout: 5000
    response-timeout: 30000
```

Además de los timeouts, el campo `metadata` puede contener cualquier clave-valor definida por el equipo de desarrollo para pasar contexto a filtros personalizados que lo necesiten:

```yaml
metadata:
  connect-timeout: 2000
  response-timeout: 8000
  equipo-responsable: plataforma
  nivel-criticidad: alto
```

Un filtro personalizado puede leer estos valores accediendo a la ruta activa en el contexto del exchange:

```java
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.cloud.gateway.route.Route;

// Dentro del método filter(ServerWebExchange exchange, GatewayFilterChain chain):
Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
if (route != null) {
    String criticidad = (String) route.getMetadata().get("nivel-criticidad");
}
```

`ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR` es una constante de tipo `String` que actúa como clave en el mapa de atributos del exchange. Spring Cloud Gateway la almacena automáticamente cuando selecciona una ruta para procesar la petición.

## Buenas y malas prácticas

Las siguientes prácticas recogen los patrones más habituales y los errores más frecuentes al definir rutas en Spring Cloud Gateway.

**Buenas prácticas:**

- Asignar siempre un `id` explícito en kebab-case. Facilita la depuración en logs y en Actuator.
- Ordenar las rutas de más específica a menos específica, asignando valores de `order` que reflejen esa jerarquía (1, 2, ..., 9999 para el catch-all).
- Usar `lb://` en lugar de IPs fijas en entornos con descubrimiento de servicios.
- Centralizar los timeouts globales en `spring.cloud.gateway.httpclient` y usar `metadata` solo para excepciones puntuales por ruta.
- Documentar en comentarios YAML el propósito de cada ruta cuando el `id` no sea suficientemente descriptivo.

**Malas prácticas:**

- Omitir el `id`: los UUIDs autogenerados son ilegibles en logs y dificultan la gestión dinámica de rutas.
- Poner una ruta catch-all con `order: 0` sin darse cuenta de que bloquea todas las demás rutas con el mismo `order`.
- Usar `http://` con IPs fijas en producción: crea acoplamiento a infraestructura y rompe el enrutamiento dinámico.
- Definir `metadata` con claves de timeout como cadenas de texto en lugar de enteros, lo que provoca que el cliente HTTP ignore los valores.

> [ADVERTENCIA] Los valores de `connect-timeout` y `response-timeout` en `metadata` deben ser enteros (sin comillas en YAML). Si se definen como cadenas (`"2000"` en lugar de `2000`), Spring Cloud Gateway no los parsea correctamente y el cliente HTTP usa los valores globales configurados en `spring.cloud.gateway.httpclient`.

## Comparación: shortcut notation vs expanded notation

Los predicates y los filters admiten dos formas de escritura en YAML. Conocer ambas es importante porque la API de Actuator usa exclusivamente la forma expanded para crear y consultar rutas dinámicamente.

La forma shortcut es la más habitual en ficheros YAML estáticos. Cada predicate o filter se escribe como una cadena con el nombre seguido de los argumentos separados por comas:

```yaml
predicates:
  - Path=/api/**
  - Method=GET,POST
filters:
  - StripPrefix=1
  - AddResponseHeader=X-Version, 1
```

La forma expanded es más verbosa pero más explícita. Cada predicate o filter se escribe como un objeto con las claves `name` y `args`:

```yaml
predicates:
  - name: Path
    args:
      patterns: /api/**
  - name: Method
    args:
      methods: GET,POST
filters:
  - name: StripPrefix
    args:
      parts: 1
  - name: AddResponseHeader
    args:
      name: X-Version
      value: "1"
```

Ambas formas son funcionalmente equivalentes cuando se cargan desde YAML. Las diferencias prácticas son las siguientes:

| Aspecto | Shortcut | Expanded |
|---|---|---|
| Legibilidad en YAML estático | Mayor | Menor |
| Requerida por la API de Actuator | No | Sí |
| Útil cuando los args tienen claves con nombre | No siempre | Sí |
| Soportada en configuración programática (`RouteLocatorBuilder`) | No aplica | Equivalente |

[LEGACY] En versiones anteriores a Spring Cloud Gateway 3.x, algunos predicates y filters solo soportaban la forma expanded porque la shortcut no había sido implementada para todos ellos. En versiones actuales, prácticamente todos los predicates y filters built-in soportan ambas formas.

> [EXAMEN] Cuando se crea una ruta mediante `POST /actuator/gateway/routes/{id}`, el cuerpo JSON debe usar la forma expanded, con los campos `name` y `args` para cada predicate y filter. La forma shortcut no es válida en el JSON de la API REST de Actuator.

---

← [6.1.7 Discovery Locator y Actuator](./06-07-gateway-discovery-actuator.md) | [Índice](./README.md) | [6.2.2 Catálogo de Predicates →](./06-09-gateway-predicates.md)
