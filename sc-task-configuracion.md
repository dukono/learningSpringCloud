# 11.3 Configuración y propiedades spring.cloud.task.*

← [11.2 TaskRepository y persistencia de ejecuciones](sc-task-repository.md) | [Índice (README.md)](README.md) | [11.4 Argumentos y parámetros de lanzamiento](sc-task-arguments.md) →

---

## Introducción

Una tarea Spring Cloud Task expone un conjunto de propiedades bajo el prefijo `spring.cloud.task.*` que controlan aspectos críticos del comportamiento de ejecución: cómo se identifica la tarea en el historial, si el contexto Spring se cierra al terminar, si se permite ejecutar más de una instancia simultánea y cómo se separan las bases de datos de negocio y de auditoría. Sin configurar correctamente estas propiedades, los problemas en producción son silenciosos: tareas que no terminan porque el contexto no se cierra, duplicaciones de ejecución en entornos de alta disponibilidad o contaminación de datos cuando varias tareas comparten DataSource. Este fichero cubre todas las propiedades `spring.cloud.task.*` que un desarrollador senior debe conocer de memoria y el patrón de múltiples DataSources que separa el almacenamiento de `TaskRepository` del DataSource de negocio.

## Representación visual

El diagrama muestra cómo Spring Cloud Task selecciona el DataSource para el `TaskRepository` cuando existen múltiples DataSources en el contexto, y el mecanismo de lock distribuido de `singleInstanceEnabled`.

```
  Contexto Spring con múltiples DataSources
  ─────────────────────────────────────────────────────────────────────
  @Bean @Primary DataSource     ──────────────────► lógica de negocio
         (negocio)                                  (JPA, JDBC propio)
  
  @Bean @TaskDatasource DataSource ────────────────► TaskRepository
         (tarea)                                    (TASK_EXECUTION, TASK_EXECUTION_PARAMS)
  ─────────────────────────────────────────────────────────────────────
  
  singleInstanceEnabled = true:
  
  [Instancia 1]              [Instancia 2]
      │                           │
      ▼                           ▼
  Consulta lock               Consulta lock
  en TASK_EXECUTION               │
      │                           │
  lock libre                  lock ocupado
      │                           │
  INSERT lock ─────────────────── │ ─► exception:
      │                           │    "Task already running"
  Ejecuta lógica              │
      │                           │
  DELETE lock                 │
      │
  (lockCheckTime: cada 10s)
  (lockTtl: expiración del lock si la instancia muere sin liberarlo)
```

## Ejemplo central

El siguiente ejemplo muestra una tarea con configuración completa: separación de DataSources, `singleInstanceEnabled`, `tablePrefix` y cierre del contexto. El escenario es una tarea que no debe ejecutarse en paralelo en entornos con múltiples réplicas.

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
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Configuración de múltiples DataSources (`DataSourceConfig.java`):**

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.task.configuration.annotation.TaskDatasource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {

    // DataSource PRIMARIO para la lógica de negocio de la tarea
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.business")
    public DataSourceProperties businessDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @Primary
    public DataSource businessDataSource() {
        return businessDataSourceProperties()
                .initializeDataSourceBuilder()
                .build();
    }

    // DataSource para TaskRepository — anotado con @TaskDatasource
    @Bean
    @TaskDatasource
    @ConfigurationProperties("spring.datasource.task")
    public DataSourceProperties taskDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @TaskDatasource
    public DataSource taskDataSource() {
        return taskDataSourceProperties()
                .initializeDataSourceBuilder()
                .build();
    }
}
```

**Clase principal (`UniqueTask.java`):**

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableTask
public class UniqueTask {
    public static void main(String[] args) {
        SpringApplication.run(UniqueTask.class, args);
    }
}

@Component
class UniqueRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Ejecutando tarea de instancia única...");
        // Si otra instancia intenta arrancar mientras esta está en curso,
        // Spring Cloud Task lanzará una excepción antes de ejecutar este método
        Thread.sleep(5000); // simula trabajo
        System.out.println("Tarea completada.");
    }
}
```

**Configuración (`application.yml`) completa con todas las propiedades clave:**

```yaml
spring:
  application:
    name: unique-task
  
  # DataSource de negocio (primario)
  datasource:
    business:
      url: jdbc:postgresql://localhost:5432/businessdb
      username: bizuser
      password: bizpassword
      driver-class-name: org.postgresql.Driver
    
    # DataSource dedicado para TaskRepository
    task:
      url: jdbc:postgresql://localhost:5432/taskdb
      username: taskuser
      password: taskpassword
      driver-class-name: org.postgresql.Driver

  cloud:
    task:
      # Nombre que aparece en TASK_EXECUTION.task_name
      name: unique-task

      # Inicializar esquema al arrancar (false en producción con Flyway/Liquibase)
      initialize-enabled: false

      # Cerrar ApplicationContext cuando terminen todos los ApplicationRunner/CommandLineRunner
      closecontextEnabled: true

      # Correlación con sistemas externos (ej: job id de Kubernetes)
      # executionid: ${EXTERNAL_JOB_ID:}

      # Prevención de ejecuciones duplicadas simultáneas
      singleInstanceEnabled: true

      # Cada cuántos milisegundos se comprueba si el lock sigue activo
      singleInstanceLockCheckTime: 10000

      # TTL del lock en milisegundos — protege contra kills abruptos que no liberan el lock
      singleInstanceLockTtl: 120000

      # Prefijo para las tablas en la BD de tareas
      tablePrefix: UNIQUE_
```

