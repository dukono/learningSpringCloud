# 6.3.2 Headers de petición: AddRequestHeader, SetRequestHeader, AddRequestHeadersIfNotPresent, MapRequestHeader, RemoveRequestHeader

← [6.3.1 Transformación de path](./06-13-gateway-filtros-path.md) | [Índice](./README.md) | [6.3.3 Headers de respuesta →](./06-15-gateway-filtros-headers-response.md)

---

El Gateway no solo enruta peticiones: también puede transformarlas antes de que lleguen al servicio de destino. La modificación de headers de petición es uno de los mecanismos más utilizados para añadir contexto interno (identificadores de ruta, versiones de API, origen de la petición), propagar identidad del usuario autenticado hacia servicios que no tienen acceso directo al sistema de autenticación, y garantizar la compatibilidad con backends legacy que esperan nombres de headers específicos que los clientes modernos no envían. Spring Cloud Gateway ofrece cinco filtros dedicados a esta tarea, cada uno con un comportamiento diferente que conviene conocer en detalle.

## Comparativa de los filtros de headers de petición

Spring Cloud Gateway ofrece cinco filtros para manipular los headers de una petición antes de que llegue al servicio de destino. La diferencia crítica está entre `AddRequestHeader` y `SetRequestHeader`: el primero acumula valores y puede producir headers multivalor, mientras que el segundo garantiza un único valor. Elegir incorrectamente puede causar comportamientos inesperados en servicios que no saben manejar headers multivalor.

| Filtro | Efecto | Caso de uso principal |
|---|---|---|
| `AddRequestHeader` | Añade el header; si ya existe, acumula un valor adicional | Inyectar contexto interno (origen, región) |
| `SetRequestHeader` | Crea el header si no existe o reemplaza todos sus valores por uno | Normalizar headers que el cliente puede enviar con valores variables |
| `AddRequestHeadersIfNotPresent` | Añade el header solo si no está presente en la petición entrante | Valores por defecto que el cliente puede sobreescribir |
| `MapRequestHeader` | Copia el valor de un header a otro header con distinto nombre | Adaptar nombres de headers entre clientes externos y servicios internos |
| `RemoveRequestHeader` | Elimina el header de la petición antes de reenviarla | Seguridad y limpieza de headers sensibles |

---

## AddRequestHeader

