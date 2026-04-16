# 11.4 Argumentos y parámetros de lanzamiento

← [11.3 Configuración y propiedades spring.cloud.task.*](sc-task-configuracion.md) | [Índice (README.md)](README.md) | [11.5 Integración con Spring Cloud Data Flow](sc-task-scdf-integration.md) →

---

## Introducción

Una tarea Spring Cloud Task raramente ejecuta siempre la misma lógica con los mismos datos. En la práctica, cada ejecución recibe parámetros que la diferencian: el rango de fechas a procesar, el identificador de un tenant, el nivel de verbosidad del log o el modo de operación. Estos parámetros llegan en el momento del lanzamiento como argumentos de línea de comandos y Spring Cloud Task los registra automáticamente en `TASK_EXECUTION_PARAMS` para garantizar la trazabilidad completa de cada ejecución. El desarrollador puede acceder a esos argumentos mediante tres mecanismos —`ApplicationArguments`, `CommandLineRunner` o `@Value`/`@ConfigurationProperties`— cuya elección afecta la testabilidad y la claridad de la interfaz de la tarea. Además, cuando la tarea se lanza desde Spring Cloud Data Flow, existe una restricción de unicidad de argumentos que obliga a estrategias de parametrización específicas.

## Representación visual

El diagrama compara los tres mecanismos de acceso a argumentos, mostrando qué interfaz usa cada uno y cómo se registran en `TASK_EXECUTION_PARAMS`.

```
  Lanzamiento: java -jar task.jar --mode=full --date=2025-01-15 tenant=ACME
                        │
                        ▼
              Spring Boot arg parsing
              ┌───────────────────────────────────────────────────────────────┐
              │ Named args:  --mode=full, --date=2025-01-15                    │
              │ Non-option:  tenant=ACME  (posicional)                         │
              └───────────────────────────────────────────────────────────────┘
                        │
              ┌─────────┴──────────────────────────────────┐
              │                                            │
              ▼                                            ▼
  TASK_EXECUTION_PARAMS                           Spring Environment
  (todas las filas)                               (properties system)
  ┌────────────────────────┐                      ┌──────────────────────────┐
  │ mode=full              │                      │ mode=full                │
  │ date=2025-01-15        │                      │ date=2025-01-15          │
  │ tenant=ACME            │                      │ (tenant=ACME no disponible│
  └────────────────────────┘                      │  como property Spring)   │
                                                  └──────────────────────────┘
                        │
       ┌────────────────┼──────────────────────┐
       ▼                ▼                       ▼
  ApplicationArguments  CommandLineRunner   @Value / @ConfigurationProperties
  (acceso tipado a      (args[])            (solo named args con --)
   named y non-option)
```

## Ejemplo central

El siguiente ejemplo muestra los tres mecanismos en una misma tarea y cómo se ve el resultado en `TASK_EXECUTION_PARAMS`. Incluye la estrategia de unicidad para relanzamientos en SCDF.

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
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Propiedades tipadas con `@ConfigurationProperties` (`TaskParams.java`):**

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.time.LocalDate;

@Component
@ConfigurationProperties(prefix = "task")
public class TaskParams {

    private String mode = "incremental";    // --task.mode=full
    private LocalDate date = LocalDate.now(); // --task.date=2025-01-15
    private String tenant = "DEFAULT";      // --task.tenant=ACME

    // Getters y setters
    public String getMode() { return mode; }
    public void setMode(String mode) { this.mode = mode; }

    public LocalDate getDate() { return date; }
    public void setDate(LocalDate date) { this.date = date; }

    public String getTenant() { return tenant; }
    public void setTenant(String tenant) { this.tenant = tenant; }
}
```

**Clase principal con los tres mecanismos de acceso (`ParametrizedTask.java`):**

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.task.configuration.EnableTask;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import java.util.List;

@SpringBootApplication
@EnableTask
public class ParametrizedTask {
    public static void main(String[] args) {
        SpringApplication.run(ParametrizedTask.class, args);
    }

    // Bean de tipo CommandLineRunner como alternativa inline
    @Bean
    public CommandLineRunner commandLineRunnerExample() {
        return args -> {
            System.out.println("CommandLineRunner recibió " + args.length + " argumentos:");
            for (String arg : args) {
                System.out.println("  arg: " + arg);
            }
        };
    }
}

// Mecanismo 1: ApplicationArguments — acceso tipado, distingue named de non-option
@Component
class ApplicationArgumentsRunner implements ApplicationRunner {

    private final TaskParams params;

    ApplicationArgumentsRunner(TaskParams params) {
        this.params = params;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("=== ApplicationArguments ===");

        // Named args: --key=value
        List<String> modeValues = args.getOptionValues("task.mode");
        System.out.println("Named --task.mode: " + modeValues);

        // Non-option args: valores sin --
        List<String> nonOption = args.getNonOptionArgs();
        System.out.println("Non-option args: " + nonOption);

        // @ConfigurationProperties — forma más limpia para acceso en lógica de negocio
        System.out.printf("Parámetros tipados: mode=%s, date=%s, tenant=%s%n",
                params.getMode(), params.getDate(), params.getTenant());

        // Lógica de negocio parametrizada
        if ("full".equals(params.getMode())) {
            System.out.println("Modo completo: procesando desde " + params.getDate());
        } else {
            System.out.println("Modo incremental: procesando incremento de " + params.getDate());
        }
    }
}
```

