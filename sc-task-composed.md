# 11.6 Composed Tasks y Composed Task Runner

← [11.5 Integración con Spring Cloud Data Flow](sc-task-scdf-integration.md) | [Índice (README.md)](README.md) | [11.7 Integración con Spring Batch](sc-task-batch-integration.md) →

---

## Introducción

Cuando un pipeline batch requiere ejecutar múltiples tareas en secuencia o en paralelo —por ejemplo, extraer datos, transformarlos y cargarlos en ese orden, o ejecutar validaciones en paralelo antes de la carga— la coordinación manual (scripts, dependencias de CI/CD) introduce acoplamiento frágil entre tareas que deberían ser artefactos independientes. Spring Cloud Data Flow resuelve este problema con las Composed Tasks: un DSL declarativo que define flujos de tareas con operadores secuenciales (`&&`), paralelos (`||`) y transiciones condicionales basadas en exit codes. El Composed Task Runner (CTR) es en sí mismo una tarea Spring Cloud Task que interpreta el DSL, lanza cada subtarea en SCDF y registra el estado de cada una en `TASK_EXECUTION` usando `parent_execution_id` para mantener la jerarquía del flujo.

## Representación visual

El diagrama muestra la arquitectura del Composed Task Runner y cómo se mapea un flujo DSL a ejecuciones en `TASK_EXECUTION`.

```
  SCDF Server
  ──────────────────────────────────────────────────────────────────────
  DSL: "extract && transform && load"
  
  task launch composed-pipeline
       │
       ▼
  CTR (Composed Task Runner) ─── es una tarea Task normal en SCDF
       │  task_execution_id=100, task_name=composed-pipeline
       │  parent_execution_id=null
       │
       ├── 1. Lanza: extract
       │         task_execution_id=101, parent_execution_id=100
       │         [espera exit_code=0]
       │
       ├── 2. Lanza: transform  (solo si extract exit_code=0)
       │         task_execution_id=102, parent_execution_id=100
       │         [espera exit_code=0]
       │
       └── 3. Lanza: load       (solo si transform exit_code=0)
                 task_execution_id=103, parent_execution_id=100
                 [fin del flujo]
  
  ──────────────────────────────────────────────────────────────────────
  DSL con paralelo: "validate-schema || validate-data && load"
  
       ├── validate-schema ────┐
       │   (en paralelo)       ├── espera ambas → load
       └── validate-data ─────┘
  
  ──────────────────────────────────────────────────────────────────────
  DSL con transición condicional:
  
  "extract 'COMPLETED'->transform 'FAILED'->notify-failure"
  
       extract ──COMPLETED──► transform
              └──FAILED─────► notify-failure
```

## Ejemplo central

El siguiente ejemplo muestra cómo definir, registrar y lanzar una Composed Task en SCDF con los tres tipos de operadores del DSL, incluyendo transiciones condicionales con exit codes personalizados.

**Dependencias Maven de cada sub-tarea (sin cambios):**

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

**Sub-tarea con exit code personalizado (`ExtractTask.java`):**

```java
import org.springframework.boot.ExitCodeGenerator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
@EnableTask
public class ExtractTask {
    public static void main(String[] args) {
        SpringApplication.run(ExtractTask.class, args);
    }

    // Exit code personalizado para transiciones condicionales en el DSL del CTR
    @Bean
    public ExitCodeGenerator extractExitCodeGenerator() {
        return () -> {
            boolean dataFound = checkDataAvailability();
            if (dataFound) {
                return 0;       // COMPLETED — el CTR lanzará la siguiente tarea
            } else {
                return 10;      // NO_DATA — el CTR puede redirigir a una tarea de notificación
            }
        };
    }

    private boolean checkDataAvailability() {
        // Lógica real de comprobación
        return true;
    }
}
```

**Sub-tarea de transformación (`TransformTask.java`):**

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableTask
public class TransformTask {
    public static void main(String[] args) {
        SpringApplication.run(TransformTask.class, args);
    }
}

