# 11.5 Integración con Spring Cloud Data Flow

← [11.4 Argumentos y parámetros de lanzamiento](sc-task-arguments.md) | [Índice (README.md)](README.md) | [11.6 Composed Tasks y Composed Task Runner](sc-task-composed.md) →

---

## Introducción

Spring Cloud Task puede ejecutarse de forma autónoma (con `java -jar`), pero su propósito más potente es ser orquestado por Spring Cloud Data Flow (SCDF): una plataforma que registra las aplicaciones de tarea, las lanza bajo demanda o de forma programada, pasa argumentos dinámicos y mantiene el historial de ejecuciones integrado con el `TaskRepository`. Sin SCDF, cada equipo implementa su propio sistema de lanzamiento (cron, scripts bash, jobs de CI/CD) sin visibilidad centralizada ni capacidad de relanzar ejecuciones fallidas con los mismos parámetros. Este fichero cubre la perspectiva de la tarea dentro de SCDF —registro, lanzamiento, correlación de IDs, parámetros dinámicos y el `TaskLauncher` SPI— sin entrar en la instalación o operación del servidor SCDF, que está fuera del alcance de este módulo.

## Representación visual

El diagrama muestra el flujo completo desde el registro de la tarea en SCDF hasta la ejecución en la plataforma destino y la correlación con `TASK_EXECUTION`.

```
  SPRING CLOUD DATA FLOW SERVER
  ─────────────────────────────────────────────────────────────────────────
  │
  │  1. App Registry                      2. Task Definition
  │  ┌─────────────────────┐              ┌──────────────────────────────┐
  │  │ nombre: purge-task  │──── registra ─│ task create purge --definition│
  │  │ uri: docker://...   │              │ "purge-task"                 │
  │  │ type: task          │              └──────────────────────────────┘
  │  └─────────────────────┘                              │
  │                                                        │ 3. Launch
  │                               ┌───────────────────────┘
  │                               ▼
  │  REST API / Shell: POST /tasks/executions?name=purge&--task.date=2025-01-15
  │                               │
  │                               ▼
  │  TaskLauncher SPI  ──────── selecciona implementación
  │  ┌──────────────────────────────────────────────────────────────┐
  │  │  LocalTaskLauncher         │  K8sTaskLauncher                │
  │  │  (local, desarrollo)       │  (Kubernetes Job)               │
  │  │                            │                                 │
  │  │  CloudFoundryTaskLauncher  │  (extensible via SPI)           │
  │  │  (CF tasks)                │                                 │
  │  └──────────────────────────────────────────────────────────────┘
  │                               │
  ─────────────────────────────── │ ─────────────────────────────────────
                                  ▼
  PROCESO DE TAREA (JVM)
  ─────────────────────────────────────────────────────────────────────────
  │  @SpringBootApplication + @EnableTask
  │                               │
  │  TaskExecution creada         │  external_execution_id = SCDF execution id
  │  en TASK_EXECUTION            │  parent_execution_id  = null (tarea raíz)
  │                               │
  │  [lógica de negocio]          │
  │                               │
  │  TaskExecution actualizada    │  exit_code, end_time, error_message
  ─────────────────────────────────────────────────────────────────────────
                                  │
                                  ▼
  SCDF monitoriza estado:  GET /tasks/executions/{id}
  correlaciona via external_execution_id ↔ TASK_EXECUTION
```

## Ejemplo central

El siguiente ejemplo muestra cómo registrar una tarea en SCDF, lanzarla via REST API y recuperar el estado. El código Java de la tarea ya fue visto en ficheros anteriores; aquí el foco está en la integración con SCDF.

**Dependencias Maven de la tarea (`pom.xml`) — sin cambios respecto a una tarea standalone:**

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
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-task</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Clase principal de la tarea (`PurgeTaskApp.java`):**

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.stereotype.Component;

import java.time.LocalDate;

@SpringBootApplication
@EnableTask
public class PurgeTaskApp {
    public static void main(String[] args) {
        SpringApplication.run(PurgeTaskApp.class, args);
    }
}

@Component
class PurgeRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // SCDF pasa los argumentos como named args con --
        String dateStr = args.getOptionValues("task.date") != null
                ? args.getOptionValues("task.date").get(0)
                : LocalDate.now().toString();

        String tenant = args.getOptionValues("task.tenant") != null
                ? args.getOptionValues("task.tenant").get(0)
                : "DEFAULT";

        System.out.printf("Purgando registros anteriores a %s para tenant %s%n", dateStr, tenant);
        // Lógica de purga...
        System.out.println("Purga completada exitosamente.");
    }
}
```

**Configuración de la tarea (`application.yml`):**

```yaml
spring:
  application:
    name: purge-task
  datasource:
    # SCDF configura el DataSource en el entorno de lanzamiento
    # En desarrollo local usar H2:
    url: jdbc:h2:mem:taskdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  cloud:
    task:
      name: purge-task
      initialize-enabled: true
      closecontextEnabled: true
```

**Operaciones REST API de SCDF para gestionar la tarea:**

```bash
# 1. Registrar la aplicación de tarea en el App Registry de SCDF
curl -X POST "http://scdf-server:9393/apps/task/purge-task" \
  -d "uri=docker://myregistry/purge-task:1.0.0"

# 2. Crear la definición de tarea
curl -X POST "http://scdf-server:9393/tasks/definitions" \
  -d "name=purge&definition=purge-task"

# 3. Lanzar la tarea con argumentos dinámicos
# SCDF pasa los argumentos como propiedades de Spring
curl -X POST "http://scdf-server:9393/tasks/executions" \
  -d "name=purge" \
  -d "arguments=--task.date=2025-01-15 --task.tenant=ACME" \
  -d "properties=deployer.purge.memory=512"

