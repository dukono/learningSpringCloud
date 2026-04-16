# 10.3.2 Plugin Maven/Gradle — propiedades de personalización

← [10.3.1 Plugin Maven/Gradle — goals, tasks y testMode](sc-contract-plugin.md) | [Índice](README.md) | [10.4.1 DSL Groovy/YAML — estructura base de contratos HTTP](sc-contract-dsl.md) →

---

## Introducción

Cuando un proyecto Spring Cloud Contract supera la configuración mínima —un solo directorio de contratos y una única clase base— la configuración por defecto del plugin deja de ser suficiente. En proyectos con múltiples servicios, múltiples dominios de contratos, o estructuras de paquetes no estándar, las propiedades de personalización del plugin determinan si los tests generados compilan y se mapean correctamente a las clases base del producer. Un error de configuración en `baseClassMappings` o `basePackageForTests` produce tests generados que no compilan, un fallo difícil de diagnosticar si no se conoce la mecánica interna del plugin.

> [CONCEPTO] **baseClassMappings**: propiedad que permite asignar una clase base distinta a cada directorio de contratos, lo que habilita usar diferentes setups de contexto (MockMvc, WebTestClient) para diferentes grupos de contratos en el mismo proyecto.

> [PREREQUISITO] Conocimiento de los goals del plugin (10.3.1) y de la estructura básica de la clase base de tests (10.5).

## Representación visual

El diagrama muestra cómo el plugin resuelve qué clase base usar para un contrato dado, aplicando primero `baseClassMappings` y cayendo a `baseClassForTests` como fallback.

```
RESOLUCIÓN DE CLASE BASE POR CONTRATO
══════════════════════════════════════

src/test/resources/contracts/
├── user/
│   └── shouldReturnUser.groovy  ──────► baseClassMappings?
│                                            │
│                                    ".*user.*" → UserBaseTest
│                                            │
│                                            ▼
│                                    UserBaseTest.java
│                                    (extiende ContractVerifierMockMvc)
│
├── order/
│   └── shouldCreateOrder.groovy ──────► baseClassMappings?
│                                            │
│                                    ".*order.*" → OrderBaseTest
│                                            │
│                                            ▼
│                                    OrderBaseTest.java
│                                    (extiende ContractVerifierMockMvc)
│
└── legacy/
    └── oldEndpoint.groovy        ──────► baseClassMappings?
                                              │
                                    Sin match → baseClassForTests
                                              │
                                              ▼
                                    DefaultBaseTest.java
```

## Ejemplo central

El ejemplo muestra un proyecto con múltiples grupos de contratos que requieren clases base distintas, configurando todas las propiedades de personalización relevantes.