## Tabla de elementos clave

La tabla siguiente reúne todas las propiedades `spring.cloud.task.*` que un desarrollador senior debe conocer, con sus valores por defecto y el efecto en producción.

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.task.name` | `String` | nombre del app | Identificador almacenado en `TASK_EXECUTION.task_name`. Permite distinguir tareas en BD compartida. |
| `spring.cloud.task.initialize-enabled` | `Boolean` | `true` | Ejecuta `schema-*.sql` al arrancar. Desactivar en producción con pipeline de migraciones. |
| `spring.cloud.task.closecontextEnabled` | `Boolean` | `true` | Cierra el `ApplicationContext` al terminar los runners. Sin esto el proceso no termina. |
| `spring.cloud.task.executionid` | `Long` | generado | Fuerza un `task_execution_id` específico. Útil para correlación con sistemas externos. |
| `spring.cloud.task.singleInstanceEnabled` | `Boolean` | `false` | Impide que dos instancias de la misma tarea se ejecuten simultáneamente. |
| `spring.cloud.task.singleInstanceLockCheckTime` | `Long` (ms) | `10000` | Intervalo de comprobación del lock. Valores bajos aumentan la contención en BD. |
| `spring.cloud.task.singleInstanceLockTtl` | `Long` (ms) | `null` | TTL del lock. Si la tarea muere abruptamente sin liberar el lock, expira tras este tiempo. |
| `spring.cloud.task.tablePrefix` | `String` | `""` | Prefijo de tablas. Permite aislar múltiples tareas en la misma base de datos. |
| `@TaskDatasource` | Anotación | — | Calificador Spring para el DataSource dedicado a `TaskRepository`. |
| Tarea huérfana | Condición | — | `end_time = null` y proceso ya no existe. Detectar con `TaskExplorer.getLatestTaskExecutionForTaskName()`. |

## Buenas y malas prácticas

**Hacer:**

- Establecer `spring.cloud.task.singleInstanceLockTtl` siempre que se use `singleInstanceEnabled: true`. Sin TTL, un kill abrupto del proceso deja el lock activo indefinidamente y ninguna instancia posterior podrá ejecutar la tarea.
- Configurar `spring.cloud.task.name` explícitamente con un valor estable. Si no se configura, el nombre por defecto es el `spring.application.name`, que puede cambiar si se renombra la aplicación, rompiendo la correlación histórica en `TASK_EXECUTION`.
- Usar `@TaskDatasource` para el DataSource dedicado al `TaskRepository` cuando la tarea tiene acceso a una BD de negocio. Evita que las transacciones largas de negocio bloqueen las escrituras del ciclo de vida de la tarea.
- Desactivar `initialize-enabled: false` en entornos de producción y gestionar el esquema de `TASK_EXECUTION` con Flyway o Liquibase. El autoarranque de DDL en producción es impredecible en despliegues concurrentes.

**Evitar:**

- Usar `singleInstanceEnabled: true` sin configurar `singleInstanceLockTtl` en entornos Kubernetes donde los pods pueden recibir un SIGKILL. Sin TTL el lock persiste en BD y la siguiente ejecución programada será bloqueada indefinidamente.
- Confiar en `closecontextEnabled: true` (valor por defecto) en tareas que también arrancan un servidor HTTP (por ejemplo, si se añade `spring-boot-starter-web` por error). El contexto se cierra al terminar el runner pero el servidor puede tener threads activos, causando errores de apagado inesperados.
- Asignar el mismo `tablePrefix` a dos tareas diferentes que comparten BD. Los locks de `singleInstanceEnabled` y el historial de `TASK_EXECUTION` se mezclan, provocando bloqueos cruzados difíciles de diagnosticar.
- Inyectar `TaskRepository` en la lógica de negocio para registrar estados intermedios. El ciclo de vida de `TaskExecution` está diseñado para start/end; los estados intermedios se modelan mejor con eventos de dominio o métricas Micrometer.

> **[ADVERTENCIA]** La propiedad `spring.cloud.task.singleInstanceLockCheckTime` controla solo el intervalo de sondeo del lock, no el tiempo de espera máximo. Si la tarea que tiene el lock tarda más de `lockTtl` en terminar, el lock expira y otra instancia puede arrancar antes de que la primera termine, causando ejecuciones solapadas.

> **[PREREQUISITO]** Para usar múltiples DataSources con `@TaskDatasource` es necesario tener `spring-boot-starter-jdbc` o `spring-boot-starter-data-jpa` en el classpath. La autoconfiguración de Spring Boot solo crea el calificador `@TaskDatasource` si detecta más de un `DataSource` bean en el contexto.

---

← [11.2 TaskRepository y persistencia de ejecuciones](sc-task-repository.md) | [Índice (README.md)](README.md) | [11.4 Argumentos y parámetros de lanzamiento](sc-task-arguments.md) →
