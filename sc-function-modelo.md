# 12.1 Modelo de programación funcional: Function, Consumer, Supplier y FunctionCatalog

← [11.9 Testing de Spring Cloud Task](sc-task-testing.md) | [Índice (README.md)](README.md) | [12.2 Configuración de funciones y exposición HTTP](sc-function-config.md) →

---

## Introducción

El problema que resuelve Spring Cloud Function es la dispersión del código de negocio en controladores HTTP, listeners de mensajería y handlers de funciones serverless. Cuando la misma transformación de datos se escribe tres veces —una para el endpoint REST, otra para el consumer Kafka y otra para el handler Lambda— cualquier corrección de lógica requiere tocar múltiples capas. Spring Cloud Function existe para separar el código de negocio puro de la infraestructura de invocación: la función se escribe una vez como un bean Java estándar y el framework se encarga de exponerla como REST, como consumer de mensajes o como función serverless sin cambiar una línea del modelo.

El modelo se necesita en cualquier escenario donde la misma lógica deba ejecutarse en más de un contexto de invocación, o cuando se quiere garantizar que el código de negocio sea testeable como POJO independiente del transporte.

> **[PREREQUISITO]** Este fichero asume conocimiento de Spring Boot 4.0.x (autoconfiguración, beans, starters) y Java 21 lambdas. Los tipos reactivos `Flux<T>` y `Mono<T>` de Project Reactor se mencionan en la sección de variantes; su manejo avanzado se documenta en el módulo Spring Cloud Stream.

## Representación visual

El modelo de programación funcional de Spring Cloud Function se articula alrededor de tres contratos Java estándar —`Function`, `Consumer` y `Supplier`— que el contexto Spring registra automáticamente y expone a través de `FunctionCatalog`. El siguiente diagrama muestra cómo los beans funcionales fluyen desde el registro hasta los adaptadores de transporte.

```
  Beans Spring
  ┌─────────────────────────────────────────────┐
  │  @Bean Function<String,String>  uppercase   │
  │  @Bean Consumer<String>         logger       │
  │  @Bean Supplier<String>         greeter      │
  └───────────────────┬─────────────────────────┘
                      │  discovery automático
                      ▼
             ┌─────────────────┐
             │  FunctionCatalog│  lookup("uppercase")
             │  (registro SCF) │  lookup("logger")
             └────────┬────────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
     HTTP REST    SC Stream    AWS Lambda
   /uppercase    Kafka topic   FunctionInvoker
```

La composición de funciones mediante el operador `|` (pipe) produce una nueva entrada en el catálogo sin necesidad de definir un nuevo bean:

```
  FunctionCatalog.lookup("uppercase|logger")
  → Function<String,Void> compuesta en runtime
```

## Ejemplo central

El ejemplo muestra el ciclo completo: definición de los tres contratos, composición y consulta al catálogo desde un `CommandLineRunner`.

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
    <artifactId>spring-cloud-function-context</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      definition: uppercase
```

### Código Java completo

```java
package com.example.scfunction;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.context.FunctionCatalog;
import org.springframework.context.annotation.Bean;

import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

@SpringBootApplication
public class FunctionModelApplication implements CommandLineRunner {

    private final FunctionCatalog catalog;

    public FunctionModelApplication(FunctionCatalog catalog) {
        this.catalog = catalog;
    }

    public static void main(String[] args) {
        SpringApplication.run(FunctionModelApplication.class, args);
    }

    // Función de transformación: String → String
    @Bean
    public Function<String, String> uppercase() {
        return value -> value.toUpperCase();
    }

    // Función de transformación encadenada: String → String
    @Bean
    public Function<String, String> trim() {
        return String::trim;
    }

    // Consumer: recibe String, no devuelve nada
    @Bean
    public Consumer<String> logger() {
        return value -> System.out.println("[LOG] " + value);
    }

    // Supplier: no recibe nada, produce String
    @Bean
    public Supplier<String> greeter() {
        return () -> "Hello, Spring Cloud Function!";
    }

