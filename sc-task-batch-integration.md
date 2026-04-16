# 11.7 Integración con Spring Batch

← [11.6 Composed Tasks y Composed Task Runner](sc-task-composed.md) | [Índice (README.md)](README.md) | [11.8 Métricas, observabilidad y trazabilidad](sc-task-observability.md) →

---

## Introducción

Spring Batch es el framework de referencia para ETL y procesamiento batch en el ecosistema Spring, con su modelo de Jobs, Steps, ItemReader, ItemProcessor e ItemWriter. Sin embargo, Spring Batch no registra la ejecución de un job en el historial de `TASK_EXECUTION` ni expone el job como una tarea orquestable desde SCDF. Spring Cloud Task resuelve esta integración: añadir `@EnableTask` junto a `@EnableBatchProcessing` hace que cada ejecución de un Batch Job quede registrada en `TASK_EXECUTION` y en la tabla de correlación `TASK_BATCH_JOB_EXECUTION`, de forma que SCDF puede lanzar, monitorizar y trazar la tarea Batch exactamente igual que cualquier tarea Spring Cloud Task ligera. Este fichero cubre solo la capa de integración que Spring Cloud Task expone; la lógica interna de Jobs, Steps y readers/writers es responsabilidad de Spring Batch y está fuera del alcance.

## Representación visual

El diagrama muestra la relación entre las tablas de Spring Batch y las de Spring Cloud Task tras la integración, y el flujo de eventos que las conecta.

```
  @SpringBootApplication
  @EnableTask              ─── activa TaskLifecycleListener
  @EnableBatchProcessing   ─── activa JobRepository, JobLauncher

  [Arranque]
       │
       ▼
  TaskLifecycleListener.onTaskStartup()
       │──── INSERT TASK_EXECUTION (start_time, task_name)
       │
       ▼
  Spring Batch JobLauncher.run(job, params)
       │──── INSERT BATCH_JOB_EXECUTION (job_name, start_time)
       │──── INSERT BATCH_JOB_EXECUTION_PARAMS
       │──── (steps se ejecutan...)
       │──── UPDATE BATCH_JOB_EXECUTION (end_time, status, exit_code)
       │
       ▼
  TaskBatchExecutionListener        ← listener registrado automáticamente
       │──── INSERT TASK_BATCH_JOB_EXECUTION
       │         (task_execution_id, job_execution_id)
       │
       ▼
  [Fin del proceso]
  TaskLifecycleListener.onTaskEnd()
       └──── UPDATE TASK_EXECUTION (end_time, exit_code)
             (si fail-on-job-failure=true y job falló → exit_code != 0)

  ──────────────────────────────────────────────────────────────────────
  Tablas relacionadas:

  TASK_EXECUTION ──────────────────────────────┐
  (task_execution_id=100, exit_code=0)         │
                                               ▼
  TASK_BATCH_JOB_EXECUTION                     │
  (task_execution_id=100, job_execution_id=50) │
                                               │
  BATCH_JOB_EXECUTION ◄────────────────────────┘
  (job_execution_id=50, status=COMPLETED)
```

## Ejemplo central

El siguiente ejemplo muestra una tarea Spring Cloud Task que ejecuta un Batch Job de importación de CSV. El ejemplo incluye la configuración de `fail-on-job-failure` y el `TaskBatchExecutionListener`.

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
    <!-- Spring Cloud Task -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-task</artifactId>
    </dependency>
    <!-- Spring Batch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
    <!-- JDBC para ambos repositorios (TaskRepository y JobRepository) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <!-- Base de datos — PostgreSQL en producción -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Clase principal (`ImportBatchTask.java`):**

```java
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;

// Ambas anotaciones en la misma clase principal activan la integración automática
@SpringBootApplication
@EnableTask
@EnableBatchProcessing
public class ImportBatchTask {
    public static void main(String[] args) {
        SpringApplication.run(ImportBatchTask.class, args);
    }
}
```

**Definición del Job Batch (`ImportJobConfig.java`):**

```java
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class ImportJobConfig {

    @Bean
    public Job importJob(JobRepository jobRepository, Step importStep) {
        return new JobBuilder("importJob", jobRepository)
                .start(importStep)
                .build();
    }

    @Bean
    public Step importStep(JobRepository jobRepository,
                           PlatformTransactionManager transactionManager) {
        // Tasklet simplificado — en producción sería un ItemReader/ItemWriter con chunks
        Tasklet importTasklet = (contribution, chunkContext) -> {
            System.out.println("Procesando importación de datos CSV...");
            // Lógica de importación (Step real tendría ItemReader, ItemProcessor, ItemWriter)
            System.out.println("Importación completada: 1000 registros procesados.");
            return RepeatStatus.FINISHED;
        };

        return new StepBuilder("importStep", jobRepository)
                .tasklet(importTasklet, transactionManager)
                .build();
    }
}
```

**Listener personalizado para correlación adicional (`ImportJobListener.java`):**

```java
import org.springframework.batch.core.BatchStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
import org.springframework.stereotype.Component;

@Component
public class ImportJobListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("Job iniciado: " + jobExecution.getJobInstance().getJobName()
                + ", jobExecutionId=" + jobExecution.getId());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            System.out.println("Job completado exitosamente: id=" + jobExecution.getId());
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            System.err.println("Job fallido: id=" + jobExecution.getId()
                    + ", exitCode=" + jobExecution.getExitStatus().getExitCode());
        }
    }
}
```

**Configuración (`application.yml`):**

