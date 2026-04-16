# 10.9 Repositorio de contratos y publicación de stubs

← [10.8 Contract Messaging — contratos de eventos](sc-contract-messaging.md) | [Índice](README.md) | [10.10 Compatibilidad, versionado y troubleshooting de contratos](sc-contract-troubleshooting.md) →

---

## Introducción

El stub JAR generado por el producer es inútil si el consumer no puede encontrarlo. En equipos pequeños funciona pasar el JAR manualmente, pero en organizaciones con decenas de microservicios la distribución de stubs requiere una estrategia: o un repositorio Maven centralizado (Nexus/Artifactory) o un repositorio Git de contratos compartidos (SCM-based). La elección determina quién tiene la responsabilidad del contrato, cómo se versiona y cómo el CI/CD de cada equipo accede a los stubs sin coordinación manual. Elegir el enfoque incorrecto produce stubs obsoletos en circulación o cuellos de botella donde un equipo bloquea los builds de otros al no publicar sus stubs a tiempo. Este fichero cubre los dos enfoques, su configuración y la gestión del versionado semántico.

> [TAREA] El desarrollador necesita publicar stubs en un repositorio centralizado para que los consumers los descarguen automáticamente en sus builds.

> [RESULTADO] Después de leer este fichero el desarrollador puede configurar `publishStubsToScm` para un repositorio Git de contratos, versionar stubs con SNAPSHOT/release y configurar el consumer para descargar la versión correcta desde Nexus/Artifactory o desde el repositorio Git.

> [PREREQUISITO] Plugin configurado (10.3.1), stub JAR generado (10.6) y acceso a un repositorio Maven o Git central.

## Representación visual

El siguiente diagrama compara los dos enfoques de distribución de stubs.

```
ENFOQUE 1: Nexus / Artifactory (repositorio Maven)
───────────────────────────────────────────────────
[Producer CI]
  mvn deploy  →  Nexus/Artifactory
                    └─ com.example:order-service:1.0.0:stubs

[Consumer CI]
  @AutoConfigureStubRunner(ids="com.example:order-service:+:stubs", stubsMode=REMOTE)
  StubRunner descarga desde Nexus

ENFOQUE 2: SCM-based contracts (repositorio Git)
──────────────────────────────────────────────────
[Consumer]
  escribe contrato en repo Git centralizado
  └─ contracts/com/example/order-service/1.0.0/
       └─ shouldReturnOrder.groovy

[Producer CI]
  mvn generate-test-sources  ← descarga contratos del repo Git
  mvn test                   ← ejecuta tests generados
  mvn verify                 ← si pasan, publica stubs al mismo repo Git

[Consumer CI]
  StubRunner descarga stubs desde el repo Git
```

## Ejemplo central

Los siguientes ejemplos muestran la configuración completa para los dos enfoques de distribución.

**Enfoque 1 — Publicación en Nexus/Artifactory con mvn deploy:**

```xml
<!-- pom.xml del producer -->
<distributionManagement>
    <repository>
        <id>releases</id>
        <url>http://nexus:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <url>http://nexus:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-contract-maven-plugin</artifactId>
            <version>${spring-cloud-contract.version}</version>
            <extensions>true</extensions>
            <configuration>
                <baseClassForTests>com.example.BaseContractTest</baseClassForTests>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```bash
# Pipeline CI del producer: genera stubs y los publica
mvn clean verify         # ejecuta tests de contrato
mvn deploy               # publica artefacto + stub JAR en Nexus
# Nexus recibe: order-service-1.0.0.jar Y order-service-1.0.0-stubs.jar
```

**Consumer descargando desde Nexus:**

```yaml
# src/test/resources/application.yml del consumer
stubrunner:
  stubs-mode: remote
  repository-root: http://nexus:8081/repository/maven-releases/
  ids: com.example:order-service:1.0.0:stubs:8090
```

**Versión + para siempre usar la última release:**

```yaml
stubrunner:
  ids: com.example:order-service:+:stubs:8090   # + = latest release
  stubs-mode: remote
  repository-root: http://nexus:8081/repository/maven-releases/
```

**Enfoque 2 — SCM-based contracts con repositorio Git centralizado:**

Estructura del repositorio Git de contratos:

```
contracts-repo/
└─ com/
    └─ example/
        └─ order-service/
            ├─ 1.0.0/
            │   └─ shouldReturnOrder.groovy
            └─ 1.1.0/
                └─ shouldReturnOrderWithTracking.groovy
```

```xml
<!-- pom.xml del producer: leer contratos desde el repo Git -->
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>${spring-cloud-contract.version}</version>
    <extensions>true</extensions>
    <configuration>
        <contractsRepositoryUrl>
            git://https://github.com/myorg/contracts-repo.git
        </contractsRepositoryUrl>
        <contractsPath>
            com/example/order-service/1.0.0
        </contractsPath>
        <baseClassForTests>com.example.BaseContractTest</baseClassForTests>
        <!-- Publicar stubs generados de vuelta al mismo repo Git -->
        <pushStubsToScm>true</pushStubsToScm>
    </configuration>