@Component
class TransformRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Transformando datos...");
        // Lógica de transformación
    }
}
```

**Sub-tarea de carga (`LoadTask.java`):**

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableTask
public class LoadTask {
    public static void main(String[] args) {
        SpringApplication.run(LoadTask.class, args);
    }
}

@Component
class LoadRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Cargando datos en destino...");
    }
}
```

**Registro y definición de la Composed Task en SCDF (curl):**

```bash
# 1. Registrar cada sub-tarea en App Registry
curl -X POST "http://scdf-server:9393/apps/task/extract-task" \
  -d "uri=docker://myregistry/extract-task:1.0.0"
curl -X POST "http://scdf-server:9393/apps/task/transform-task" \
  -d "uri=docker://myregistry/transform-task:1.0.0"
curl -X POST "http://scdf-server:9393/apps/task/load-task" \
  -d "uri=docker://myregistry/load-task:1.0.0"

# Registrar el Composed Task Runner (proporcionado por Spring Cloud)
curl -X POST "http://scdf-server:9393/apps/task/composed-task-runner" \
  -d "uri=docker://springcloudtask/composed-task-runner:latest"

# 2. Crear Composed Task con DSL secuencial
curl -X POST "http://scdf-server:9393/tasks/definitions" \
  -d "name=etl-pipeline" \
  -d "definition=extract-task && transform-task && load-task"

# 3. Crear Composed Task con transición condicional basada en exit code
curl -X POST "http://scdf-server:9393/tasks/definitions" \
  -d "name=etl-conditional" \
  -d "definition=extract-task 'COMPLETED'->transform-task 'NO_DATA'->notify-task"

# 4. Crear Composed Task con paralelo
curl -X POST "http://scdf-server:9393/tasks/definitions" \
  -d "name=parallel-validate-load" \
  -d "definition=<validate-schema || validate-data> && load-task"

# 5. Lanzar la Composed Task
curl -X POST "http://scdf-server:9393/tasks/executions" \
  -d "name=etl-pipeline" \
  -d "arguments=--task.date=2025-01-15"
```

**Consulta del estado del flujo via TaskExplorer (en una tarea de diagnóstico):**

