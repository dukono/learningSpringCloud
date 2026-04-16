# 10.1 Arquitectura CDC y modelo Consumer-Driven Contracts

← [9.12 Testing de Spring Cloud Kubernetes](sc-kubernetes-testing.md) | [Índice](README.md) | [10.2 Ciclo de vida CDC y workflow CI/CD](sc-contract-workflow.md) →

---

## Introducción

En arquitecturas de microservicios, la comunicación entre servicios se prueba habitualmente con tests de integración de extremo a extremo que levantan todos los servicios en un entorno compartido. Este enfoque tiene un coste real en producción: entornos costosos, pipelines lentos, fallos intermitentes por dependencias de red y, lo más grave, la detección tardía de *breaking changes* solo cuando el cambio ya está desplegado. Consumer-Driven Contracts (CDC) resuelve este problema invirtiendo la responsabilidad: el consumer describe exactamente qué necesita del producer en forma de contrato; el producer genera tests a partir de ese contrato y publica stubs que el consumer usa en sus propios tests. El resultado es que ambos lados pueden testear de forma aislada con la garantía de que el contrato compartido actúa como fuente de verdad.

> [CONCEPTO] **Consumer-Driven Contracts**: metodología en la que el consumidor de una API define formalmente sus expectativas sobre el productor mediante un contrato verificable automáticamente en el pipeline CI/CD.

> [PREREQUISITO] Se asume conocimiento de Spring Boot 4.0.x, JUnit 5 y conceptos básicos de testing de integración con MockMvc o REST Assured.

## Representación visual

El diagrama siguiente muestra los tres actores del modelo CDC y el flujo de artefactos entre ellos. El contrato es el único artefacto compartido; ningún servicio depende del otro en tiempo de test.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      FLUJO CDC — SPRING CLOUD CONTRACT                  │
│                                                                         │
│  CONSUMER TEAM                       PRODUCER TEAM                      │
│  ─────────────────                   ───────────────────                │
│  1. Escribe contrato                 3. Recibe contrato                 │
│     (Groovy / YAML)                     (vía SCM o Nexus)               │
│         │                                    │                          │
│         ▼                                    ▼                          │
│  2. Publica contrato ──────────────► 4. Plugin genera tests             │
│     en repositorio                     (build/generated-test-sources)   │
│     compartido                              │                           │
│                                        5. Tests se ejecutan             │
│                                           contra implementación real    │
│                                              │                          │
│                                        6. Plugin publica Stub JAR       │
│                                           (WireMock mappings)           │
│                                              │                          │
│  8. Tests del consumer  ◄──────────── 7. Consumer descarga Stub JAR    │
│     usan WireMock stubs                   vía StubRunner                │
│     (sin llamar al                                                       │
│      producer real)                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

## Ejemplo central

El ejemplo muestra el mínimo viable del lado del producer: dependencias, un contrato Groovy simple y la clase base de test necesaria para que el plugin genere y ejecute el test automáticamente.

**Dependencias Maven (producer):**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-verifier</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-contract-maven-plugin</artifactId>
            <version>${spring-cloud-contract.version}</version>
            <extensions>true</extensions>
            <configuration>
                <baseClassForTests>com.example.producer.BaseContractTest</baseClassForTests>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Dependencias Maven (consumer):**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Contrato Groovy** (`src/test/resources/contracts/shouldReturnUser.groovy`):

```groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return user by id"
    request {
        method GET()
        url '/api/users/1'
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
            id  : 1,
            name: "Alice",
            email: anyNonEmptyString()
        ])
    }
}
```

**Clase base del producer** (`src/test/java/com/example/producer/BaseContractTest.java`):

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

**Controller del producer** (`src/main/java/com/example/producer/controller/UserController.java`):

```java
package com.example.producer.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<Map<String, Object>> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(Map.of(
            "id", id,
            "name", "Alice",
            "email", "alice@example.com"
        ));
    }
}
```

**application.yml** (producer):

```yaml
spring:
  application:
    name: user-service
server:
  port: 8080
```

**Test del consumer** (`src/test/java/com/example/consumer/UserClientContractTest.java`):

