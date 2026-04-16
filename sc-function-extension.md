# 12.8 Puntos de extensión avanzados: FunctionAroundWrapper y FunctionRegistration

← [12.7 Integración con Spring Cloud Stream](sc-function-stream.md) | [Índice (README.md)](README.md) | [12.9 Observabilidad y operación de funciones expuestas](sc-function-operations.md) →

---

## Introducción

El problema que resuelven los puntos de extensión avanzados es la necesidad de comportamiento transversal en la invocación de funciones sin modificar el código de negocio. En producción aparecen tres escenarios concretos: (1) añadir logging, métricas o validación a todas las funciones sin duplicar código en cada bean; (2) registrar funciones cuyo tipo genérico no puede inferirse en runtime (lambdas sin contexto de tipo, beans cargados dinámicamente); (3) controlar el ciclo de vida de funciones que requieren inicialización o limpieza de recursos.

`FunctionAroundWrapper` es el mecanismo de interceptor de Spring Cloud Function: cualquier bean que implemente esta interfaz se aplica a todas las invocaciones de funciones del catálogo, de forma análoga a un `Filter` de Servlet o un `Aspect` de AOP. `FunctionRegistration<T>` es el mecanismo de registro explícito que permite asociar un bean funcional con su tipo genérico y nombre cuando el discovery automático falla.

> **[PREREQUISITO]** Requiere `sc-function-modelo.md` (12.1) y `sc-function-config.md` (12.2). El registro con `FunctionRegistration` resuelve directamente el problema de type erasure descrito en `sc-function-tipos.md` (12.3).