```java
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
class ComposedTaskStatusChecker {

    private final TaskExplorer taskExplorer;

    ComposedTaskStatusChecker(TaskExplorer taskExplorer) {
        this.taskExplorer = taskExplorer;
    }

    public void printFlowStatus(long parentExecutionId) {
        // Recuperar la ejecución del CTR
        TaskExecution parent = taskExplorer.getTaskExecution(parentExecutionId);
        System.out.printf("CTR: id=%d, exitCode=%d, end=%s%n",
                parent.getExecutionId(), parent.getExitCode(), parent.getEndTime());

        // Las sub-tareas tienen parent_execution_id = parentExecutionId
        // Se consultan por nombre de tarea (el CTR crea nombres como "etl-pipeline-extract-task-1")
        Page<TaskExecution> children = taskExplorer.findTaskExecutionsByName(
                "etl-pipeline-extract-task", PageRequest.of(0, 10));
        children.getContent().forEach(child ->
                System.out.printf("  sub-tarea: id=%d, name=%s, exitCode=%d%n",
                        child.getExecutionId(), child.getTaskName(), child.getExitCode()));
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge el DSL del CTR, los operadores y las propiedades de configuración más relevantes.

| Elemento | Valor/Tipo | Descripción |
|---|---|---|
| `&&` (DSL) | Operador secuencial | Lanza la tarea siguiente solo si la anterior terminó con exit code 0 (COMPLETED) |
| `\|\|` (DSL) | Operador paralelo | Lanza ambas tareas simultáneamente; la siguiente tarea del flujo espera a que ambas terminen |
| `'LABEL'->tarea` (DSL) | Transición condicional | Si la tarea anterior termina con la etiqueta/exit code indicado, lanza la tarea destino |
| `<tarea1 \|\| tarea2>` (DSL) | Grupo paralelo | Sintaxis de agrupación para combinar paralelo con secuencial |
| `exit_code = 0` | Convención | Corresponde a la etiqueta `COMPLETED` en el DSL del CTR |
| `exit_code != 0` | Convención | Corresponde a la etiqueta `FAILED` o al valor numérico del exit code |
| `parent_execution_id` | `TASK_EXECUTION` column | FK que apunta al `task_execution_id` del CTR padre |
| `composed-task-runner` | App | Aplicación Task que interpreta el DSL; se registra como cualquier otra tarea en SCDF |
| `ExitCodeGenerator` | Interfaz Spring Boot | Permite definir el exit code de la tarea para controlar las transiciones del CTR |
| `composed-task-runner.split-thread-core-pool-size` | Property | Número de threads para ejecución paralela de sub-tareas en el CTR |
| `composed-task-runner.max-wait-time` | Property (ms) | Tiempo máximo que el CTR espera a que una sub-tarea termine antes de fallar el flujo |

## Buenas y malas prácticas

**Hacer:**

- Implementar `ExitCodeGenerator` en las sub-tareas que tienen transiciones condicionales en el DSL. Sin exit codes personalizados, el CTR solo distingue entre `COMPLETED` (exit 0) y `FAILED` (exit != 0), lo que limita los flujos a bifurcaciones binarias.
- Mantener las sub-tareas de una Composed Task como artefactos independientes con versionado propio. La Composed Task es solo la definición del flujo en SCDF; las sub-tareas deben poder ejecutarse de forma standalone para facilitar el testing.
- Registrar el `composed-task-runner` con la misma versión que la versión de Spring Cloud Task usada en las sub-tareas. Desajustes de versión pueden causar incompatibilidades en la interpretación de exit codes.
- Configurar `max-wait-time` del CTR con un valor superior al tiempo máximo esperado de la sub-tarea más lenta. Sin límite, un timeout por defecto puede abortar flujos legítimamente largos.

**Evitar:**

- Crear Composed Tasks muy profundas (más de 5–6 niveles) en producción. Cada nivel de jerarquía añade overhead de polling y latencia de arranque; para flujos muy complejos considerar Spring Batch con Steps configurados como Job.
- Usar el operador `||` para paralelismo masivo sin configurar `split-thread-core-pool-size`. El pool por defecto del CTR puede ser insuficiente, causando que tareas "paralelas" se ejecuten en realidad de forma secuencial por falta de threads disponibles.
- Ignorar el `parent_execution_id` al consultar el historial de ejecuciones de una Composed Task. Sin filtrar por padre, la lista de ejecuciones mezcla ejecuciones de distintos lanzamientos del mismo flujo.
- Definir el DSL de una Composed Task con nombres de sub-tareas que no están registradas en el App Registry. SCDF lanza el CTR pero este falla inmediatamente al intentar lanzar la sub-tarea desconocida, sin indicación clara del problema en el `error_message`.

> **[EXAMEN]** El Composed Task Runner no es un framework Spring especializado: es una tarea Spring Cloud Task normal que recibe el DSL como argumento de lanzamiento (`--composed-task-properties=...`). Al terminar de ejecutar todas las sub-tareas del flujo, el CTR termina con exit code 0 (COMPLETED) o distinto de 0 (FAILED), igual que cualquier tarea.

> **[PREREQUISITO]** Para lanzar Composed Tasks es necesario tener instalado y configurado Spring Cloud Data Flow Server. Las Composed Tasks no pueden ejecutarse de forma standalone sin SCDF porque dependen del `TaskLauncher` SPI para lanzar las sub-tareas y del polling de `TASK_EXECUTION` para conocer el estado de cada una.

---

← [11.5 Integración con Spring Cloud Data Flow](sc-task-scdf-integration.md) | [Índice (README.md)](README.md) | [11.7 Integración con Spring Batch](sc-task-batch-integration.md) →
