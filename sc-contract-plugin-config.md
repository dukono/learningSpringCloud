# 10.5 Spring Cloud Contract — Plugin Maven y Gradle

← [10.4 Contratos Mensajería](sc-contract-mensajeria.md) | [Índice](README.md) | [10.6 Clases Base Productor](sc-contract-base-class.md) →

---

## Introducción

El `spring-cloud-contract-maven-plugin` y su equivalente Gradle son el núcleo del lado productor en Spring Cloud Contract. Estos plugins leen los contratos del directorio configurado, generan automáticamente tests JUnit 5 (o Spock) que verifican que el productor cumple los contratos, y empaquetan los stubs WireMock resultantes en un JAR clasificador para que el consumidor los descargue. Conocer todas las propiedades del plugin y las fases que ejecuta es contenido de examen.

> [PREREQUISITO] Este nodo requiere haber comprendido el flujo productor-consumidor de [10.1 Fundamentos CDC](sc-contract-fundamentos.md).

## Configuración del plugin Maven

La configuración mínima del plugin en Maven requiere declararlo en la sección `<build><plugins>` del `pom.xml` del productor. El siguiente ejemplo muestra todas las propiedades relevantes con sus valores por defecto y descripción.

```xml
<!-- pom.xml del PRODUCTOR -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-contract-maven-plugin</artifactId>
            <!-- La versión la gestiona el BOM de Spring Cloud -->
            <extensions>true</extensions>
            <configuration>
                <!--
                  contractsDslDir: directorio donde el plugin busca los contratos.
                  DEFAULT: src/test/resources/contracts
                  Se puede cambiar si los contratos están en otro directorio.
                -->
                <contractsDslDir>${project.basedir}/src/test/resources/contracts</contractsDslDir>

                <!--
                  baseClassForTests: clase base que extienden TODOS los tests generados.
                  Se usa cuando todos los contratos comparten la misma clase base.
                  NO se puede combinar con baseClassMappings.
                -->
                <baseClassForTests>com.example.BaseContractTest</baseClassForTests>

                <!--
                  baseClassMappings: permite mapear distintos contratos a distintas
                  clases base según el paquete/directorio del contrato (regex sobre el nombre del contrato).
                  Tiene prioridad sobre baseClassForTests.
                -->
                <!--
                <baseClassMappings>
                    <baseClassMapping>
                        <contractPackageRegex>.*order.*</contractPackageRegex>
                        <baseClassFQN>com.example.order.BaseOrderTest</baseClassFQN>
                    </baseClassMapping>
                    <baseClassMapping>
                        <contractPackageRegex>.*payment.*</contractPackageRegex>
                        <baseClassFQN>com.example.payment.BasePaymentTest</baseClassFQN>
                    </baseClassMapping>
                </baseClassMappings>
                -->

                <!--
                  generatedTestSourcesDir: directorio donde se generan los tests.
                  DEFAULT: ${project.build.directory}/generated-test-sources/contracts
                -->
                <generatedTestSourcesDir>
                    ${project.build.directory}/generated-test-sources/contracts
                </generatedTestSourcesDir>

                <!--
                  testFramework: framework para los tests generados.
                  Valores: JUNIT5 (default), SPOCK, JUNIT (JUnit 4 — [LEGACY])
                -->
                <testFramework>JUNIT5</testFramework>

                <!--
                  outputDirectory: directorio donde se empaquetan los stubs WireMock.
                  DEFAULT: ${project.build.directory}/stubs
                -->
                <outputDirectory>${project.build.directory}/stubs</outputDirectory>

                <!--
                  excludedFiles: patrones de contratos a excluir del procesamiento.
                -->
                <excludedFiles>
                    <excludedFile>**/draft/**</excludedFile>
                </excludedFiles>

                <!--
                  testMode: MOCKMVC (default), EXPLICIT (RestTemplate real), WEBTESTCLIENT (reactivo)
                -->
                <testMode>MOCKMVC</testMode>
            </configuration>
        </plugin>
    </plugins>
</build>

<!-- Dependencias necesarias en el productor -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
    <!--
      spring-cloud-contract-wiremock: módulo estándar del stub runner en SC 4.x.
      Incluye WireMock como motor de stubs. Es la dependencia que provee
      @AutoConfigureWireMock y el soporte de ejecución de stubs WireMock.
      NOTA: spring-cloud-contract-wiremock ≠ Spring Cloud Contract completo;
      es el módulo específico de WireMock del ecosistema SCC.
    -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-contract-wiremock</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> [CONCEPTO] `baseClassMappings` tiene prioridad sobre `baseClassForTests`. Cuando un proyecto tiene contratos de distintos dominios (orders, payments, notifications), cada grupo puede mapear a su propia clase base con una regex sobre el directorio del contrato. Esto es más flexible que una única clase base global.

## Fases del ciclo de vida Maven

El plugin se enlaza a dos fases del ciclo de vida Maven que se ejecutan automáticamente durante `mvn verify`.

```
mvn verify ejecuta en orden:
──────────────────────────────────────────────────────────────
generate-test-sources  →  generateTests
                           Lee contratos de contractsDslDir
                           Genera clases JUnit 5 en generatedTestSourcesDir
                           Las clases generadas extienden baseClassForTests

