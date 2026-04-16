# 11.1 Fundamentos de Spring Cloud Task

← [10.12 Testing / Verificación de Spring Cloud Contract](sc-contract-testing.md) | [Índice (README.md)](README.md) | [11.2 TaskRepository y persistencia de ejecuciones](sc-task-repository.md) →

---

## Introducción

En un sistema de microservicios el modelo habitual es el de servicios de larga duración: el proceso arranca, escucha peticiones y no termina. Sin embargo, existe una clase de cargas de trabajo —importaciones nocturnas, generación de informes, purgas de datos, migraciones— que deben ejecutarse una vez, registrar su resultado y terminar. Spring Boot base no ofrece ningún mecanismo para registrar ese ciclo de vida, saber si la tarea terminó con éxito, cuánto tardó o qué argumentos recibió. Spring Cloud Task resuelve exactamente ese problema: añade al contexto de Spring Boot las abstracciones necesarias para que una aplicación de vida corta registre su ejecución en una tabla relacional y pueda ser consultada, monitorizida y relanzada desde plataformas como Spring Cloud Data Flow. Sin Spring Cloud Task, cada equipo implementa su propia solución ad-hoc de registro de ejecuciones, lo que fragmenta la visibilidad operativa en sistemas con decenas de tareas programadas.

## Representación visual

El siguiente diagrama muestra el ciclo de vida completo de una TaskExecution desde el arranque del proceso hasta su registro final en base de datos, incluyendo los puntos de extensión que ofrece el módulo.

```
  [JVM arranca]
       │
       ▼
  @SpringBootApplication
  @EnableTask  ──────────────────────────────────────────────────┐
       │                                                          │
       ▼                                                          │
  TaskLifecycleListener                                   TaskRepository
  (autoconfigurado)                                       (SimpleTaskRepository)
       │                                                          │
       ├── onTaskStartup() ──── crea TaskExecution ─────────────►│ INSERT TASK_EXECUTION
       │       │                                                  │  start_time = now()
       │       ▼                                                  │  exit_code = null
       │  TaskExecutionListener                                   │
       │  (personalizados del usuario)                            │
       │       │                                                  │
       ▼       ▼                                                  │
  [Lógica de negocio — CommandLineRunner / ApplicationRunner]     │
       │                                                          │
       ├── éxito ─────────────────────────────────────────────── │ UPDATE TASK_EXECUTION
       │                                                          │  exit_code = 0
       │                                                          │  end_time = now()
       └── excepción ────────────────────────────────────────────►│ UPDATE TASK_EXECUTION
               │                                                  │  exit_code != 0
               ▼                                                  │  error_message = ...
       onTaskFailed()                                             │
               │                                                  │
               ▼                                                  │
       [JVM termina]                                              │
               │                                                  │
               └──────────────── onTaskEnd() ────────────────────►│ (fin de transacción)
```

## Ejemplo central

El siguiente ejemplo muestra una tarea Spring Cloud Task mínima pero completa: una aplicación Spring Boot 4.0.x que procesa registros pendientes, registra su ejecución y termina.

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
    <!-- Spring Boot base -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- Spring Cloud Task -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-task</artifactId>
    </dependency>

    <!-- DataSource para TaskRepository (H2 en desarrollo, PostgreSQL en producción) -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- JDBC para SimpleTaskRepository -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
</dependencies>
```

**Clase principal (`PurgeTask.java`):**

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import org.springframework.stereotype.Component;

import java.util.List;

@SpringBootApplication
@EnableTask  // activa autoconfiguración de TaskRepository, TaskLifecycleListener y TaskMetrics
public class PurgeTask {

    public static void main(String[] args) {
        SpringApplication.run(PurgeTask.class, args);
    }
}

@Component
class PurgeRunner implements ApplicationRunner {

    private final TaskExplorer taskExplorer;

    PurgeRunner(TaskExplorer taskExplorer) {
        this.taskExplorer = taskExplorer;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Iniciando purga de registros...");

        // Simula la lógica de negocio
        int purged = purgeOldRecords();

        // Consulta la ejecución actual via TaskExplorer
        long executionCount = taskExplorer.getTaskExecutionCount();
        System.out.println("Ejecuciones registradas hasta ahora: " + executionCount);
        System.out.println("Registros purgados: " + purged);
    }

    private int purgeOldRecords() {
        // Lógica real de purga iría aquí
        return 42;
    }
}
```

**Listener personalizado (`PurgeTaskListener.java`):**

```java
import org.springframework.cloud.task.listener.TaskExecutionListener;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.stereotype.Component;

@Component
public class PurgeTaskListener implements TaskExecutionListener {

    @Override
    public void onTaskStartup(TaskExecution taskExecution) {
        System.out.println("Tarea iniciada: id=" + taskExecution.getExecutionId()
                + ", nombre=" + taskExecution.getTaskName());
    }

    @Override
    public void onTaskEnd(TaskExecution taskExecution) {
        System.out.println("Tarea finalizada: exitCode=" + taskExecution.getExitCode()
                + ", duración=" +
                (taskExecution.getEndTime().toEpochMilli()
                - taskExecution.getStartTime().toEpochMilli()) + "ms");
    }

    @Override
    public void onTaskFailed(TaskExecution taskExecution, Throwable throwable) {
        System.err.println("Tarea fallida: " + throwable.getMessage());
        System.err.println("errorMessage registrado: " + taskExecution.getErrorMessage());
    }
}
```

**Configuración (`application.yml`):**

