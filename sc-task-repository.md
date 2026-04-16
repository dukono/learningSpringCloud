# 11.2 TaskRepository y persistencia de ejecuciones

← [11.1 Fundamentos de Spring Cloud Task](sc-task-fundamentos.md) | [Índice (README.md)](README.md) | [11.3 Configuración y propiedades spring.cloud.task.*](sc-task-configuracion.md) →

---

## Introducción

Cada vez que una aplicación Spring Cloud Task arranca, su ciclo de vida queda registrado en un esquema relacional que persiste entre ejecuciones. Sin ese esquema, no es posible auditar qué tareas se ejecutaron, cuándo terminaron, con qué argumentos y si tuvieron éxito o error. El módulo define dos tablas principales —`TASK_EXECUTION` y `TASK_EXECUTION_PARAMS`— y dos interfaces Java para operar con ellas: `TaskRepository` para escritura y `TaskExplorer` para lectura. Entender el esquema completo es indispensable para diagnosticar problemas en producción (tareas huérfanas, ejecuciones duplicadas, argumentos incorrectos) y para usarlo como fuente de verdad en tests de integración que verifican que la tarea se ejecutó correctamente.

## Representación visual

El siguiente diagrama muestra la relación entre las dos tablas del esquema Task y las interfaces Java que las gestionan.

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                        TASK_EXECUTION                               │
  ├──────────────────────────┬──────────────────────────────────────────┤
  │ task_execution_id (PK)   │ BIGINT IDENTITY — identificador único    │
  │ task_name                │ VARCHAR(100) — valor de spring.cloud.task.name │
  │ start_time               │ DATETIME(6) — instante de arranque       │
  │ end_time                 │ DATETIME(6) — instante de fin (null si en curso o huérfana) │
  │ exit_code                │ INTEGER — 0=éxito, !=0=fallo             │
  │ exit_message             │ VARCHAR(2500) — mensaje de salida        │
  │ error_message            │ TEXT — stack trace si hubo excepción     │
  │ last_updated             │ DATETIME(6) — última modificación        │
  │ external_execution_id    │ VARCHAR(255) — id externo (SCDF, k8s job)│
  │ parent_execution_id      │ BIGINT — FK a task_execution_id (CTR)   │
  └──────────────────────────┴──────────────────────────────────────────┘
                │ 1
                │
                │ N
  ┌─────────────────────────────────────────────────────────────────────┐
  │                      TASK_EXECUTION_PARAMS                          │
  ├──────────────────────────┬──────────────────────────────────────────┤
  │ task_execution_id (FK)   │ BIGINT — referencia a TASK_EXECUTION     │
  │ task_param               │ VARCHAR(2500) — argumento de lanzamiento │
  └──────────────────────────┴──────────────────────────────────────────┘

  Interfaces Java:
  ┌────────────────────┐        ┌──────────────────────────────────────┐
  │  TaskRepository    │        │  TaskExplorer                        │
  │ (escritura)        │        │ (lectura)                            │
  ├────────────────────┤        ├──────────────────────────────────────┤
  │ createTaskExecution│        │ getTaskExecution(id)                 │
  │ updateTaskExecution│        │ getTaskExecutionCount()              │
  │ completeTaskExecution│      │ getTaskExecutionCountByTaskName(name)│
  └─────────┬──────────┘        │ findTaskExecutionsByName(name, page) │
            │                   │ getLatestTaskExecutionForTaskName()  │
            ▼                   └──────────────────────────────────────┘
  SimpleTaskRepository
  (implementación JDBC por defecto)
