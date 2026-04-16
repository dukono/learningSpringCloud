# 12.10 Testing y verificación de Spring Cloud Function

← [12.9 Observabilidad y operación de funciones expuestas](sc-function-operations.md) | [Índice (README.md)](README.md) | [13.1 CQRS con Spring Cloud Stream y Gateway](sc-patrones-cqrs.md) →

---

## Introducción

El problema de testing en Spring Cloud Function es la tentación de testear directamente el adaptador (HTTP, Lambda, Stream) en lugar de la función de negocio, lo que produce tests lentos, frágiles y dependientes de infraestructura. La fortaleza del modelo funcional de Spring Cloud Function es que la función es un POJO Java estándar: `Function<Order,Invoice>` puede testarse sin Spring, sin HTTP y sin Kafka. Este fichero establece la estrategia de testing por nivel: unitario puro para la lógica, integración con `FunctionCatalog` para la configuración, e integración HTTP con `MockMvc` o `WebTestClient` para el adaptador web.

> **[PREREQUISITO]** Requiere `sc-function-modelo.md` (12.1) y `sc-function-config.md` (12.2). JUnit 5 es provisto por `spring-boot-starter-test`; ver la [documentación de JUnit 5](https://junit.org/junit5/docs/current/user-guide/) para el framework de testing.

## Representación visual

La pirámide de testing de Spring Cloud Function tiene cuatro niveles con distintos costes y cobertura.

```
              ┌──────────────────────────┐
              │ Test HTTP (MockMvc /      │  Lento, tests del adaptador web
              │ WebTestClient)            │  Verifica: serialización, path, status HTTP
              └──────────────────────────┘
             ┌────────────────────────────┐
             │ Test de integración        │  Contexto Spring reducido
             │ FunctionCatalog            │  Verifica: composición, lookup, binding
             └────────────────────────────┘
            ┌──────────────────────────────┐
            │ Test de composición           │  Sin contexto Spring
            │ Function.andThen()            │  Verifica: cadenas de transformación
            └──────────────────────────────┘
           ┌────────────────────────────────┐
           │ Test unitario puro              │  Más rápido, más aislado
           │ Function<T,R> como POJO         │  Verifica: lógica de negocio
           └────────────────────────────────┘
```

## Ejemplo central

El ejemplo muestra los cuatro niveles de test para una función `processOrder` completa.

### Dependencias Maven

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### application.yml (src/main/resources)

```yaml
spring:
  cloud:
    function:
      definition: processOrder
```

### application-test.yml (src/test/resources)

```yaml
spring:
  cloud:
    function:
      definition: processOrder
```

### Código Java completo — Producción

```java
package com.example.scfunction.testing;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.util.function.Function;

@SpringBootApplication
public class FunctionTestingApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionTestingApplication.class, args);
    }

    public record Order(Long id, String item, double price) {}
    public record Invoice(Long orderId, String description, double total) {}

    @Bean
    public Function<Order, Invoice> processOrder() {
        return order -> {
            if (order.price() <= 0) {
                throw new IllegalArgumentException("El precio debe ser positivo");
            }
            return new Invoice(
                order.id(),
                "Factura: " + order.item(),
                order.price() * 1.21
            );
        };
    }

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

### Código Java completo — Tests

```java
package com.example.scfunction.testing;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.function.context.FunctionCatalog;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

// ============================================================
// NIVEL 1 — Test unitario puro (sin contexto Spring)
// ============================================================
class ProcessOrderFunctionTest {

    // Instancia directa del bean funcional como POJO — sin Spring
    private final Function<FunctionTestingApplication.Order, FunctionTestingApplication.Invoice>
        processOrder = new FunctionTestingApplication().processOrder();

    @Test
    void deberiaCalcularIVACorrectamente() {
        var order = new FunctionTestingApplication.Order(1L, "libro", 10.0);

        var invoice = processOrder.apply(order);

        assertThat(invoice.orderId()).isEqualTo(1L);
        assertThat(invoice.total()).isEqualTo(12.1);
        assertThat(invoice.description()).contains("libro");
    }

    @Test
    void deberiaLanzarExcepcionConPrecioNegativo() {
        var order = new FunctionTestingApplication.Order(2L, "item", -5.0);

        assertThatThrownBy(() -> processOrder.apply(order))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("positivo");
    }
}

// ============================================================
// NIVEL 2 — Test de composición (sin contexto Spring)
// ============================================================
class FunctionCompositionTest {

    private final Function<String, String> uppercase = new FunctionTestingApplication().uppercase();
    private final Function<String, String> trim = new FunctionTestingApplication().trim();

    @Test
    void deberiaComponerTrimConUppercase() {
        // Composición manual con andThen (equivalente al pipe | de SCF)
        Function<String, String> composed = trim.andThen(uppercase);

        String result = composed.apply("  hello world  ");

        assertThat(result).isEqualTo("HELLO WORLD");
    }

    @Test
    void deberiaAplicarUppercaseSoloSiNecesario() {
        Function<String, String> composed = trim.andThen(uppercase);

        assertThat(composed.apply("already")).isEqualTo("ALREADY");
        assertThat(composed.apply("  spaces  ")).isEqualTo("SPACES");
    }
}

// ============================================================
// NIVEL 3 — Test de integración con FunctionCatalog
// ============================================================
@SpringBootTest(classes = FunctionTestingApplication.class)
class FunctionCatalogIntegrationTest {

    @Autowired
    private FunctionCatalog catalog;

    @Test
    void deberiaRegistrarFuncionProcessOrderEnCatalogo() {
        Function<FunctionTestingApplication.Order, FunctionTestingApplication.Invoice> fn =
            catalog.lookup(Function.class, "processOrder");

        assertThat(fn).isNotNull();

        var order = new FunctionTestingApplication.Order(1L, "prueba", 100.0);
        var invoice = fn.apply(order);

        assertThat(invoice.total()).isEqualTo(121.0);
    }

    @Test
    void deberiaRessolverComposicionTrimUppercaseDesdeCalatogo() {
        Function<String, String> composed = catalog.lookup(Function.class, "trim|uppercase");

        assertThat(composed).isNotNull();
        assertThat(composed.apply("  test  ")).isEqualTo("TEST");
    }
}

// ============================================================
// NIVEL 4 — Test HTTP con MockMvc (contexto web completo)
// ============================================================
@SpringBootTest(classes = FunctionTestingApplication.class,
    webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
class FunctionHttpTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void deberiaResponder200ConFacturaAlInvocarProcessOrderViaHTTP() throws Exception {
        var order = new FunctionTestingApplication.Order(1L, "libro", 10.0);
        String requestBody = objectMapper.writeValueAsString(order);

        mockMvc.perform(
                post("/processOrder")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestBody)
            )
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.orderId").value(1))
            .andExpect(jsonPath("$.total").value(12.1))
            .andExpect(jsonPath("$.description").value("Factura: libro"));
    }

    @Test
    void deberiaResponder500ConPrecioNegativoViaHTTP() throws Exception {
        var order = new FunctionTestingApplication.Order(2L, "item", -5.0);
        String requestBody = objectMapper.writeValueAsString(order);

        mockMvc.perform(
                post("/processOrder")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestBody)
            )
            .andExpect(status().is5xxServerError());
    }
}
```

> **[EXAMEN]** La ventaja de testear `Function<T,R>` como POJO puro (Nivel 1) es que el test no arranca el contexto Spring, no abre puertos y ejecuta en milisegundos. Un test de integración `@SpringBootTest` puede tardar 3-8 segundos en arrancar el contexto. En un proyecto con 200 funciones, la diferencia entre 200 tests unitarios puros (~2 segundos total) y 200 tests `@SpringBootTest` (~600 segundos total) es determinante para el feedback loop del desarrollador.

## Tabla de elementos clave

Los elementos de testing específicos de Spring Cloud Function se recogen en la siguiente tabla.

| Elemento | Nivel | Descripción |
|---|---|---|
| Test unitario puro | Nivel 1 | `new MyApp().myFunction().apply(input)` — sin Spring, sin anotaciones |
| `Function.andThen()` | Nivel 2 | Composición manual para testear cadenas de transformación sin catálogo |
| `@SpringBootTest` | Nivel 3 y 4 | Carga el `ApplicationContext` completo; necesario para `FunctionCatalog` |
| `FunctionCatalog.lookup()` | Nivel 3 | Verifica que la función está registrada con el nombre y tipo correctos |
| `@AutoConfigureMockMvc` | Nivel 4 | Activa `MockMvc` para tests HTTP sin servidor real |
| `MockMvc.perform(post(...))` | Nivel 4 | Invoca el endpoint HTTP de la función vía MockMvc |
| `WebTestClient` | Nivel 4 (reactivo) | Alternativa reactiva a `MockMvc` para `spring-cloud-function-web` con WebFlux |
| `SpringBootTest.WebEnvironment.MOCK` | Configuración | Contexto web simulado; sin puerto abierto, más rápido que `RANDOM_PORT` |
| `SpringBootTest.WebEnvironment.RANDOM_PORT` | Configuración | Servidor real en puerto aleatorio; necesario para tests de integración completa |

## Buenas y malas prácticas

**Hacer:**
- Comenzar siempre por el test unitario puro (Nivel 1) para toda la lógica de negocio: es el test más rápido, más aislado y el que aporta mayor densidad de cobertura por segundo de ejecución.
- Incluir un test de `FunctionCatalog` (Nivel 3) para cada bean funcional para verificar que el nombre, el tipo y la composición están correctamente configurados: este test detecta errores de `spring.cloud.function.definition` incorrectos.
- Usar `WebEnvironment.MOCK` con `MockMvc` en lugar de `WebEnvironment.RANDOM_PORT` para tests HTTP de funciones: es más rápido y no requiere un puerto disponible en el entorno de CI.
- Testear los casos de error (excepción en la función) en el Nivel 1 con `assertThatThrownBy` y en el Nivel 4 con `status().is5xxServerError()` para verificar el comportamiento del adaptador HTTP ante excepciones.

**Evitar:**
- Evitar testear únicamente vía HTTP (`@SpringBootTest` + `MockMvc`): si el test HTTP pasa pero la función tiene un bug en un caso borde, el test de nivel 4 puede no detectarlo porque el adaptador serializa el error silenciosamente.
- Evitar `@SpringBootTest` para tests que solo necesitan verificar la lógica de la función: añade entre 3 y 8 segundos de overhead de arranque de contexto innecesario.
- Evitar omitir `spring.cloud.function.definition` en el `application-test.yml`: si el contexto de test tiene múltiples beans funcionales y la propiedad no está definida, el test lanzará `IllegalStateException` con un mensaje de ambigüedad que puede confundirse con un error de test.
- Evitar testear `FunctionAroundWrapper` de forma aislada sin registrar también la función que envuelve: el wrapper depende de `FunctionInvocationWrapper` que solo existe en el contexto de una invocación real a través del catálogo.

---

← [12.9 Observabilidad y operación de funciones expuestas](sc-function-operations.md) | [Índice (README.md)](README.md) | [13.1 CQRS con Spring Cloud Stream y Gateway](sc-patrones-cqrs.md) →
