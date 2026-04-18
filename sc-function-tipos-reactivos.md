# 12.9 Spring Cloud Function — Tipos reactivos (Flux/Mono)

← [12.8 Integración con Spring Cloud Stream](sc-function-integracion-stream.md) | [Índice](README.md) | [12.10 Testing y packaging →](sc-function-testing.md)

---

## Introducción

Spring Cloud Function soporta de forma nativa los tipos reactivos de Project Reactor: `Flux<T>` y `Mono<T>`. Cuando una función declara `Function<Flux<I>, Flux<O>>`, el ciclo de vida de la invocación cambia radicalmente respecto a la función imperativa: en lugar de procesar un elemento por llamada, SCF pasa el stream completo a la función y deja que Reactor gestione la suscripción, el backpressure y la ejecución asíncrona. El adaptador HTTP reactivo (`function-webflux`) es necesario para exponer estas funciones como endpoints.

> [CONCEPTO] Una función imperativa `Function<String, String>` se invoca una vez por elemento. Una función reactiva `Function<Flux<String>, Flux<String>>` recibe un stream infinito potencial y devuelve un stream transformado; la ejecución real ocurre cuando el subscriber se suscribe.

> [CONCEPTO] `Mono<T>` modela cero o un elemento (respuesta única). `Flux<T>` modela cero o N elementos (stream). En el contexto de SCF, `Function<Mono<I>, Mono<O>>` es equivalente a la función imperativa en semántica, pero con composición reactiva.

> [PREREQUISITO] Las funciones reactivas requieren `spring-cloud-starter-function-webflux` para el adaptador HTTP. Con el adaptador MVC (`function-web`) se puede usar `Mono<T>` pero no `Flux<T>` en modo streaming.

## Diagrama: imperativa vs reactiva

El siguiente diagrama compara el ciclo de vida de una función imperativa frente a una reactiva.

```
Función imperativa Function<String,String>:
  Petición 1 → apply("a") → "A"   (invocación 1)
  Petición 2 → apply("b") → "B"   (invocación 2)
  Petición 3 → apply("c") → "C"   (invocación 3)
  Cada petición HTTP = una invocación = un resultado

Función reactiva Function<Flux<String>,Flux<String>>:
  SCF construye Flux["a","b","c","d",...]
         │
         ▼
  Function recibe el Flux completo
  Aplica operadores Reactor: map, filter, flatMap...
         │
         ▼
  Devuelve Flux transformado
         │
         ▼
  SCF suscribe y emite los elementos al cliente (SSE/streaming)
  Backpressure gestionado automáticamente
```

## Ejemplo central

El siguiente ejemplo muestra una función reactiva con `Flux`, una función con `Mono` y la diferencia en la configuración del adaptador.

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.function.Function;
import java.util.function.Supplier;

@SpringBootApplication
public class ReactiveApplication {

    public static void main(String[] args) {
        SpringApplication.run(ReactiveApplication.class, args);
    }

    /**
     * Función imperativa: un elemento por invocación.
     * Accesible en POST /uppercase (con function-web o function-webflux)
     */
    @Bean
    public Function<String, String> uppercase() {
        return String::toUpperCase;
    }

    /**
     * Función reactiva con Mono: semántica 1:1 pero con composición reactiva.
     * Accesible en POST /processAsync (requiere function-webflux)
     */
    @Bean
    public Function<Mono<String>, Mono<String>> processAsync() {
        return inputMono -> inputMono
                .map(String::toUpperCase)
                .doOnNext(result -> System.out.println("Processed: " + result));
    }

    /**
     * Función reactiva con Flux: N elementos de entrada → N transformados de salida.
     * Accesible en POST /transformStream (requiere function-webflux)
     * Útil para Server-Sent Events o procesamiento de lotes.
     */
    @Bean
    public Function<Flux<String>, Flux<String>> transformStream() {
        return inputFlux -> inputFlux
                .filter(s -> !s.isBlank())
                .map(String::toUpperCase)
                .map(s -> "[" + s + "]");
    }