test-compile           →  Compila las clases generadas junto con las clases base

test                   →  Ejecuta los tests generados
                           Si algún test falla → el build falla → NO se publican stubs

package                →  generateStubs
                           Empaqueta los stubs WireMock en un JAR clasificador 'stubs'
                           Ubica el JAR en outputDirectory (target/stubs por defecto)
──────────────────────────────────────────────────────────────
```

> [ADVERTENCIA] El JAR de stubs solo se genera si `mvn package` o `mvn install` se ejecutan correctamente. En un pipeline CI típico se usa `mvn install` para que el JAR de stubs quede en el repositorio Maven local (útil para `StubsMode.LOCAL`) o `mvn deploy` para publicarlo en Nexus/Artifactory.

## Configuración del plugin Gradle

La configuración equivalente en Gradle usa el plugin `spring-cloud-contract` que se aplica al `build.gradle` del productor.

```groovy
// build.gradle del PRODUCTOR
plugins {
    id 'groovy'
    id 'org.springframework.cloud.contract' version "${springCloudContractVersion}"
}

contracts {
    // contractsDslDir: directorio de los contratos
    // DEFAULT: src/test/resources/contracts
    contractsDslDir = file("src/test/resources/contracts")

    // baseClassForTests: clase base global para todos los tests generados
    baseClassForTests = "com.example.BaseContractTest"

    // baseClassMappings: mapeo de contratos a clases base por regex
    // baseClassMappings {
    //     baseClassMapping(".*order.*", "com.example.order.BaseOrderTest")
    //     baseClassMapping(".*payment.*", "com.example.payment.BasePaymentTest")
    // }

    // generatedTestSourcesDir: directorio de tests generados
    generatedTestSourcesDir = project.file("${project.buildDir}/generated-test-sources/contracts")

    // testFramework: JUNIT5 (default) o SPOCK
    testFramework = TestFramework.JUNIT5

    // testMode: MOCKMVC (default), EXPLICIT, WEBTESTCLIENT
    testMode = 'MOCKMVC'
}

