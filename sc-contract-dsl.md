# 10.2 Spring Cloud Contract — DSL Groovy y YAML

← [10.1 Fundamentos CDC](sc-contract-fundamentos.md) | [Índice](README.md) | [10.3 Contratos HTTP](sc-contract-http.md) →

---

## Introducción

Spring Cloud Contract ofrece dos lenguajes para definir contratos: el **DSL Groovy**, que es la sintaxis nativa y más expresiva del framework, y el **DSL YAML**, que es su equivalente sin dependencias de Groovy. Ambos producen exactamente el mismo resultado — tests generados en el productor y stubs WireMock en el consumidor — y pueden coexistir en el mismo proyecto. Conocer ambas sintaxis es imprescindible para el examen de certificación.

> [PREREQUISITO] Este nodo requiere haber comprendido el flujo productor-consumidor descrito en [10.1 Fundamentos CDC](sc-contract-fundamentos.md).

## Estructura del DSL Groovy

El DSL Groovy está centrado en la clase `Contract` del módulo `spring-cloud-contract-spec`. Todo contrato comienza con `Contract.make {}` y contiene uno de dos bloques: `request`/`response` para contratos HTTP, o `input`/`outputMessage` para contratos de mensajería.

```groovy
// src/test/resources/contracts/payment/shouldCreatePayment.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    // Campo opcional: descripción legible del escenario
    description "should create a payment and return 201 with location header"

    // Para contratos HTTP: bloques request y response
    request {
        method POST()
        url "/payments"
        headers {
            contentType(applicationJson())
        }
        body([
            amount  : 150.00,
            currency: "EUR"
        ])
    }
    response {
        status CREATED()
        headers {
            header("Location", "/payments/1")
            contentType(applicationJson())
        }
        body([
            id    : 1,
            status: "PENDING"
        ])
    }
}
```

> [CONCEPTO] El bloque `request` describe lo que el consumidor enviará. El bloque `response` describe lo que el productor **debe** devolver cuando reciba ese request. El plugin generará un test que llama al productor con ese request exacto y verifica la respuesta.

## Bloques de mensajería en Groovy

Para contratos de mensajería asíncrona, los bloques `request`/`response` se sustituyen por `input` y `outputMessage`. El bloque `input` define cómo se desencadena la producción del mensaje, y `outputMessage` define el mensaje resultante que el productor debe enviar.

```groovy
// src/test/resources/contracts/notification/shouldSendNotificationOnOrderConfirmed.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should send notification when order is confirmed via method call"

    input {
        // triggeredBy invoca un método en la clase base del test del productor
        triggeredBy("confirmOrder()")
    }

    outputMessage {
        // sentTo define el destino: canal de Spring Cloud Stream o topic
        sentTo("notifications")
        messageHeaders {
            header("eventType", "ORDER_CONFIRMED")
        }
        messageBody([
            orderId: 1,
            message: "Your order has been confirmed"
        ])
    }
}
```

```groovy
// Variante: contrato activado por un mensaje entrante (no por llamada a método)
Contract.make {
    input {
        messageFrom("orders")         // canal de entrada
        messageHeaders {
            header("action", "confirm")
        }
        messageBody([
            orderId: 1
        ])
    }
    outputMessage {
        sentTo("notifications")
        messageBody([
            orderId: 1,
            message: "Your order has been confirmed"
        ])
    }
}
```

> [CONCEPTO] `triggeredBy("methodName()")` hace que el test generado invoque el método `methodName()` definido en la clase base del productor, en lugar de enviar un mensaje entrante. `messageFrom` es para el escenario inverso: el productor reacciona a un mensaje recibido.

## Equivalente YAML

El mismo contrato HTTP del ejemplo anterior expresado en YAML sigue exactamente la misma semántica. YAML no requiere Groovy en el classpath y es más legible para equipos sin experiencia en Groovy.

```yaml
# src/test/resources/contracts/payment/shouldCreatePayment.yml
description: "should create a payment and return 201 with location header"
request:
  method: POST
  url: /payments
  headers:
    Content-Type: application/json
  body:
    amount: 150.00
    currency: EUR
response:
  status: 201
  headers:
    Location: /payments/1
    Content-Type: application/json
  body:
    id: 1
    status: PENDING
```

```yaml
# Contrato de mensajería en YAML
description: "should send notification on order confirmed"
input:
  triggeredBy: confirmOrder()
outputMessage:
  sentTo: notifications
  headers:
    eventType: ORDER_CONFIRMED
  body:
    orderId: 1
    message: "Your order has been confirmed"
```

## Comparación Groovy vs YAML

Las dos sintaxis son funcionalmente equivalentes, pero tienen diferencias prácticas importantes para elegir una u otra en un proyecto.

