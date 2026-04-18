# 11.4 Spring Cloud Task — ApplicationRunner, CommandLineRunner y Argumentos

← [11.3 Configuración y Datasource](sc-task-configuracion.md) | [Índice](README.md) | [11.5 Integración con Spring Batch](sc-task-batch.md) →

---

## Introducción

La lógica de negocio de una Spring Cloud Task se implementa mediante beans que implementan `ApplicationRunner` o `CommandLineRunner`. Estas interfaces de Spring Boot son el punto de entrada de la ejecución y su comportamiento (éxito o excepción) determina el `EXIT_CODE` que se persiste en `TASK_EXECUTION`. Spring Cloud Task añade sobre este mecanismo el registro automático de los argumentos de lanzamiento en la tabla `TASK_EXECUTION_PARAMS` y la posibilidad de inyectar el contexto de ejecución actual mediante `TaskExecutionContext`.

> [CONCEPTO] La diferencia entre `ApplicationRunner` y `CommandLineRunner` es el tipo del parámetro: `ApplicationRunner.run(ApplicationArguments)` recibe argumentos ya parseados con soporte para opciones `--key=value`; `CommandLineRunner.run(String...)` recibe el array de strings crudo.

## Flujo de ejecución con múltiples runners

Cuando existen varios runners en el contexto, Spring Cloud Task los ejecuta en orden. La anotación `@Order` controla la secuencia. Si cualquiera de ellos lanza una excepción no capturada, la tarea termina con `EXIT_CODE ≠ 0` y los runners posteriores no se ejecutan.

```
ApplicationContext arranca → @EnableTask activa registro START
        │
        ▼
Runners se ejecutan por orden @Order (menor número = primero)
        │
        ├─── @Order(1) Runner A ──→ OK
        ├─── @Order(2) Runner B ──→ OK
        └─── @Order(3) Runner C ──→ Excepción → EXIT_CODE=1, runners posteriores NO se ejecutan
```

## Ejemplo central

El siguiente ejemplo muestra tres runners con orden definido, el uso de `ApplicationArguments` para acceder a argumentos parseados, y la inyección de `TaskExecutionContext` para acceder al ID de la ejecución actual. El código también demuestra cómo los argumentos quedan registrados automáticamente en `TASK_EXECUTION_PARAMS`.

```java
package com.example.task;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.cloud.task.listener.TaskExecutionContext;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.context.annotation.Bean;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableTask
public class RunnersTaskApp {

    public static void main(String[] args) {
        // Lanzar con argumentos: --report.date=2025-01-01 --dry-run
        SpringApplication.run(RunnersTaskApp.class, args);
    }
}

// Runner principal — se ejecuta primero
@Component
@Order(1)
class ValidationRunner implements ApplicationRunner {

    private final TaskExecutionContext taskExecutionContext;

    public ValidationRunner(TaskExecutionContext taskExecutionContext) {
        this.taskExecutionContext = taskExecutionContext;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        TaskExecution execution = taskExecutionContext.getTaskExecution();
        System.out.println("Ejecución ID: " + execution.getExecutionId());
        System.out.println("Task Name: " + execution.getTaskName());

        // Acceso a argumentos parseados
        if (args.containsOption("report.date")) {
            String date = args.getOptionValues("report.date").get(0);
            System.out.println("Procesando fecha: " + date);
        }

        // Acceso a argumentos no-option (posicionales)
        args.getNonOptionArgs().forEach(arg ->
            System.out.println("Argumento posicional: " + arg)
        );

        // Si falla aquí, @Order(2) y @Order(3) NO se ejecutan
        validateEnvironment();
    }

    private void validateEnvironment() {
        // Lanza IllegalStateException si el entorno no es válido
        // → EXIT_CODE = 1, ERROR_MESSAGE con stacktrace
    }
}

// Runner secundario — procesa datos
@Component
@Order(2)
class ProcessingRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        boolean dryRun = args.containsOption("dry-run");
        if (dryRun) {
            System.out.println("Modo dry-run: simulando procesamiento");
            return;
        }
        System.out.println("Procesando datos reales...");
    }
}

// CommandLineRunner — recibe args como String[] crudo
@Component
@Order(3)
class CleanupRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        System.out.println("Limpieza completada. Args recibidos: " + args.length);
    }
}
```

Al lanzar con `java -jar app.jar --report.date=2025-01-01 --dry-run input.csv`, la tabla `TASK_EXECUTION_PARAMS` registrará:

```
TASK_EXECUTION_ID | TASK_PARAM
------------------+------------------------
1                 | --report.date=2025-01-01
1                 | --dry-run
1                 | input.csv
```