`AddRequestHeader` añade un header con el nombre y valor indicados a la petición saliente hacia el servicio. Si la petición ya contiene un header con ese nombre, el filtro no lo elimina sino que añade el nuevo valor, de modo que el header puede terminar con múltiples valores. Este filtro soporta variables de path capturadas por predicates `Path`, lo que permite inyectar en un header información extraída directamente de la URL.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: http://servicio-productos:8080
          predicates:
            - Path=/api/productos/**
          filters:
            - AddRequestHeader=X-Internal-Source, gateway
            - AddRequestHeader=X-Request-Region, EU
```

Con una variable capturada del path:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-versionado-route
          uri: http://servicio-productos:8080
          predicates:
            - Path=/api/{version}/productos/**
          filters:
            - AddRequestHeader=X-API-Version, {version}
```

En este caso, si la petición llega a `/api/v2/productos/123`, el servicio recibirá el header `X-API-Version: v2`.

### Parámetros

`AddRequestHeader` acepta dos parámetros obligatorios: el nombre del header que se inyectará en la petición saliente y el valor que llevará, que puede ser un literal o una variable de path capturada por el predicate.

| Parámetro | Descripción |
|---|---|
| `name` | Nombre del header que se va a añadir |
| `value` | Valor del header; acepta variables de path entre llaves como `{variable}` |

---

## SetRequestHeader

`SetRequestHeader` crea el header si no existe en la petición o reemplaza todos sus valores existentes por el valor indicado. La diferencia fundamental con `AddRequestHeader` es que `Set` garantiza que el header tendrá exactamente un valor al llegar al servicio, independientemente de cuántos valores tuviera en la petición entrante. También soporta variables de path capturadas por predicates.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api-publica-route
          uri: http://servicio-backend:8080
          predicates:
            - Path=/api/**
          filters:
            - SetRequestHeader=X-Forwarded-Host, api.miempresa.com
```

> **Diferencia clave con AddRequestHeader:** si la petición entrante ya contiene `X-Forwarded-Host: otro-valor`, `AddRequestHeader` produciría un header con dos valores (`api.miempresa.com` y `otro-valor`), mientras que `SetRequestHeader` reemplazaría `otro-valor` dejando únicamente `api.miempresa.com`.

### Parámetros

`SetRequestHeader` acepta los mismos dos parámetros que `AddRequestHeader`, pero la semántica del campo `value` cambia: en lugar de acumularse, el valor reemplaza cualquier instancia previa del header, garantizando que el servicio reciba exactamente uno.

| Parámetro | Descripción |
|---|---|
| `name` | Nombre del header que se va a crear o reemplazar |
| `value` | Valor que tendrá el header; acepta variables de path entre llaves como `{variable}` |

---

## AddRequestHeadersIfNotPresent

`AddRequestHeadersIfNotPresent` añade uno o varios headers a la petición, pero solo cuando esos headers no están presentes en la petición entrante. Si el cliente ya envía el header, el filtro no lo modifica. Esto lo hace especialmente útil para inyectar valores por defecto que el cliente tiene la opción de sobreescribir, como trace IDs generados automáticamente o indicadores de origen.

> **Nota sobre la sintaxis:** este filtro usa `:` para separar el nombre del valor dentro de cada par, y `,` para separar múltiples pares. Esta sintaxis difiere de los otros filtros de headers, que usan `,` entre nombre y valor.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: trazabilidad-route
          uri: http://servicio-pedidos:8080
          predicates:
            - Path=/pedidos/**
          filters:
            - AddRequestHeadersIfNotPresent=X-Trace-Id:auto-generated,X-Source:gateway
```

Si la petición entrante ya lleva `X-Trace-Id: abc-123`, el servicio recibirá `X-Trace-Id: abc-123` (el valor original del cliente). Si no lleva ese header, recibirá `X-Trace-Id: auto-generated`.

### Parámetros

`AddRequestHeadersIfNotPresent` tiene una interfaz diferente a los demás filtros de headers: en lugar de recibir nombre y valor por separado, acepta un único parámetro compuesto que agrupa todos los pares en una sola cadena con su sintaxis propia de dos puntos y coma.

| Parámetro | Descripción |
|---|---|
| `headersValues` | Lista de pares `Name:Value` separados por coma; cada par usa `:` entre nombre y valor |

---

## MapRequestHeader

`MapRequestHeader` copia el valor de un header de la petición entrante a un nuevo header con un nombre diferente. El header original se mantiene en la petición; el filtro no lo elimina. Este filtro es útil cuando los clientes externos utilizan nombres de headers distintos a los que esperan los servicios internos, permitiendo una adaptación transparente sin modificar ni el cliente ni el servicio.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: adaptacion-headers-route
          uri: http://servicio-usuarios:8080
          predicates:
            - Path=/usuarios/**
          filters:
            - MapRequestHeader=X-External-User-Id, X-Internal-User-Id
```

Si la petición entrante contiene `X-External-User-Id: 42`, el servicio recibirá tanto `X-External-User-Id: 42` como `X-Internal-User-Id: 42`. Si el header de origen no está presente en la petición, el filtro no añade el header de destino.

### Parámetros

`MapRequestHeader` requiere exactamente dos parámetros que definen el renombrado: el header del que se lee el valor y el nuevo header que recibirá ese valor en la petición enviada al servicio.

| Parámetro | Descripción |
|---|---|
| `fromHeader` | Nombre del header de origen cuyo valor se va a copiar |
| `toHeader` | Nombre del nuevo header de destino que recibirá el valor copiado |

---

## RemoveRequestHeader

`RemoveRequestHeader` elimina un header de la petición antes de reenviarla al servicio de destino. El header desaparece completamente: el servicio no recibe ni el nombre ni el valor. Es el mecanismo principal para evitar que headers sensibles enviados por el cliente lleguen a los servicios internos.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: limpieza-headers-route
          uri: http://servicio-catalogo:8080
          predicates:
            - Path=/catalogo/**
          filters:
            - RemoveRequestHeader=X-Client-Secret
            - RemoveRequestHeader=Cookie
```

En este ejemplo, aunque el cliente envíe `X-Client-Secret` o `Cookie`, el servicio de catálogo no recibirá ninguno de esos headers.

### Parámetros

`RemoveRequestHeader` acepta un único parámetro: el nombre del header que debe desaparecer de la petición saliente. Para eliminar varios headers es necesario declarar el filtro una vez por cada header.

| Parámetro | Descripción |
|---|---|
| `name` | Nombre del header que se va a eliminar de la petición saliente |

---

## Buenas y malas prácticas

Hacer:
- Usar `RemoveRequestHeader` para eliminar headers sensibles del cliente que no deben llegar al servicio. Si el cliente envía por error un token interno o una credencial en un header, el Gateway puede interceptarlo y eliminarlo antes de que alcance cualquier servicio interno.
- Preferir `SetRequestHeader` sobre `AddRequestHeader` cuando el header debe tener exactamente un valor. Usando `Add`, si el cliente ya envía ese header, el servicio recibirá múltiples valores; usando `Set`, el valor queda normalizado a uno solo, lo que evita comportamientos inesperados en servicios que no saben manejar headers multivalor.
- Usar `AddRequestHeadersIfNotPresent` para inyectar trace IDs u otros identificadores de correlación solo cuando el cliente no los proporciona. Esto respeta los IDs de correlación generados por sistemas de trazabilidad upstream (como Zipkin o Jaeger corriendo en el cliente) sin sobreescribirlos.

Evitar:
- Usar `AddRequestHeader` para propagar secretos o credenciales internas hacia los servicios. Si el valor del header se registra en logs de cualquier sistema intermedio (proxy, balanceador, sistema de auditoría), la credencial queda expuesta en texto plano. Para la comunicación segura entre servicios, usar mecanismos de autenticación de servicio a servicio como mutual TLS o tokens de servicio dedicados.
- Combinar `AddRequestHeader` y `SetRequestHeader` para el mismo header en la misma ruta. El resultado depende del orden de ejecución de los filtros, que puede no ser intuitivo. El comportamiento puede cambiar entre versiones de Spring Cloud Gateway y produce configuraciones difíciles de razonar y de testear.

---

← [6.3.1 Transformación de path](./06-13-gateway-filtros-path.md) | [Índice](./README.md) | [6.3.3 Headers de respuesta →](./06-15-gateway-filtros-headers-response.md)
