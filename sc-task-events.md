# 11.7 Spring Cloud Task — Task Events y TaskExecutionListener

← [11.6 Integración con Spring Cloud Data Flow](sc-task-data-flow.md) | [Índice](README.md) | [11.8 Composed Tasks](sc-task-composed.md) →

---

## Introducción

Spring Cloud Task ofrece dos mecanismos para reaccionar a los eventos del ciclo de vida de una tarea: la interfaz `TaskExecutionListener` para callbacks síncronos locales, y la publicación de `TaskExecutionEvent` a un `MessageChannel` para integraciones reactivas distribuidas. El listener local es ideal para auditoría y métricas dentro del mismo proceso; los eventos de mensajería permiten que sistemas externos sean notificados cuando una Task arranca, termina o falla.

> [CONCEPTO] `TaskExecutionListener` proporciona callbacks síncronos que se ejecutan dentro del mismo contexto de Spring. Los `TaskExecutionEvent` son mensajes enviados a un `MessageChannel` (Kafka/RabbitMQ vía Spring Cloud Stream) cuando `spring-cloud-task-stream` está presente.

## Flujo de eventos del ciclo de vida

Los tres métodos de `TaskExecutionListener` se invocan en momentos precisos del ciclo de vida. El siguiente diagrama muestra cuándo se llama cada callback.

```
Task arranca
    │
    ▼
TaskLifecycleListener.onTaskStartup(TaskExecution)
   → @EnableTask persiste START_TIME en TASK_EXECUTION
   → [OPCIONAL] TaskExecutionEvent(TaskStartedEvent) → MessageChannel
    │
    ▼
ApplicationRunner.run() ejecuta lógica de negocio
    │
    ├─── sin excepción
    │       │
    │       ▼
    │   onTaskEnd(TaskExecution)
    │      → EXIT_CODE=0, END_TIME actualizado
    │      → [OPCIONAL] TaskExecutionEvent(TaskEndedEvent) → MessageChannel
    │
    └─── con excepción
            │
            ▼
        onTaskFailed(TaskExecution, Throwable)
           → EXIT_CODE≠0, ERROR_MESSAGE con stacktrace
           → [OPCIONAL] TaskExecutionEvent(TaskFailedEvent) → MessageChannel
```

## Ejemplo central

El siguiente ejemplo muestra la implementación completa de `TaskExecutionListener` para auditoría y, a continuación, la configuración opcional para publicar eventos a un MessageChannel usando Spring Cloud Task Stream.

```java
package com.example.task;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.cloud.task.listener.TaskExecutionListener;
import org.springframework.cloud.task.repository.TaskExecution;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;

@SpringBootApplication
@EnableTask
public class TaskEventsApp {
    public static void main(String[] args) {
        SpringApplication.run(TaskEventsApp.class, args);
    }
}

// Listener de auditoría — se registra automáticamente como bean
@Component
class AuditTaskListener implements TaskExecutionListener {

    @Override
    public void onTaskStartup(TaskExecution taskExecution) {
        System.out.printf("[AUDIT] Task '%s' iniciada (ID=%d) a las %s%n",
            taskExecution.getTaskName(),
            taskExecution.getExecutionId(),
            taskExecution.getStartTime()
        );
        // Aquí se puede escribir en una tabla de auditoría,
        // enviar una notificación a Slack, etc.
    }

    @Override
    public void onTaskEnd(TaskExecution taskExecution) {
        System.out.printf("[AUDIT] Task '%s' completada (ID=%d) | exitCode=%d%n",
            taskExecution.getTaskName(),
            taskExecution.getExecutionId(),
            taskExecution.getExitCode()
        );
        // Calcular duración
        long durationMs = taskExecution.getEndTime().getTime()
            - taskExecution.getStartTime().getTime();
        System.out.println("[AUDIT] Duración: " + durationMs + " ms");
    }

    @Override
    public void onTaskFailed(TaskExecution taskExecution, Throwable throwable) {
        System.err.printf("[AUDIT] Task '%s' FALLIDA (ID=%d): %s%n",
            taskExecution.getTaskName(),
            taskExecution.getExecutionId(),
            throwable.getMessage()
        );
        // Enviar alerta a sistema de monitorización
    }
}

// Runner que puede fallar
@Component
class BusinessRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Ejecutando lógica de negocio...");
        // Si falla: se invoca onTaskFailed en el listener
    }
}
```

Para publicar eventos a un MessageChannel (requiere `spring-cloud-task-stream`), la configuración mínima:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-task-stream</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>
```

```yaml
# application.yml con spring-cloud-task-stream
spring:
  cloud:
    stream:
      bindings:
        task-events:
          destination: task-lifecycle-events
          content-type: application/json
    task:
      events:
        enabled: true  # activa publicación de TaskExecutionEvent
