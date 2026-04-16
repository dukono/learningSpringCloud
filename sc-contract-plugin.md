# 10.3.1 Plugin Maven/Gradle — goals, tasks y testMode

← [10.2 Ciclo de vida CDC y workflow CI/CD](sc-contract-workflow.md) | [Índice](README.md) | [10.3.2 Plugin Maven/Gradle — propiedades de personalización](sc-contract-plugin-config.md) →

---

## Introducción

El plugin de Spring Cloud Contract es el motor que convierte contratos Groovy/YAML en tests JUnit ejecutables y en Stub JARs publicables. Sin él, el ciclo CDC no existe: es la pieza que conecta el contrato (artefacto estático) con la verificación real de la implementación del producer. Comprender qué hace cada goal/task, en qué fase del ciclo de build se ejecuta y qué modo de test usar según la tecnología del producer es imprescindible para configurar correctamente el pipeline CI y diagnosticar fallos de generación.

> [CONCEPTO] **testMode**: propiedad del plugin que determina qué librería de testing HTTP se usa en los tests generados. El modo incorrecto provoca tests que no compilan o que no arrancan el contexto correcto.

> [PREREQUISITO] Familiaridad con el ciclo de build de Maven (fases) o Gradle (tasks). Conocimiento básico de MockMvc, REST Assured y WebTestClient para entender las implicaciones de cada modo.

## Representación visual

El diagrama muestra la integración del plugin en el ciclo de build Maven y qué artefactos produce cada goal.

```
CICLO MAVEN (mvn verify)
═══════════════════════
  generate-test-sources
       │
       ▼
  [spring-cloud-contract:generateTests]
       │
       └─► build/generated-test-sources/
           └─► ContractVerifierTest.java  (JUnit 5, extiende BaseContractTest)
       │
  test-compile ──── compila los tests generados
       │
  test
       │
       ▼
  [JUnit 5 ejecuta ContractVerifierTest]
       │
       └─► Si falla → build falla (breaking change detectado)
       │
  package
       │
       ▼
  [spring-cloud-contract:generateStubs]
       │
       └─► target/[artifactId]-[version]-stubs.jar
           └─► META-INF/[groupId]/[artifactId]/[version]/
               └─► mappings/
                   └─► shouldReturnUser.json  (WireMock mapping)
       │
  install / deploy
       │
       ▼
  [Stub JAR publicado en Nexus/Maven local]
```

## Ejemplo central

El ejemplo muestra la declaración completa del plugin tanto en Maven como en Gradle, con los cuatro modos de test y sus dependencias respectivas.

**pom.xml — declaración del plugin Maven con testMode MockMvc:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>user-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.0</version>
    </parent>

    <properties>
        <spring-cloud.version>2025.1.1</spring-cloud.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- Dependencia de test para el producer -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-contract-verifier</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Para testMode=MOCKMVC -->
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>spring-mock-mvc</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-contract-maven-plugin</artifactId>
                <version>${spring-cloud.version}</version>
                <extensions>true</extensions>
                <configuration>
                    <!-- Clase base para todos los tests generados -->
                    <baseClassForTests>
                        com.example.producer.BaseContractTest
                    </baseClassForTests>
                    <!-- testMode: MOCKMVC | WEBTESTCLIENT | RESTASSURED | EXPLICIT -->
                    <testMode>MOCKMVC</testMode>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

**build.gradle — declaración del plugin Gradle con testMode WebTestClient (para Spring WebFlux):**

```groovy
plugins {
    id 'org.springframework.boot' version '4.0.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
    id 'org.springframework.cloud.contract' version '4.2.0'
}

group = 'com.example'
version = '1.0.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2025.1.1"
    }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    testImplementation 'org.springframework.cloud:spring-cloud-starter-contract-verifier'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    // Para testMode=WEBTESTCLIENT con WebFlux
    testImplementation 'io.projectreactor:reactor-test'
}

// Configuración del plugin Gradle equivalente al pom.xml
contracts {
    baseClassForTests = 'com.example.producer.BaseContractTest'
    testMode = 'WEBTESTCLIENT'
}
```

**Clase base para testMode=MOCKMVC** (`src/test/java/com/example/producer/BaseContractTest.java`):

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
        // Standalone setup: solo el controller, sin servidor completo
        RestAssuredMockMvc.standaloneSetup(userController);
    }
}
```

**Clase base para testMode=WEBTESTCLIENT** (Spring WebFlux):

```java
package com.example.producer;

import com.example.producer.controller.ReactiveUserController;
import io.restassured.module.webtestclient.RestAssuredWebTestClient;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
public abstract class BaseContractTest {

    @Autowired
    private ReactiveUserController reactiveUserController;

    @BeforeEach
    public void setup() {
        RestAssuredWebTestClient.standaloneSetup(reactiveUserController);
    }
}
```

**Ejecución de goals Maven individuales:**

```bash
# Solo generar tests (sin ejecutarlos)
mvn spring-cloud-contract:generateTests

# Generar Stub JAR sin ejecutar tests de contrato
mvn spring-cloud-contract:generateStubs

# Publicar stubs al SCM Git configurado
mvn spring-cloud-contract:publishStubsToScm

# Ciclo completo: genera tests, los ejecuta y empaqueta el Stub JAR
mvn verify

# Instalar en repositorio Maven local (para desarrollo)
mvn install

# Publicar en Nexus/Artifactory
mvn deploy
```

**application.yml del producer:**

```yaml
spring:
  application:
    name: user-service

