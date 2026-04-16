# 10.2 Ciclo de vida CDC y workflow CI/CD

← [10.1 Arquitectura CDC y modelo Consumer-Driven Contracts](sc-contract-fundamentos.md) | [Índice](README.md) | [10.3.1 Plugin Maven/Gradle — goals, tasks y testMode](sc-contract-plugin.md) →

---

## Introducción

Adoptar Consumer-Driven Contracts en un equipo no requiere solo conocer las herramientas, sino entender el protocolo que hace funcionar el flujo entre equipos distintos. Sin un workflow definido, los contratos acaban siendo un artefacto que nadie mantiene y que rompe CI de forma inexplicable. Este fichero cubre el ciclo de vida completo: quién escribe el contrato primero, cómo se negocia un cambio, cómo el pipeline CI del producer y del consumer coordinan sus builds, y cómo se detectan los breaking changes antes de que lleguen a staging.

> [CONCEPTO] **Contract-first**: el consumer escribe el contrato antes de que el producer implemente el endpoint. Es la estrategia recomendada porque garantiza que la API se diseña desde las necesidades reales del consumidor, no desde lo que al producer le resulta cómodo exponer.

> [PREREQUISITO] Este fichero asume que el lector conoce el modelo CDC explicado en 10.1. Se asume también familiaridad básica con pipelines CI/CD (Jenkins, GitHub Actions, GitLab CI o equivalente).

## Representación visual

El diagrama muestra el flujo completo CDC con las fases del pipeline CI y los puntos de detección de breaking changes. Las flechas sólidas son flujos de artefactos; las flechas punteadas son notificaciones de estado.

```
CONSUMER TEAM                       REPOSITORIO COMPARTIDO        PRODUCER TEAM
─────────────────                   ──────────────────────        ─────────────────
1. Consumer necesita                                              
   nuevo endpoint                                                 
        │                                                         
        ▼                                                         
2. Consumer escribe        ──────► 3. Contrato en Git ──────────► 4. Producer recibe
   contrato Groovy/YAML               (PR o rama)                    contrato (pull)
   (contract-first)                                                       │
        │                                                            5. Plugin genera
        │                                                               tests desde
        │                                                               contratos
        │                                                                 │
        │                                                            6. Tests ejecutan
        │                                                               contra impl.
        │                                                               real del producer
        │                                                                 │
        │                                                            7. Producer
        │                                                               implementa
        │                                                               endpoint
        │                                                                 │
        │                                                            8. Build publica
        │                                                               Stub JAR en
        │                         ◄────────────────────────────────     Nexus/Maven
        │                              Stub JAR disponible
        ▼
9. CI del consumer         ◄──── 10. StubRunner descarga
   ejecuta tests con               Stub JAR automáticamente
   @AutoConfigureStubRunner
        │
        ▼
11. Tests pasan/fallan
    ← detección temprana
      de breaking changes
```

## Ejemplo central

El ejemplo muestra una configuración completa de GitHub Actions que implementa el pipeline CDC para el producer: genera tests, ejecuta verificaciones de contrato, empaqueta y publica el Stub JAR.

**Pipeline CI del producer** (`.github/workflows/contract-producer.yml`):

```yaml
name: Contract Producer CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  contract-verify:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Run contract tests (generate + verify)
        run: mvn verify -pl producer-service

      - name: Publish Stub JAR to local Maven repository
        run: mvn install -pl producer-service -DskipTests

      - name: Publish Stub JAR to Nexus
        if: github.ref == 'refs/heads/main'
        run: mvn deploy -pl producer-service -DskipTests
        env:
          MAVEN_USERNAME: ${{ secrets.NEXUS_USER }}
          MAVEN_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
```

**Pipeline CI del consumer** (`.github/workflows/contract-consumer.yml`):

```yaml
name: Contract Consumer CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  consumer-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Run consumer contract tests
        run: mvn test -pl consumer-service
        env:
          STUB_RUNNER_REPOSITORY_ROOT: http://nexus.company.com/repository/maven-releases/
```

**application.yml del consumer para tests CI:**

```yaml
spring:
  application:
    name: order-service

stubrunner:
  stubsMode: REMOTE
  repositoryRoot: ${STUB_RUNNER_REPOSITORY_ROOT:http://localhost:8081/repository/maven-releases/}
  ids: com.example:user-service:+:stubs:8090
```

**Clase de test del consumer con configuración de entorno:**

```java
package com.example.consumer;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;
import org.springframework.web.client.RestClient;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureStubRunner(
    ids = "${stubrunner.ids:com.example:user-service:+:stubs:8090}",
    stubsMode = StubRunnerProperties.StubsMode.REMOTE,
    repositoryRoot = "${stubrunner.repositoryRoot:}"
)
class OrderServiceContractTest {

    @Value("${stubrunner.runnedStubs.com.example:user-service:8090:}")
    private int stubPort;

    @Test
    void shouldGetUserForOrder() {
        RestClient client = RestClient.builder()
            .baseUrl("http://localhost:" + stubPort)
            .build();

        Map<?, ?> user = client.get()
            .uri("/api/users/1")
            .retrieve()
            .body(Map.class);

        assertThat(user).isNotNull();
        assertThat(user.get("id")).isEqualTo(1);
    }
}
```

## Tabla de elementos clave

La siguiente tabla resume las propiedades de configuración y conceptos de workflow que un senior debe conocer para diseñar un pipeline CDC robusto.