## Tabla de elementos clave

Los elementos involucrados en la ejecución de runners dentro de una Task tienen roles diferenciados según el alcance de la información que manejan.

| Elemento | Tipo | Responsabilidad |
|---|---|---|
| `ApplicationRunner` | Interfaz Spring Boot | Lógica principal; recibe `ApplicationArguments` parseados |
| `CommandLineRunner` | Interfaz Spring Boot | Alternativa; recibe `String[]` crudo de la línea de comandos |
| `@Order(n)` | Anotación | Define el orden de ejecución (menor n = antes) |
| `ApplicationArguments` | Bean | Argumentos parseados: options (`--key=value`) y non-options |
| `TaskExecutionContext` | Bean inyectable | Provee la `TaskExecution` actual con ID, nombre y metadatos |
| `TASK_EXECUTION_PARAMS` | Tabla BD | Almacena cada argumento como fila separada vinculada al `TASK_EXECUTION_ID` |

## Propagación de excepciones y EXIT_CODE

El comportamiento del `EXIT_CODE` depende directamente de cómo se propagan las excepciones desde los runners. Este es uno de los temas más importantes para la certificación.

Spring Cloud Task envuelve la ejecución de todos los runners en un bloque try-catch a nivel de `TaskLifecycleListener`. Si cualquier runner lanza una excepción no capturada, el `TaskLifecycleListener` la intercepta, invoca `ExitCodeExceptionMapper` (si existe en el contexto) para determinar el código de salida, y persiste `EXIT_CODE` y `ERROR_MESSAGE` en `TASK_EXECUTION`.

```java
// Ejemplo de ExitCodeExceptionMapper para distinguir tipos de error
@Component
public class BusinessExitCodeMapper implements ExitCodeExceptionMapper {

    @Override
    public int getExitCode(Throwable exception) {
        if (exception instanceof IllegalArgumentException) {
            return 2;  // Error de validación de argumentos
        }
        if (exception instanceof java.sql.SQLException) {
            return 3;  // Error de base de datos
        }
        return 1;  // Error genérico
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `ApplicationRunner` en lugar de `CommandLineRunner` cuando se necesita acceder a argumentos con nombre (`--key=value`): la interfaz `ApplicationArguments` facilita su parseo.
- Definir `@Order` explícito cuando hay múltiples runners para evitar depender del orden no determinista de beans Spring.
- Inyectar `TaskExecutionContext` para obtener el `executionId` en lugar de consultar la BD directamente.

**Malas prácticas:**
- Capturar todas las excepciones dentro del runner con `try-catch` genérico y loguearlas sin relanzar: Spring Cloud Task registrará `EXIT_CODE=0` aunque la lógica haya fallado.
- Ejecutar lógica de larga duración (horas) dentro de un `ApplicationRunner` sin mecanismo de checkpoint: si la JVM muere, no hay forma de retomar el trabajo.
- Depender del orden de ejecución sin `@Order` explícito cuando hay múltiples runners.

> [ADVERTENCIA] Si el runner captura la excepción y no la relanza, el `EXIT_CODE` en `TASK_EXECUTION` será 0 aunque el trabajo haya fallado. Esto es un antipatrón de observabilidad crítico.

> [PREREQUISITO] `TaskExecutionContext` solo está disponible como bean dentro del ciclo de vida de la tarea. No se puede inyectar en beans que se inicializan antes de que `@EnableTask` haya registrado la ejecución.

## Verificación y práctica

> [EXAMEN] **Pregunta 1:** ¿Qué diferencia hay entre `ApplicationRunner` y `CommandLineRunner` respecto al tipo de argumento que reciben?

> [EXAMEN] **Pregunta 2:** ¿En qué tabla de base de datos quedan registrados los argumentos de línea de comandos y cuál es su estructura?

> [EXAMEN] **Pregunta 3:** Si hay tres `ApplicationRunner` beans con `@Order(3)`, `@Order(1)` y `@Order(2)`, y el segundo lanza una excepción, ¿cuál de los tres se ejecuta?

> [EXAMEN] **Pregunta 4:** ¿Qué bean se inyecta para obtener el `executionId` de la Task que está ejecutándose actualmente?

> [EXAMEN] **Pregunta 5:** ¿Qué interfaz permite mapear tipos de excepción a `EXIT_CODE` específicos en lugar del genérico 1?

---

← [11.3 Configuración y Datasource](sc-task-configuracion.md) | [Índice](README.md) | [11.5 Integración con Spring Batch](sc-task-batch.md) →