```yaml
spring:
  application:
    name: purge-task
  datasource:
    url: jdbc:h2:mem:taskdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
    driver-class-name: org.h2.Driver
    username: sa
    password:
  cloud:
    task:
      name: purge-task          # nombre que aparece en TASK_EXECUTION.task_name
      initialize-enabled: true  # crea las tablas TASK_EXECUTION* si no existen
      closecontextEnabled: true # cierra el ApplicationContext al terminar la lógica
```

## Tabla de elementos clave

La tabla siguiente recoge las propiedades de autoconfiguración y las clases que un desarrollador senior debe conocer para trabajar con Spring Cloud Task.

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `@EnableTask` | Anotación | — | Activa la autoconfiguración de TaskRepository, TaskLifecycleListener y TaskMetrics |
| `spring.cloud.task.name` | String | nombre de la app | Valor almacenado en `TASK_EXECUTION.task_name` |
| `spring.cloud.task.initialize-enabled` | Boolean | `true` | Ejecuta los scripts `schema-*.sql` para crear las tablas al arrancar |
| `spring.cloud.task.closecontextEnabled` | Boolean | `true` | Cierra el `ApplicationContext` al terminar `ApplicationRunner`/`CommandLineRunner` |
| `TaskExecutionListener` | Interfaz | — | Puntos de extensión: `onTaskStartup`, `onTaskEnd`, `onTaskFailed` |
| `TaskExecution` | Clase | — | Representa una ejecución: id, nombre, tiempos, exit code, mensajes |
| `TaskRepository` | Interfaz | — | CRUD de `TaskExecution`; implementación por defecto: `SimpleTaskRepository` |
| `TaskExplorer` | Interfaz | — | API de consulta de ejecuciones: `getTaskExecution`, `findTaskExecutionsByName` |
| `spring-cloud-starter-task` | Starter | — | Incluye `spring-cloud-task-core` y sus dependencias transitivas |
| Ciclo de vida | — | — | `start → (update)* → end`; la transición `end` persiste `exit_code` y `end_time` |

## Buenas y malas prácticas

**Hacer:**

- Declarar `@EnableTask` en la clase `@SpringBootApplication` principal. Colocarlo en una clase de configuración separada funciona pero dificulta la búsqueda cuando se revisa el proyecto.
- Implementar `TaskExecutionListener` para registrar métricas de negocio (registros procesados, errores) en `onTaskEnd`, de modo que estén disponibles en el historial de ejecuciones sin necesidad de parsear logs.
- Configurar un DataSource dedicado para el `TaskRepository` cuando la tarea también tiene su propio DataSource de negocio. Mezclar ambos en el mismo DataSource complica la gestión de migraciones.
- Propagar excepciones desde `ApplicationRunner.run()` sin atraparlas silenciosamente: Spring Cloud Task intercepta la excepción, la registra en `TASK_EXECUTION.error_message` y asigna el exit code correspondiente.

**Evitar:**

- Llamar a `System.exit()` directamente dentro de la lógica de negocio. Spring Cloud Task no tendrá oportunidad de actualizar `end_time` y `exit_code`, dejando la `TaskExecution` huérfana con `end_time = null`.
- Usar `spring.cloud.task.closecontextEnabled=false` sin razón justificada en tareas que no son servicios web. El contexto no se cerrará y el proceso quedará colgado esperando que terminen threads del servidor.
- Registrar múltiples `@Bean` de tipo `ApplicationRunner` sin un orden definido con `@Order`. El listener de ciclo de vida de Task solo captura la excepción que provoca el fallo general; si varios runners lanzan excepciones, solo la primera queda registrada en `error_message`.
- Omitir el DataSource en tests: sin DataSource configurado, `SimpleTaskRepository` lanza `BeanCreationException` al arrancar. Siempre configurar H2 en memoria en el scope de test.

## Comparación: tarea vs. servicio vs. job Batch

La tabla siguiente aclara cuándo usar cada modelo para evitar decisiones arquitectónicas incorrectas en entrevistas técnicas.

| Criterio | Spring Boot (servicio) | Spring Cloud Task | Spring Batch Job |
|---|---|---|---|
| Duración del proceso | Indefinida | Acotada (segundos–horas) | Acotada (minutos–horas) |
| Registro de ejecución | Manual | Automático (TASK_EXECUTION) | Automático (BATCH_JOB_EXECUTION) |
| Modelo de datos | Sin esquema propio | 2 tablas (TASK_EXECUTION, PARAMS) | 6+ tablas (JOB, STEP, etc.) |
| Reintentos / restart | No | No nativo | Sí, a nivel de Step |
| Orquestación | No | Vía SCDF | Vía SCDF o Quartz |
| Caso de uso típico | API REST, consumidor Kafka | Importación, purga, migración ligera | ETL grande con chunks y restart |

> **[CONCEPTO]** La diferencia clave entre una tarea y un servicio no es el volumen de datos sino la **semántica de terminación**: una tarea que termina sin error es un resultado exitoso; un servicio que termina es un fallo operativo.

> **[ADVERTENCIA]** Spring Cloud Task no implementa reintentos automáticos de la tarea completa. Si la tarea falla, es responsabilidad del orquestador (SCDF, cron, Kubernetes Job) relanzarla. No confundir con Spring Retry, que opera dentro de un método.

---

← [10.12 Testing / Verificación de Spring Cloud Contract](sc-contract-testing.md) | [Índice (README.md)](README.md) | [11.2 TaskRepository y persistencia de ejecuciones](sc-task-repository.md) →
