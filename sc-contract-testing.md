# 10.12 Testing / Verificación de Spring Cloud Contract

← [10.11 Soporte GraphQL en Spring Cloud Contract](sc-contract-graphql.md) | [Índice (README.md)](README.md) | [11.1 Fundamentos de Spring Cloud Task](sc-task-fundamentos.md) →

---

Spring Cloud Contract divide la verificación en dos responsabilidades complementarias: el producer demuestra que su implementación satisface el contrato mediante tests generados automáticamente; el consumer demuestra que su código funciona cuando el servicio se comporta como promete el contrato usando el stub JAR. Estos dos pasos no son opcionales — si el producer no verifica sus contratos, los stubs no son fiables; si el consumer no usa los stubs en sus tests, los contratos no protegen la interfaz real que usa. Este fichero cubre las tres estrategias de verificación: test unitario del consumer con stub local, test del producer con base class y test del consumer con StubRunner remoto, y la verificación end-to-end del ciclo CDC con Testcontainers.

> [PREREQUISITO] Requiere `spring-cloud-starter-contract-verifier` en el producer y `spring-cloud-starter-contract-stub-runner` en el consumer. Para tests de integración con broker real: `org.testcontainers:testcontainers` y Docker disponible.

## Niveles de verificación CDC

Cada nivel verifica un aspecto distinto del ciclo CDC. La cobertura completa requiere los tres.

| Nivel | Quién lo ejecuta | Qué verifica |
|---|---|---|
| 1 — Tests del producer (generados) | Pipeline del producer | Producer implementa el contrato correctamente |
| 2 — Tests del consumer con StubRunner | Pipeline del consumer | Consumer funciona contra la interfaz prometida |
| 3 — Verificación CDC end-to-end | Pipeline integrado / Testcontainers | El ciclo completo: publicación, consumo y compatibilidad |

## Estrategia 1: Tests generados del producer

El plugin Maven/Gradle genera tests a partir de los contratos. Cada contrato produce un método de test que verifica que el endpoint del producer devuelve exactamente lo especificado en el contrato.

```java
// src/test/java/com/example/producer/ContractBase.java
// Clase base que el plugin usa para los tests generados
package com.example.producer;

import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest
@AutoConfigureMockMvc
public abstract class ContractBase {

    @Autowired
    private MockMvc mockMvc;

    @BeforeEach
    void setup() {
        RestAssuredMockMvc.mockMvc(mockMvc);
    }
}
```

```xml
<!-- pom.xml del producer: configuración del plugin -->
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <extensions>true</extensions>
    <configuration>
        <!-- El plugin usa esta clase como base para los tests generados -->
        <baseClassForTests>
            com.example.producer.ContractBase
        </baseClassForTests>
        <!-- Modo MockMvc: no requiere servidor HTTP real -->
        <testMode>MOCKMVC</testMode>
    </configuration>
</plugin>
```

El test generado (no se edita manualmente) tiene este aspecto:

```java
// target/generated-test-sources/.../ContractVerifierTest.java (generado)
// Este test lo crea el plugin; se ejecuta en `mvn verify`
public class ContractVerifierTest extends ContractBase {

    @Test
    public void validate_getOrderById() throws Exception {
        // RestAssured realiza la petición al MockMvc configurado en ContractBase
        // y verifica que el response coincide con el contrato
        Response response = given()
            .header("Content-Type", "application/json")
            .when()
            .get("/orders/1");

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.header("Content-Type"))
            .matches("application/json.*");
        DocumentContext parsedJson =
            JsonPath.parse(response.getBody().asString());
        assertThatJson(parsedJson).field("['id']").isEqualTo(1);
        assertThatJson(parsedJson).field("['total']").matches("[0-9]+\\.?[0-9]*");
    }
}
```

## Estrategia 2: Tests del consumer con `@AutoConfigureStubRunner`

El consumer usa `@AutoConfigureStubRunner` para cargar el stub JAR del producer. El stub simula el producer con las respuestas definidas en los contratos. Este es el test que verifica que el consumer usa la API del producer de forma compatible.

```xml
<!-- pom.xml del consumer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.consumer;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.StubRunnerProperties;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureStubRunner(
    // Especificar groupId:artifactId:version:classifier:puerto
    ids = "com.example:order-service:+:stubs:8090",
    // LOCAL = stubs desde ~/.m2; REMOTE = desde repositorio de artefactos
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderConsumerTest {

    @Autowired
    private OrderServiceClient orderServiceClient;

    @Test
    void whenGetOrder_thenReturnsOrderFromStub() {
        // WireMock está levantado en localhost:8090 con los mappings del stub JAR
        Order order = orderServiceClient.getOrder("1");

        assertThat(order).isNotNull();
        assertThat(order.getId()).isEqualTo(1);
        assertThat(order.getTotal()).isGreaterThan(0);
    }

    @Test
    void whenGetNonExistingOrder_thenReturnsNull() {
        // Si no hay stub que matchee esta petición, WireMock devuelve 404
        // El test verifica que el consumer maneja el 404 correctamente
        Order order = orderServiceClient.getOrder("999");

        assertThat(order).isNull();
    }
}
```