| Concepto / Propiedad | Tipo | Descripción |
|---------------------|------|-------------|
| `contract-first` | Estrategia | Consumer define el contrato antes de que el producer implemente; garantiza API orientada al cliente |
| `code-first` | Estrategia | Producer implementa primero; el contrato se extrae de la implementación existente |
| `publishStubsToScm` | Maven goal | Publica el Stub JAR al repositorio SCM (Git) en lugar de Nexus |
| `stubrunner.ids` | Propiedad YAML | Formato `groupId:artifactId:version:classifier:port`; `+` significa última versión disponible |
| `stubrunner.stubsMode` | Enum | CLASSPATH, LOCAL, REMOTE o SERVICE; determina de dónde se descargan los stubs |
| `stubrunner.repositoryRoot` | URL | Repositorio Maven donde el StubRunner busca los Stub JARs en modo REMOTE |
| Breaking change detection | Mecanismo | El pipeline CI del producer falla si la implementación no satisface algún contrato existente |
| Rollback strategy | Procedimiento | Revertir el cambio del producer o versionar el contrato nuevo sin eliminar el anterior |
| Contrato compartido en Git | Patrón | Un repositorio Git centralizado almacena todos los contratos; el producer hace pull antes del build |
| `spring-cloud-contract.version` | Property Maven | Versión del plugin; debe alinearse con la BOM de Spring Cloud (2025.1.1 para Oakwood) |

## Buenas y malas prácticas

**Hacer:**

- Usar `contract-first` como estrategia predeterminada: el consumer escribe el contrato como PR en el repositorio del producer antes de que el equipo producer comience la implementación; esto evita implementaciones que no satisfacen las necesidades reales.
- Configurar el pipeline CI del producer para que falle si los tests de contrato no pasan (`mvn verify`); nunca publicar el Stub JAR si los tests de contrato fallan, o los consumers recibirán stubs que no corresponden a la implementación real.
- Usar versión `+` en `stubrunner.ids` solo en desarrollo local; en CI usar la versión exacta (`1.2.3`) para reproducibilidad y evitar que un nuevo Stub JAR rompa el consumer silenciosamente.
- Mantener los contratos en el mismo repositorio del producer bajo control de versiones; cada cambio de contrato debe tener su propio commit para facilitar el bisect en caso de regresión.
- Notificar al equipo consumer antes de modificar o eliminar un contrato existente; el protocolo recomendado es deprecar el contrato antiguo durante un sprint antes de eliminarlo.

**Evitar:**

- Evitar publicar Stub JARs desde ramas de feature sin convenio de versioning: contamina el repositorio remoto con stubs que corresponden a código no mergeado y pueden ser descargados por otros consumers.
- Evitar depender del build local del producer en el pipeline del consumer: si el Stub JAR no está en Nexus/Artifactory, el pipeline del consumer falla con una excepción de resolución de artefacto, no con un fallo de contrato claro.
- Evitar mezclar `StubsMode.CLASSPATH` en CI y `StubsMode.REMOTE` en local: los tests pueden pasar en local con stubs obsoletos del classpath y fallar en CI porque el Stub JAR remoto tiene una versión más nueva con cambios incompatibles.
- Evitar aprobar un PR del producer que rompe contratos existentes sin verificar con el equipo consumer que la migración es coordinada: una ruptura de contrato en producción obliga a un hotfix coordinado entre equipos.
- Evitar usar la misma versión de Stub JAR (`1.0.0-SNAPSHOT`) para múltiples iteraciones en CI: los SNAPSHOTs se sobreescriben en Nexus y pueden causar que un consumer en CI use un stub diferente al que le corresponde según el estado del producer.

## Comparación: workflows según estrategia de repositorio de contratos

La elección del repositorio de contratos impacta en el tiempo de feedback y la autonomía de los equipos.

| Estrategia | Dónde viven los contratos | Ventaja | Inconveniente |
|-----------|--------------------------|---------|---------------|
| Repositorio del producer | `src/test/resources/contracts/` del producer | Setup mínimo; contrato y código en el mismo repo | El consumer no puede hacer PR al contrato sin acceso al repo del producer |
| Repositorio compartido Git | Repo Git independiente con todos los contratos | Visibilidad centralizada; el consumer puede hacer PR | Requiere gestión de un repo adicional y sincronización de builds |
| Nexus/Artifactory (stubs remotos) | Stub JAR publicado como artefacto Maven | Consumer descarga stub sin acceso al repo del producer | La publicación del stub requiere un build exitoso del producer primero |
| SCM-based (publishStubsToScm) | Git SCM referenciado desde el plugin | Combina código y stubs en SCM con versionado semántico | Configuración más compleja del plugin y del acceso SCM en CI |

> [EXAMEN] Pregunta senior habitual: "¿Cómo gestionas un breaking change en un contrato cuando hay varios consumers?" — La respuesta correcta es: versionar el contrato (v1 y v2 coexisten), publicar el nuevo Stub JAR con versión mayor, migrar los consumers gradualmente y deprecar el contrato v1 con un deadline acordado. No se elimina el contrato antiguo hasta que todos los consumers hayan migrado.

> [ADVERTENCIA] En pipelines con múltiples consumers del mismo producer, un fallo de contrato en el producer bloquea el build de todos los consumers que dependen de ese Stub JAR. El monitoring del repositorio de contratos es crítico para detectar la causa raíz rápidamente y no confundir un fallo de contrato con un fallo de infraestructura.

---

← [10.1 Arquitectura CDC y modelo Consumer-Driven Contracts](sc-contract-fundamentos.md) | [Índice](README.md) | [10.3.1 Plugin Maven/Gradle — goals, tasks y testMode](sc-contract-plugin.md) →
