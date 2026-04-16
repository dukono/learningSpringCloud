# 10.11 Soporte GraphQL en Spring Cloud Contract

← [10.10 Compatibilidad, versionado y troubleshooting de contratos](sc-contract-troubleshooting.md) | [Índice (README.md)](README.md) | [10.12 Testing / Verificación de Spring Cloud Contract](sc-contract-testing.md) →

---

Spring Cloud Contract 4.x incluye soporte experimental para contratos GraphQL: un contrato puede describir una query o mutation GraphQL y generar tanto el test del producer como el stub WireMock para el consumer. El soporte es más limitado que el de REST — no existe DSL específico de GraphQL, sino una adaptación del DSL HTTP que mapea la petición como POST al endpoint `/graphql` con body estructurado como `{ "query": "...", "variables": {...} }`. El valor principal es mantener el contrato como fuente de verdad para operaciones GraphQL del mismo modo que para APIs REST, con la misma mecánica de stub JARs y StubRunner.

> [PREREQUISITO] Requiere `spring-cloud-starter-contract-verifier` en el producer y `spring-cloud-starter-contract-stub-runner` en el consumer. El servidor GraphQL puede ser cualquier implementación Spring (Spring for GraphQL 1.x), ya que el contrato interactúa a nivel HTTP POST contra `/graphql`.

## Estructura del contrato GraphQL

Los contratos GraphQL se expresan como contratos HTTP donde el body de la petición sigue el formato estándar de GraphQL over HTTP. El producer genera un test que realiza la petición al controlador, y el consumer recibe un stub WireMock que simula la respuesta.

```
contracts/
  graphql/
    get-product-query.groovy    ← contrato para una query
    create-order-mutation.groovy ← contrato para una mutation
```

## Ejemplo central: contrato de query y mutation GraphQL

El contrato mapea la operación GraphQL a una petición HTTP POST con body JSON. El DSL es el mismo DSL Groovy de contratos HTTP.

### Dependencias Maven

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Producer: verifier genera el test -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Spring for GraphQL -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-graphql</artifactId>
    </dependency>
    <!-- Consumer: StubRunner usa el stub JAR -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Contrato para una query GraphQL

```groovy
// contracts/graphql/get-product-query.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Query GraphQL: obtener producto por ID"
    request {
        method POST()
        url '/graphql'
        headers {
            contentType(applicationJson())
        }
        body([
            query: 'query GetProduct($id: ID!) { product(id: $id) { id name price } }',
            variables: [id: '42']
        ])
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            data: [
                product: [
                    id: '42',
                    name: value(consumer('Laptop Pro'), producer(regex('[A-Za-z ]+'))),
                    price: value(consumer(999.99), producer(regex('[0-9]+\\.?[0-9]*')))
                ]
            ]
        ])
    }
}
```

### Contrato para una mutation GraphQL

```groovy
// contracts/graphql/create-order-mutation.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Mutation GraphQL: crear pedido"
    request {
        method POST()
        url '/graphql'
        headers {
            contentType(applicationJson())
        }
        body([
            query: 'mutation CreateOrder($input: OrderInput!) { createOrder(input: $input) { id status } }',
            variables: [
                input: [
                    productId: '42',
                    quantity: 2
                ]
            ]
        ])
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            data: [
                createOrder: [
                    id: value(consumer('order-123'), producer(regex('[a-z0-9-]+'))),
                    status: 'CREATED'
                ]
            ]
        ])
    }
}
```

### Clase base del producer para contratos GraphQL

```java
package com.example.graphql;

import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

// La clase base se configura como graphql en pom.xml:
// <baseClassForTests>com.example.graphql.GraphQlContractBase</baseClassForTests>
@SpringBootTest
@AutoConfigureMockMvc
public abstract class GraphQlContractBase {

    @Autowired
    private MockMvc mockMvc;

    @BeforeEach
    void setup() {
        RestAssuredMockMvc.mockMvc(mockMvc);
    }
}
```

```yaml
# application.yml del consumer usando el stub GraphQL
spring:
  cloud:
    contract:
      stub-runner:
        ids:
          - com.example:product-service:+:stubs:8090
        stubs-mode: LOCAL
```

> [CONCEPTO] El stub WireMock generado por un contrato GraphQL registra un mapping que coincide con el body POST exacto (incluyendo la query string). Si el consumer usa una query diferente (aunque semánticamente equivalente, por ejemplo con distintos campos solicitados), el stub no matcheará — es la principal limitación del soporte GraphQL.

> [ADVERTENCIA] El soporte GraphQL en Spring Cloud Contract 4.x es funcional pero no diferencia entre errores GraphQL (`{ "errors": [...] }`) y errores HTTP. Para contratos que verifican respuestas de error GraphQL (con `errors` en el body y status 200), hay que expresar el response body completo con el campo `errors` mapeado explícitamente.

## Tabla de elementos clave

| Elemento | Descripción |
|---|---|
| Endpoint `/graphql` | Todos los contratos GraphQL usan POST a `/graphql`; el tipo de operación (query/mutation) va dentro del body |
| `body([query: '...', variables: [...]])` | Estructura del request body para GraphQL over HTTP en el DSL de contratos |
| `data: [...]` en response body | Estructura estándar de respuesta GraphQL; el contrato debe modelarla completa |
| `value(consumer(...), producer(...))` | Permite valores distintos en el stub y en el test generado; útil para valores dinámicos (IDs, timestamps) |
| `regex(...)` en producer | El test generado valida con regex el valor real devuelto por el producer |
| Stub JAR con GraphQL mappings | El JAR de stubs contiene mappings WireMock que coinciden con el body POST exacto; menos flexible que REST |

## Buenas y malas prácticas

**Hacer:**
- Usar `value(consumer('valor-fijo'), producer(regex('...')))` en los campos del response que varían entre llamadas (IDs generados, timestamps): el stub devuelve un valor fijo predecible y el test del producer valida con el regex.
- Organizar los contratos GraphQL en un subdirectorio `contracts/graphql/` separado de los contratos REST; facilita la configuración de `baseClassForTests` por paquete.
- Incluir el campo `variables` en la petición del contrato siempre que la query/mutation los use; un stub sin variables solo matcheará requests que envíen el mismo body exacto.

**Evitar:**
- Usar contratos GraphQL para verificar operaciones con fragmentos, aliases, o introspección: el soporte 4.x no incluye parsing del lenguaje GraphQL — el contrato compara strings literales.
- Compartir la clase base con contratos REST cuando los contratos GraphQL requieren configuración específica del contexto MockMvc con GraphQL; usar herencia o clases base separadas por paquete.
- Ignorar el formato `{ "data": {...} }` en el response del contrato: si el contrato devuelve el body sin la envoltura `data`, el stub generado no matcheará la respuesta real del servidor GraphQL.

---

← [10.10 Compatibilidad, versionado y troubleshooting de contratos](sc-contract-troubleshooting.md) | [Índice (README.md)](README.md) | [10.12 Testing / Verificación de Spring Cloud Contract](sc-contract-testing.md) →