**pom.xml — configuración completa del plugin con baseClassMappings:**

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-contract-maven-plugin</artifactId>
            <version>${spring-cloud.version}</version>
            <extensions>true</extensions>
            <configuration>
                <!-- Directorio donde se buscan los contratos -->
                <contractsDirectory>
                    ${project.basedir}/src/test/resources/contracts
                </contractsDirectory>
                <!-- Clase base fallback si ningún mapping coincide -->
                <baseClassForTests>
                    com.example.producer.DefaultBaseTest
                </baseClassForTests>
                <!-- Mappings por directorio de contratos (regex → clase base) -->
                <baseClassMappings>
                    <baseClassMapping>
                        <contractPackageRegex>.*user.*</contractPackageRegex>
                        <baseClassFQN>com.example.producer.UserBaseTest</baseClassFQN>
                    </baseClassMapping>
                    <baseClassMapping>
                        <contractPackageRegex>.*order.*</contractPackageRegex>
                        <baseClassFQN>com.example.producer.OrderBaseTest</baseClassFQN>
                    </baseClassMapping>
                    <baseClassMapping>
                        <contractPackageRegex>.*reactive.*</contractPackageRegex>
                        <baseClassFQN>com.example.producer.ReactiveBaseTest</baseClassFQN>
                    </baseClassMapping>
                </baseClassMappings>
                <!-- Paquete base de los tests generados -->
                <basePackageForTests>com.example.producer.contracts</basePackageForTests>
                <!-- Sufijo del nombre de la clase generada (por defecto: Test) -->
                <nameSuffixForTests>ContractTest</nameSuffixForTests>
                <!-- Ficheros a excluir (soporta ant-style patterns) -->
                <excludedFiles>
                    <param>**/experimental/**</param>
                    <param>**/deprecated/**</param>
                </excludedFiles>
                <!-- testMode para todos los contratos (MOCKMVC por defecto) -->
                <testMode>MOCKMVC</testMode>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Clases base específicas por dominio:**

```java
package com.example.producer;

import com.example.producer.controller.UserController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public abstract class UserBaseTest {

    @Autowired
    private UserController userController;

    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(userController);
    }
}
```

```java
package com.example.producer;

import com.example.producer.controller.OrderController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public abstract class OrderBaseTest {

    @Autowired
    private OrderController orderController;

    @BeforeEach
    public void setup() {
        RestAssuredMockMvc.standaloneSetup(orderController);
    }
}
```

```java
package com.example.producer;

import com.example.producer.controller.ReactiveProductController;
import io.restassured.module.webtestclient.RestAssuredWebTestClient;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public abstract class ReactiveBaseTest {

    @Autowired
    private ReactiveProductController productController;

    @BeforeEach
    public void setup() {
        RestAssuredWebTestClient.standaloneSetup(productController);
    }
}
```

**Equivalente en build.gradle:**

```groovy
contracts {
    // Directorio de contratos
    contractsDslDir = file("${project.rootDir}/src/test/resources/contracts")

    // Clase base fallback
    baseClassForTests = 'com.example.producer.DefaultBaseTest'

    // Mappings por regex de paquete
    baseClassMappings {
        baseClassMapping('.*user.*', 'com.example.producer.UserBaseTest')
        baseClassMapping('.*order.*', 'com.example.producer.OrderBaseTest')
        baseClassMapping('.*reactive.*', 'com.example.producer.ReactiveBaseTest')
    }

    // Paquete de los tests generados
    basePackageForTests = 'com.example.producer.contracts'

    // Sufijo del nombre de test generado
    nameSuffixForTests = 'ContractTest'

    // Exclusiones
    excludedFiles = ['**/experimental/**', '**/deprecated/**']

    // testMode
    testMode = 'MOCKMVC'
}
```

**Estructura de directorios resultante:**

```
src/test/resources/contracts/
├── user/
│   ├── shouldReturnUser.groovy
│   └── shouldReturnAllUsers.groovy
├── order/
│   ├── shouldCreateOrder.groovy
│   └── shouldCancelOrder.groovy
└── reactive/
    └── shouldStreamProducts.groovy

build/generated-test-sources/
└── com/example/producer/contracts/
    ├── UserContractTest.java     ← extiende UserBaseTest
    ├── OrderContractTest.java    ← extiende OrderBaseTest
    └── ReactiveContractTest.java ← extiende ReactiveBaseTest
```

**application.yml del producer:**

```yaml
spring:
  application:
    name: multi-domain-service

# No se requiere configuración de Spring Cloud Contract en application.yml
# Toda la configuración está en pom.xml / build.gradle
```

## Tabla de elementos clave

La siguiente tabla recoge todas las propiedades de personalización del plugin con su valor por defecto y el impacto de una configuración incorrecta.

| Propiedad | Maven | Gradle | Default | Descripción |
|-----------|-------|--------|---------|-------------|
| `contractsDirectory` | `<contractsDirectory>` | `contractsDslDir` | `src/test/resources/contracts` | Directorio raíz donde el plugin busca contratos DSL |
| `baseClassForTests` | `<baseClassForTests>` | `baseClassForTests` | Ninguno (obligatorio) | FQN de la clase base para todos los tests generados |
| `baseClassMappings` | `<baseClassMappings>` | `baseClassMappings { }` | Vacío | Regex de paquete → FQN de clase base; tiene prioridad sobre `baseClassForTests` |
| `basePackageForTests` | `<basePackageForTests>` | `basePackageForTests` | `org.springframework.cloud.contract.verifier.tests` | Paquete Java de las clases generadas |
| `nameSuffixForTests` | `<nameSuffixForTests>` | `nameSuffixForTests` | `Test` | Sufijo del nombre de la clase generada |
| `excludedFiles` | `<excludedFiles>` | `excludedFiles` | Vacío | Patrones ant-style de contratos a ignorar |
| `testMode` | `<testMode>` | `testMode` | `MOCKMVC` | Modo de test: MOCKMVC, WEBTESTCLIENT, RESTASSURED, EXPLICIT |
| `contractsRepositoryUrl` | `<contractsRepositoryUrl>` | `contractsRepositoryUrl` | Vacío | URL del repositorio remoto de contratos (si no están en el proyecto) |
| `stubsSuffix` | `<stubsSuffix>` | `stubsSuffix` | `stubs` | Classifier del Stub JAR generado |
| `contractsPath` | `<contractsPath>` | `contractsPath` | Directorio del proyecto | Subdirectorio dentro del repositorio remoto de contratos |

## Buenas y malas prácticas

**Hacer:**

- Usar `baseClassMappings` en proyectos con múltiples dominios de API para asignar la clase base correcta a cada grupo de contratos; evita errores de compilación por usar MockMvc en contratos que necesitan WebTestClient.
- Definir `basePackageForTests` explícitamente para que los tests generados se ubiquen en el paquete del proyecto (ej: `com.example.producer.contracts`) y no en el paquete por defecto del framework; facilita la navegación en el IDE.
- Usar `excludedFiles` para excluir contratos experimentales o en proceso de desarrollo del ciclo de build sin borrarlos físicamente del proyecto; permite trabajar en contratos nuevos sin que rompan el pipeline.
- Validar los patrones `baseClassMappings` con `mvn spring-cloud-contract:generateTests` en local antes de commitear; un regex incorrecto hace que todos los contratos caigan al fallback `baseClassForTests`.
- Documentar en el README del proyecto la convención de directorios de contratos y el mapeo a clases base; sin esta documentación, los desarrolladores nuevos generan contratos en el lugar incorrecto y el plugin los ignora silenciosamente.

**Evitar:**

- Evitar usar la misma clase base para contratos que requieren `MockMvc` y contratos que requieren `WebTestClient`: los tests generados heredan de una sola superclase y el uso mixto provoca `IllegalStateException` en tiempo de ejecución al intentar configurar el contexto de test.
- Evitar usar `contractsDirectory` apuntando a `src/main/resources`: los contratos quedan incluidos en el JAR de producción, lo que expone detalles de testing en el artefacto desplegado.
- Evitar dejar `nameSuffixForTests` en el valor por defecto `Test` si el proyecto ya tiene clases de test con ese sufijo en el mismo paquete: puede causar conflictos de naming o sobreescritura accidental de clases existentes.
- Evitar configurar `contractsRepositoryUrl` sin autenticación correcta en el entorno CI: el build falla con un error de resolución de artefacto difícil de distinguir de un fallo de red.
- Evitar mezclar versiones del plugin entre módulos de un proyecto multi-módulo: si `module-a` usa la versión `4.1.0` y `module-b` usa `4.2.0`, los stubs generados pueden tener formatos de mapping WireMock incompatibles.

## Comparación: estrategias de organización de contratos

| Estrategia | Estructura | `baseClassMappings` necesario | Complejidad |
|-----------|-----------|------------------------------|-------------|
| Plana (todos en `/contracts`) | Un solo directorio con todos los contratos | No (una sola clase base) | Baja; válida para proyectos pequeños |
| Por dominio (`/contracts/user`, `/contracts/order`) | Un directorio por dominio de API | Sí (regex por dominio) | Media; recomendada para proyectos medianos |
| Por tecnología (`/contracts/mvc`, `/contracts/reactive`) | Separación por tecnología de test | Sí (regex por tecnología) | Media; necesaria si el producer mezcla MVC y WebFlux |
| Por equipo consumer | Un directorio por consumer que define contratos | Sí (regex por consumer) | Alta; útil para contratos propiedad del consumer en repositorio compartido |

> [EXAMEN] Pregunta de entrevista: "¿Qué ocurre si `baseClassMappings` no tiene ningún regex que coincida con el paquete de un contrato?" — El plugin usa `baseClassForTests` como fallback. Si `baseClassForTests` tampoco está configurado, el test generado hereda de `Object` y no compila porque no tiene los métodos de setup necesarios.

> [ADVERTENCIA] Los patrones de `excludedFiles` en Maven usan sintaxis ant-style (`**/experimental/**`) pero en Gradle usan la notación de Gradle `FileTree` glob. Copiar un patrón de pom.xml a build.gradle sin adaptarlo puede hacer que el exclusion no funcione silenciosamente.

---

← [10.3.1 Plugin Maven/Gradle — goals, tasks y testMode](sc-contract-plugin.md) | [Índice](README.md) | [10.4.1 DSL Groovy/YAML — estructura base de contratos HTTP](sc-contract-dsl.md) →
