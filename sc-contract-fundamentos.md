# 10.1 Spring Cloud Contract — Fundamentos CDC

← [9.10 Spring Cloud Kubernetes — Testing](sc-kubernetes-testing.md) | [Índice](README.md) | [10.2 DSL Groovy y YAML](sc-contract-dsl.md) →

---

## Introducción

Consumer-Driven Contract Testing (CDC) es una técnica de verificación de contratos entre microservicios donde el **consumidor** define sus expectativas sobre el productor en forma de contratos, y el **productor** las verifica automáticamente. Spring Cloud Contract implementa este patrón generando tests en el lado del productor y stubs WireMock en el lado del consumidor, permitiendo que ambos servicios se prueben de forma completamente aislada.

> [CONCEPTO] Un **contrato** en Spring Cloud Contract es un fichero (Groovy o YAML) que describe una interacción: dado un request concreto, el productor debe devolver una respuesta específica. El contrato es la única fuente de verdad compartida entre productor y consumidor.

## Flujo productor-consumidor

El flujo CDC con Spring Cloud Contract sigue cuatro etapas claramente diferenciadas que garantizan el aislamiento de los tests de ambas partes.

```
Consumidor                    Contrato                   Productor
─────────                     ────────                   ─────────
  define              →    .groovy / .yaml   →         verifica
  expectativas              (fuente única                genera tests
                              de verdad)                  genera stubs
                                 │
                                 ▼
                          stubs.jar publicado
                                 │
                                 ▼
Consumidor  ←──────────  Stub Runner descarga
tests con                   y ejecuta stubs
stubs locales              WireMock en puerto local
```

El productor ejecuta `mvn verify`, que genera tests automáticos a partir de los contratos. Si los tests pasan, el productor empaqueta los stubs en un JAR clasificador y los publica en el repositorio de artefactos. El consumidor, en su CI, descarga ese JAR y usa `@AutoConfigureStubRunner` para levantar un servidor WireMock local que responde exactamente según el contrato.

> [PREREQUISITO] Para trabajar con Spring Cloud Contract se necesita conocer conceptos básicos de testing con JUnit 5 y MockMvc/RestAssured en Spring Boot. No es necesario conocer WireMock directamente, ya que SCC gestiona su configuración.

## Diferencia con integration testing end-to-end

Los tests de integración end-to-end requieren que todos los servicios involucrados estén desplegados y disponibles simultáneamente. Esta dependencia genera tests lentos, frágiles y difíciles de mantener en entornos de CI/CD con muchos microservicios.

| Criterio | Integration Test E2E | Consumer-Driven Contract Testing |
|---|---|---|
| Servicios requeridos | Todos desplegados | Solo el servicio bajo test |
| Velocidad de ejecución | Lenta (minutos) | Rápida (segundos) |
| Entorno necesario | Infraestructura completa | Local / CI básico |
| Detección de rotura | Al final del pipeline | En el build del productor |
| Quién define el contrato | Nadie / acuerdo informal | El consumidor formalmente |
| Mantenimiento | Alto — cambios en cualquier servicio rompen | Bajo — contratos versionados |

> [EXAMEN] La clave diferencial es que en CDC el consumidor **define** el contrato, no el productor. Si el productor cambia su API de forma incompatible, los tests generados fallarán **en el build del productor**, antes de que el artefacto llegue a ningún entorno.

## Roles y responsabilidades

Cada parte tiene un rol estricto en el proceso CDC. Comprender estos roles es fundamental para entender por qué el testing contractual funciona.

El **consumidor** es el servicio que llama a otro servicio. Es responsable de escribir los contratos que expresan qué necesita del productor — ni más ni menos. Un consumidor que sobrespecifica el contrato (validando campos que no usa) crea fragilidad innecesaria.

El **productor** es el servicio que expone la API. Es responsable de mantener los contratos en su repositorio bajo `src/test/resources/contracts/`, ejecutar los tests generados y publicar los stubs resultantes. El productor no escribe los contratos; los acepta del consumidor.

> [ADVERTENCIA] En la práctica, los contratos pueden vivir en el repositorio del productor o en un repositorio compartido. Spring Cloud Contract soporta ambos enfoques mediante la propiedad `repositoryRoot`. El enfoque más común es que los contratos vivan en el repo del productor bajo control de versiones.

## Beneficios del aislamiento

El aislamiento que proporciona CDC resuelve varios problemas concretos de los microservicios.

