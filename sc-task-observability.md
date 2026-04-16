# 11.8 Métricas, observabilidad y trazabilidad

← [11.7 Integración con Spring Batch](sc-task-batch-integration.md) | [Índice (README.md)](README.md) | [11.9 Testing de Spring Cloud Task](sc-task-testing.md) →

---

Una tarea de Spring Cloud Task deja rastro en la base de datos (`TASK_EXECUTION`), pero ese rastro no es suficiente para operar en producción: no informa de la latencia real de la tarea, no correlaciona con trazas distribuidas de otros servicios, y no propaga el resultado de la tarea al orquestador de forma semántica. Spring Cloud Task resuelve los tres aspectos mediante integración con Micrometer Observations (métricas y trazas) y a través del mecanismo de exit codes con `ExitCodeExceptionMapper`. Estos tres mecanismos son ortogonales —pueden activarse de forma independiente— pero comparten la finalidad de hacer la ejecución de una tarea observable y diagnosticable en producción.

> [PREREQUISITO] Requiere `spring-cloud-starter-task`, `micrometer-core` para observaciones y métricas, y `micrometer-tracing-bridge-otel` con un exportador (Zipkin, OTLP) para trazas distribuidas. Los exit codes requieren solo `spring-cloud-starter-task`.

## Mecanismos de observabilidad de Spring Cloud Task

Los tres pilares de la observabilidad en Spring Cloud Task actúan en capas distintas del ciclo de vida de la tarea, desde el registro en base de datos hasta la propagación al orquestador.

```
┌─────────────────────────────────────────────────────────────┐
│                   Spring Cloud Task                         │
│                                                             │
│  Ejecución de la tarea                                      │
│  ├── TaskObservation (Micrometer) → métricas + traza        │
│  │     spring.cloud.task.observation.enabled = true         │
│  │     Observation: "spring.cloud.task"                     │
│  │     - duration, status (SUCCEEDED/FAILED)                │
│  │     - trace ID propagado al TaskExecution                │
│  │                                                          │
│  ├── TaskExecution                                          │
│  │     → TASK_EXECUTION en DB (id, start/end, exit_code)    │
│  │     → exit_message para descripción del resultado        │
│  │                                                          │
│  └── Exit codes                                             │
│        ExitCodeExceptionMapper → RuntimeException → 2       │
│        ExitCodeGenerator       → lógica de negocio → 3..N   │
│        DefaultApplicationContext exit code → OS process     │
└─────────────────────────────────────────────────────────────┘
```

## Ejemplo central: observaciones, tracing y exit codes

El siguiente ejemplo configura los tres mecanismos en una tarea Spring Cloud Task: observaciones Micrometer habilitadas, tracing con Zipkin, y exit codes semánticos para comunicar el resultado al orquestador.

### Dependencias Maven

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
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-task</artifactId>
    </dependency>
    <!-- Observaciones y métricas -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-core</artifactId>
    </dependency>
    <!-- Tracing con OpenTelemetry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-zipkin</artifactId>
    </dependency>
    <!-- Actuator para endpoint /actuator/metrics -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  cloud:
    task:
      observation:
        enabled: true     # habilita TaskObservation en Micrometer
  application:
    name: inventory-sync-task

management:
  tracing:
    sampling:
      probability: 1.0    # 100% de muestreo en dev/test
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

### Tarea con ExitCodeGenerator y ExitCodeExceptionMapper

```java
package com.example.task;

import org.springframework.boot.ExitCodeGenerator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.context.annotation.Bean;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.ExitCodeExceptionMapper;

@SpringBootApplication
@EnableTask
public class InventorySyncTaskApplication {

    public static void main(String[] args) {
        // SpringApplication.exit() propaga el exit code al proceso OS
        System.exit(SpringApplication.exit(
            SpringApplication.run(InventorySyncTaskApplication.class, args)
        ));
    }

    // ExitCodeGenerator: lógica de negocio → exit code semántico
    @Bean
    public ExitCodeGenerator inventorySyncExitCodeGenerator(InventorySyncRunner runner) {
        return () -> runner.getExitCode();
    }

    // ExitCodeExceptionMapper: excepción → exit code numérico
    // Permite al orquestador (SCDF) saber qué tipo de error ocurrió
    @Bean
    public ExitCodeExceptionMapper taskExitCodeExceptionMapper() {
        return exception -> {
            if (exception.getCause() instanceof IllegalArgumentException) {
                return 2; // configuración incorrecta
            }
            if (exception.getCause() instanceof java.io.IOException) {
                return 3; // error de I/O (red, fichero)
            }
            return 1; // error genérico
        };
    }
}
```