# No se necesita configuración especial para el plugin
# Los contratos se buscan en src/test/resources/contracts/ por defecto
```

## Tabla de elementos clave

La siguiente tabla recoge los goals Maven, tasks Gradle y el testMode con sus características para decisiones de configuración en entrevistas técnicas.

| Goal/Task | Fase Maven | Equivalente Gradle | Descripción |
|-----------|-----------|-------------------|-------------|
| `generateTests` | generate-test-sources | `generateContractTests` | Genera clases JUnit 5 en `build/generated-test-sources/` desde los contratos |
| `generateStubs` | package | `verifierStubsJar` | Empaqueta los mappings WireMock en un JAR clasificado como `stubs` |
| `publishStubsToScm` | — (manual) | `publishStubsToScm` | Publica el Stub JAR al repositorio SCM (Git) configurado |
| `convert` | generate-test-sources | `convertContracts` | Convierte contratos DSL a mappings WireMock sin generar tests |
| **testMode=MOCKMVC** | — | — | Usa MockMvc de Spring Test; válido para apps Servlet (Spring MVC) |
| **testMode=WEBTESTCLIENT** | — | — | Usa WebTestClient; válido para apps reactivas (Spring WebFlux) |
| **testMode=RESTASSURED** | — | — | Usa REST Assured en modo HTTP real contra servidor levantado en puerto random |
| **testMode=EXPLICIT** | — | — | Genera tests sin ningún framework de test HTTP; la clase base configura el cliente manualmente |
| `<extensions>true</extensions>` | — | — | Activa los goals del plugin en las fases estándar del ciclo de build |
| `spring-cloud-contract.version` | — | — | Debe coincidir con la versión de la BOM (`2025.1.1` para Oakwood) |

## Buenas y malas prácticas

**Hacer:**

- Usar `<extensions>true</extensions>` en el plugin Maven para que los goals `generateTests` y `generateStubs` se ejecuten automáticamente en las fases `generate-test-sources` y `package`; sin esto hay que invocarlos manualmente en cada build.
- Elegir `testMode=MOCKMVC` para aplicaciones Spring MVC (Servlet) y `testMode=WEBTESTCLIENT` para Spring WebFlux; la elección incorrecta genera tests que no compilan porque la superclase generada hereda de una clase inexistente.
- Verificar que la versión del plugin en `pom.xml` coincide con la versión declarada en la BOM de Spring Cloud; versiones desincronizadas generan tests que no compilan por incompatibilidad de APIs internas.
- Ejecutar `mvn verify` (no solo `mvn test`) en CI para garantizar que el Stub JAR se genera y empaqueta correctamente además de que los tests pasan.
- Usar `mvn install` en el pipeline del producer antes de ejecutar los tests del consumer en entornos de desarrollo local; en CI usar `mvn deploy` al repositorio remoto.

**Evitar:**

- Evitar declarar el plugin sin `<extensions>true</extensions>`: los goals no se ejecutan automáticamente y el build parece correcto pero no genera tests ni stubs, dejando el contrato sin verificar.
- Evitar usar `testMode=RESTASSURED` en tests de CI que arrancan el servidor en puerto fijo: puede causar conflictos de puerto si varios jobs de CI se ejecutan en el mismo agente; preferir `MOCKMVC` o `WEBTESTCLIENT` que no levantan servidor HTTP real.
- Evitar mezclar `MOCKMVC` con controladores WebFlux o `WEBTESTCLIENT` con controladores MVC Servlet: la generación de tests funciona pero los tests generados lanzan `ClassCastException` o `NoSuchBeanDefinitionException` en tiempo de ejecución.
- Evitar invocar `publishStubsToScm` sin configurar las credenciales SCM en el entorno CI: el goal falla con error de autenticación y bloquea el pipeline sin mensaje claro sobre la causa.
- Evitar usar `mvn spring-cloud-contract:generateTests` de forma aislada para verificar contratos: el goal solo genera el código pero no ejecuta los tests; usar siempre `mvn verify` para la verificación real.

## Comparación: testMode disponibles

La elección del testMode es una de las decisiones más frecuentes en entrevistas sobre Spring Cloud Contract.

| testMode | Tecnología | Contexto Spring | Servidor HTTP | Caso de uso |
|---------|-----------|----------------|--------------|-------------|
| MOCKMVC | MockMvc (Spring Test) | Se carga (slice o full) | No | API REST Servlet estándar; el más rápido |
| WEBTESTCLIENT | WebTestClient (Spring WebFlux) | Se carga | No (MockWebServer interno) | API REST reactiva con WebFlux |
| RESTASSURED | REST Assured HTTP | Se carga (full @SpringBootTest) | Sí (puerto random) | Necesidad de probar filtros Servlet o SecurityFilterChain reales |
| EXPLICIT | Sin framework HTTP | Opcional | Configurable manualmente | Casos avanzados donde el desarrollador gestiona el cliente HTTP |

> [EXAMEN] Pregunta de entrevista: "¿Por qué el plugin incluye `<extensions>true</extensions>` y qué pasa si se omite?" — La extensión registra los goals en el ciclo de vida estándar de Maven. Sin ella, `generateTests` y `generateStubs` no se ejecutan automáticamente; el build compila y los tests pasan, pero sin verificación de contrato ni generación de stubs, invalidando todo el ciclo CDC.

> [ADVERTENCIA] Spring Cloud Contract 2025.1.1 (Oakwood) requiere Java 21 como mínimo. Si el proyecto usa `maven.compiler.source=17`, el plugin generará código con APIs de Java 21 que no compilará con el source de Java 17. Verificar la compatibilidad antes de actualizar la BOM.

---

← [10.2 Ciclo de vida CDC y workflow CI/CD](sc-contract-workflow.md) | [Índice](README.md) | [10.3.2 Plugin Maven/Gradle — propiedades de personalización](sc-contract-plugin-config.md) →