> [CONCEPTO] `StubsMode.LOCAL` busca el stub JAR en `~/.m2/repository`. Para que funcione, el producer debe haber ejecutado `mvn install` antes de que el consumer ejecute sus tests. En CI, esto se logra orquestando el pipeline del producer antes del del consumer, o publicando el JAR en un repositorio de artefactos y usando `StubsMode.REMOTE`.

## Estrategia 3: Verificación end-to-end del ciclo CDC con Testcontainers

Cuando el proyecto necesita verificar el ciclo CDC completo —incluyendo que los contratos en Git son consistentes con la implementación del producer y con el uso del consumer— un test de integración con Testcontainers puede orquestar los dos servicios reales.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.cdc;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.StubRunnerProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

// Test que verifica la compatibilidad CDC antes de hacer merge al trunk
// Se ejecuta después de que el producer haya publicado el stub JAR en CI
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE,
    properties = {
        // Apuntar al stub JAR publicado en el pipeline del producer
        "spring.cloud.contract.stub-runner.stubs-mode=REMOTE",
        "spring.cloud.contract.stub-runner.repository-root=https://repo.example.com/repository/maven-releases/"
    }
)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:LATEST.RELEASE:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.REMOTE
)
class CdcCompatibilityTest {

    @Autowired
    private OrderServiceClient orderServiceClient;

    @Test
    void consumerIsCompatibleWithLatestProducerContract() {
        // Este test falla si el producer publicó un contrato que rompe al consumer
        Order order = orderServiceClient.getOrder("1");

        assertThat(order.getId()).isNotNull();
        assertThat(order.getTotal()).isGreaterThanOrEqualTo(0.0);
    }
}
```

### Tabla resumen: cuándo usar cada estrategia

| Situación | Estrategia recomendada |
|---|---|
| Verificar que el producer implementa el contrato | 1 — Tests generados del producer (`mvn verify`) |
| Verificar que el consumer usa la API correctamente | 2 — @AutoConfigureStubRunner con StubsMode.LOCAL |
| Verificar compatibilidad antes de merge | 2 o 3 — StubRunner con versión fija o LATEST.RELEASE |
| Detectar breaking changes del producer en el pipeline del consumer | 3 — StubRunner REMOTE apuntando al artefacto del producer |
| Pipeline de CDC puro (contracts en Git compartido) | 1 + 2 orquestados: producer publica, consumer descarga |

## Tabla de elementos clave

| Componente / Propiedad | Descripción |
|---|---|
| `spring-cloud-contract-maven-plugin` | Genera los tests del producer a partir de los contratos; goal `generateTests` en fase `generate-test-sources` |
| `baseClassForTests` | Clase abstracta que configura MockMvc/RestAssured para los tests generados |
| `testMode=MOCKMVC` | Los tests generados usan MockMvc; alternativas: `WEBTESTCLIENT` (WebFlux), `EXPLICIT` (RestAssured real HTTP) |
| `@AutoConfigureStubRunner` | Anotación del consumer que levanta WireMock con los stubs del producer |
| `stubsMode=LOCAL` | Carga stubs desde `~/.m2`; requiere `mvn install` previo en el producer |
| `stubsMode=REMOTE` | Descarga stubs desde repositorio Maven; requiere `repository-root` configurado |
| `ids = "g:a:v:classifier:port"` | Formato del identificador de stub; `+` como version = última instalada; `LATEST.RELEASE` = último release en remoto |
| `mvn verify` en el producer | Ejecuta los tests generados por el plugin; genera el stub JAR en `target/` |
| `mvn install` en el producer | Publica el stub JAR en `~/.m2` para que `StubsMode.LOCAL` funcione |

## Buenas y malas prácticas

**Hacer:**
- Ejecutar `mvn install` (o `./gradlew publishToMavenLocal`) en el producer antes de los tests del consumer en pipelines locales o de CI sin repositorio de artefactos; si el stub JAR no está en `~/.m2`, StubRunner fallará con "unable to find stub".
- Usar `testMode=MOCKMVC` para la mayoría de contratos REST: es el modo más rápido y no requiere servidor HTTP. Cambiar a `testMode=EXPLICIT` solo cuando el contrato involucra comportamiento que MockMvc no puede simular (filtros de Servlet, comportamiento de red).
- Configurar `StubsMode.REMOTE` con `LATEST.RELEASE` en el pipeline del consumer para detectar automáticamente breaking changes del producer; si el producer publica un contrato que rompe el consumer, el job del consumer falla en CI antes de llegar a producción.

**Evitar:**
- Saltar la ejecución de `mvn verify` en el producer y publicar el JAR directamente sin tests: invalida el propósito de CDC, ya que el stub puede no reflejar el comportamiento real del producer.
- Usar `StubsMode.LOCAL` con versión `+` (latest local) en CI multi-agente: distintos agentes pueden tener versiones diferentes instaladas en `~/.m2`, lo que produce tests no deterministas.
- Testear el consumer con mocks manuales (Mockito) del cliente HTTP en lugar de StubRunner: los mocks manuales no están ligados al contrato y no detectan roturas cuando el producer cambia su API.

---

← [10.11 Soporte GraphQL en Spring Cloud Contract](sc-contract-graphql.md) | [Índice (README.md)](README.md) | [11.1 Fundamentos de Spring Cloud Task](sc-task-fundamentos.md) →