```

## Ejemplo central

El ejemplo completo muestra cómo configurar el esquema, consultar ejecuciones pasadas con `TaskExplorer` y detectar tareas huérfanas con `end_time = null`.

**Dependencias Maven (`pom.xml`):**

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
    <!-- Producción: PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Clase principal con consulta de ejecuciones (`ReportTask.java`):**

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Component;

import java.util.List;

@SpringBootApplication
@EnableTask
public class ReportTask {
    public static void main(String[] args) {
        SpringApplication.run(ReportTask.class, args);
    }
}

@Component
class ReportRunner implements ApplicationRunner {

    private final TaskExplorer taskExplorer;

    ReportRunner(TaskExplorer taskExplorer) {
        this.taskExplorer = taskExplorer;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        String taskName = "report-task";

        // Número total de ejecuciones registradas para esta tarea
        long total = taskExplorer.getTaskExecutionCountByTaskName(taskName);
        System.out.println("Ejecuciones previas de '" + taskName + "': " + total);

        // Última ejecución para determinar si la anterior fue exitosa
        TaskExecution last = taskExplorer.getLatestTaskExecutionForTaskName(taskName);
        if (last != null) {
            System.out.println("Última ejecución: id=" + last.getExecutionId()
                    + ", exitCode=" + last.getExitCode()
                    + ", start=" + last.getStartTime()
                    + ", end=" + last.getEndTime());

            // Detectar ejecución huérfana (kill abrupto en ejecución anterior)
            if (last.getEndTime() == null) {
                System.err.println("ADVERTENCIA: ejecución huérfana detectada con id="
                        + last.getExecutionId() + ". Considera limpiarla antes de continuar.");
            }
        }

        // Paginación de ejecuciones — útil para auditoría masiva
        Page<TaskExecution> page = taskExplorer.findTaskExecutionsByName(
                taskName, PageRequest.of(0, 10));
        page.getContent().forEach(e ->
                System.out.printf("  id=%d, exitCode=%s, start=%s%n",
                        e.getExecutionId(), e.getExitCode(), e.getStartTime()));

        System.out.println("Lógica de generación de informe completada.");
    }
}
```

**Configuración (`application.yml`) con esquema en PostgreSQL:**

```yaml
spring:
  application:
    name: report-task
  datasource:
    url: jdbc:postgresql://localhost:5432/taskdb
    username: taskuser
    password: taskpassword
    driver-class-name: org.postgresql.Driver
  cloud:
    task:
      name: report-task
      initialize-enabled: true     # Spring Cloud Task ejecuta schema-postgresql.sql automáticamente
      tablePrefix: REPORT_          # aisla las tablas: REPORT_TASK_EXECUTION, REPORT_TASK_EXECUTION_PARAMS
```

**Script de esquema manual (si `initialize-enabled: false`):**

Los scripts se incluyen en el JAR de Spring Cloud Task en `org/springframework/cloud/task/schema-postgresql.sql`. El esquema mínimo es:

```sql
-- TASK_EXECUTION
CREATE TABLE IF NOT EXISTS TASK_EXECUTION (
    TASK_EXECUTION_ID     BIGSERIAL PRIMARY KEY,
    TASK_NAME             VARCHAR(100),
    START_TIME            TIMESTAMP(6),
    END_TIME              TIMESTAMP(6),
    EXIT_CODE             INTEGER,
    EXIT_MESSAGE          VARCHAR(2500),
    ERROR_MESSAGE         TEXT,
    LAST_UPDATED          TIMESTAMP(6),
    EXTERNAL_EXECUTION_ID VARCHAR(255),
    PARENT_EXECUTION_ID   BIGINT
);

-- TASK_EXECUTION_PARAMS
CREATE TABLE IF NOT EXISTS TASK_EXECUTION_PARAMS (
    TASK_EXECUTION_ID BIGINT NOT NULL REFERENCES TASK_EXECUTION(TASK_EXECUTION_ID),
    TASK_PARAM        VARCHAR(2500)
);
```

## Tabla de elementos clave

La siguiente tabla recoge las columnas de `TASK_EXECUTION` y los métodos de `TaskExplorer` que un desarrollador senior necesita memorizar para entrevistas y diagnóstico en producción.

