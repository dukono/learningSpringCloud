# 10.6 Stub JAR y modos de StubRunner

← [10.5 Clase base de tests del producer](sc-contract-producer-tests.md) | [Índice](README.md) | [10.7 Stub Runner en el consumer](sc-contract-stub-runner.md) →

---

## Introducción

Cuando el producer ha verificado que su implementación cumple los contratos, necesita empaquetar esa verificación en un artefacto que los consumers puedan usar: el stub JAR. Este JAR contiene los mappings WireMock generados automáticamente desde los contratos, de modo que cualquier consumer puede arrancar un servidor WireMock local sin necesitar el producer real. El mecanismo que arranca ese servidor en los tests del consumer es el StubRunner, y exige una decisión crítica: ¿de dónde descarga los stubs? La respuesta depende del entorno —classpath local, repositorio Maven local, Nexus/Artifactory remoto o Eureka— y elegir el modo incorrecto causa fallos intermitentes en CI o dependencias circulares entre builds. Este fichero cubre la anatomía del stub JAR y la tabla de decisión de los cuatro modos.

> [TAREA] El desarrollador necesita seleccionar el modo de StubRunner apropiado para garantizar la resolución correcta de stubs en cada entorno.

> [RESULTADO] Después de leer este fichero el desarrollador puede inspeccionar la estructura interna de un stub JAR, entender cómo WireMock carga los mappings y elegir el `stubsMode` correcto para entornos de desarrollo local, CI y producción.

> [PREREQUISITO] Plugin configurado (10.3.1) y contrato DSL escrito (10.4.1).

## Representación visual

El siguiente diagrama muestra el ciclo completo desde el contrato hasta el stub en uso por el consumer.

```
[src/test/resources/contracts/orders/shouldReturnOrder.groovy]
         │
         ▼ (mvn package / ./gradlew build)
[spring-cloud-contract-maven-plugin]
         │
         ├─── genera ──► target/generated-test-sources/  (tests del producer)
         │
         └─── empaqueta ► order-service-1.0.0-stubs.jar
                              └─ META-INF/
                                  └─ com.example/
                                      └─ order-service/
                                          └─ 1.0.0/
                                              └─ mappings/
                                                  └─ orders/
                                                      └─ shouldReturnOrder.json  ← WireMock mapping
```

**Tabla de modos de StubRunner:**

```
StubsMode        Origen de los stubs         Cuándo usar
─────────────────────────────────────────────────────────────────────────
CLASSPATH        JAR en el classpath         CI / tests unitarios offline
LOCAL            ~/.m2 (repositorio local)   Desarrollo local activo
REMOTE           Nexus / Artifactory         Integración entre equipos
SERVICE          Eureka + repositorio         Entornos desplegados
```

## Ejemplo central

El siguiente ejemplo muestra la estructura interna del stub JAR, el mapping WireMock generado y cómo configurar el StubRunner en los cuatro modos.

**Estructura interna del stub JAR (modo CLASSPATH):**

```
order-service-1.0.0-stubs.jar
└─ META-INF/
    └─ com.example/
        └─ order-service/
            └─ 1.0.0/
                ├─ mappings/
                │   └─ orders/
                │       └─ shouldReturnOrder.json
                └─ contracts/
                    └─ orders/
                        └─ shouldReturnOrder.groovy
```

**Contenido de shouldReturnOrder.json (mapping WireMock generado):**

```json
{
  "id" : "a3f4c2e1-1234-5678-abcd-000000000001",
  "request" : {
    "method" : "GET",
    "url" : "/orders/42",
    "headers" : {
      "Content-Type" : {
        "matches" : "application/json.*"
      }
    }
  },
  "response" : {
    "status" : 200,
    "body" : "{\"id\":42,\"nif\":\"12345678Z\",\"amount\":99.99,\"createdAt\":\"2025-01-15T10:00:00Z\"}",
    "headers" : {
      "Content-Type" : "application/json"
    },
    "transformers" : [ "response-template" ]
  }
}
```

**StubsMode.CLASSPATH — para tests CI sin repositorio:**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:1.0.0:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderConsumerClasspathTest {

    @Autowired
    private OrderClient orderClient;

    @Test
    void shouldFetchOrder() {
        OrderDTO order = orderClient.getOrder(42L);
        assertThat(order.getId()).isEqualTo(42L);
    }
}
```

**pom.xml del consumer — añadir el stub JAR como dependencia test:**

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>
    <classifier>stubs</classifier>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>*</groupId>
            <artifactId>*</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

**StubsMode.LOCAL — para desarrollo local con `mvn install` del producer:**

```java
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderConsumerLocalTest { ... }
```

**StubsMode.REMOTE — para integración con Nexus/Artifactory:**

```java
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.REMOTE,
    repositoryRoot = "http://nexus:8081/repository/maven-releases/"
)
class OrderConsumerRemoteTest { ... }
```

**Alternativamente via application.yml para evitar datos de infra en el código:**

```yaml
stubrunner:
  stubs-mode: remote
  repository-root: http://nexus:8081/repository/maven-releases/
  ids: com.example:order-service:+:stubs:8090
