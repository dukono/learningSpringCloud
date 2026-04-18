# 10.3 Spring Cloud Contract — Contratos HTTP

← [10.2 DSL Groovy y YAML](sc-contract-dsl.md) | [Índice](README.md) | [10.4 Contratos Mensajería](sc-contract-mensajeria.md) →

---

## Introducción

Los contratos HTTP son el tipo más frecuente en Spring Cloud Contract y cubren toda la casuística de las APIs REST: métodos HTTP, URLs estáticas y dinámicas con regex, cabeceras, query parameters, cookies, body con matchers, multipart y respuestas asíncronas. Dominar la estructura completa del bloque `request` y `response` es imprescindible para el examen de certificación.

> [PREREQUISITO] Este nodo asume conocimiento del DSL Groovy y YAML de [10.2 DSL Groovy y YAML](sc-contract-dsl.md).

## Estructura del bloque request

El bloque `request` define todo lo que el consumidor enviará al productor. A continuación se muestra un contrato exhaustivo con todos los campos posibles del bloque `request` en un único ejemplo anotado.

```groovy
// src/test/resources/contracts/product/shouldSearchProducts.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should search products by category with pagination"
    request {
        // Método HTTP: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
        method GET()

        // url: URL exacta, sin regex
        // url "/products"

        // urlPath: URL exacta (alternativa a url)
        // urlPath "/products"

        // urlPathPattern: URL con regex (útil para path variables dinámicos)
        urlPathPattern "/products/[a-zA-Z0-9-]+"

        // queryParameters: parámetros de query string
        queryParameters {
            parameter("category", "electronics")
            parameter("page", "0")
            parameter("size", "10")
        }

        headers {
            header("Accept", "application/json")
            header("Authorization", matching("Bearer .+"))
        }

        // cookies: cookies enviadas en el request
        cookies {
            cookie("sessionId", "abc123")
        }
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            content: [[id: 1, name: "Laptop"]],
            totalElements: 1,
            totalPages: 1
        ])
    }
}
```

> [CONCEPTO] `url` acepta una cadena literal exacta. `urlPathPattern` acepta una expresión regular completa que se aplica solo al path (sin query string). Usar `urlPathPattern` es obligatorio cuando el path contiene segmentos dinámicos como IDs variables.

## Estructura del bloque response

El bloque `response` define el comportamiento esperado del productor. Incluye el status HTTP, headers de respuesta, body, matchers dinámicos y soporte para respuestas asíncronas.

```groovy
// src/test/resources/contracts/order/shouldGetOrderWithDynamicFields.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return order with dynamic createdAt and UUID traceId"
    request {
        method GET()
        url "/orders/1"
        headers {
            header("Accept", "application/json")
        }
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
            header("X-Request-Id", $(anyNonEmptyString()))
        }
        body([
            id       : 1,
            status   : "CONFIRMED",
            createdAt: $(anyIsoDate()),
            traceId  : $(anyUuid()),
            total    : $(anyDouble())
        ])
        // bodyMatchers: validaciones adicionales sobre el body de la respuesta
        // Complementan o sustituyen los matchers inline del body
        bodyMatchers {
            jsonPath('$.status', byRegex("CONFIRMED|PENDING|CANCELLED"))
            jsonPath('$.total', byRegex("[0-9]+(\\.[0-9]{1,2})?"))
        }
        // async: indica al generador que el response puede ser asíncrono (Deferred/DeferredResult)
        // async()
    }
}
```

> [CONCEPTO] `bodyMatchers` permite añadir validaciones adicionales sobre campos del body usando jsonPath. Es especialmente útil cuando el valor del body en el contrato es estático (para el stub del consumidor) pero en el productor necesita validarse con un patrón. Los matchers inline en `body` y los de `bodyMatchers` son complementarios.

## URL estática vs URL dinámica

La elección entre `url`, `urlPath` y `urlPathPattern` tiene implicaciones directas en cómo el stub WireMock y el test generado del productor validan la URL.

```groovy
// URL exacta — el stub solo responde a /orders/1 exactamente
Contract.make {
    request {
        method GET()
        url "/orders/1"
    }
    response { status OK() }
}

// urlPathPattern con regex — el stub responde a cualquier /orders/{id numérico}
Contract.make {
    request {
        method GET()
        urlPathPattern "/orders/[0-9]+"
    }
    response { status OK() }
}

// url con queryParameters — el stub responde a /orders?status=CONFIRMED
Contract.make {
    request {
        method GET()
        url "/orders"
        queryParameters {
            parameter("status", "CONFIRMED")
        }
    }
    response { status OK() }
}
```

> [ADVERTENCIA] `urlPathPattern` no admite query string. Los query parameters deben especificarse siempre en el bloque `queryParameters`, tanto si se usa `url` como `urlPathPattern`. Mezclar query params en la cadena de `url` es un error habitual.