**Detección temprana de incompatibilidades**: cuando el productor modifica su contrato de forma incompatible (cambia un campo, elimina un endpoint), los tests generados fallan inmediatamente en el build del productor, sin necesidad de desplegar nada.

**Tests del consumidor deterministas**: el consumidor prueba contra un stub que refleja exactamente el comportamiento acordado, sin depender de disponibilidad de red, datos de prueba compartidos ni versiones del productor en entornos.

**Documentación ejecutable**: los contratos son documentación que siempre está actualizada porque es verificada automáticamente en cada build.

## Ejemplo central

El siguiente ejemplo muestra el flujo mínimo completo: un contrato HTTP, el fragmento de configuración del productor y el test del consumidor con Stub Runner.

```groovy
// Contrato: src/test/resources/contracts/order/shouldReturnOrder.groovy
// (reside en el repositorio del PRODUCTOR)
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return an order by id"
    request {
        method GET()
        url "/orders/1"
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            id    : 1,
            status: "CONFIRMED"
        ])
    }
}
```

```java
// Clase base en el PRODUCTOR (src/test/java/.../BaseOrderTest.java)
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public abstract class BaseOrderTest {

    @Autowired
    OrderController orderController;

    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(orderController);
    }
}
```

```java
// Test del CONSUMIDOR (src/test/java/.../OrderClientTest.java)
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.StubRunnerProperties;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
public class OrderClientTest {

    @Autowired
    OrderClient orderClient;  // cliente HTTP que llama a localhost:8090

    @Test
    void shouldRetrieveConfirmedOrder() {
        Order order = orderClient.getOrder(1L);
        assertThat(order.getId()).isEqualTo(1L);
        assertThat(order.getStatus()).isEqualTo("CONFIRMED");
    }
}
```

## Tabla de elementos clave

Los siguientes son los artefactos y anotaciones centrales que todo desarrollador debe conocer para trabajar con Spring Cloud Contract.

| Elemento | Rol | Dónde se usa |
|---|---|---|
| Fichero `.groovy` / `.yaml` | Define el contrato | Repositorio del productor |
| `spring-cloud-contract-maven-plugin` | Genera tests y stubs en el productor | `pom.xml` del productor |
| `@AutoConfigureStubRunner` | Activa Stub Runner en el consumidor | Test class del consumidor |
| `stubs.jar` | JAR con stubs WireMock empaquetados | Nexus/Artifactory/Git |
| `RestAssuredMockMvc.standaloneSetup()` | Configura contexto en clase base | Clase base del productor |
| `StubRunnerProperties.StubsMode` | Controla cómo se resuelven los stubs | `@AutoConfigureStubRunner` |

## Buenas y malas prácticas

**Buenas prácticas**:
- El consumidor escribe contratos con solo los campos que realmente usa, sin sobrespecificar la respuesta.
- El productor mantiene los contratos bajo control de versiones con el mismo ciclo de vida que el código.
- Se versiona el JAR de stubs con semántica clara (MAJOR.MINOR.PATCH) para que el consumidor pueda fijar versiones compatibles.
- Se usan matchers dinámicos (`byType`, `byRegex`) para campos no deterministas como timestamps o UUIDs.

**Malas prácticas**:
- Copiar el JSON completo de una respuesta real como body esperado sin usar matchers — rompe con cualquier campo opcional nuevo.
- Mantener contratos en un repositorio separado sin integración continua — los contratos se vuelven obsoletos.
- Usar `stubsMode = REMOTE` en CI sin cache — genera dependencia de red y tests lentos.
- Escribir el contrato desde el productor en lugar de desde el consumidor — pierde el beneficio del enfoque CDC.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia fundamental entre Consumer-Driven Contract Testing y un test de integración end-to-end tradicional?

> [EXAMEN] 2. ¿Quién es responsable de escribir los contratos en CDC: el productor o el consumidor? ¿Por qué?

> [EXAMEN] 3. Describe las cuatro etapas del flujo CDC con Spring Cloud Contract: desde la definición del contrato hasta la ejecución del test del consumidor.

> [EXAMEN] 4. ¿Qué contiene el JAR clasificador `stubs` que genera el plugin de Spring Cloud Contract?

> [EXAMEN] 5. ¿Qué ventaja tiene CDC respecto al e2e testing en cuanto a la detección temprana de incompatibilidades entre servicios?

---

← [9.10 Spring Cloud Kubernetes — Testing](sc-kubernetes-testing.md) | [Índice](README.md) | [10.2 DSL Groovy y YAML](sc-contract-dsl.md) →