```java
package com.example.task;

import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.atomic.AtomicInteger;

@Component
public class InventorySyncRunner implements CommandLineRunner {

    private static final Logger log = LoggerFactory.getLogger(InventorySyncRunner.class);
    private final AtomicInteger exitCode = new AtomicInteger(0);

    @Override
    public void run(String... args) throws Exception {
        log.info("Iniciando sincronización de inventario");
        try {
            // Lógica de negocio de la tarea
            int processed = syncInventory();
            log.info("Sincronización completada: {} registros procesados", processed);
            exitCode.set(0); // éxito
        } catch (Exception e) {
            log.error("Error en sincronización: {}", e.getMessage(), e);
            exitCode.set(1); // fallo — ExitCodeExceptionMapper puede refinarlo
            throw e;
        }
    }

    private int syncInventory() {
        // Implementación real omitida para brevedad
        return 42;
    }

    public int getExitCode() {
        return exitCode.get();
    }
}
```

> [CONCEPTO] `ExitCodeExceptionMapper` transforma una excepción no capturada en un exit code numérico. `ExitCodeGenerator` permite a la lógica de negocio decidir el exit code de forma programática, incluso cuando no hay excepción. Ambos son beans de Spring Boot estándar que Spring Cloud Task integra y persiste en `TASK_EXECUTION.EXIT_CODE`.

> [ADVERTENCIA] Si no se llama a `System.exit(SpringApplication.exit(...))` en el `main`, el proceso termina siempre con exit code 0, ignorando el `ExitCodeGenerator`. Spring Cloud Data Flow monitoriza el exit code del proceso OS para determinar el estado de la tarea en el pipeline.

## Tabla de elementos clave

Los parámetros y componentes que un desarrollador senior debe conocer para operar tareas en producción:

| Componente / Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.task.observation.enabled` | boolean | false | Activa TaskObservation en Micrometer; requiere `micrometer-core` en classpath |
| `ExitCodeGenerator` | interfaz | — | Bean que devuelve el exit code numérico del proceso; usado por `SpringApplication.exit()` |
| `ExitCodeExceptionMapper` | interfaz | DefaultExitCodeMapper (→1) | Mapea excepciones no capturadas a exit codes numéricos |
| `TASK_EXECUTION.EXIT_CODE` | columna DB | — | Exit code persistido por Spring Cloud Task en la base de datos |
| `TASK_EXECUTION.EXIT_MESSAGE` | columna DB | — | Mensaje de error o descripción del resultado; poblado automáticamente en fallos |
| `management.tracing.sampling.probability` | double | 0.1 | Fracción de trazas enviadas al backend; 1.0 para trazas completas en dev |
| Observation `spring.cloud.task` | Micrometer | — | Nombre de la observación creada por Spring Cloud Task; genera timer y traza |

## Buenas y malas prácticas

**Hacer:**
- Llamar siempre a `System.exit(SpringApplication.exit(...))` en el `main` cuando se usen `ExitCodeGenerator`; sin esta llamada, el exit code siempre es 0 y los orquestadores no detectan fallos.
- Registrar `ExitCodeExceptionMapper` para distinguir tipos de error en el exit code; facilita el troubleshooting en pipelines de SCDF y permite políticas de reintento selectivas por tipo de error.
- Usar `spring.cloud.task.observation.enabled=true` en producción para obtener métricas de duración y correlación de trazas; es la única forma de correlacionar una tarea con la traza distribuida que la originó.
- Configurar `management.tracing.sampling.probability=0.1` en producción para no saturar el backend de trazas; usar 1.0 solo en dev/test.

**Evitar:**
- Usar exit code 0 para indicar "tarea completada con advertencias": el orquestador lo interpreta como éxito completo. Usar exit codes distintos de 0 para cualquier estado que no sea éxito total.
- Ignorar el campo `EXIT_MESSAGE` en el diagnóstico de producción: es el primer lugar donde buscar la causa de un fallo sin abrir logs.
- Depender solo de la tabla `TASK_EXECUTION` para monitorización en producción sin integrar Micrometer: la tabla no informa de tendencias de latencia ni permite alertas en tiempo real.

---

← [11.7 Integración con Spring Batch](sc-task-batch-integration.md) | [Índice (README.md)](README.md) | [11.9 Testing de Spring Cloud Task](sc-task-testing.md) →