# Respuesta: { "executionId": 42 }

# 4. Consultar el estado de la ejecución
curl "http://scdf-server:9393/tasks/executions/42"
# Respuesta incluye: taskName, startTime, endTime, exitCode, arguments

# 5. Listar todas las ejecuciones de una tarea
curl "http://scdf-server:9393/tasks/executions?name=purge&page=0&size=20"

# Workaround de unicidad — añadir timestamp para relanzar con mismos parámetros
curl -X POST "http://scdf-server:9393/tasks/executions" \
  -d "name=purge" \
  -d "arguments=--task.date=2025-01-15 --task.tenant=ACME --run.timestamp=$(date +%s)"
```

**Comando equivalente con el SCDF Shell:**

```
dataflow:>app register --name purge-task --type task --uri docker://myregistry/purge-task:1.0.0
dataflow:>task create purge --definition "purge-task"
dataflow:>task launch purge --arguments "--task.date=2025-01-15 --task.tenant=ACME"
dataflow:>task execution list --name purge
```

## Tabla de elementos clave

La siguiente tabla recoge los conceptos de integración Task-SCDF y el `TaskLauncher` SPI que un desarrollador senior debe conocer.

| Concepto | Tipo | Descripción |
|---|---|---|
| App Registry | SCDF concept | Catálogo de aplicaciones registradas con nombre, tipo (`task`/`source`/`processor`/`sink`) y URI (Docker, Maven) |
| Task Definition | SCDF concept | Asociación nombre-lógico ↔ app registrada: `task create purge --definition "purge-task"` |
| `POST /tasks/executions` | REST endpoint | Lanza una tarea. Parámetros: `name`, `arguments` (args de la tarea), `properties` (propiedades del deployer) |
| `external_execution_id` | `TASK_EXECUTION` column | ID de SCDF asignado a la ejecución; permite correlacionar el historial de la tarea con el registro de SCDF |
| `parent_execution_id` | `TASK_EXECUTION` column | ID de la tarea padre en Composed Tasks; `null` para tareas raíz |
| `TaskLauncher` SPI | Interfaz | SPI de extensión para implementar nuevas plataformas de lanzamiento; los métodos son `launch()`, `cancel()`, `getStatus()` |
| `LocalTaskLauncher` | Implementación | Lanza la tarea como proceso local en la misma máquina; solo para desarrollo |
| `KubernetesTaskLauncher` | Implementación | Crea un Kubernetes Job por cada ejecución; la implementación más usada en producción |
| `CloudFoundryTaskLauncher` | Implementación | Lanza un CF Task en Cloud Foundry; gestiona el ciclo de vida del proceso CF |
| Unicidad de argumentos | Restricción SCDF | SCDF impide lanzar con exactamente los mismos argumentos que una ejecución previa; workaround: añadir `--run.timestamp=$(date +%s)` |

## Buenas y malas prácticas

**Hacer:**

- Registrar la tarea en SCDF con URI de Docker o Maven en lugar de rutas de archivo locales. Las URIs de registro son reproducibles en cualquier entorno y forman parte del proceso de release.
- Usar `properties` del lanzamiento (no `arguments`) para configuraciones de infraestructura como memoria, CPU o tolerations de Kubernetes. Los `arguments` son para lógica de negocio; los `properties` son para el deployer.
- Leer `external_execution_id` desde `TaskExecution` en los listeners para enriquecer logs y métricas con el contexto de la plataforma de orquestación. Facilita la correlación al diagnosticar fallos.
- Añadir un argumento de `timestamp` como política estándar de lanzamiento cuando la tarea puede relanzarse con los mismos parámetros de negocio. Documentarlo como parte de la interfaz de la tarea.

**Evitar:**

- Pasar secretos como `arguments` de lanzamiento en la llamada REST a SCDF. Los argumentos quedan registrados en `TASK_EXECUTION_PARAMS` en texto plano y son visibles en el dashboard de SCDF. Usar propiedades de deployer con referencias a secretos de Kubernetes o Vault.
- Usar `LocalTaskLauncher` en producción. Lanza la tarea en el mismo proceso que SCDF, lo que puede causar OOM si la tarea consume mucha memoria y no ofrece aislamiento de fallos.
- Ignorar el `exit_code` devuelto por la tarea en el flujo CI/CD de SCDF. Un exit code distinto de cero indica fallo; si SCDF no está configurado para alertar o relanzar en ese caso, los fallos silenciosos se acumulan.
- Asumir que el `executionId` de SCDF (`external_execution_id`) y el `task_execution_id` de `TASK_EXECUTION` son el mismo valor. Son identificadores distintos en sistemas distintos; la correlación se hace a través de la columna `external_execution_id`.

> **[CONCEPTO]** `TaskLauncher` es el SPI que separa "qué ejecutar" (la lógica de la tarea) de "dónde ejecutar" (la plataforma). Esta separación permite que el mismo artefacto de tarea se despliegue en local (desarrollo), Kubernetes (staging) o Cloud Foundry (producción) sin modificar el código de la tarea.

> **[ADVERTENCIA]** Spring Cloud Data Flow como plataforma (servidor, base de datos, UI) está fuera del alcance de este módulo. Este fichero solo cubre la perspectiva de la tarea Task dentro de SCDF. La instalación, configuración y operación del servidor SCDF se documenta en dataflow.spring.io.

---

← [11.4 Argumentos y parámetros de lanzamiento](sc-task-arguments.md) | [Índice (README.md)](README.md) | [11.6 Composed Tasks y Composed Task Runner](sc-task-composed.md) →
