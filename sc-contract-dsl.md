# 10.4.1 DSL Groovy/YAML — estructura base de contratos HTTP

← [10.3.2 Plugin Maven/Gradle — propiedades de personalización](sc-contract-plugin-config.md) | [Índice](README.md) | [10.4.2 Matchers estándar en contratos](sc-contract-matchers.md) →

---

## Introducción

El DSL de Spring Cloud Contract es el lenguaje con el que un equipo consumer formaliza sus expectativas sobre el producer. Un contrato mal escrito —con valores hardcoded donde debería haber matchers, o con matchers demasiado permisivos donde se necesita precisión— produce stubs que no reproducen el comportamiento real del producer y tests de productor que pasan aunque la implementación sea incorrecta. Dominar la estructura base del DSL, tanto en Groovy como en YAML, es la habilidad fundamental del ciclo CDC: todo lo demás —matchers avanzados, mensajería, troubleshooting— parte de entender bien este bloque estructural.

> [CONCEPTO] **Dynamic values**: el DSL permite definir valores distintos para el lado del consumer (stub) y para el lado del producer (test generado) usando `$(consumer(...), producer(...))`. Esto permite que el stub devuelva un valor concreto al consumer mientras el test del producer verifica un patrón más flexible.

> [PREREQUISITO] Familiaridad con Groovy básico (closures, mapas literales) y con la estructura de una petición/respuesta HTTP REST. Conocimiento de los goals del plugin (10.3.1).

## Representación visual

El diagrama muestra la dualidad del contrato: un único fichero genera dos artefactos con roles distintos.

```
CONTRATO DSL (Groovy o YAML)
════════════════════════════
        │
        ├──────────────────────────────────────────────────────┐
        ▼                                                      ▼
LADO PRODUCER                                         LADO CONSUMER
(test generado por el plugin)                         (mapping WireMock en Stub JAR)
─────────────────────────────                         ─────────────────────────────
Test JUnit 5 que verifica:                            Servidor WireMock que devuelve:

  request enviado:                                      cuando recibe:
  GET /api/users/1                                      GET /api/users/1
  Accept: application/json                              Accept: application/json

  response esperada:                                    responde:
  status 200                                            status 200
  Content-Type: application/json                        Content-Type: application/json
  body.id == 1 (producer value)                         body = { id: 1, name: "Alice",
  body.name != null (matcher)                                     email: "alice@example.com" }
  body.email matches [a-z]+@[a-z]+\.[a-z]+             (consumer value concreto)
```

## Ejemplo central

El ejemplo muestra la misma definición de contrato en Groovy DSL y YAML DSL, con todos los bloques principales: request, response, dynamic values, priority e ignored.

**Contrato Groovy completo** (`src/test/resources/contracts/user/shouldReturnUser.groovy`):

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    // Descripción opcional pero recomendada para trazabilidad
    description "should return user by id when user exists"

    // Prioridad: WireMock usa el contrato de mayor prioridad cuando
    // varios contratos coinciden con el mismo request
    priority 1

    request {
        method GET()
        url '/api/users/1'
        headers {
            accept(applicationJson())
        }
    }

    response {
        status OK()  // 200
        headers {
            contentType(applicationJson())
        }
        body([
            id    : value(producer(1), consumer(1)),
            name  : value(producer(regex('[A-Za-z ]+')), consumer('Alice')),
            email : value(producer(regex('[a-z]+@[a-z]+\\.[a-z]+')), consumer('alice@example.com')),
            active: true
        ])
    }
}
```

**Contrato YAML equivalente** (`src/test/resources/contracts/user/shouldReturnUser.yml`):

```yaml
description: "should return user by id when user exists"
priority: 1
request:
  method: GET
  url: /api/users/1
  headers:
    Accept: application/json