```yaml
spring:
  application:
    name: import-batch-task
  datasource:
    url: jdbc:postgresql://localhost:5432/batchdb
    username: batchuser
    password: batchpassword
    driver-class-name: org.postgresql.Driver
  
  batch:
    jdbc:
      initialize-schema: never   # gestionar esquema Batch con Flyway/Liquibase en producción
    job:
      enabled: true              # arranca el Job automáticamente al iniciar el contexto
  
  cloud:
    task:
      name: import-batch-task
      initialize-enabled: false  # gestionar esquema Task con Flyway/Liquibase
      closecontextEnabled: true  # cerrar contexto al terminar el Job
      batch:
        fail-on-job-failure: true  # propagar fallo del Job como fallo de la Task (exit_code != 0)
```

## Tabla de elementos clave

La siguiente tabla recoge los elementos de integración Spring Cloud Task — Spring Batch que un desarrollador senior debe conocer.

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `@EnableTask` + `@EnableBatchProcessing` | Anotaciones | — | Combinar ambas activa `TaskBatchExecutionListener` automáticamente |
| `TASK_BATCH_JOB_EXECUTION` | Tabla BD | — | Tabla de correlación con columnas `task_execution_id` y `job_execution_id` |
| `TaskBatchExecutionListener` | Listener (auto) | — | Registra la correlación Task↔Job en `TASK_BATCH_JOB_EXECUTION` al arrancar el Job |
| `spring.cloud.task.batch.fail-on-job-failure` | `Boolean` | `true` | Si el Job Batch falla, la TaskExecution termina con exit_code != 0 |
| `@EnableBatchProcessing` | Anotación | — | Activa autoconfiguración de Batch: `JobRepository`, `JobLauncher`, `JobExplorer` |
| `JobRepository` | Spring Batch | — | Persiste el estado de `BATCH_JOB_EXECUTION` y `BATCH_STEP_EXECUTION` |
| `spring.batch.job.enabled` | `Boolean` | `true` | Si es `true`, Spring Boot lanza el Job automáticamente al arrancar el contexto |
| `spring.batch.jdbc.initialize-schema` | Enum | `embedded` | `always`/`never`/`embedded` — controla la creación del esquema Batch |
| Exit code propagation | Comportamiento | — | Con `fail-on-job-failure=true`, el exit code de la `TaskExecution` refleja el del Job |
| `TaskExplorer.findTaskExecutionsByName()` | Método | — | Permite auditar todas las ejecuciones de la tarea incluyendo las que contienen Jobs |

## Buenas y malas prácticas

**Hacer:**

- Activar `spring.cloud.task.batch.fail-on-job-failure: true` (valor por defecto) en todas las tareas que ejecutan Jobs Batch. Sin esta propiedad, un Job fallido deja la `TaskExecution` con exit_code=0, lo que oculta el fallo a SCDF y a cualquier sistema de monitorización que evalúe el historial de tareas.
- Usar un único DataSource para `TaskRepository`, `JobRepository` y la base de datos de negocio solo en entornos de desarrollo. En producción, separar al menos `JobRepository` y `TaskRepository` del DataSource de negocio para evitar conflictos de transacciones.
- Configurar `spring.batch.job.enabled: false` si la tarea lanza el Job programáticamente (mediante `JobLauncher.run()` desde un `ApplicationRunner`). Sin esto, Spring Boot intenta lanzar el Job automáticamente Y el runner lo lanza de nuevo, causando una doble ejecución.
- Gestionar el esquema de Batch (`BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION`, etc.) y el esquema de Task (`TASK_EXECUTION`) con el mismo pipeline de migraciones (Flyway, Liquibase). Mantenerlos sincronizados evita errores de versión de esquema al actualizar Spring Batch o Spring Cloud Task.

**Evitar:**

- Asumir que la relación `TASK_BATCH_JOB_EXECUTION` es 1:1. Una tarea puede lanzar múltiples Jobs (por ejemplo, un Job por tenant), y cada uno generará una fila separada en `TASK_BATCH_JOB_EXECUTION` con el mismo `task_execution_id`.
- Usar `spring.batch.jdbc.initialize-schema: always` en producción. En despliegues concurrentes, múltiples instancias pueden intentar ejecutar el DDL simultáneamente, causando errores de constraint o alteraciones de tabla inesperadas.
- Ignorar el campo `exit_message` de `TASK_EXECUTION` cuando `fail-on-job-failure=true`. Spring Cloud Task copia el mensaje de error del Job en `exit_message`, lo que facilita el diagnóstico sin necesidad de acceder a las tablas de Spring Batch.
- Implementar lógica de reintentos de Jobs dentro de la tarea Task. Spring Batch tiene su propio mecanismo de restart (`JobRepository` guarda el estado del Step para reanudar desde el punto de fallo); duplicar reintentos a nivel Task crea ejecuciones en `TASK_EXECUTION` con Job IDs inconsistentes.

> **[ADVERTENCIA]** Con Spring Boot 4.0.x (Jakarta EE namespace), la autoconfiguración de `@EnableBatchProcessing` ha cambiado respecto a versiones anteriores. En Spring Boot 4.x la anotación es opcional si hay un `DataSource` configurado; Spring Boot autoconfigura el `JobRepository` automáticamente. Verificar en la documentación oficial de Spring Batch 6.x (compatible con Spring Boot 4.0.x) si `@EnableBatchProcessing` es necesario o redundante en tu configuración.

> **[CONCEPTO]** La tabla `TASK_BATCH_JOB_EXECUTION` es la única "capa de integración" entre los dos mundos. No existe acoplamiento en el código Java: `@EnableTask` y `@EnableBatchProcessing` se combinan, y el `TaskBatchExecutionListener` detecta automáticamente la presencia de Spring Batch en el classpath para registrar la correlación.

---

← [11.6 Composed Tasks y Composed Task Runner](sc-task-composed.md) | [Índice (README.md)](README.md) | [11.8 Métricas, observabilidad y trazabilidad](sc-task-observability.md) →