</plugin>
```

**Comando para publicar stubs al SCM manualmente:**

```bash
mvn spring-cloud-contract:pushStubsToScm
# equivalente Gradle:
./gradlew pushStubsToScm
```

**Consumer leyendo stubs desde el repo Git:**

```yaml
# src/test/resources/application.yml del consumer
stubrunner:
  stubs-mode: remote
  repository-root: git://https://github.com/myorg/contracts-repo.git
  ids: com.example:order-service:1.0.0:stubs:8090
```

**Versionado semántico — gestión de SNAPSHOT vs release:**

```yaml
# Durante desarrollo activo del producer (rama feature):
stubrunner:
  ids: com.example:order-service:1.1.0-SNAPSHOT:stubs:8090
  stubs-mode: remote
  repository-root: http://nexus:8081/repository/maven-snapshots/

# En pipeline CI de release del consumer:
stubrunner:
  ids: com.example:order-service:1.0.0:stubs:8090  # versión fija
  stubs-mode: remote
  repository-root: http://nexus:8081/repository/maven-releases/
```

**Gestión de múltiples versiones de API en paralelo:**

```yaml
# Consumer que soporta tanto v1.0.0 como v1.1.0 del producer:
stubrunner:
  ids:
    - com.example:order-service:1.0.0:stubs:8090
    - com.example:order-service:1.1.0:stubs:8091
  stubs-mode: remote
```

## Tabla de elementos clave

Los parámetros de publicación y descarga de stubs que un profesional senior debe conocer.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `pushStubsToScm` | boolean | `false` | Publica stubs al repositorio Git de contratos configurado |
| `contractsRepositoryUrl` | String | — | URL del repo Git de contratos; prefijo `git://` para SCM-based |
| `contractsPath` | String | — | Subdirectorio dentro del repo Git con los contratos de este servicio |
| `repositoryRoot` | String | — | URL Nexus/Artifactory o Git para `StubsMode.REMOTE` |
| `SNAPSHOT` | Sufijo versión | — | Versión en desarrollo; publicada en repositorio de snapshots |
| `+` en version | String | — | Resuelve a la última versión release disponible en el repositorio |
| `mvn deploy` | Comando | — | Publica el artefacto y el stub JAR clasificado `stubs` en Nexus |
| `spring-cloud-contract:pushStubsToScm` | Goal Maven | — | Publica stubs al repo SCM configurado sin hacer `mvn deploy` |
| `verifierStubsJar` | Task Gradle | — | Genera el stub JAR; debe ejecutarse antes de `pushStubsToScm` |
| `contracts-repo/groupId/artifactId/version/` | Convenio | — | Estructura de directorios obligatoria en el repo Git de contratos |

## Buenas y malas prácticas

**Hacer:**

- Usar el enfoque SCM-based (Git de contratos) cuando el consumer escribe los contratos antes de que el producer los implemente: el consumer hace un PR al repo de contratos, el producer lo acepta e implementa. El flujo es explícito y trazable.
- Usar Nexus/Artifactory cuando el producer es el propietario del contrato y el consumer simplemente descarga los stubs del artefacto publicado: es el flujo Maven estándar y no requiere infraestructura adicional.
- Fijar la versión exacta del stub (`1.0.0`, no `+`) en los pipelines de release del consumer: `+` puede resolver a una versión publicada por el producer en las últimas horas que no ha sido validada todavía.
- Activar la purga de SNAPSHOTs antiguos en Nexus (política de retención); sin purga, el repositorio acumula cientos de versiones `-SNAPSHOT` de stubs que nunca se usarán y aumentan los tiempos de resolución de `+`.

**Evitar:**

- No usar el mismo repositorio Nexus para SNAPSHOTs y releases de stubs: los consumers que apuntan a `maven-releases` nunca verán los SNAPSHOTs, causando `StubNotFoundException` intermitentes cuando el producer solo ha publicado un SNAPSHOT.
- No comprometer los stubs generados (`.json` WireMock) directamente en el repositorio de código del producer: los stubs son artefactos de build, no código fuente. Pertenecen al stub JAR publicado, no al control de versiones.
- No usar `contractsRepositoryUrl` sin configurar credenciales Git en el entorno CI: el plugin intenta un `git clone` durante la fase `generate-test-sources` y falla silenciosamente si no tiene acceso, produciendo 0 tests generados sin error visible.
- No mezclar contratos de v1 y v2 de la API en el mismo directorio del repo Git: usar subdirectorios por versión (`/1.0.0/`, `/2.0.0/`) permite mantener ambas versiones en producción mientras se migra el consumer.

## Comparación: Nexus/Artifactory vs SCM-based contracts

La elección del enfoque de distribución determina quién es el propietario del contrato y cómo fluye el cambio entre equipos.

| Criterio | Nexus / Artifactory | SCM-based (Git) |
|---|---|---|
| Propietario del contrato | Producer | Consumer (PR al repo) |
| Momento de escritura del contrato | Después de implementar | Antes de implementar (contract-first) |
| Trazabilidad de cambios | Via versión Maven | Via historial Git (PR, reviews) |
| Infraestructura requerida | Nexus/Artifactory | Repositorio Git adicional |
| Complejidad de setup | Baja | Media |
| Flujo recomendado por Spring Cloud | Ambos soportados | Preferido para CDC estricto |

---

← [10.8 Contract Messaging — contratos de eventos](sc-contract-messaging.md) | [Índice](README.md) | [10.10 Compatibilidad, versionado y troubleshooting de contratos](sc-contract-troubleshooting.md) →