response:
  status: 200
  headers:
    Content-Type: application/json
  body:
    id: 1
    name: Alice
    email: alice@example.com
    active: true
  matchers:
    body:
      - path: $.name
        type: by_regex
        value: "[A-Za-z ]+"
      - path: $.email
        type: by_regex
        value: "[a-z]+@[a-z]+\\.[a-z]+"
```

**Contrato con queryParameters y cuerpo de request (POST)** (`src/test/resources/contracts/user/shouldCreateUser.groovy`):

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should create user and return created resource"

    request {
        method POST()
        url '/api/users'
        headers {
            contentType(applicationJson())
            accept(applicationJson())
        }
        body([
            name : value(consumer('Bob'), producer(regex('[A-Za-z ]+'))),
            email: value(consumer('bob@example.com'), producer(regex('[a-z]+@[a-z]+\\.[a-z]+')))
        ])
    }

    response {
        status CREATED()  // 201
        headers {
            contentType(applicationJson())
        }
        body([
            id   : value(producer(regex('[0-9]+')), consumer('42')),
            name : fromRequest().body('$.name'),  // repite el valor del request
            email: fromRequest().body('$.email')
        ])
    }
}
```

**Contrato con queryParameters** (`src/test/resources/contracts/user/shouldSearchUsers.groovy`):

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should search users by status"

    request {
        method GET()
        urlPath('/api/users') {
            queryParameters {
                parameter('status', 'active')
                parameter('page', value(consumer('0'), producer(regex('[0-9]+'))))
                parameter('size', value(consumer('10'), producer(regex('[1-9][0-9]*'))))
            }
        }
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
            content: [[id: 1, name: 'Alice', status: 'active']],
            totalElements: value(producer(anyPositiveInt()), consumer(1)),
            page: 0,
            size: 10
        ])
    }
}
```

**Contrato marcado como ignored** (contrato en desarrollo, excluido de la generación):

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "DRAFT - endpoint not implemented yet"
    ignored()  // Este contrato no genera test ni stub

    request {
        method DELETE()
        url '/api/users/1'
    }

    response {
        status NO_CONTENT()  // 204
    }
}
```

**Contrato YAML con ignored:**

```yaml
description: "DRAFT - endpoint not implemented yet"
ignored: true
request:
  method: DELETE
  url: /api/users/1
response:
  status: 204
```

**application.yml del producer:**

```yaml
spring:
  application:
    name: user-service
server:
  port: 8080
```

**Clase base mínima del producer:**

