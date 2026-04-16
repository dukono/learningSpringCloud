# 11.9 Testing de Spring Cloud Task

← [11.8 Métricas, observabilidad y trazabilidad](sc-task-observability.md) | [Índice (README.md)](README.md) | [12.1 Modelo de programación funcional: Function, Consumer, Supplier y FunctionCatalog](sc-function-modelo.md) →

---

Una tarea de Spring Cloud Task tiene dos responsabilidades verificables: ejecutar correctamente su lógica de negocio y registrar correctamente su ejecución en la tabla `TASK_EXECUTION`. El test unitario cubre la primera; el test de integración cubre la segunda. La peculiaridad de testear Spring Cloud Task es que el ciclo de vida —`TaskLifecycleListener` iniciando y cerrando la ejecución— solo actúa con el contexto completo de Spring y una `DataSource` disponible. Un test que no incluye `@EnableTask` ni `DataSource` no está testeando Spring Cloud Task: está testeando solo el `CommandLineRunner`.

> [PREREQUISITO] Requiere `spring-cloud-starter-task` y una `DataSource` para tests de integración. Usar H2 en memoria (`com.h2database:h2` con `scope test`) para aislar los tests de la base de datos de producción.

## Niveles de testing de Spring Cloud Task

La elección de estrategia depende de qué aspecto del ciclo de vida se quiere verificar.

| Estrategia | Contexto Spring | DataSource | Verifica |
|---|---|---|---|
| 1 — Test unitario del runner | No | No | Lógica de negocio del CommandLineRunner en aislamiento |
| 2 — Test de integración con @SpringBootTest | Sí | H2 en memoria | Registro en TASK_EXECUTION, exit code, TaskLifecycle |
| 3 — Test de integración con TaskExplorer | Sí | H2 en memoria | Aserciones sobre TaskExecution: estado, duración, exit code |

## Estrategia 1: Test unitario del `CommandLineRunner`

El runner es un `@Component` estándar. Su lógica de negocio puede testearse en aislamiento sin levantar el contexto de Spring ni necesitar base de datos.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.task;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class InventorySyncRunnerTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @InjectMocks
    private InventorySyncRunner runner;

    @Test
    void whenRunSucceeds_thenExitCodeIsZero() throws Exception {
        when(inventoryRepository.findPending()).thenReturn(java.util.List.of());

        runner.run();

        assertThat(runner.getExitCode()).isEqualTo(0);
        verify(inventoryRepository).findPending();
    }

    @Test
    void whenRepositoryThrows_thenExitCodeIsOneAndExceptionPropagated() {
        when(inventoryRepository.findPending())
            .thenThrow(new RuntimeException("DB error"));

        assertThatThrownBy(() -> runner.run())
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("DB error");

        assertThat(runner.getExitCode()).isEqualTo(1);
    }
}
```

## Estrategia 2: Test de integración con `@SpringBootTest` y H2

Este nivel verifica que `@EnableTask` registra correctamente la ejecución en `TASK_EXECUTION` y que el `TaskLifecycleListener` funciona con el contexto real.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-task</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.task;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(properties = {
    // H2 en memoria — los schemas de Task se crean automáticamente
    "spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1",
    "spring.datasource.driver-class-name=org.h2.Driver",
    "spring.datasource.username=sa",
    "spring.datasource.password=",
    "spring.sql.init.schema-locations=classpath:org/springframework/cloud/task/schema-h2.sql",
    // Nombre único por test para no colisionar en la tabla TASK_EXECUTION
    "spring.application.name=test-inventory-sync"
})
class InventorySyncTaskIntegrationTest {

    @Autowired
    private TaskExplorer taskExplorer;

    @Test
    void whenTaskRuns_thenExecutionRegisteredInDatabase() {
        // La tarea se ejecuta automáticamente al arrancar el contexto Spring
        // (CommandLineRunner se invoca durante el refresh del contexto)
        List<TaskExecution> executions =
            taskExplorer.findTaskExecutions("test-inventory-sync", null).getContent();

        assertThat(executions).isNotEmpty();

        TaskExecution execution = executions.get(0);
        assertThat(execution.getTaskName()).isEqualTo("test-inventory-sync");
        // Exit code 0 = éxito; null = tarea aún en ejecución (no debería ocurrir aquí)
        assertThat(execution.getExitCode()).isEqualTo(0);
        assertThat(execution.getStartTime()).isNotNull();
        assertThat(execution.getEndTime()).isNotNull();
        assertThat(execution.getEndTime()).isAfterOrEqualTo(execution.getStartTime());
    }
}
```

> [CONCEPTO] `TaskExplorer` es el bean que Spring Cloud Task proporciona para leer las ejecuciones registradas en `TASK_EXECUTION`. Es el equivalente de `JdbcTemplate` pero tipado para el modelo de Task: `findTaskExecutions(name, pageable)`, `getTaskExecutionCount()`, `getRunningTaskExecutionCount()`.

## Estrategia 3: Verificación de `TaskExecution` con aserciones avanzadas

Cuando el test necesita verificar el `EXIT_MESSAGE`, los argumentos de lanzamiento o el manejo de errores, se usan aserciones más específicas sobre el objeto `TaskExecution`.

