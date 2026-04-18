# 11.9 Spring Cloud Task — Testing

← [11.8 Composed Tasks](sc-task-composed.md) | [Índice](README.md) | [11.10 Troubleshooting](sc-task-troubleshooting.md) →

---

## Introducción

El testing de Spring Cloud Task verifica que la lógica de negocio del runner se ejecuta correctamente Y que el ciclo de vida de la ejecución queda registrado en la base de datos. Los tests de integración usan H2 en memoria para simular la persistencia sin necesidad de una base de datos real. `TaskExplorer` permite hacer assertions sobre las `TaskExecution` registradas, verificando nombre, `EXIT_CODE`, argumentos y timestamps.

> [CONCEPTO] Un test completo de Spring Cloud Task debe verificar dos aspectos: (1) la lógica de negocio del runner produce el resultado esperado, y (2) la `TaskExecution` se registró correctamente en la base de datos con los valores esperados de `EXIT_CODE`, `taskName` y argumentos.

## Estrategia de testing

Los tests de Spring Cloud Task se organizan en tres niveles según el alcance de verificación y la velocidad de ejecución.

```
Nivel 1 — Tests unitarios del runner:
  → Mock de dependencias externas
  → Sin contexto Spring, sin BD
  → Verifican lógica de negocio pura
  → Rápidos: ms

Nivel 2 — Tests de integración con H2:
  → @SpringBootTest con H2 en memoria
  → TASK_EXECUTION se crea automáticamente
  → TaskExplorer verifica lo que se persistió
  → Velocidad: segundos

Nivel 3 — Tests con Spring Batch integrado:
  → @SpringBootTest con H2 para Task + Batch
  → Verifica TASK_TASK_BATCH además de TASK_EXECUTION
  → TaskExplorer + JobExplorer para verificación completa
  → Velocidad: segundos
```

## Ejemplo central

El siguiente ejemplo muestra un test de integración completo que arranca el contexto Spring con H2, ejecuta una Task y verifica mediante `TaskExplorer` que la ejecución se registró correctamente con el `EXIT_CODE` y los argumentos esperados.

```java
package com.example.task;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Component;
import org.springframework.test.context.TestPropertySource;
import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;

// La aplicación Task bajo test
@SpringBootApplication
@EnableTask
class SampleTaskApp {
    public static void main(String[] args) {
        SpringApplication.run(SampleTaskApp.class, args);
    }
}

@Component
class SampleRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Lógica de la tarea
        System.out.println("Ejecutando tarea de ejemplo");
    }
}

// Test de integración
@SpringBootTest(
    classes = SampleTaskApp.class,
    args = {"--report.date=2025-01-01", "--dry-run"}
)
@TestPropertySource(properties = {
    "spring.cloud.task.name=test-task",
    "spring.cloud.task.initialize-enabled=true",
    // H2 en memoria configurado por spring-boot-test automáticamente
    // cuando h2 está en test scope del classpath
})
class SampleTaskIntegrationTest {

    @Autowired
    private TaskExplorer taskExplorer;

    @Test
    void taskExecutionShouldBeRegisteredWithExitCodeZero() {
        // Consulta todas las ejecuciones de la tarea "test-task"
        Page<TaskExecution> executions = taskExplorer
            .findTaskExecutionsByName("test-task", PageRequest.of(0, 10));

        assertThat(executions.getContent()).isNotEmpty();

        TaskExecution execution = executions.getContent().get(0);

        // Verifica EXIT_CODE exitoso
        assertThat(execution.getExitCode()).isEqualTo(0);

        // Verifica nombre de la tarea
        assertThat(execution.getTaskName()).isEqualTo("test-task");

        // Verifica que START_TIME y END_TIME están registrados
        assertThat(execution.getStartTime()).isNotNull();
        assertThat(execution.getEndTime()).isNotNull();

        // Verifica que los argumentos se registraron en TASK_EXECUTION_PARAMS
        List<String> arguments = execution.getArguments();
        assertThat(arguments).contains("--report.date=2025-01-01", "--dry-run");

        // Verifica que no hay mensaje de error
        assertThat(execution.getErrorMessage()).isNull();
    }

    @Test
    void latestTaskExecutionShouldBeRetrievable() {
        TaskExecution latest = taskExplorer
            .getLatestTaskExecutionForTaskName("test-task");

        assertThat(latest).isNotNull();
        assertThat(latest.getExitCode()).isEqualTo(0);
    }
}
```

Test adicional que verifica que una tarea con runner que falla registra `EXIT_CODE ≠ 0`:

```java
package com.example.task;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import org.springframework.stereotype.Component;
import org.springframework.test.context.TestPropertySource;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootApplication
@EnableTask
class FailingTaskApp {
    public static void main(String[] args) {
        // SpringApplication no lanza excepción al main;
        // registra el fallo en TASK_EXECUTION y termina con exit code 1
        SpringApplication.run(FailingTaskApp.class, args);
    }
}

@Component
class FailingRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        throw new IllegalStateException("Fallo de prueba intencional");
    }
}

@SpringBootTest(
    classes = FailingTaskApp.class,
    // Spring Boot captura la excepción del runner pero la Task la registra
    properties = "spring.cloud.task.name=failing-task"
)
@TestPropertySource(properties = "spring.cloud.task.initialize-enabled=true")
class FailingTaskIntegrationTest {

    @Autowired
    private TaskExplorer taskExplorer;

    @Test
    void failedTaskShouldRegisterNonZeroExitCode() {
        TaskExecution execution = taskExplorer
            .getLatestTaskExecutionForTaskName("failing-task");

        assertThat(execution).isNotNull();
        assertThat(execution.getExitCode()).isNotEqualTo(0);
        assertThat(execution.getErrorMessage()).contains("Fallo de prueba intencional");
    }
}
```