dependencies {
    testImplementation 'org.springframework.cloud:spring-cloud-starter-contract-verifier'
    testImplementation 'org.springframework.cloud:spring-cloud-contract-wiremock'
}
```

## Tabla de propiedades del plugin

Las propiedades del plugin son el conocimiento más frecuente en preguntas de examen relacionadas con el lado productor de Spring Cloud Contract.

| Propiedad Maven | Propiedad Gradle | Default | Descripción |
|---|---|---|---|
| `contractsDslDir` | `contractsDslDir` | `src/test/resources/contracts` | Directorio de contratos |
| `baseClassForTests` | `baseClassForTests` | — | Clase base para todos los tests generados |
| `baseClassMappings` | `baseClassMappings {}` | — | Mapeo regex → clase base (múltiples) |
| `generatedTestSourcesDir` | `generatedTestSourcesDir` | `target/generated-test-sources/contracts` | Destino de tests generados |
| `testFramework` | `testFramework` | `JUNIT5` | Framework: JUNIT5, SPOCK |
| `outputDirectory` | — | `target/stubs` | Destino de stubs WireMock |
| `excludedFiles` | `excludedFiles` | — | Patrones de contratos a excluir |
| `testMode` | `testMode` | `MOCKMVC` | Modo de test: MOCKMVC, EXPLICIT, WEBTESTCLIENT |

> [EXAMEN] La propiedad `contractsDslDir` tiene como valor por defecto `src/test/resources/contracts`. Este es uno de los datos más frecuentes en el examen. El directorio `generatedTestSourcesDir` por defecto es `target/generated-test-sources/contracts` (Maven) — no confundir con `outputDirectory` (target/stubs) que es donde van los stubs.

## spring-cloud-contract-wiremock

`spring-cloud-contract-wiremock` es el módulo de Spring Cloud Contract que integra WireMock como motor de stubs. Es la dependencia estándar del stub runner en Spring Cloud Contract 4.x (2025.x) y provee `@AutoConfigureWireMock` para levantar un servidor WireMock en tests.

```java
// Ejemplo de uso de @AutoConfigureWireMock (provisto por spring-cloud-contract-wiremock)
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import static com.github.tomakehurst.wiremock.client.WireMock.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)  // puerto aleatorio
public class WireMockTest {

    @Test
    void shouldConfigureWireMockStub() {
        // Configura un stub WireMock directamente (sin contrato SCC)
        stubFor(get(urlEqualTo("/external/api"))
            .willReturn(aResponse()
                .withStatus(200)
                .withBody("{\"result\": \"ok\"}")));
    }
}
```

> [ADVERTENCIA] `spring-cloud-contract-wiremock` es el módulo que incluye WireMock como motor de stubs en el ecosistema SCC. No confundir con Spring Cloud Contract completo. El stub runner usa `spring-cloud-contract-wiremock` internamente, pero ambos son módulos distintos con responsabilidades distintas.

## Buenas y malas prácticas

**Buenas prácticas**:
- Usar `baseClassMappings` en proyectos con contratos de múltiples dominios para no tener una única clase base monolítica.
- Mantener el directorio de contratos bajo `src/test/resources/contracts` (default) para evitar configuración adicional.
- Verificar que `mvn verify` pasa localmente antes de crear un PR — los tests generados deben pasar en el build.
- Usar `testMode = WEBTESTCLIENT` para productores reactivos (WebFlux).

**Malas prácticas**:
- Usar `baseClassForTests` y `baseClassMappings` simultáneamente — `baseClassMappings` tiene prioridad y `baseClassForTests` se ignora.
- Commitear los tests generados (`generatedTestSourcesDir`) al repositorio — son artefactos generados, no código fuente.
- Cambiar `outputDirectory` a un directorio no estándar sin documentarlo — el Stub Runner puede no encontrar los stubs.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es el valor por defecto de `contractsDslDir` en el `spring-cloud-contract-maven-plugin`?

> [EXAMEN] 2. ¿Cuál es la diferencia entre `baseClassForTests` y `baseClassMappings` en la configuración del plugin?

> [EXAMEN] 3. ¿En qué fase del ciclo de vida Maven se generan los tests automáticos a partir de los contratos?

> [EXAMEN] 4. ¿Qué propiedad del plugin controla el framework de test que se usará para los tests generados, y cuáles son los valores posibles?

> [EXAMEN] 5. ¿Qué módulo de Spring Cloud Contract integra WireMock como motor de stubs y qué anotación provee para tests?

---

← [10.4 Contratos Mensajería](sc-contract-mensajeria.md) | [Índice](README.md) | [10.6 Clases Base Productor](sc-contract-base-class.md) →