    @Override
    public void run(String... args) {
        // Lookup de función simple
        Function<String, String> fn = catalog.lookup(Function.class, "uppercase");
        System.out.println("uppercase: " + fn.apply("hello"));

        // Composición mediante pipe en runtime
        Function<String, String> composed = catalog.lookup(Function.class, "trim|uppercase");
        System.out.println("trim|uppercase: " + composed.apply("  spaces  "));

        // Supplier invocado programáticamente
        Supplier<String> sup = catalog.lookup(Supplier.class, "greeter");
        System.out.println("greeter: " + sup.get());

        // Consumer invocado programáticamente
        Consumer<String> cons = catalog.lookup(Consumer.class, "logger");
        cons.accept("mensaje de prueba");
    }
}
```

La salida esperada al arrancar:
```
uppercase: HELLO
trim|uppercase: SPACES
greeter: Hello, Spring Cloud Function!
[LOG] mensaje de prueba
```

> **[CONCEPTO]** `FunctionCatalog` es el registro central de Spring Cloud Function. Resuelve beans por nombre (con `lookup`) y soporta la composición mediante el operador `|` sin requerir la definición de un nuevo bean. Si no se llama a `lookup` explícitamente, el framework resuelve la función activa mediante `spring.cloud.function.definition`.

## Tabla de elementos clave

La siguiente tabla recoge los contratos y conceptos del modelo que un profesional senior debe dominar para entrevistas y evaluaciones técnicas.

| Elemento | Tipo / Firma | Contrato | Descripción |
|---|---|---|---|
| `Function<T,R>` | `java.util.function.Function` | Transformación T → R | Contrato principal de procesamiento con entrada y salida |
| `Consumer<T>` | `java.util.function.Consumer` | Consumo T → void | Procesamiento sin retorno; típico para side-effects (logging, DB) |
| `Supplier<T>` | `java.util.function.Supplier` | Producción void → T | Generación de datos sin entrada; típico para polling |
| `FunctionCatalog` | Interface SCF | Registro central | Lookup por nombre y composición de funciones en runtime |
| Operador pipe `\|` | Sintaxis de definición | Composición | `"uppercase\|trim"` compone dos funciones en una sin nuevo bean |
| `Flux<T>` entrada/salida | `reactor.core.publisher.Flux` | Reactivo N-elementos | `Function<Flux<T>,Flux<R>>` para procesamiento reactivo streaming |
| `Mono<T>` entrada/salida | `reactor.core.publisher.Mono` | Reactivo 0-1 elemento | `Function<Mono<T>,Mono<R>>` para procesamiento reactivo singular |
| Discovery automático | Mecanismo de registro | Auto-registro | Beans `Function`/`Consumer`/`Supplier` sin `@FunctionScan` explícito se registran automáticamente |
| `spring.cloud.function.definition` | Propiedad String | Selección | Define qué función está activa; obligatorio cuando hay más de un bean funcional |
| `FunctionRegistration<T>` | Clase SCF | Registro explícito | Registro programático con tipo y nombre cuando el discovery automático falla (ej. tipos genéricos complejos) |

> **[ADVERTENCIA]** Spring Cloud 2025.1.1 (Oakwood) con Spring Boot 4.0.x puede introducir cambios en la API de `FunctionCatalog`. Verificar en [docs.spring.io/spring-cloud-function](https://docs.spring.io/spring-cloud-function) antes de asumir compatibilidad total con versiones anteriores.

## Buenas y malas prácticas

**Hacer:**
- Definir funciones como beans `@Bean` en clases `@Configuration` para que el discovery automático las registre sin configuración extra.
- Usar `spring.cloud.function.definition` siempre que existan dos o más beans funcionales en el contexto; la ambigüedad produce `IllegalStateException` en arranque que bloquea producción.
- Implementar lógica de negocio pura en la lambda sin dependencias de infraestructura (HttpServletRequest, MessageHeaders) para garantizar portabilidad entre adaptadores.
- Usar `Function<Flux<T>,Flux<R>>` cuando el procesamiento sea inherentemente reactivo o cuando se integre con Spring Cloud Stream para evitar bloqueos de hilo.
- Nombrar los beans con nombres descriptivos del dominio (`orderProcessor`, `invoiceValidator`) en lugar de nombres técnicos (`myFunction`, `fn1`) porque el nombre del bean determina el path HTTP y el nombre del binding de mensajería.

**Evitar:**
- Evitar acceder a `FunctionCatalog` desde dentro de otra función registrada: produce dependencia circular en el contexto y puede causar inicialización incompleta del catálogo.
- Evitar usar tipos raw sin genéricos (`Function` sin `<T,R>`): el framework pierde la capacidad de inferir tipos para la serialización, lo que produce `ClassCastException` en runtime durante la conversión de payloads.
- Evitar combinar lógica de negocio con manejo de `Message<T>` salvo cuando sea estrictamente necesario acceder a headers: añade acoplamiento al transporte y rompe la portabilidad.
- Evitar registrar el mismo bean funcional con múltiples anotaciones de framework (ej. `@KafkaListener` y como `@Bean Function`): se producen dos consumidores independientes que pueden provocar procesamiento duplicado.

## Comparación: tipos de contrato funcional

Los tres contratos sirven casos de uso distintos. Elegir el incorrecto provoca que el framework no pueda deducir el tipo de adaptador adecuado.

| Contrato | Firma | Caso de uso típico | Adaptador HTTP | Adaptador Stream |
|---|---|---|---|---|
| `Function<T,R>` | `T → R` | Transformación de datos, enriquecimiento | POST con body → respuesta | Input channel → Output channel |
| `Consumer<T>` | `T → void` | Persistencia, logging, notificación | POST sin respuesta (204) | Input channel solo (sink) |
| `Supplier<T>` | `void → T` | Generación de datos, polling | GET sin body → respuesta | Output channel solo (source) |
| `Function<Flux<T>,Flux<R>>` | `Flux<T> → Flux<R>` | Procesamiento reactivo streaming | POST streaming | Input/Output reactivo |
| `Function<Mono<T>,Mono<R>>` | `Mono<T> → Mono<R>` | Procesamiento reactivo singular | POST reactivo | Input/Output Mono |

---

← [11.9 Testing de Spring Cloud Task](sc-task-testing.md) | [Índice (README.md)](README.md) | [12.2 Configuración de funciones y exposición HTTP](sc-function-config.md) →