**Configuración (`application.yml`):**

```yaml
spring:
  application:
    name: parametrized-task
  datasource:
    url: jdbc:h2:mem:taskdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  cloud:
    task:
      name: parametrized-task
      initialize-enabled: true
      closecontextEnabled: true

# Valores por defecto para los parámetros (se sobreescriben en lanzamiento)
task:
  mode: incremental
  date: 2025-01-01
  tenant: DEFAULT
```

**Lanzamiento con argumentos (línea de comandos):**

```bash
# Modo completo para un tenant específico
java -jar parametrized-task.jar \
  --task.mode=full \
  --task.date=2025-01-15 \
  --task.tenant=ACME

# Con timestamp único para relanzar en SCDF (workaround de unicidad)
java -jar parametrized-task.jar \
  --task.mode=full \
  --task.date=2025-01-15 \
  --task.tenant=ACME \
  --task.run.timestamp=1705305600000
```

## Tabla de elementos clave

La siguiente tabla compara los tres mecanismos de acceso a argumentos y recoge las propiedades relacionadas con parametrización en SCDF.

| Mecanismo | Tipo de arg | Tipado | Registrado en TASK_EXECUTION_PARAMS | Caso de uso |
|---|---|---|---|---|
| `CommandLineRunner` | `String[]` | No | Sí (todos) | Scripts simples, prototipado rápido |
| `ApplicationArguments` | Named + non-option | Parcial | Sí (todos) | Distinción explícita entre tipos de arg |
| `@Value("${key}")` | Solo named (`--key=val`) | Sí (por conversión) | Sí | Parámetros simples con default |
| `@ConfigurationProperties` | Solo named (`--prefix.key=val`) | Sí (convención) | Sí | Grupos de parámetros relacionados, testeable |
| Arg posicional (non-option) | Sin `--` | No | Sí | Parámetros de posición (poco recomendado) |
| Unicidad SCDF | Restricción | — | — | SCDF impide relanzar con exactamente los mismos args; añadir `--timestamp` |

## Buenas y malas prácticas

**Hacer:**

- Preferir `@ConfigurationProperties` con un prefijo específico (como `task.*`) para todos los parámetros que la tarea necesita. El binding tipado incluye validación con `@Validated`, documentación implícita de la interfaz y tests unitarios sin Spring context.
- Incluir un argumento de `timestamp` o `runId` cuando la tarea se lanza desde SCDF. SCDF impide relanzar una tarea con exactamente los mismos argumentos que una ejecución anterior; el timestamp hace cada lanzamiento único sin cambiar la lógica de negocio.
- Documentar la interfaz de argumentos de la tarea en el mismo fichero que la clase `@ConfigurationProperties`. La tarea es un artefacto que otros equipos lanzan; sin documentación de parámetros, cada lanzamiento requiere inspeccionar el código fuente.
- Validar los argumentos obligatorios al inicio de `run()` y lanzar una excepción con mensaje claro si faltan. Spring Cloud Task registrará el mensaje en `error_message` y el exit code será distinto de cero, haciendo el fallo auditable.

**Evitar:**

- Usar argumentos posicionales (`args[0]`, `args[1]`) en tareas que serán lanzadas desde SCDF o plataformas de orquestación. El orden de los argumentos posicionales no está garantizado en todas las plataformas; los argumentos con nombre son inequívocos.
- Leer argumentos directamente del array de `CommandLineRunner` para lógica de negocio compleja. Sin tipado ni validación, un argumento mal formado lanza una `NumberFormatException` o `NullPointerException` sin mensaje útil en el log de errores de `TASK_EXECUTION`.
- Pasar secretos (contraseñas, tokens API) como argumentos de lanzamiento. Spring Cloud Task registra todos los argumentos en `TASK_EXECUTION_PARAMS` en texto plano; los secretos deben inyectarse como variables de entorno o mediante un gestor de secretos externo.

> **[EXAMEN]** La diferencia entre `ApplicationArguments.getOptionValues(key)` y `@Value("${key}")` es que el primero devuelve todos los valores de un argumento con nombre (puede haber múltiples `--key=v1 --key=v2`) mientras que `@Value` solo resuelve el último valor del Environment.

> **[ADVERTENCIA]** Los argumentos posicionales (non-option args) NO son accesibles mediante `@Value` ni `@ConfigurationProperties`. Solo son accesibles via `ApplicationArguments.getNonOptionArgs()` o el array de `CommandLineRunner`. En entornos SCDF, los non-option args se pasan con sintaxis `key=value` sin doble guión y se almacenan en `TASK_EXECUTION_PARAMS`.

---

← [11.3 Configuración y propiedades spring.cloud.task.*](sc-task-configuracion.md) | [Índice (README.md)](README.md) | [11.5 Integración con Spring Cloud Data Flow](sc-task-scdf-integration.md) →