```java
package com.example.consumer;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;
import org.springframework.web.client.RestTemplate;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:user-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class UserClientContractTest {

    @Autowired
    private RestTemplate restTemplate;

    @Test
    void shouldGetUserById() {
        Map response = restTemplate.getForObject("http://localhost:8090/api/users/1", Map.class);

        assertThat(response).isNotNull();
        assertThat(response.get("id")).isEqualTo(1);
        assertThat(response.get("name")).isEqualTo("Alice");
        assertThat(response.get("email")).isNotNull();
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge la terminología fundamental del modelo CDC que todo desarrollador senior debe manejar con precisión en una entrevista técnica.

| Término | Rol | Descripción |
|---------|-----|-------------|
| Contrato | Compartido | Fichero Groovy/YAML que describe la interacción esperada entre producer y consumer |
| Producer | Proveedor | Servicio que implementa la API; ejecuta tests generados para verificar el contrato |
| Consumer | Cliente | Servicio que consume la API; usa stubs WireMock para testear sin llamar al producer real |
| Stub JAR | Artefacto | JAR publicado por el producer con los mappings WireMock generados desde los contratos |
| StubRunner | Infraestructura test | Componente que descarga y levanta el Stub JAR como servidor WireMock en el consumer |
| CDC testing | Estrategia | Verificación de compatibilidad basada en contratos definidos por el consumer |
| @AutoConfigureStubRunner | Anotación | Activa el StubRunner en tests del consumer con la configuración declarada |
| ContractVerifierMockMvc | Clase base | Superclase que configura el contexto MockMvc para los tests generados del producer |
| WireMock mapping | Artefacto interno | JSON dentro del Stub JAR que reproduce la respuesta del producer para un request definido |
| Breaking change | Evento | Modificación del producer que invalida uno o más contratos existentes |
| Contract-first | Estrategia | El consumer escribe el contrato antes de que el producer implemente el endpoint |
| Code-first | Estrategia | El producer implementa y luego extrae el contrato de la implementación existente |

## Buenas y malas prácticas

**Hacer:**

- Situar los contratos en el repositorio del producer bajo `src/test/resources/contracts/` para que el plugin los detecte automáticamente sin configuración adicional de `contractsDirectory`.
- Usar matchers (`anyNonNull()`, `byRegex()`) en los campos que cambian entre ejecuciones (UUIDs, timestamps, tokens) para evitar que los tests fallen por valores concretos no predecibles.
- Versionar los contratos en el mismo repositorio Git del producer; esto garantiza que el historial de cambios del contrato y de la implementación estén sincronizados.
- Definir una única clase base por modo de test (MockMvc, WebTestClient, RestAssured) y usar `baseClassMappings` para asignarla por directorio de contratos.
- Publicar el Stub JAR en Nexus/Artifactory como parte del pipeline CI del producer; los consumers no deben depender de builds locales del producer.

**Evitar:**

- Evitar usar valores literales (`"email": "alice@example.com"`) en campos variables del contrato: provoca fallos por acoplamiento a datos de test, no por incompatibilidad real de la API.
- Evitar omitir el Stub JAR en el artefacto publicado: sin el JAR de stubs, ningún consumer puede ejecutar sus tests de contrato en CI, paralizando los builds dependientes.
- Evitar usar `StubsMode.CLASSPATH` en CI si el Stub JAR no está en el classpath del proyecto consumer: el test pasa en local pero falla en pipeline por ausencia del artefacto.
- Evitar escribir contratos demasiado restrictivos que validen el orden de campos JSON o el formato exacto de fechas: genera fallos de contrato por cambios de serialización irrelevantes para el negocio.
- Evitar que el producer modifique un contrato existente sin coordinar con el equipo consumer: un cambio en el contrato rompe los stubs que el consumer ya usa en producción.

## Comparación: CDC vs otras estrategias de testing

La tabla siguiente muestra por qué CDC es la estrategia óptima para equipos que trabajan con microservicios independientes desplegados con frecuencia.

| Criterio | CDC (Spring Cloud Contract) | E2E Testing | Integration Testing clásico |
|---------|----------------------------|-------------|----------------------------|
| Entorno necesario | Ninguno en runtime de test | Todos los servicios activos | Servicio real o mock manual |
| Velocidad de feedback | Segundos (tests aislados) | Minutos/horas | Minutos |
| Detección de breaking changes | En CI del producer, antes del merge | En entorno de staging, post-deploy | Solo si el mock es correcto |
| Mantenimiento | Contrato como artefacto versionado | Scripts de entorno complejos | Mocks duplicados en cada consumer |
| Cobertura de comportamiento | Contrato define los casos explícitamente | Alta pero lenta y frágil | Limitada al mock manual |
| Acoplamiento entre equipos | Mínimo (solo vía contrato) | Total (deploy sincronizado) | Medio (mock compartido informalmente) |

> [EXAMEN] Pregunta frecuente en entrevistas: "¿Qué diferencia hay entre un stub y un mock en el contexto de Spring Cloud Contract?" — El stub es un artefacto generado automáticamente desde el contrato; el mock es una implementación manual. El stub garantiza que lo que el consumer testea corresponde al comportamiento real del producer porque ambos son verificados contra el mismo contrato.

> [ADVERTENCIA] Spring Cloud Contract 2025.1.1 (Oakwood) usa WireMock 3.x como motor de stubs. Las extensiones de WireMock 2.x usan el paquete `com.github.tomakehurst.wiremock`; WireMock 3.x usa `org.wiremock`. Los transformers personalizados escritos para versiones anteriores deben migrarse antes de actualizar a Oakwood.

---

← [9.12 Testing de Spring Cloud Kubernetes](sc-kubernetes-testing.md) | [Índice](README.md) | [10.2 Ciclo de vida CDC y workflow CI/CD](sc-contract-workflow.md) →