```java
package com.example.producer;

import com.example.producer.controller.UserController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public abstract class BaseContractTest {

    @Autowired
    private UserController userController;

    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(userController);
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge las construcciones del DSL que todo profesional senior debe manejar de memoria.

| Construcción | DSL Groovy | DSL YAML | Descripción |
|-------------|-----------|----------|-------------|
| Método HTTP | `method GET()` | `method: GET` | Verbos: GET, POST, PUT, DELETE, PATCH, HEAD |
| URL fija | `url '/api/users/1'` | `url: /api/users/1` | URL exacta sin parámetros de query |
| URL con query params | `urlPath('/api/users') { queryParameters { ... } }` | `urlPath: /api/users` + `queryParameters:` | URL con parámetros de query dinámicos |
| Status HTTP | `status OK()` | `status: 200` | Constantes: OK, CREATED, NOT_FOUND, BAD_REQUEST, etc. |
| Cabeceras request | `headers { accept(applicationJson()) }` | `headers: { Accept: application/json }` | Cabeceras de la petición esperada |
| Cabeceras response | `headers { contentType(applicationJson()) }` | `headers: { Content-Type: application/json }` | Cabeceras de la respuesta devuelta |
| Body literal | `body([key: 'value'])` | `body: { key: value }` | Cuerpo con valores concretos |
| Dynamic value | `value(producer(...), consumer(...))` | `matchers.body[].type: by_regex` | Valor diferente en test del producer y en stub del consumer |
| fromRequest() | `fromRequest().body('$.field')` | `fromRequest: { body: '$.field' }` | Copia un campo del request en el response |
| priority | `priority 1` | `priority: 1` | Mayor número = mayor prioridad en WireMock |
| ignored | `ignored()` | `ignored: true` | Excluye el contrato de la generación |
| description | `description "texto"` | `description: texto` | Descripción legible del contrato |

## Buenas y malas prácticas

**Hacer:**

- Usar `description` en todos los contratos: el mensaje de error del test generado incluye la descripción, lo que acelera el diagnóstico cuando el test falla en CI.
- Usar `value(producer(...), consumer(...))` para campos que varían (UUIDs, timestamps, emails generados) en lugar de valores hardcoded: evita fallos de contrato por datos de test no predecibles.
- Preferir YAML para contratos escritos por teams de QA o equipos no familiarizados con Groovy; el Groovy DSL es más expresivo pero tiene una curva de aprendizaje mayor.
- Usar `urlPath` con `queryParameters` en lugar de construir la URL con la query string manualmente (`/api/users?status=active`): el plugin parsea los parámetros correctamente y genera matchers de query param en el WireMock mapping.
- Aplicar `priority` cuando múltiples contratos pueden coincidir con el mismo request (ej: contrato genérico y contrato específico para el mismo endpoint): sin prioridad, WireMock puede servir el contrato incorrecto.

**Evitar:**

- Evitar usar `url` con query params embebidos en la cadena (`url '/api/users?status=active'`): WireMock trata la query string como parte de la URL y el matching puede fallar si el orden de los parámetros difiere.
- Evitar contratos con cuerpos totalmente hardcoded para campos variables: un UUID diferente en cada ejecución hace que el test del producer falle de forma intermitente.
- Evitar mezclar Groovy DSL y YAML DSL para el mismo dominio de contratos: aunque ambos son soportados en el mismo directorio, dificulta la revisión de contratos en PRs y el onboarding de nuevos desarrolladores.
- Evitar usar `ignored()` como solución permanente para contratos que no pasan: el contrato ignorado deja de aportar valor al ciclo CDC y acumula deuda técnica de testing no verificada.
- Evitar omitir cabeceras obligatorias del request en el contrato: si el producer requiere `Content-Type: application/json` en un POST y el contrato no lo incluye, el test generado enviará el request sin esa cabecera y la respuesta real puede ser un 415 Unsupported Media Type.

## Comparación: Groovy DSL vs YAML DSL

| Característica | Groovy DSL | YAML DSL |
|---------------|-----------|----------|
| Expresividad | Alta (closures, métodos, lógica) | Media (declarativo puro) |
| Curva de aprendizaje | Mayor (requiere Groovy básico) | Menor (legible por no programadores) |
| Matchers inline | `value(producer(...), consumer(...))` | Separados en sección `matchers:` |
| Validación en IDE | Sí (con plugin Groovy) | Sí (con schema JSON) |
| Contratos complejos | Más conciso | Más verboso |
| Recomendación | Desarrolladores Java/Groovy | Equipos mixtos o QA |

> [EXAMEN] Pregunta de entrevista: "¿Cuál es la diferencia entre `url` y `urlPath` en el DSL de Spring Cloud Contract?" — `url` define una URL exacta (path + query string como string). `urlPath` define solo el path y permite declarar `queryParameters` como elementos individuales con sus propios matchers. Usar `urlPath` es la práctica correcta para endpoints con parámetros de query.

> [ADVERTENCIA] En proyectos multi-módulo donde los contratos están en un submódulo separado, la propiedad `contractsDirectory` debe apuntar a la ruta absoluta o relativa correcta desde el módulo del producer. Una ruta incorrecta hace que el plugin no encuentre contratos y genere cero tests, lo que no produce error de build sino silencio, pasando todos los checks de CI sin verificación real.

---

← [10.3.2 Plugin Maven/Gradle — propiedades de personalización](sc-contract-plugin-config.md) | [Índice](README.md) | [10.4.2 Matchers estándar en contratos](sc-contract-matchers.md) →