| Aspecto | DSL Groovy | YAML |
|---|---|---|
| Sintaxis para métodos HTTP | `method GET()`, `method POST()` | `method: GET`, `method: POST` |
| Estado HTTP | `status OK()`, `status CREATED()` | `status: 200`, `status: 201` |
| Content-Type | `contentType(applicationJson())` | `Content-Type: application/json` |
| Matchers dinámicos | `$(consumer(regex('[0-9]+')))` | `matchers:` bloque separado |
| Expresividad | Alta — closures, lógica condicional | Baja — solo datos estáticos |
| Curva de aprendizaje | Media — requiere sintaxis Groovy | Baja — YAML estándar |
| Recomendado para | Contratos complejos con matchers | Contratos simples o equipos sin Groovy |

> [ADVERTENCIA] En YAML, los matchers dinámicos (`byRegex`, `byUuid`, etc.) no se expresan inline en el `body`, sino en un bloque `matchers` separado a nivel del `request` o `response`. Esto hace que los contratos YAML complejos sean más verbosos que sus equivalentes Groovy.

## Matchers básicos en DSL Groovy

Cuando los valores del contrato no son fijos — por ejemplo un UUID generado o una fecha actual — se usan matchers. El DSL Groovy los expresa directamente con `$(consumer(...), producer(...))` o con los helpers del contrato.

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    request {
        method GET()
        url "/users/1"
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            // El id es fijo para el stub (consumidor) y el test del productor lo valida como regex
            id       : $(consumer(1), producer(regex("[0-9]+"))),
            // El createdAt es ignorado en el stub pero validado como timestamp en el productor
            createdAt: $(consumer("2024-01-01T00:00:00Z"), producer(regex(isoDate()))),
            // El UUID se valida tanto en el stub como en el test del productor
            traceId  : $(anyUuid())
        ])
    }
}
```

> [CONCEPTO] En cada matcher, el valor `consumer(...)` es lo que el **stub** devuelve al consumidor (valor fijo), y `producer(...)` es lo que el **test generado** verifica en el productor (pattern o regex). Cuando se usa `anyUuid()` o `anyNonEmptyString()`, el mismo valor se aplica a ambos lados.

## Tabla de helpers del DSL Groovy

Spring Cloud Contract proporciona helpers directos en el DSL para los tipos de datos más comunes. Estos helpers son atajos que generan matchers predefinidos.

| Helper | Descripción | Equivale a |
|---|---|---|
| `anyAlphaNumeric()` | Cualquier cadena alfanumérica | `regex("[a-zA-Z0-9]+")`|
| `anyNonBlankString()` | Cadena no vacía ni solo espacios | `regex(".*\\S.*")` |
| `anyNonEmptyString()` | Cadena con al menos un carácter | `regex(".+")` |
| `anyUuid()` | UUID formato estándar | `regex(uuid())` |
| `anyPositiveInt()` | Entero positivo | `regex("[1-9][0-9]*")` |
| `anyDouble()` | Número decimal | `regex(aDouble())` |
| `anyBoolean()` | true o false | `regex("(true|false)")` |
| `isoDate()` | Fecha ISO 8601 | Regex de fecha ISO |
| `isoDateTime()` | Fecha y hora ISO 8601 | Regex de datetime ISO |
| `anyEmail()` | Email válido | Regex de email |
| `anyUrl()` | URL válida | Regex de URL |
| `anyIpAddress()` | IP v4 válida | Regex de IP |

## Buenas y malas prácticas

**Buenas prácticas**:
- Usar helpers como `anyUuid()` o `anyPositiveInt()` en lugar de regex escritos a mano para los tipos comunes.
- Organizar los contratos en subdirectorios por entidad o funcionalidad bajo `src/test/resources/contracts/`.
- Elegir YAML para contratos simples de equipos mixtos; Groovy para contratos con lógica de matchers compleja.
- Incluir `description` en cada contrato para documentar el escenario que representa.

**Malas prácticas**:
- Usar valores fijos para campos dinámicos como UUIDs o timestamps sin matchers — los tests del productor fallarán en valores reales.
- Mezclar bloques HTTP (`request`/`response`) con bloques de mensajería (`input`/`outputMessage`) en el mismo contrato — son mutuamente excluyentes.
- Definir contratos demasiado detallados con todos los campos del response cuando el consumidor solo usa dos o tres campos.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia entre `triggeredBy` y `messageFrom` en un contrato de mensajería en Groovy?

> [EXAMEN] 2. En el DSL Groovy, ¿qué significa el valor `consumer(...)` y el valor `producer(...)` dentro de un matcher `$(consumer(...), producer(...))`?

> [EXAMEN] 3. ¿Cómo se expresa un campo de tipo UUID dinámico en el body de respuesta de un contrato Groovy?

> [EXAMEN] 4. ¿Qué bloque se usa en un contrato YAML para definir matchers dinámicos y por qué no van inline en el `body`?

> [EXAMEN] 5. ¿En qué directorio se ubican por defecto los contratos que lee el plugin de Spring Cloud Contract?

---

← [10.1 Fundamentos CDC](sc-contract-fundamentos.md) | [Índice](README.md) | [10.3 Contratos HTTP](sc-contract-http.md) →
