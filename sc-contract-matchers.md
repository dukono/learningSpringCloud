# 10.4.2 Matchers estándar en contratos

← [10.4.1 DSL Groovy/YAML — estructura base de contratos HTTP](sc-contract-dsl.md) | [Índice](README.md) | [10.4.3 Custom matchers y extensión del DSL](sc-contract-custom-matchers.md) →

---

## Introducción

Un contrato sin matchers es un contrato frágil: cualquier cambio en un UUID, un timestamp o un email generado rompe el test del producer aunque la API sea completamente compatible. Los matchers estándar de Spring Cloud Contract resuelven este problema permitiendo expresar reglas de validación —tipo, formato, rango— en lugar de valores concretos. El lado del producer usa el matcher para verificar que la implementación devuelve datos del tipo correcto; el lado del consumer usa el valor concreto del stub para ejecutar su lógica sin sorpresas. Dominar estos ocho matchers es imprescindible para escribir contratos robustos que sobreviven al ciclo de vida de la API.

> [CONCEPTO] **BodyMatchers**: bloque del DSL que permite aplicar matchers JsonPath sobre campos específicos del body del response (o request), separando la validación del producer de lo que el consumer recibe en el stub.

> [PREREQUISITO] Conocimiento del DSL base (10.4.1): structure request/response, dynamic values y la distinción producer/consumer.

## Representación visual

La siguiente tabla-diagrama muestra cómo cada matcher se aplica en el lado del producer (test generado) y qué valor concreto aparece en el stub del consumer.

```
MATCHER              PRODUCER TEST VERIFICA           CONSUMER STUB RECIBE
════════════════     ═══════════════════════           ════════════════════════
anyNonNull()     →   value != null                     "someValue"
anyAlphaNumeric()→   matches [a-zA-Z0-9]+              "abc123"
byRegex("...")   →   value.matches(regex)              valor del consumer (o generado)
byDate()         →   matches YYYY-MM-DD                "2025-04-16"
byTime()         →   matches HH:mm:ss                  "14:30:00"
byTimestamp()    →   matches YYYY-MM-DDTHH:mm:ss.SSS   "2025-04-16T14:30:00.000"
byIso8601...()   →   matches ISO 8601 with offset      "2025-04-16T14:30:00.000+02:00"
byType()         →   value is Integer/String/etc.      valor del consumer
byCommand()      →   executes static method (custom)   valor del consumer
```

## Ejemplo central

El ejemplo muestra un contrato completo usando todos los matchers estándar en el response body, con la sintaxis Groovy DSL y su equivalente YAML con la sección `matchers:`.

**Contrato Groovy con matchers estándar** (`src/test/resources/contracts/order/shouldReturnOrder.groovy`):

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return order with all fields validated by matchers"

    request {
        method GET()
        url '/api/orders/ORD-001'
        headers {
            accept(applicationJson())
        }
    }

    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            // anyNonNull: el campo existe y no es null
            orderId     : value(producer(anyNonNull()), consumer('ORD-001')),

            // anyAlphaNumeric: solo letras y números
            referenceCode: value(producer(anyAlphaNumeric()), consumer('REF123ABC')),

            // byRegex: expresión regular personalizada
            customerEmail: value(producer(regex('[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}')), consumer('alice@example.com')),

            // byDate: formato YYYY-MM-DD
            orderDate   : value(producer(byDate()), consumer('2025-04-16')),

            // byTime: formato HH:mm:ss
            deliveryTime: value(producer(byTime()), consumer('14:30:00')),

            // byTimestamp: formato ISO 8601 sin offset
            createdAt   : value(producer(byTimestamp()), consumer('2025-04-16T10:00:00.000')),

            // byIso8601WithOffset: formato ISO 8601 con offset de timezone
            updatedAt   : value(producer(byIso8601WithOffset()), consumer('2025-04-16T10:00:00.000+02:00')),

            // byType: tipo Java esperado
            totalAmount : value(producer(byType { instanceOf(BigDecimal.class) }), consumer(99.99)),

            // Valor concreto sin matcher (campos estables)
            status      : 'PENDING',
            currency    : 'EUR'
        ])
    }
}
```

**Contrato YAML equivalente** (`src/test/resources/contracts/order/shouldReturnOrder.yml`):

```yaml
description: "should return order with all fields validated by matchers"
request:
  method: GET
  url: /api/orders/ORD-001
  headers:
    Accept: application/json