```

Con esta configuración, cada vez que una Task arranca, termina o falla, se publica un mensaje al topic `task-lifecycle-events` de Kafka con el payload del `TaskExecutionEvent`.

## Tabla de elementos clave

Los dos mecanismos de eventos tienen características distintas que determinan cuándo usar cada uno.

| Mecanismo | Tipo | Alcance | Dependencia |
|---|---|---|---|
| `TaskExecutionListener` | Interfaz | Local, síncrono, mismo JVM | Solo `spring-cloud-starter-task` |
| `TaskExecutionEvent` | Mensaje | Distribuido, asíncrono, pub/sub | `spring-cloud-task-stream` + binder |
| `onTaskStartup` | Callback | Inicio de la tarea | — |
| `onTaskEnd` | Callback | Fin exitoso de la tarea | — |
| `onTaskFailed` | Callback | Fin con excepción | — |
| `TaskStartedEvent` | Evento | Equivalente a `onTaskStartup` en mensajería | — |
| `TaskEndedEvent` | Evento | Equivalente a `onTaskEnd` en mensajería | — |
| `TaskFailedEvent` | Evento | Equivalente a `onTaskFailed` en mensajería | — |

## Diferencia entre listener local y evento de mensajería

La elección entre `TaskExecutionListener` y `TaskExecutionEvent` depende del alcance de la notificación y de los requisitos de disponibilidad.

`TaskExecutionListener` es síncrono: los callbacks se ejecutan en el mismo hilo y contexto de Spring que la Task. Si el listener falla, puede afectar al EXIT_CODE de la Task. Es apropiado para auditoría local, métricas internas o logging enriquecido.

`TaskExecutionEvent` es asíncrono y desacoplado: el mensaje se envía al broker de mensajería y los consumidores son independientes de la Task. Si el broker no está disponible, la publicación puede fallar pero la Task continúa (según la configuración del binder). Es apropiado para notificaciones a sistemas externos, dashboards de monitorización o pipelines event-driven.

> [ADVERTENCIA] Si `onTaskFailed` o `onTaskEnd` en el `TaskExecutionListener` lanzan una excepción, esta excepción puede sobreescribir el mensaje de error original y modificar el EXIT_CODE almacenado en `TASK_EXECUTION`. Implementar los listeners con manejo de excepciones defensivo.

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `TaskExecutionListener` para logging, métricas y auditoría que deben ejecutarse síncronamente con la Task.
- Usar `TaskExecutionEvent` con `spring-cloud-task-stream` cuando otros sistemas necesitan ser notificados del ciclo de vida de la Task de forma desacoplada.
- Envolver el cuerpo de los listeners en try-catch para evitar que un fallo en el listener afecte al registro final de la Task.

**Malas prácticas:**
- Implementar lógica de negocio pesada dentro de `onTaskStartup` o `onTaskEnd`: estos callbacks son síncronos y retrasan el ciclo de vida de la Task.
- Asumir que `spring-cloud-task-stream` publica eventos por defecto sin configurar `spring.cloud.task.events.enabled=true`.
- Registrar múltiples `TaskExecutionListener` beans sin comprobar si el orden de invocación es relevante para la lógica de negocio.

> [PREREQUISITO] Para usar `TaskExecutionEvent` con mensajería, se necesita `spring-cloud-task-stream` en el classpath más un binder de Spring Cloud Stream (Kafka o RabbitMQ). Sin el binder configurado, la publicación fallará silenciosamente o lanzará excepciones al arrancar.

## Verificación y práctica

> [EXAMEN] **Pregunta 1:** ¿Qué interfaz permite reaccionar a los eventos del ciclo de vida de una Task y cuáles son sus tres métodos?

> [EXAMEN] **Pregunta 2:** ¿Qué dependencia Maven adicional es necesaria para publicar `TaskExecutionEvent` a un MessageChannel?

> [EXAMEN] **Pregunta 3:** ¿Cuál es la diferencia principal entre `TaskExecutionListener` y `TaskExecutionEvent` en términos de sincronía y alcance?

> [EXAMEN] **Pregunta 4:** Si `onTaskFailed` en un `TaskExecutionListener` lanza una excepción, ¿qué efecto puede tener sobre la persistencia en `TASK_EXECUTION`?

> [EXAMEN] **Pregunta 5:** ¿Es necesario implementar `TaskExecutionListener` para que la Task registre su ciclo de vida en base de datos?

---

← [11.6 Integración con Spring Cloud Data Flow](sc-task-data-flow.md) | [Índice](README.md) | [11.8 Composed Tasks](sc-task-composed.md) →