## Contratos POST con body validado

Los contratos de escritura (POST, PUT, PATCH) requieren definir el body del request. Spring Cloud Contract valida que el productor rechace o acepte el body exacto especificado en el contrato.

```groovy
// src/test/resources/contracts/order/shouldCreateOrder.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should create a new order and return 201 with the created resource"
    request {
        method POST()
        url "/orders"
        headers {
            contentType(applicationJson())
        }
        body([
            customerId: 42,
            items: [
                [productId: 1, quantity: 2],
                [productId: 3, quantity: 1]
            ]
        ])
        // bodyMatchers en el request: validan lo que el productor recibe
        bodyMatchers {
            jsonPath('$.customerId', byRegex("[0-9]+"))
            jsonPath('$.items[*].quantity', byRegex("[1-9][0-9]*"))
        }
    }
    response {
        status CREATED()
        headers {
            contentType(applicationJson())
            header("Location", matching("/orders/[0-9]+"))
        }
        body([
            id       : 1,
            customerId: 42,
            status   : "PENDING"
        ])
    }
}
```

## Equivalente YAML completo

El mismo contrato de creación de order en formato YAML muestra la diferencia en cómo se expresan los matchers (en un bloque separado `matchers`).

```yaml
# src/test/resources/contracts/order/shouldCreateOrder.yml
description: "should create a new order and return 201 with the created resource"
request:
  method: POST
  url: /orders
  headers:
    Content-Type: application/json
  body:
    customerId: 42
    items:
      - productId: 1
        quantity: 2
      - productId: 3
        quantity: 1
  matchers:
    body:
      - path: $.customerId
        type: by_regex
        value: "[0-9]+"
      - path: $.items[*].quantity
        type: by_regex
        value: "[1-9][0-9]*"
response:
  status: 201
  headers:
    Content-Type: application/json
    Location: /orders/1
  body:
    id: 1
    customerId: 42
    status: PENDING
  matchers:
    headers:
      - key: Location
        regex: /orders/[0-9]+
```

## Tabla de campos del bloque request

Los campos disponibles en el bloque `request` de un contrato HTTP cubren todos los aspectos de una petición HTTP estándar.

| Campo | Tipo | Descripción |
|---|---|---|
| `method` | Obligatorio | Método HTTP: `GET()`, `POST()`, `PUT()`, `DELETE()`, `PATCH()` |
| `url` | Obligatorio* | URL exacta del endpoint |
| `urlPath` | Obligatorio* | Alias de `url` |
| `urlPathPattern` | Obligatorio* | URL como regex, solo path sin query |
| `headers` | Opcional | Mapa de cabeceras con valores exactos o matchers |
| `body` | Opcional | Cuerpo del request como mapa, lista o cadena |
| `bodyMatchers` | Opcional | Validaciones jsonPath adicionales sobre el body |
| `queryParameters` | Opcional | Parámetros de query string |
| `cookies` | Opcional | Cookies del request |
| `multipart` | Opcional | Partes de un multipart/form-data |

*Se especifica exactamente uno de: `url`, `urlPath` o `urlPathPattern`.

## Buenas y malas prácticas

**Buenas prácticas**:
- Usar `urlPathPattern` para endpoints con IDs en el path y así evitar contratos duplicados por cada ID.
- Definir solo los headers que el productor realmente valida; headers opcionales no necesitan aparecer en el contrato.
- Usar `bodyMatchers` para añadir validaciones de formato sobre campos del response que son dinámicos en el productor.
- Separar contratos por escenario (happy path, error 404, error 400) en ficheros distintos con `description` clara.

**Malas prácticas**:
- Incluir query params en la cadena literal de `url` en lugar de usar el bloque `queryParameters`.
- Omitir `bodyMatchers` en responses con campos como UUIDs o timestamps, lo que hace fallar los tests del productor.
- Escribir un único contrato "todo en uno" con múltiples escenarios — cada fichero debe representar un único escenario.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia entre `url` y `urlPathPattern` en el bloque `request` de un contrato HTTP?

> [EXAMEN] 2. ¿Cómo se especifican los query parameters en un contrato Spring Cloud Contract para un GET con filtros?

> [EXAMEN] 3. ¿Qué rol cumple el bloque `bodyMatchers` en el bloque `response` y cuándo es necesario usarlo?

> [EXAMEN] 4. En un contrato YAML, ¿dónde se colocan los matchers dinámicos de un campo del body de respuesta?

> [EXAMEN] 5. ¿Qué método se usa en Groovy para definir que la URL del contrato acepta cualquier ID numérico en el path?

---

← [10.2 DSL Groovy y YAML](sc-contract-dsl.md) | [Índice](README.md) | [10.4 Contratos Mensajería](sc-contract-mensajeria.md) →
