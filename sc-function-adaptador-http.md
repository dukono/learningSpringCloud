# 12.3 Spring Cloud Function — Adaptador HTTP

← [12.2 FunctionCatalog](sc-function-catalog.md) | [Índice](README.md) | [12.4 Adaptadores cloud →](sc-function-adaptadores-cloud.md)

---

## Introducción

El adaptador HTTP de Spring Cloud Function expone automáticamente cada función registrada como un endpoint REST sin necesidad de escribir un `@RestController`. Basta con añadir el starter `spring-cloud-starter-function-web` (o `function-webflux` para la variante reactiva) para que SCF publique `POST /{functionName}` y maneje la serialización, negociación de contenido y mapeo de headers HTTP a `MessageHeaders`.

> [CONCEPTO] El adaptador HTTP es un componente de auto-configuración que intercepta peticiones HTTP entrantes, resuelve la función correspondiente en `FunctionCatalog` y delega la invocación. No requiere código de controlador.

> [PREREQUISITO] El adaptador MVC (`function-web`) requiere Spring MVC en el classpath. El adaptador reactivo (`function-webflux`) requiere Spring WebFlux. No se pueden mezclar en la misma aplicación.

## Diagrama del adaptador HTTP

El siguiente diagrama muestra el flujo de una petición HTTP hasta la invocación de la función.

```
Cliente HTTP
  POST /uppercase
  Content-Type: application/json
  Body: "hello world"
       │
       ▼
  FunctionController (auto-configurado por SCF)
       │  lookup("uppercase") en FunctionCatalog
       ▼
  Function<String,String> uppercase
       │  apply("hello world")
       ▼
  "HELLO WORLD"
       │
       ▼
  HTTP 200 OK
  Content-Type: application/json
  Body: "HELLO WORLD"
```

## Ejemplo central

El siguiente ejemplo muestra la configuración mínima para exponer una función vía HTTP con el adaptador MVC, incluyendo el mapeo de headers HTTP y la habilitación del modo debug.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-function-web</artifactId>
</dependency>
```

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@SpringBootApplication
public class HttpAdapterApplication {

    public static void main(String[] args) {
        SpringApplication.run(HttpAdapterApplication.class, args);
    }

    /**
     * Función simple: accesible en POST /uppercase
     */
    @Bean
    public Function<String, String> uppercase() {
        return value -> value.toUpperCase();
    }

    /**
     * Función con acceso a headers HTTP: los headers de la petición
     * se mapean automáticamente a MessageHeaders cuando se usa Message<T>.
     */
    @Bean
    public Function<Message<String>, Message<String>> enrichedUppercase() {
        return message -> {
            String originalHeader = (String) message.getHeaders().get("X-Request-Id");
            String result = message.getPayload().toUpperCase();
            return MessageBuilder.withPayload(result)
                    .setHeader("X-Processed-By", "enrichedUppercase")
                    .setHeader("X-Request-Id", originalHeader)
                    .build();
        };
    }
}
```

Configuración en `application.yml`:

```yaml
spring:
  cloud:
    function:
      definition: uppercase
      web:
        export:
          debug: true   # habilita log de headers y body para depuración
```

Ejemplo de invocación:

```bash
# Invocar función simple
curl -X POST http://localhost:8080/uppercase \
  -H "Content-Type: text/plain" \
  -d "hello world"
# Respuesta: "HELLO WORLD"

# Invocar función con composición (pipe)
# spring.cloud.function.definition: uppercase|trim
curl -X POST "http://localhost:8080/uppercase%7Ctrim" \
  -H "Content-Type: text/plain" \
  -d "  hello world  "
# Respuesta: "HELLO WORLD"
```

> [ADVERTENCIA] La URL de una función compuesta usa el separador `|` codificado como `%7C` en la URL. Asegurarse de codificar correctamente el carácter pipe en los clientes HTTP.

## Tabla de elementos clave

La siguiente tabla compara los dos starters del adaptador HTTP.

| Elemento | `function-web` (MVC) | `function-webflux` (Reactivo) |
|---|---|---|
| Stack | Spring MVC (servlet) | Spring WebFlux (netty) |
| Tipos de función | `Function<T,R>` imperativa | `Function<Flux<T>,Flux<R>>` reactiva |
| Thread model | Bloqueante | No bloqueante |
| Starter | `spring-cloud-starter-function-web` | `spring-cloud-starter-function-webflux` |
| Debug | `web.export.debug=true` | `web.export.debug=true` |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `spring.cloud.function.web.export.debug=true` solo en entornos de desarrollo/staging para evitar el overhead de logging.
- Declarar `spring.cloud.function.definition` explícitamente para evitar ambigüedad con múltiples funciones.
- Usar `Function<Message<String>, Message<String>>` cuando se necesiten acceder o propagar headers HTTP.
- Usar `function-webflux` para aplicaciones que ya usen WebFlux y funciones con tipos reactivos.

**Malas prácticas:**
- Incluir tanto `function-web` como `function-webflux` en el mismo classpath — generará conflicto de auto-configuración.
- Asumir que todos los headers HTTP se propagan automáticamente sin usar `Message<T>`.
- Exponer funciones internas (lógica de negocio sensible) sin protección de seguridad vía Spring Security.

## Verificación y práctica

> [EXAMEN] ¿Qué starter activa el endpoint HTTP automático para funciones en Spring Cloud Function con el stack MVC?

> [EXAMEN] ¿Cuál es la convención de URL para invocar una función llamada `processOrder` mediante el adaptador HTTP?

> [EXAMEN] ¿Cómo accede una función a los headers de la petición HTTP cuando usa el adaptador HTTP de SCF?

> [EXAMEN] ¿Qué propiedad habilita el modo debug del adaptador HTTP para registrar headers y bodies de petición y respuesta?

> [EXAMEN] ¿Cuándo es necesario usar `spring-cloud-starter-function-webflux` en lugar de `spring-cloud-starter-function-web`?

---

← [12.2 FunctionCatalog](sc-function-catalog.md) | [Índice](README.md) | [12.4 Adaptadores cloud →](sc-function-adaptadores-cloud.md)