response:
  status: 200
  headers:
    Content-Type: application/json
  body:
    orderId: ORD-001
    referenceCode: REF123ABC
    customerEmail: alice@example.com
    orderDate: "2025-04-16"
    deliveryTime: "14:30:00"
    createdAt: "2025-04-16T10:00:00.000"
    updatedAt: "2025-04-16T10:00:00.000+02:00"
    totalAmount: 99.99
    status: PENDING
    currency: EUR
  matchers:
    body:
      - path: $.orderId
        type: by_regex
        value: ".+"
      - path: $.referenceCode
        type: by_regex
        value: "[a-zA-Z0-9]+"
      - path: $.customerEmail
        type: by_regex
        value: "[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}"
      - path: $.orderDate
        type: by_date
      - path: $.deliveryTime
        type: by_time
      - path: $.createdAt
        type: by_timestamp
      - path: $.updatedAt
        type: by_iso8601_with_offset
      - path: $.totalAmount
        type: by_type
```

**Contrato con matchers en el cuerpo de la petición (request body matchers):**

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should create order accepting flexible input"

    request {
        method POST()
        url '/api/orders'
        headers {
            contentType(applicationJson())
            accept(applicationJson())
        }
        body([
            customerId   : value(consumer('CUST-1'), producer(anyNonNull())),
            productCode  : value(consumer('PROD-ABC'), producer(anyAlphaNumeric())),
            requestedDate: value(consumer('2025-06-01'), producer(byDate())),
            amount       : value(consumer(150.00), producer(byType { instanceOf(BigDecimal.class) }))
        ])
        bodyMatchers {
            jsonPath('$.customerId', byRegex('[A-Z]+-[0-9]+'))
            jsonPath('$.amount', byType())
        }
    }

    response {
        status CREATED()
        headers {
            contentType(applicationJson())
        }
        body([
            orderId : value(producer(byRegex('[A-Z]{3}-[0-9]+')), consumer('ORD-001')),
            status  : 'CREATED',
            createdAt: value(producer(byTimestamp()), consumer('2025-04-16T10:00:00.000'))
        ])
    }
}
```

**Dependencias Maven** (mismas que en 10.3.1):

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>spring-mock-mvc</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**application.yml:**

```yaml
spring:
  application:
    name: order-service
```

## Tabla de elementos clave

La siguiente tabla es la referencia de los matchers estándar que un profesional senior debe conocer de memoria.

| Matcher | DSL Groovy | YAML type | Pattern verificado |
|---------|-----------|-----------|-------------------|
| `anyNonNull()` | `anyNonNull()` | `by_regex: .+` | Cualquier valor no nulo |
| `anyNonEmptyString()` | `anyNonEmptyString()` | `by_regex: .+` | String no vacío |
| `anyAlphaNumeric()` | `anyAlphaNumeric()` | `by_regex: [a-zA-Z0-9]+` | Solo letras y dígitos |
| `anyAlpha()` | `anyAlpha()` | `by_regex: [a-zA-Z]+` | Solo letras |
| `anyNumber()` | `anyNumber()` | `by_type` | Tipo numérico (Integer, Double, etc.) |
| `anyInteger()` | `anyInteger()` | `by_type` | Tipo entero (Integer o Long) |
| `anyPositiveInt()` | `anyPositiveInt()` | `by_regex: [0-9]+` | Entero positivo |
| `byRegex(pattern)` | `byRegex('[0-9]+')` | `by_regex: [0-9]+` | Coincidencia con expresión regular |
| `byDate()` | `byDate()` | `by_date` | Formato YYYY-MM-DD |
| `byTime()` | `byTime()` | `by_time` | Formato HH:mm:ss |
| `byTimestamp()` | `byTimestamp()` | `by_timestamp` | Formato YYYY-MM-DD'T'HH:mm:ss.SSS |
| `byIso8601WithOffset()` | `byIso8601WithOffset()` | `by_iso8601_with_offset` | ISO 8601 con timezone offset |
| `byType()` | `byType { instanceOf(X.class) }` | `by_type` | Tipo Java o JSON |

## Buenas y malas prácticas

**Hacer:**

