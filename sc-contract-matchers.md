# 10.9 Spring Cloud Contract — Matchers Personalizados

← [10.8 Repositorio de Stubs](sc-contract-stubs-repo.md) | [Índice](README.md) | [10.10 Contratos GraphQL](sc-contract-graphql.md) →

---

## Introducción

Los matchers en Spring Cloud Contract resuelven el problema de los campos dinámicos en contratos: valores que cambian en cada ejecución, como UUIDs, timestamps o IDs generados. La API `BodyMatchers` permite expresar expectativas sobre el formato o tipo de un campo sin fijar su valor exacto, tanto en el bloque `request` (para validar lo que el productor recibe) como en el bloque `response` (para validar lo que el productor devuelve). Comprender la diferencia entre matchers del lado productor y del lado consumidor es esencial para el examen.

> [PREREQUISITO] Este nodo requiere conocimiento del DSL de contratos de [10.2](sc-contract-dsl.md) y de los contratos HTTP de [10.3](sc-contract-http.md).

## BodyMatchers API: jsonPath + matchers encadenados

El bloque `bodyMatchers {}` del DSL Groovy permite definir validaciones sobre campos específicos del body usando expresiones jsonPath. Cada validación combina un path con un matcher.

```groovy
// src/test/resources/contracts/product/shouldReturnProductWithMatchers.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return product with dynamic fields validated by matchers"
    request {
        method GET()
        url "/products/1"
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        // body: valores usados en el STUB (lo que recibe el consumidor)
        // Para campos dinámicos, se pone un valor representativo
        body([
            id         : 1,
            name       : "Laptop Pro",
            price      : 999.99,
            uuid       : "550e8400-e29b-41d4-a716-446655440000",
            createdAt  : "2024-01-15T10:30:00Z",
            category   : "ELECTRONICS",
            available  : true,
            tags       : ["tech", "computer"]
        ])
        // bodyMatchers: validaciones que el TEST DEL PRODUCTOR aplica sobre el response real
        // Verifican que el productor devuelve el formato correcto, no el valor exacto del stub
        bodyMatchers {
            // byRegex: valida que el campo cumple una expresión regular
            jsonPath('$.id', byRegex("[0-9]+"))

            // byType: valida que el campo es del tipo especificado
            jsonPath('$.price', byType())  // cualquier número

            // byUuid: valida que el campo tiene formato UUID estándar
            jsonPath('$.uuid', byUuid())

            // byTimestamp: valida formato datetime ISO 8601
            jsonPath('$.createdAt', byTimestamp())

            // byDate: valida formato fecha ISO 8601 (YYYY-MM-DD)
            // jsonPath('$.birthDate', byDate())

            // byRegex con constante predefinida: isoDateTimeWithMillis()
            // jsonPath('$.updatedAt', byRegex(isoDateTime()))

            // byCommand: valida usando un método custom de la clase base del test
            // El método es invocado con el valor real del campo
            jsonPath('$.category', byCommand('assertCategoryIsValid($it)'))

            // equalToJson: valida que el campo es igual al JSON especificado
            // Útil para validar objetos anidados completos
            // jsonPath('$.address', equalToJson('{"city":"Madrid","country":"ES"}'))
        }
    }
}
```

> [CONCEPTO] La diferencia fundamental entre los valores del `body` y los de `bodyMatchers`: el `body` define el **valor que el stub devuelve al consumidor** (fijo, determinista). Los `bodyMatchers` definen las **validaciones que el test del productor aplica sobre el response real** (dinámico, basado en regex/tipo). Son complementarios, no alternativos.

## Matchers en el bloque request

Los matchers también se aplican en el bloque `request` para validar qué recibe el productor. En este caso, `bodyMatchers` dentro de `request {}` valida el body entrante.

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should accept order creation with validated request body"
    request {
        method POST()
        url "/orders"
        headers {
            contentType(applicationJson())
        }
        body([
            customerId: 42,
            amount    : 150.00,
            currency  : "EUR"
        ])
        // bodyMatchers en el REQUEST: el stub WireMock usa estas reglas para
        // decidir si la petición entrante coincide con este stub
        bodyMatchers {
            jsonPath('$.customerId', byRegex("[0-9]+"))
            jsonPath('$.amount', byRegex("[0-9]+(\\.[0-9]{1,2})?"))
            jsonPath('$.currency', byRegex("[A-Z]{3}"))
        }
    }
    response {
        status CREATED()
        body([id: 1, status: "PENDING"])
    }
}
```

## HeaderMatchers API

`HeaderMatchers` permite validar cabeceras con expresiones regulares en lugar de valores exactos. Se define dentro del bloque `headers {}` usando `matching()` o `$(...)`.

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    request {
        method GET()
        url "/secured/resource"
        headers {
            // Header con valor exacto
            header("Accept", "application/json")
            // Header con regex: acepta cualquier token Bearer
            header("Authorization", matching("Bearer [A-Za-z0-9._-]+"))
            // Header con matcher dinámico
            header("X-Correlation-Id", $(consumer(anyNonEmptyString()), producer(anyUuid())))
        }
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
            // Header en response con regex: el test del productor valida el formato
            header("X-Request-Id", $(anyUuid()))
            header("Cache-Control", matching("max-age=[0-9]+"))
        }
        body([data: "secured content"])
    }
}
```

## byCommand() para validación custom

`byCommand()` es el matcher más flexible: permite delegar la validación a un método custom definido en la clase base del test del productor. El método recibe el valor real del campo como argumento (`$it` en el DSL).