## Tabla de elementos clave

Los componentes disponibles para testing son los siguientes, agrupados por su responsabilidad.

| Componente | Uso en Test | Descripción |
|---|---|---|
| `@SpringBootTest` | Test de integración | Levanta contexto Spring completo con H2 si está en classpath |
| `TaskExplorer` | Assertions | API de solo lectura para consultar `TASK_EXECUTION`; se inyecta como `@Autowired` |
| `TaskExecutionDao` | Assertions avanzadas | Acceso directo a DAO si se necesitan operaciones que `TaskExplorer` no expone |
| `SimpleTaskRepository` | Tests unitarios | Se puede instanciar sin contexto Spring para tests de integración ligeros |
| `H2` | Datasource de test | Base de datos en memoria; `spring.cloud.task.initialize-enabled=true` crea el schema |
| `TaskExecutionDaoConfiguration` | Configuración | Importar explícitamente si el contexto de test no lo detecta automáticamente |

## Verificación de integración Task + Batch

Cuando la Task incluye Spring Batch, el test debe verificar también la tabla `TASK_TASK_BATCH`. Para esto se inyecta tanto `TaskExplorer` como `JobExplorer` de Spring Batch.

```java
package com.example.task;

import org.junit.jupiter.api.Test;
import org.springframework.batch.core.explore.JobExplorer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;

@SpringBootTest(properties = {
    "spring.cloud.task.name=batch-task",
    "spring.cloud.task.initialize-enabled=true",
    "spring.batch.initialize-schema=always"
})
class BatchTaskIntegrationTest {

    @Autowired
    private TaskExplorer taskExplorer;

    @Autowired
    private JobExplorer jobExplorer;

    @Test
    void batchJobShouldBeLinkedToTaskExecution() {
        // Verifica que se creó al menos una TaskExecution
        TaskExecution taskExecution = taskExplorer
            .getLatestTaskExecutionForTaskName("batch-task");
        assertThat(taskExecution).isNotNull();
        assertThat(taskExecution.getExitCode()).isEqualTo(0);

        // Verifica que el Job de Batch se ejecutó
        assertThat(jobExplorer.getJobNames()).contains("reportJob");
        assertThat(jobExplorer.getJobInstances("reportJob", 0, 1)).isNotEmpty();
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `@TestPropertySource(properties = "spring.cloud.task.name=test-task")` para fijar el nombre de la tarea en tests: evita dependencias del nombre del artefacto.
- Verificar tanto `EXIT_CODE` como `getErrorMessage()` en los tests de escenarios de fallo.
- Añadir H2 como dependencia de test (`<scope>test</scope>`) para que el schema de Task se cree automáticamente sin BD real.

**Malas prácticas:**
- Asumir que el contexto Spring se reinicia entre tests en la misma clase: `@SpringBootTest` reutiliza el contexto si las propiedades son iguales; usar `@DirtiesContext` si se necesita aislamiento.
- Hacer assertions sobre timestamps exactos en `START_TIME` y `END_TIME`: usar `isNotNull()` en lugar de comparar fechas concretas.
- Olvidar `spring.cloud.task.initialize-enabled=true` en tests con H2: el schema no se crea y los tests fallan con error de tabla no encontrada.

> [ADVERTENCIA] En tests de integración con `@SpringBootTest`, si el runner lanza una excepción, `SpringApplication.run()` puede retornar un código de salida no cero. Dependiendo de la configuración, esto puede o no propagar una excepción en el test. Verificar el comportamiento con `SpringApplication.exit()`.

## Verificación y práctica

> [EXAMEN] **Pregunta 1:** ¿Qué interfaz se inyecta en los tests de Spring Cloud Task para consultar las `TaskExecution` registradas en base de datos?

> [EXAMEN] **Pregunta 2:** ¿Qué propiedad debe estar en `true` para que Spring Cloud Task cree el schema de tablas automáticamente en H2 durante los tests?

> [EXAMEN] **Pregunta 3:** ¿Cómo se verifica en un test que los argumentos de línea de comandos se registraron correctamente en `TASK_EXECUTION_PARAMS`?

> [EXAMEN] **Pregunta 4:** En un test de integración que combina Task y Batch, ¿qué dos `Explorer` se inyectan para verificar la relación en `TASK_TASK_BATCH`?

> [EXAMEN] **Pregunta 5:** ¿Por qué es recomendable usar `@TestPropertySource` para fijar `spring.cloud.task.name` en tests de integración?

---

← [11.8 Composed Tasks](sc-task-composed.md) | [Índice](README.md) | [11.10 Troubleshooting](sc-task-troubleshooting.md) →