> **[ADVERTENCIA]** Spring Cloud 2025.1.1 (Oakwood) con Spring Boot 4.0.x puede introducir cambios en la API de `FunctionCatalog` y en el lifecycle de funciones. Verificar en [docs.spring.io/spring-cloud-function](https://docs.spring.io/spring-cloud-function) antes de asumir compatibilidad total con la API de `FunctionAroundWrapper` documentada aquí.

## Representación visual

El pipeline de invocación con `FunctionAroundWrapper` activo envuelve cada llamada a `Function.apply()` con el código del wrapper, similar al patrón Decorator.

```
  Invocación HTTP / Stream / Lambda
          │
          ▼
  FunctionCatalog.lookup("processOrder")
          │
          ▼
  FunctionAroundWrapper.apply(
    invocable,    ← la función original (processOrder)
    functionArgs  ← los argumentos (Order payload)
  )
          │
          ├── PRE: logging, validación, métricas, MDC
          │
          ▼
  invocable.apply(functionArgs)  ← invocación real
          │
          ▼
  POST: logging de resultado, timing, error handling
          │
          ▼
  Resultado devuelto al adaptador
```

## Ejemplo central

El ejemplo implementa un `FunctionAroundWrapper` que añade logging de entrada/salida y cronometrado a todas las funciones, y un `FunctionRegistration` para un bean funcional con tipo no inferible automáticamente.

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
    <artifactId>spring-cloud-function-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      definition: processOrder
```

### Código Java completo

```java
package com.example.scfunction.extension;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.context.FunctionProperties;
import org.springframework.cloud.function.context.FunctionRegistration;
import org.springframework.cloud.function.context.FunctionType;
import org.springframework.cloud.function.context.catalog.FunctionAroundWrapper;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;

import java.util.function.Function;

@SpringBootApplication
public class FunctionExtensionApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionExtensionApplication.class, args);
    }

    public record Order(Long id, String item, double price) {}
    public record Invoice(Long orderId, String description, double total) {}

    // Bean funcional estándar (discovery automático funciona)
    @Bean
    public Function<Order, Invoice> processOrder() {
        return order -> new Invoice(
            order.id(),
            "Factura: " + order.item(),
            order.price() * 1.21
        );
    }

    // FunctionAroundWrapper: interceptor aplicado a TODAS las funciones del catálogo
    @Bean
    public FunctionAroundWrapper loggingWrapper() {
        return new FunctionAroundWrapper() {
            @Override
            public Object apply(Object value, FunctionInvocationWrapper invocable) {
                long start = System.currentTimeMillis();
                String functionName = invocable.getFunctionDefinition();

                // PRE-procesamiento
                System.out.println("[WRAPPER PRE] Invocando: " + functionName
                    + " | Input type: " + (value != null ? value.getClass().getSimpleName() : "null"));

                Object result;
                try {
                    // Invocación real de la función
                    result = invocable.apply(value);
                } catch (Exception e) {
                    System.err.println("[WRAPPER ERROR] Función: " + functionName
                        + " | Error: " + e.getMessage());
                    throw e;
                }

                // POST-procesamiento
                long elapsed = System.currentTimeMillis() - start;
                System.out.println("[WRAPPER POST] Función: " + functionName
                    + " | Tiempo: " + elapsed + "ms"
                    + " | Output type: " + (result != null ? result.getClass().getSimpleName() : "null"));

                return result;
            }
        };
    }

    // FunctionRegistration: registro explícito con tipo para una lambda sin tipo inferible
    // Caso de uso: la función se construye dinámicamente o proviene de un factory method
    @Bean
    public FunctionRegistration<Function<Order, Invoice>> discountFunction() {
        // Lambda almacenada en variable: el tipo genérico se pierde en bytecode
        Function<Order, Invoice> discountFn = order -> new Invoice(
            order.id(),
            "Descuento 10%: " + order.item(),
            order.price() * 0.90
        );

        return new FunctionRegistration<>(discountFn, "discountFunction")
            .type(FunctionType.from(Order.class).to(Invoice.class).getType());
    }

    // FunctionProperties: acceso programático a las propiedades configuradas
    @Bean
    public FunctionConfigPrinter configPrinter(FunctionProperties functionProperties) {
        return new FunctionConfigPrinter(functionProperties);
    }

    // Componente que usa FunctionProperties para introspección en arranque
    static class FunctionConfigPrinter {
        FunctionConfigPrinter(FunctionProperties props) {
            System.out.println("[CONFIG] Función definida: " + props.getDefinition());
        }
    }
}
```

Salida de ejemplo al invocar `POST /processOrder`:
```
[CONFIG] Función definida: processOrder
[WRAPPER PRE] Invocando: processOrder | Input type: Order
[WRAPPER POST] Función: processOrder | Tiempo: 3ms | Output type: Invoice
```

> **[CONCEPTO]** `FunctionInvocationWrapper` (parámetro del método `apply` de `FunctionAroundWrapper`) expone `getFunctionDefinition()` para obtener el nombre de la función activa, y `apply(value)` para delegar la invocación real. Es el objeto central del interceptor y permite conocer el contexto de la invocación sin acceder al `FunctionCatalog`.

## Tabla de elementos clave

Los puntos de extensión y sus propósitos se recogen en la siguiente tabla.

| Elemento | Tipo | Propósito |
|---|---|---|
| `FunctionAroundWrapper` | Interface SCF | Interceptor aplicado a todas las invocaciones del catálogo; patrón Decorator |
| `FunctionInvocationWrapper` | Clase SCF | Parámetro del wrapper; provee nombre de función y método `apply()` para delegar |
| `FunctionRegistration<T>` | Clase SCF | Registro programático con tipo explícito y nombre; resuelve type erasure |
| `FunctionType` | Clase SCF | Builder de tipo para `FunctionRegistration`; usa `from(A.class).to(B.class)` |
| `FunctionProperties` | Clase SCF | Bean auto-configurado con las propiedades `spring.cloud.function.*` activas |
| `FunctionProperties.getDefinition()` | Método | Retorna el valor de `spring.cloud.function.definition` configurado |
| Inicialización lazy | Concepto | `@Lazy` en beans funcionales pesados aplaza su creación a la primera invocación |

## Buenas y malas prácticas

**Hacer:**
- Implementar `FunctionAroundWrapper` para comportamiento transversal (logging, métricas, validación de entrada) en lugar de duplicar ese código en cada bean funcional: un único wrapper afecta a todas las funciones del catálogo.
- Usar `FunctionRegistration` con `FunctionType.from(A.class).to(B.class)` cuando la función se crea mediante factory methods o lambdas almacenadas en variables: garantiza que el converter de mensajes conozca los tipos en runtime.
- Verificar que el wrapper no introduce latencia observable en el path crítico: si el wrapper realiza operaciones síncronas costosas (llamadas a base de datos, validaciones complejas), moverlas a operaciones reactivas o asíncronas para no bloquear el hilo de invocación.
- Usar `FunctionProperties` para introspección en lugar de leer directamente `Environment.getProperty("spring.cloud.function.definition")`: `FunctionProperties` está tipado y validado por el framework.

**Evitar:**
- Evitar usar `FunctionAroundWrapper` para implementar lógica de negocio que pertenece a la función: el wrapper debe ser transversal (logging, métricas, seguridad); la lógica de dominio debe estar en la función.
- Evitar registrar múltiples `FunctionAroundWrapper` beans sin documentar el orden de aplicación: el orden de aplicación de múltiples wrappers no está garantizado por defecto y puede producir comportamiento inesperado en producción.
- Evitar usar `FunctionRegistration` para todos los beans como práctica general: el discovery automático es suficiente para la mayoría de casos y más mantenible. Reservar `FunctionRegistration` para casos donde la inferencia falla demostrable.

---

← [12.7 Integración con Spring Cloud Stream](sc-function-stream.md) | [Índice (README.md)](README.md) | [12.9 Observabilidad y operación de funciones expuestas](sc-function-operations.md) →