```

**StubsMode.SERVICE — con Eureka activo:**

```yaml
stubrunner:
  stubs-mode: service
  ids: com.example:order-service:+:stubs
eureka:
  client:
    service-url:
      default-zone: http://localhost:8761/eureka
```

**Stub adicional registrado manualmente con WireMock DSL (sin contrato):**

```java
import static com.github.tomakehurst.wiremock.client.WireMock.*;

@Autowired
private StubFinder stubFinder;  // inyectado por @AutoConfigureStubRunner

@BeforeEach
void addExtraStub() {
    stubFinder.findStubUrl("com.example", "order-service")
        .ifPresent(url -> {
            // registra stub extra que no existe en el contrato
            configureFor("localhost", url.getPort());
            stubFor(get(urlEqualTo("/orders/999"))
                .willReturn(aResponse().withStatus(404)));
        });
}
```

## Tabla de elementos clave

Los parámetros críticos para configurar correctamente el stub JAR y el StubRunner.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `stubsMode` | enum | `CLASSPATH` | Origen de los stubs: CLASSPATH, LOCAL, REMOTE, SERVICE |
| `ids` | String | — | Formato: `groupId:artifactId:version:classifier:port` |
| `repositoryRoot` | String | — | URL de Nexus/Artifactory para StubsMode.REMOTE |
| `classifier` | String | `stubs` | Classifier del JAR; debe coincidir con el generado por el plugin |
| `port` | int | aleatorio | Puerto WireMock; 0 = dinámico (usar `@StubRunnerPort`) |
| `mappings/` | Directorio en JAR | — | Ubicación de los mappings WireMock dentro del stub JAR |
| `META-INF/groupId/artifactId/version/` | Ruta en JAR | — | Convención de rutas; debe coincidir exactamente con groupId/artifactId |
| `response-template` | Transformer WireMock | activo | Permite valores dinámicos en responses (Handlebars) |
| `<verifierStubsJar>` | Task Gradle | — | Task que genera el stub JAR; equivale al goal `generateStubs` Maven |
| `<generateStubs>` | Goal Maven | — | Genera el stub JAR como artefacto clasificado `stubs` |

## Buenas y malas prácticas

**Hacer:**

- Usar `StubsMode.CLASSPATH` en el pipeline CI con el stub JAR añadido como dependencia `test` con `<exclusions><exclusion>*:*</exclusion></exclusions>`; así los tests no descargan nada de la red y el pipeline es reproducible offline.
- Usar `version=+` (latest) en `stubrunner.ids` durante el desarrollo local con `StubsMode.LOCAL`; el consumer siempre usa el stub más reciente del producer sin cambiar la versión manualmente.
- Fijar la versión exacta del stub JAR en el pipeline CI de release (`1.0.0` en lugar de `+`); el `+` en CI puede causar que una build tome un stub SNAPSHOT recién publicado que rompe los tests del consumer por una razón no relacionada.
- Añadir `<exclusions>` al stub JAR en el `pom.xml` del consumer para evitar que las dependencias transitivas del producer contaminen el classpath del consumer y causen conflictos de versiones.

**Evitar:**

- No usar `StubsMode.REMOTE` con `version=+` en un repositorio que mezcla SNAPSHOTs y releases: el resolver puede tomar una versión SNAPSHOT en un entorno de release, introduciendo comportamientos no verificados en producción.
- No editar manualmente los mappings `.json` dentro del stub JAR: el plugin los regenera en cada build, borrando cualquier cambio manual. Si necesitas un stub diferente, crea un contrato nuevo.
- No usar `StubsMode.SERVICE` (Eureka) en entornos CI efímeros: el test requiere Eureka activo y el stub JAR registrado, lo que introduce una dependencia de infraestructura que causa fallos intermitentes cuando Eureka no arranca a tiempo.
- No omitir el transformer `response-template` en stubs con valores dinámicos: WireMock sin el transformer devuelve literalmente `{{randomValue()}}` como string, y el test del consumer falla al parsear un valor que no es el tipo esperado.

## Comparación: cuatro modos de StubRunner

La decisión de qué modo usar determina la velocidad de feedback y el nivel de realismo del test del consumer.

| Criterio | CLASSPATH | LOCAL | REMOTE | SERVICE |
|---|---|---|---|---|
| Necesita red | No | No | Sí | Sí (Eureka) |
| Refleja último producer | No (snapshot fijo) | Sí (mvn install) | Sí (+) | Sí |
| Velocidad CI | Muy alta | Alta | Media | Baja |
| Complejidad config | Baja | Baja | Media | Alta |
| Entorno recomendado | CI/CD pipeline | Dev local | Integración | Staging |

---

← [10.5 Clase base de tests del producer](sc-contract-producer-tests.md) | [Índice](README.md) | [10.7 Stub Runner en el consumer](sc-contract-stub-runner.md) →