    /**
     * Supplier reactivo: genera un stream infinito de eventos.
     * Integrado con Spring Cloud Stream para publicar en un topic.
     */
    @Bean
    public Supplier<Flux<String>> eventStream() {
        return () -> Flux.interval(Duration.ofSeconds(1))
                .map(tick -> "event-" + tick);
    }
}
```

Configuración `application.yml` para el adaptador WebFlux:

```yaml
spring:
  cloud:
    function:
      definition: transformStream
```

Dependencia en `pom.xml`:

```xml
<!-- Adaptador reactivo: requerido para Function<Flux<T>,Flux<R>> -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-function-webflux</artifactId>
</dependency>
```

Invocación del endpoint streaming:

```bash
# Enviar un JSON array como input (el adaptador lo convierte a Flux)
curl -X POST http://localhost:8080/transformStream \
  -H "Content-Type: application/json" \
  -d '["hello", "world", "spring"]'
# Respuesta: ["[HELLO]","[WORLD]","[SPRING]"]
```

> [ADVERTENCIA] No bloquear dentro de una función reactiva usando `.block()` en la lambda — esto provoca `BlockingOperationError` en el scheduler reactivo de Netty y puede causar deadlocks. Toda la lógica dentro de `Function<Flux<I>, Flux<O>>` debe ser no bloqueante.

## Tabla de elementos clave

La siguiente tabla compara las variantes de función según el tipo de dato.

| Firma de función | Semántica | Adaptador HTTP | Caso de uso |
|---|---|---|---|
| `Function<String, String>` | 1 entrada → 1 salida | Web o WebFlux | Transformación simple |
| `Function<Mono<I>, Mono<O>>` | 1 async → 1 async | WebFlux | Operaciones async (DB reactiva) |
| `Function<Flux<I>, Flux<O>>` | N → N stream | WebFlux | Procesamiento de streams, SSE |
| `Supplier<Flux<O>>` | Sin entrada → N | WebFlux / Stream | Generación de eventos continuos |
| `Consumer<Flux<I>>` | N → sin salida | Stream | Sink de stream (persistencia) |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `Function<Flux<I>, Flux<O>>` para procesamiento de streams donde el backpressure sea importante.
- Usar `Function<Mono<I>, Mono<O>>` cuando la operación es asíncrona pero con semántica 1:1 (por ejemplo, llamada a base de datos reactiva).
- Preferir la función imperativa `Function<T, R>` cuando no hay beneficio real del modelo reactivo — es más simple y fácil de testear.
- Usar `function-webflux` consistentemente cuando cualquier función en el contexto use tipos reactivos.

**Malas prácticas:**
- Llamar a `.block()` dentro de una función reactiva — causa deadlock en el scheduler de Netty.
- Mezclar `function-web` (MVC) con funciones `Flux` — el adaptador MVC no soporta streaming reactivo.
- Usar `Flux` para transformaciones simples de un solo elemento cuando `String` o `Mono` son suficientes.

## Verificación y práctica

> [EXAMEN] ¿Qué diferencia hay en el ciclo de vida de invocación entre `Function<String, String>` y `Function<Flux<String>, Flux<String>>`?

> [EXAMEN] ¿Qué starter es necesario para exponer una función `Function<Flux<String>, Flux<String>>` como endpoint HTTP?

> [EXAMEN] ¿Por qué no se debe llamar a `.block()` dentro de una función reactiva `Function<Flux<I>, Flux<O>>`?

> [EXAMEN] ¿Cuándo es preferible usar `Function<Mono<I>, Mono<O>>` frente a `Function<Flux<I>, Flux<O>>`?

---

← [12.8 Integración con Spring Cloud Stream](sc-function-integracion-stream.md) | [Índice](README.md) | [12.10 Testing y packaging →](sc-function-testing.md)