- Usar `byDate()`, `byTime()` y `byTimestamp()` para todos los campos temporales en lugar de hardcodear fechas: una fecha hardcoded como `"2025-01-01"` convierte el test del producer en un test que falla fuera de esa fecha si la lógica genera la fecha actual.
- Combinar matchers en el request body (`bodyMatchers { jsonPath(...) }`) cuando el consumer envía datos dinámicos: permite que el test del producer verifique que el endpoint acepta correctamente datos variables, no solo los valores concretos del stub.
- Usar `byRegex()` con expresiones específicas del dominio para campos como codes, IDs o referencias: `byRegex('[A-Z]{3}-[0-9]+')` es más informativo que `anyNonNull()` y produce mensajes de error más claros cuando falla.
- Añadir matchers solo a los campos realmente variables; los campos estables (`status`, `currency`) deben permanecer como valores literales para detectar regressions en valores específicos.
- Verificar que las expresiones regulares son compatibles con Java regex (el backend del matcher); patrones PCRE avanzados (lookbehind variable, `\K`) no están soportados.

**Evitar:**

- Evitar usar `anyNonNull()` en campos que deben tener un formato específico: si `orderId` debe seguir el patrón `ORD-[0-9]+`, un `anyNonNull()` acepta `"hello"` como válido en el test del producer, dejando pasar una implementación incorrecta.
- Evitar usar `byType()` sin especificar el tipo esperado (`instanceOf(Integer.class)`): el matcher acepta cualquier tipo JSON (string, number, boolean), haciendo la verificación inútil para contratos de tipos estrictos.
- Evitar mezclar matchers en YAML con la sintaxis Groovy (`$(producer(...), consumer(...))` no existe en YAML): en YAML los matchers siempre van en la sección `matchers:` separada del body.
- Evitar `byRegex()` con expresiones que usan Java-style escaping en YAML sin el escape correcto: en YAML, `\\d+` debe escribirse como `\\d+` (double backslash) en strings sin comillas; en strings con comillas simples se pierde el escape.
- Evitar usar `byTimestamp()` para campos `date-only`: el matcher espera la forma completa `YYYY-MM-DDTHH:mm:ss.SSS` y rechazará un valor como `"2025-04-16"` aunque sea una fecha válida.

## Comparación: matchers disponibles por categoría

| Categoría | Matchers disponibles | Caso de uso típico |
|-----------|---------------------|-------------------|
| Nulidad | `anyNonNull()`, `anyNonEmptyString()` | IDs, tokens, campos obligatorios |
| Tipo alfanumérico | `anyAlphaNumeric()`, `anyAlpha()`, `anyNumber()`, `anyInteger()`, `anyPositiveInt()` | Códigos de producto, contadores |
| Expresión regular | `byRegex(pattern)` | Emails, teléfonos, referencias con formato propio |
| Temporal | `byDate()`, `byTime()`, `byTimestamp()`, `byIso8601WithOffset()` | Fechas de creación, timestamps de auditoría |
| Tipo Java | `byType { instanceOf(X.class) }` | Decimales, booleanos, arrays |
| Custom | `byCommand('methodName')` | Validaciones de negocio propias (ver 10.4.3) |

> [EXAMEN] Pregunta de entrevista: "¿Qué diferencia hay entre `byTimestamp()` y `byIso8601WithOffset()` en Spring Cloud Contract?" — `byTimestamp()` valida el formato `YYYY-MM-DDTHH:mm:ss.SSS` sin offset de timezone (UTC implícito). `byIso8601WithOffset()` valida el formato ISO 8601 completo con offset explícito (`+02:00`, `Z`). Para APIs que devuelven fechas con timezone, usar `byIso8601WithOffset()` para no hacer pasar valores en UTC que deberían llevar offset.

> [ADVERTENCIA] Los matchers de la sección `bodyMatchers { jsonPath(...) }` en el DSL Groovy son para el response body del producer. Si se necesita validar el request body en el producer, se usa también `bodyMatchers` dentro del bloque `request`. No confundir con el `body()` del bloque `request`, que define el valor que el consumer enviará en el stub.

---

← [10.4.1 DSL Groovy/YAML — estructura base de contratos HTTP](sc-contract-dsl.md) | [Índice](README.md) | [10.4.3 Custom matchers y extensión del DSL](sc-contract-custom-matchers.md) →