| Elemento | Tipo/Columna | Default | Descripción |
|---|---|---|---|
| `task_execution_id` | `BIGINT` PK | autoincremento | Identificador único de la ejecución |
| `task_name` | `VARCHAR(100)` | nombre del app | Valor de `spring.cloud.task.name` |
| `start_time` | `DATETIME(6)` | — | Instante de arranque del contexto Spring |
| `end_time` | `DATETIME(6)` | `null` | `null` indica ejecución en curso o huérfana |
| `exit_code` | `INTEGER` | `null` → `0` | 0 = éxito; distinto de 0 = fallo |
| `error_message` | `TEXT` | `null` | Stack trace completo si hubo excepción no capturada |
| `external_execution_id` | `VARCHAR(255)` | `null` | ID asignado por SCDF o el orquestador externo |
| `parent_execution_id` | `BIGINT` FK | `null` | Apunta al CTR padre en Composed Tasks |
| `task_param` | `VARCHAR(2500)` | — | Cada argumento de lanzamiento en fila separada |
| `TaskExplorer.getTaskExecution(id)` | Método | — | Recupera una ejecución por su ID primario |
| `TaskExplorer.getTaskExecutionCountByTaskName(name)` | Método | — | Número histórico de ejecuciones para una tarea |
| `TaskExplorer.findTaskExecutionsByName(name, page)` | Método | — | Lista paginada de ejecuciones de una tarea |
| `TaskExplorer.getLatestTaskExecutionForTaskName(name)` | Método | — | Última ejecución (por `start_time`) de una tarea |
| `spring.cloud.task.initialize-enabled` | `Boolean` | `true` | Ejecuta `schema-*.sql` al arrancar |
| `spring.cloud.task.tablePrefix` | `String` | `""` | Prefijo de tablas para aislar múltiples tareas en la misma BD |

## Buenas y malas prácticas

**Hacer:**

- Usar `spring.cloud.task.tablePrefix` cuando varias tareas comparten la misma base de datos. El prefijo evita colisiones de nombres y permite gestionar el ciclo de vida de datos de cada tarea de forma independiente.
- Verificar al arranque si existe una ejecución huérfana (`end_time = null`) para el nombre de la tarea actual. Una tarea huérfana indica un kill abrupto anterior; relanzarla sin limpiar el registro puede confundir las métricas de tasa de éxito.
- Poner `spring.cloud.task.initialize-enabled: false` en producción una vez que el esquema está creado por el pipeline de migraciones (Flyway, Liquibase). El autoarranque de DDL en producción es un antipatrón en entornos con DBA.
- Indexar `TASK_EXECUTION(task_name, start_time)` en bases de datos con volumen alto de ejecuciones. `TaskExplorer` emite `ORDER BY start_time DESC` en varias consultas y sin índice el coste crece linealmente.

**Evitar:**

- Usar el DataSource de negocio para el `TaskRepository` si las transacciones de negocio son largas. Una ejecución de tarea que inicia una transacción global puede bloquear las escrituras de `TaskLifecycleListener` sobre `TASK_EXECUTION`, resultando en deadlocks o en que el `end_time` no se persiste correctamente.
- Truncar `TASK_EXECUTION` manualmente en producción sin política de retención acordada. Si SCDF está integrado, perder el historial rompe la correlación entre `external_execution_id` y las ejecuciones de la plataforma.
- Modificar directamente `TASK_EXECUTION.exit_code` en base de datos para "limpiar" una ejecución fallida. El estado incorrecto en la tabla puede hacer que `singleInstanceEnabled` bloquee el relanzamiento o que el CTR interprete un estado de flujo erróneo.
- Confiar en `getTaskExecutionCount()` (total global) para lógica de negocio. Usar siempre `getTaskExecutionCountByTaskName(name)` para acotar la consulta al nombre de tarea específico.

> **[ADVERTENCIA]** Con Spring Cloud 2025.1.1 (Spring Boot 4.0.x / Jakarta EE namespace), los scripts de esquema se encuentran en `org/springframework/cloud/task/schema-*.sql` dentro del JAR. Si copias un script de una versión anterior basada en `javax.*`, las clases `SimpleTaskRepository` y `TaskBatchExecutionListener` pueden fallar porque el driver JDBC espera el esquema exacto de la versión activa.

> **[EXAMEN]** `TaskExplorer` es de solo lectura. Para crear o actualizar ejecuciones desde código de aplicación usa `TaskRepository`, pero en la práctica el ciclo de vida lo gestiona `TaskLifecycleListener` automáticamente; rara vez se inyecta `TaskRepository` directamente en lógica de negocio.

---

← [11.1 Fundamentos de Spring Cloud Task](sc-task-fundamentos.md) | [Índice (README.md)](README.md) | [11.3 Configuración y propiedades spring.cloud.task.*](sc-task-configuracion.md) →
