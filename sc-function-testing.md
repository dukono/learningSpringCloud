# 12.10 Spring Cloud Function — Testing y packaging

← [12.9 Tipos reactivos](sc-function-tipos-reactivos.md) | [Índice](README.md) | [13.1 Patrones: Descomposición por dominio →](sc-patrones-descomposicion-dominio.md)

---

## Introducción

Spring Cloud Function facilita tres niveles de testing: el test unitario del bean funcional como POJO puro sin contexto Spring (el más rápido), el test de integración con contexto mínimo usando `FunctionalSpringBootTest` para verificar el registro en `FunctionCatalog`, y el test de extremo a extremo del endpoint HTTP con `WebTestClient`. Para el packaging, SCF soporta thin JAR, fat JAR y compilación nativa con GraalVM a través del soporte Spring AOT.

> [CONCEPTO] Un bean `Function<I,O>` es un POJO Java estándar. Se puede instanciar directamente en un test unitario sin necesidad de Spring, simplemente llamando a `new MyFunction().apply(input)`. Esta es la forma más rápida de probar la lógica de negocio.

> [CONCEPTO] `FunctionalSpringBootTest` es una anotación de SCF que levanta solo el contexto funcional mínimo (sin servidor HTTP embebido, sin autoconfiguración completa), adecuada para verificar el registro en `FunctionCatalog` y la composición de funciones.

> [PREREQUISITO] `FunctionalSpringBootTest` requiere la dependencia `spring-cloud-function-context` en scope `test`.

## Diagrama de estrategia de testing

El siguiente diagrama muestra los tres niveles de testing y su alcance.

```
Nivel 1 — Test unitario (más rápido):
  new UppercaseFunction().apply("hello") → "HELLO"
  Sin Spring, sin contexto, sin dependencias
  ✓ Verifica lógica de negocio pura

Nivel 2 — Test de integración con FunctionalSpringBootTest:
  @FunctionalSpringBootTest
  @Autowired FunctionCatalog catalog
  catalog.lookup("uppercase").apply("hello") → "HELLO"
  Contexto Spring mínimo (solo SCF, sin HTTP)
  ✓ Verifica registro, composición y tipos genéricos

Nivel 3 — Test de extremo a extremo (más lento):
  @SpringBootTest(webEnvironment = RANDOM_PORT)
  WebTestClient.post().uri("/uppercase").bodyValue("hello")
  Contexto completo con servidor HTTP
  ✓ Verifica la integración HTTP completa
```

## Ejemplo central

El siguiente ejemplo muestra los tres niveles de testing para el mismo bean funcional.

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.Function;

@Configuration
public class FunctionConfig {

    @Bean
    public Function<String, String> uppercase() {
        return String::toUpperCase;
    }

    @Bean
    public Function<String, String> trim() {
        return String::trim;
    }
}
```

Test unitario puro (sin Spring):

```java
package com.example.demo;

import org.junit.jupiter.api.Test;

import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;

class UppercaseFunctionTest {

    private final Function<String, String> uppercase = String::toUpperCase;

    @Test
    void shouldConvertToUppercase() {
        String result = uppercase.apply("hello world");
        assertThat(result).isEqualTo("HELLO WORLD");
    }

    @Test
    void shouldHandleEmptyString() {
        String result = uppercase.apply("");
        assertThat(result).isEqualTo("");
    }
}
```

Test de integración con `FunctionalSpringBootTest`:

```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.function.context.FunctionCatalog;
import org.springframework.cloud.function.context.test.FunctionalSpringBootTest;

import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;

@FunctionalSpringBootTest
class FunctionCatalogIntegrationTest {

    @Autowired
    private FunctionCatalog catalog;

    @Test
    void shouldRegisterUppercaseFunction() {
        Function<String, String> uppercase = catalog.lookup("uppercase");
        assertThat(uppercase).isNotNull();
        assertThat(uppercase.apply("hello")).isEqualTo("HELLO");
    }

    @Test
    void shouldLookupComposedFunction() {
        // Verifica que la composición "uppercase|trim" está registrada
        // cuando spring.cloud.function.definition: uppercase|trim
        Function<String, String> composed = catalog.lookup("uppercase");
        Function<String, String> trim = catalog.lookup("trim");
        assertThat(composed.andThen(trim).apply("  hello  ")).isEqualTo("HELLO");
    }

    @Test
    void shouldReturnNullForUnknownFunction() {
        Function<String, String> unknown = catalog.lookup("nonExistent");
        assertThat(unknown).isNull();
    }
}
```

Test de extremo a extremo con `WebTestClient`:

```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class HttpAdapterE2ETest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldExposeUppercaseEndpoint() {
        webTestClient.post()
                .uri("/uppercase")
                .bodyValue("hello world")
                .exchange()
                .expectStatus().isOk()
                .expectBody(String.class)
                .isEqualTo("HELLO WORLD");
    }
}
```

> [ADVERTENCIA] `@FunctionalSpringBootTest` no levanta el servidor HTTP embebido. Para testear el adaptador HTTP se necesita `@SpringBootTest(webEnvironment = RANDOM_PORT)` con `WebTestClient` o `MockMvc`.

## Tabla de elementos clave

La siguiente tabla resume las opciones de packaging de SCF.

| Formato | Descripción | Cuándo usar |
|---|---|---|
| Fat JAR (`spring-boot-maven-plugin`) | JAR auto-contenido con Tomcat/Netty embebido | Despliegue como microservicio en contenedor |
| Shaded JAR (`maven-shade-plugin`) | JAR plano sin classloader de Spring Boot | AWS Lambda (obligatorio) |
| Thin JAR | JAR sin dependencias (se descargan en runtime) | Reduce tamaño de imagen Docker |
| GraalVM native image | Ejecutable nativo compilado con AOT | Cold start mínimo en serverless |

## Buenas y malas prácticas

**Buenas prácticas:**
- Empezar con tests unitarios puros para la lógica de la función — son los más rápidos y fáciles de mantener.
- Usar `@FunctionalSpringBootTest` para verificar que el bean se registra correctamente y que la composición funciona.
- Reservar los tests `@SpringBootTest` con servidor HTTP para la integración completa del adaptador.
- Para AWS Lambda, validar el empaquetado con el shaded JAR en un entorno SAM local antes del despliegue.

**Malas prácticas:**
- Depender exclusivamente de tests `@SpringBootTest` para probar la lógica de negocio — son lentos e innecesariamente pesados.
- Omitir la validación del lookup en `FunctionCatalog` con valor null — puede causar `NullPointerException` en producción.
- Usar el fat JAR de Spring Boot como artefacto de AWS Lambda — causa error de classloader.

## Verificación y práctica

> [EXAMEN] ¿Cómo se prueba un bean `Function<String,String>` sin levantar el contexto Spring y qué ventaja ofrece este enfoque?

> [EXAMEN] ¿Qué anotación levanta el contexto funcional mínimo de Spring Cloud Function para tests de integración sin servidor HTTP?

> [EXAMEN] ¿Cómo se verifica en un test de integración que un bean funcional está correctamente registrado en `FunctionCatalog`?

> [EXAMEN] ¿Qué tipo de JAR se requiere para desplegar una aplicación Spring Cloud Function en AWS Lambda y por qué no sirve el fat JAR estándar de Spring Boot?

> [EXAMEN] ¿Qué anotación de test se usa para verificar el adaptador HTTP completo de Spring Cloud Function con `WebTestClient`?

---

← [12.9 Tipos reactivos](sc-function-tipos-reactivos.md) | [Índice](README.md) | [13.1 Patrones: Descomposición por dominio →](sc-patrones-descomposicion-dominio.md)
