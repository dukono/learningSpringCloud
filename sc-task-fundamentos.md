# 11.1 Spring Cloud Task — Fundamentos y Ciclo de Vida

← [10.12 Spring Cloud Contract — Troubleshooting](sc-contract-troubleshooting.md) | [Índice](README.md) | [11.2 TaskExecution y Persistencia](sc-task-execution-bd.md) →

---

## Introducción

Spring Cloud Task resuelve el problema de los trabajos de corta duración en el ecosistema Spring: aplicaciones que arrancan, realizan una operación concreta y terminan, necesitando dejar trazabilidad de su ejecución. A diferencia de un microservicio que corre indefinidamente, una Task registra su inicio, fin y resultado en base de datos, lo que permite monitorización, reintentos auditados y orquestación por parte de herramientas como Spring Cloud Data Flow.

> [CONCEPTO] Una Spring Cloud Task es un proceso de corta duración cuyo ciclo de vida completo queda registrado en la tabla `TASK_EXECUTION` de una base de datos relacional.

## Arquitectura y ciclo de vida

La activación de Spring Cloud Task se realiza con una sola anotación. Al arrancar el ApplicationContext, el framework intercepta el inicio y persiste una fila en la tabla `TASK_EXECUTION`. Al terminar la aplicación, actualiza esa misma fila con el resultado.

El siguiente diagrama muestra el flujo completo del ciclo de vida:

```
@SpringBootApplication arranca
        │
        ▼
@EnableTask activa TaskLifecycleConfiguration
        │
        ▼
SimpleTaskRepository.createTaskExecution()
   → INSERT TASK_EXECUTION (START_TIME, TASK_NAME, EXIT_CODE=null)
        │
        ▼
ApplicationRunner / CommandLineRunner ejecuta lógica de negocio
        │
        ├─── sin excepción ──→ EXIT_CODE = 0
        │                     SimpleTaskRepository.update() → END_TIME, EXIT_CODE=0
        │
        └─── con excepción ──→ EXIT_CODE = 1 (o según ExitCodeExceptionMapper)
                              SimpleTaskRepository.update() → END_TIME, EXIT_CODE≠0, ERROR_MESSAGE
```

## Ejemplo central

El siguiente ejemplo muestra la configuración mínima de una Spring Cloud Task. Se necesita la anotación `@EnableTask` sobre la clase principal y al menos un bean que implemente `ApplicationRunner` o `CommandLineRunner` con la lógica a ejecutar.

```java
package com.example.task;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableTask  // activa TaskLifecycleConfiguration y TaskRepositoryInitializer
public class HelloTaskApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloTaskApplication.class, args);
    }
}

@Component
class HelloTaskRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Tarea ejecutada. Args: " + args.getSourceArgs().length);
        // Si este método lanza excepción, EXIT_CODE será ≠ 0
    }
}
```

El `pom.xml` mínimo necesita:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-task</artifactId>
</dependency>
<!-- Datasource para persistir TASK_EXECUTION -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

## Tabla de elementos clave

Los componentes que Spring Cloud Task registra automáticamente en el contexto al detectar `@EnableTask` son los siguientes. Cada uno cumple un papel específico en el ciclo de vida.

| Componente | Tipo | Responsabilidad |
|---|---|---|
| `@EnableTask` | Anotación | Importa `TaskLifecycleConfiguration`; habilita registro automático |
| `TaskLifecycleConfiguration` | Configuración | Registra `TaskLifecycleListener` que intercepta inicio y fin |
| `SimpleTaskRepository` | Implementación | Coordina START y END delegando en `TaskExecutionDao` |
| `SimpleTaskNameResolver` | Implementación | Resuelve el nombre de la tarea desde `spring.cloud.task.name` o nombre del artefacto |
| `TaskExecution` | Entidad | Objeto Java que representa la ejecución; se persiste en `TASK_EXECUTION` |
| `TaskRepositoryInitializer` | Inicializador | Crea el schema de tablas si `spring.cloud.task.initialize-enabled=true` |

## Resolución del nombre de tarea

Spring Cloud Task necesita un nombre para identificar cada tipo de tarea en la tabla `TASK_EXECUTION`. La resolución sigue el siguiente orden de prioridad.

`SimpleTaskNameResolver` busca primero la propiedad `spring.cloud.task.name` en el entorno Spring. Si no existe, usa el nombre del artefacto (campo `spring.application.name`). Si tampoco está configurado, usa el nombre de la clase principal anotada con `@SpringBootApplication`.

```yaml
spring:
  application:
    name: my-report-task
  cloud:
    task:
      name: nightly-report  # tiene precedencia sobre spring.application.name
```

## Buenas y malas prácticas

**Buenas prácticas:**
- Siempre configurar `spring.application.name` o `spring.cloud.task.name` para identificar claramente la tarea en la tabla `TASK_EXECUTION`.
- Asegurarse de que cualquier excepción que deba provocar un `EXIT_CODE` distinto de 0 se propague correctamente hacia afuera del `ApplicationRunner.run()` en lugar de capturarse silenciosamente.
- Usar un datasource dedicado para las tablas de Task en entornos de producción, separado del datasource de negocio.

**Malas prácticas:**
- Capturar todas las excepciones dentro del runner sin relanzarlas: Spring Cloud Task registrará `EXIT_CODE=0` aunque la lógica haya fallado.
- Olvidar la anotación `@EnableTask`: la aplicación arrancará pero no registrará nada en `TASK_EXECUTION`.
- Usar `@EnableTask` en un servicio de larga duración (microservicio REST): está diseñado para procesos de corta duración que terminan.

> [ADVERTENCIA] Si el `ApplicationRunner` captura la excepción y no la relanza, Spring Cloud Task registrará `EXIT_CODE=0` aunque la lógica haya fallado. Esto es un antipatrón de observabilidad.

## Verificación y práctica

> [EXAMEN] **Pregunta 1:** ¿Qué anotación activa el mecanismo de registro de Spring Cloud Task y en qué clase debe colocarse?

> [EXAMEN] **Pregunta 2:** Describe el ciclo de vida completo de una Task: ¿qué ocurre en la base de datos cuando la aplicación arranca y cuando termina correctamente?

> [EXAMEN] **Pregunta 3:** ¿Cuál es el `EXIT_CODE` registrado en `TASK_EXECUTION` cuando el `ApplicationRunner` termina sin excepción? ¿Y cuando lanza una excepción?

> [EXAMEN] **Pregunta 4:** ¿Qué componente resuelve el nombre de la tarea y cuál es el orden de prioridad de resolución?

> [EXAMEN] **Pregunta 5:** Si una Task no registra nada en `TASK_EXECUTION` a pesar de ejecutarse, ¿cuáles son las dos causas más probables?

---

← [10.12 Spring Cloud Contract — Troubleshooting](sc-contract-troubleshooting.md) | [Índice](README.md) | [11.2 TaskExecution y Persistencia](sc-task-execution-bd.md) →