```groovy
// Contrato con byCommand para validación custom
Contract.make {
    request {
        method POST()
        url "/products"
        body([name: "Laptop", category: "TECH", price: 999.99])
    }
    response {
        status CREATED()
        body([id: 1, name: "Laptop", status: "ACTIVE"])
        bodyMatchers {
            // byCommand invoca assertStatusIsValid con el valor real de $.status
            // '$it' es el marcador del valor actual del campo en el DSL
            jsonPath('$.status', byCommand('assertStatusIsValid($it)'))
        }
    }
}
```

```java
// Clase base del productor — debe definir el método referenciado en byCommand
public abstract class BaseProductTest {

    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(new ProductController());
    }

    // Método invocado por byCommand('assertStatusIsValid($it)')
    // Puede contener cualquier lógica de validación
    protected void assertStatusIsValid(String status) {
        assertThat(status).isIn("ACTIVE", "INACTIVE", "PENDING");
    }
}
```

> [CONCEPTO] `byCommand('methodName($it)')` funciona como un hook de validación en la clase base. El framework sustituye `$it` por el valor real del campo durante la ejecución del test del productor. El método debe ser accesible desde la clase base (no privado).

## Matchers en YAML

En YAML, los matchers se declaran en un bloque `matchers` separado bajo `request` o `response`. El campo `type` especifica el tipo de matcher y `value` el patrón o clase.

```yaml
response:
  status: 200
  body:
    id: 1
    uuid: "550e8400-e29b-41d4-a716-446655440000"
    price: 999.99
    createdAt: "2024-01-15T10:30:00Z"
  matchers:
    body:
      - path: $.id
        type: by_regex
        value: "[0-9]+"
      - path: $.uuid
        type: by_regex
        value: "[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}"
      - path: $.price
        type: by_type
      - path: $.createdAt
        type: by_timestamp
```

## Tabla de matchers disponibles

La siguiente tabla recoge todos los matchers de la API `BodyMatchers` con su descripción y caso de uso.

| Matcher Groovy | Tipo YAML | Valida | Ejemplo de uso |
|---|---|---|---|
| `byRegex("pattern")` | `by_regex` | Cadena que cumple la regex | `byRegex("[0-9]+")` para IDs numéricos |
| `byType()` | `by_type` | Tipo Java del campo | `byType()` para cualquier número |
| `byUuid()` | `by_regex` (uuid regex) | Formato UUID estándar | `byUuid()` para IDs UUID |
| `byTimestamp()` | `by_timestamp` | Datetime ISO 8601 | `byTimestamp()` para campos de fecha-hora |
| `byDate()` | `by_date` | Fecha ISO 8601 (YYYY-MM-DD) | `byDate()` para fechas sin hora |
| `byTime()` | `by_time` | Hora ISO 8601 (HH:MM:SS) | `byTime()` para campos de hora |
| `byCommand("method($it)")` | `by_command` | Delegado a método custom | `byCommand("validate($it)")` |
| `equalToJson("{ }")` | `by_equality` | Igual al JSON exacto | Para objetos anidados completos |

## Diferencia productor vs consumidor en matchers

El uso del matcher determina qué validación se aplica en cada lado del contrato.

```
PRODUCTOR (test generado):
──────────────────────────────────────────────────────────────
body.price = 999.99 (en el stub)
bodyMatchers: jsonPath('$.price', byType())
→ El test llama al productor real y verifica que $.price sea un número

CONSUMIDOR (stub WireMock):
──────────────────────────────────────────────────────────────
body.price = 999.99 (valor fijo del stub)
→ El stub devuelve SIEMPRE 999.99 al consumidor
→ El consumidor puede asumir que el campo price es un número
──────────────────────────────────────────────────────────────
```

> [ADVERTENCIA] Si un campo tiene `byType()` en `bodyMatchers` pero el productor real devuelve una cadena donde se espera un número, el test del productor fallará. El cuerpo del stub (valores fijos en `body`) no afecta este fallo — el fallo ocurre en el test del productor, no en el consumidor.

## Buenas y malas prácticas

**Buenas prácticas**:
- Usar `byUuid()`, `byTimestamp()`, `byDate()` para campos de formato conocido en lugar de escribir regex complejos.
- Usar `byCommand()` solo para validaciones de negocio que no tienen un matcher predefinido.
- Definir valores representativos en `body` (no vacíos) aunque los matchers los validen dinámicamente — el stub los devuelve al consumidor.
- Aplicar `bodyMatchers` en el bloque `request` cuando el stub WireMock debe validar el body de la petición entrante.

**Malas prácticas**:
- Omitir `bodyMatchers` para campos UUID, timestamp o IDs dinámicos — los tests del productor fallarán con valores reales.
- Usar `equalToJson` para campos que tienen subcampos dinámicos — rompe en valores reales.
- Confundir `byType()` (valida el tipo Java) con `byRegex()` (valida el patrón de la cadena) — `byType()` no valida formato, solo tipo.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia entre el valor del campo en `body` y la validación en `bodyMatchers` en un contrato Spring Cloud Contract?

> [EXAMEN] 2. ¿Qué matcher se usa para validar que un campo UUID tiene el formato estándar en el response del productor?

> [EXAMEN] 3. ¿Cómo funciona `byCommand('assertValid($it)')` y dónde se debe definir el método `assertValid`?

> [EXAMEN] 4. ¿Cómo se expresa un matcher `byType()` en un contrato YAML?

> [EXAMEN] 5. ¿En qué bloque se colocan los `bodyMatchers` cuando se quiere validar el body del request (no del response) en un contrato HTTP?

---

← [10.8 Repositorio de Stubs](sc-contract-stubs-repo.md) | [Índice](README.md) | [10.10 Contratos GraphQL](sc-contract-graphql.md) →