```java
package com.example.task;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.cloud.task.repository.TaskExplorer;
import org.springframework.data.domain.PageRequest;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    args = {"--mode=full", "--source=catalog"},
    properties = {
        "spring.datasource.url=jdbc:h2:mem:testdb2;DB_CLOSE_DELAY=-1",
        "spring.datasource.driver-class-name=org.h2.Driver",
        "spring.datasource.username=sa",
        "spring.datasource.password=",
        "spring.sql.init.schema-locations=classpath:org/springframework/cloud/task/schema-h2.sql",
        "spring.application.name=test-task-args"
    }
)
class TaskExecutionArgumentsTest {

    @Autowired
    private TaskExplorer taskExplorer;

    @Test
    void whenTaskRunsWithArgs_thenArgumentsPersistedInExecution() {
        TaskExecution execution = taskExplorer
            .findTaskExecutions("test-task-args", PageRequest.of(0, 1))
            .getContent()
            .get(0);

        // Los args de @SpringBootTest(args=...) deben persistirse
        assertThat(execution.getArguments())
            .contains("--mode=full", "--source=catalog");

        // Exit code 0 indica éxito
        assertThat(execution.getExitCode()).isEqualTo(0);

        // EXIT_MESSAGE solo se rellena en fallos
        assertThat(execution.getExitMessage()).isNullOrEmpty();
    }
}
```

> [ADVERTENCIA] Cada `@SpringBootTest` lanza la tarea al iniciar el contexto. Si múltiples tests de integración de Task comparten el mismo contexto (`spring.application.name` y `DataSource` iguales), las aserciones sobre `findTaskExecutions` pueden encontrar más de una ejecución. Usar nombres de aplicación distintos (`spring.application.name` en `properties`) para aislar cada test.

### Tabla resumen: cuándo usar cada estrategia

| Situación | Estrategia recomendada |
|---|---|
| Verificar lógica del CommandLineRunner con mocks | 1 — Test unitario con Mockito |
| Verificar registro en TASK_EXECUTION | 2 — @SpringBootTest con H2 y TaskExplorer |
| Verificar exit code y manejo de excepciones | 2 — @SpringBootTest + assertThat(execution.getExitCode()) |
| Verificar argumentos persistidos en TASK_EXECUTION | 3 — @SpringBootTest(args=...) con aserciones sobre execution.getArguments() |
| Verificar EXIT_MESSAGE en casos de fallo | 2/3 — simular fallo con mock del repositorio + @SpringBootTest |

## Tabla de elementos clave

| Componente / Método | Descripción |
|---|---|
| `TaskExplorer` | Bean de Spring Cloud Task para consultar TASK_EXECUTION; métodos: `findTaskExecutions`, `getTaskExecutionCount`, `getRunningTaskExecutionCount` |
| `TaskExecution` | POJO con los campos de una ejecución: `taskName`, `startTime`, `endTime`, `exitCode`, `exitMessage`, `arguments` |
| `schema-h2.sql` | Schema SQL de Spring Cloud Task para H2; en `spring-cloud-task-core.jar` bajo `org/springframework/cloud/task/` |
| `spring.application.name` | Determina el `taskName` en `TASK_EXECUTION`; esencial para identificar la tarea en `TaskExplorer` |
| `@SpringBootTest(args=...)` | Permite pasar argumentos de lanzamiento al `CommandLineRunner` en tests de integración |

## Buenas y malas prácticas

**Hacer:**
- Usar H2 en memoria con el schema DDL de Spring Cloud Task (`schema-h2.sql`) para tests de integración; evita dependencias de infraestructura en CI y garantiza aislamiento entre tests.
- Usar `spring.application.name` diferente en cada test de integración de Task para evitar colisiones en `TASK_EXECUTION` cuando varios tests comparten el mismo contexto.
- Testear el exit code explícitamente con `assertThat(execution.getExitCode()).isEqualTo(0)`: es la única forma de verificar que `ExitCodeGenerator` y `ExitCodeExceptionMapper` están configurados correctamente.
- Añadir tests de integración que verifican el campo `arguments` de `TaskExecution` cuando la tarea acepta argumentos: garantiza que el parsing de argumentos y su persistencia funcionan de extremo a extremo.

**Evitar:**
- Testear solo el `CommandLineRunner` con tests unitarios y omitir los tests de integración: no detecta problemas de configuración de `@EnableTask`, esquema de base de datos incorrecto, o errores en el `TaskLifecycleListener`.
- Usar la misma `DataSource` de producción en tests de integración: contamina `TASK_EXECUTION` con registros de test y puede interferir con el monitoring de producción.
- Omitir la llamada a `System.exit(SpringApplication.exit(...))` en tests que verifican exit codes: en contexto de `@SpringBootTest`, el exit code no se propaga al OS, pero sí debe registrarse correctamente en `TASK_EXECUTION`.

---

← [11.8 Métricas, observabilidad y trazabilidad](sc-task-observability.md) | [Índice (README.md)](README.md) | [12.1 Modelo de programación funcional: Function, Consumer, Supplier y FunctionCatalog](sc-function-modelo.md) →
