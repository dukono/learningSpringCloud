# 12.3 Manejo de tipos, inferencia y serialización de payloads

← [12.2 Configuración de funciones y exposición HTTP](sc-function-config.md) | [Índice (README.md)](README.md) | [12.4 Routing dinámico con RoutingFunction y expresiones SpEL](sc-function-routing.md) →

---

## Introducción

El problema que cubre este fichero es la brecha entre el tipo genérico declarado en la firma Java y el payload serializado que llega al adaptador en runtime. En Java, los genéricos se borran en compilación (type erasure): `Function<Order,Invoice>` en bytecode es simplemente `Function`. Spring Cloud Function necesita conocer `Order` e `Invoice` en runtime para deserializar el body JSON entrante y serializar la respuesta. Cuando el framework no puede inferir los tipos, aplica la conversión más genérica posible (`LinkedHashMap`), lo que produce `ClassCastException` o pérdidas de datos en producción.

Este problema se agrava en tres escenarios: funciones compuestas (la inferencia debe seguir la cadena), funciones lambda anónimas (sin información de tipo en la clase anónima) y adaptadores remotos como AWS Lambda (el payload llega como `String` JSON que debe convertirse al tipo concreto). Entender `FunctionTypeUtils`, `Message<T>` y la negociación de `Content-Type` es prerequisito para comprender cómo los adaptadores del módulo 12.5 y 12.7 invocan las funciones.

> **[PREREQUISITO]** Requiere `sc-function-modelo.md` (12.1) y `sc-function-config.md` (12.2).

## Representación visual

El siguiente diagrama muestra el camino de resolución de tipos desde el payload entrante hasta la invocación de la función, incluyendo el punto donde `FunctionTypeUtils` interviene cuando la inferencia automática falla.

```
  Payload entrante (HTTP body / mensaje)
  Content-Type: application/json
  {"id":1,"item":"book"}
          │
          ▼
  MessageConverter (Jackson / StringMessageConverter)
          │
          │  ¿Conoce el tipo destino?
          │
     SÍ ──┴── NO
     │              │
     ▼              ▼
  Deserializa   FunctionTypeUtils.getFunctionType()
  a Order.class   │
                  ▼
            Resuelve tipo via
            ParameterizedTypeReference
            o TypeUtils.getReturnType
                  │
                  ▼
          Deserializa correctamente
                  │
                  ▼
  Function<Order, Invoice>.apply(order)
          │
          ▼
  Serializa Invoice → JSON
  Content-Type: application/json (negociado)
```

## Ejemplo central

El ejemplo muestra tres patrones de manejo de tipos: inferencia automática con POJO, uso explícito de `Message<T>` para acceder a headers, y `FunctionRegistration` para forzar el tipo cuando la inferencia falla con lambdas.

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
package com.example.scfunction.tipos;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.context.FunctionRegistration;
import org.springframework.cloud.function.context.FunctionType;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@SpringBootApplication
public class FunctionTypesApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionTypesApplication.class, args);
    }

    // POJO de entrada
    public record Order(Long id, String item, double price) {}

    // POJO de salida
    public record Invoice(Long orderId, String description, double total) {}

    // Patrón 1: inferencia automática con clase concreta (el más común)
    // Spring Cloud Function resuelve Order e Invoice via reflexión sobre el bean
    @Bean
    public Function<Order, Invoice> processOrder() {
        return order -> new Invoice(
            order.id(),
            "Factura por: " + order.item(),
            order.price() * 1.21
        );
    }

    // Patrón 2: acceso a headers via Message<T>
    // Útil cuando el adaptador de mensajería añade metadata en headers
    @Bean
    public Function<Message<Order>, Message<Invoice>> processOrderWithHeaders() {
        return message -> {
            Order order = message.getPayload();
            String correlationId = (String) message.getHeaders().get("X-Correlation-Id");
            Invoice invoice = new Invoice(
                order.id(),
                "Factura: " + order.item() + " [corr:" + correlationId + "]",
                order.price() * 1.21
            );
            return MessageBuilder.withPayload(invoice)
                .setHeader("X-Correlation-Id", correlationId)
                .setHeader("X-Processed-By", "invoiceService")
                .build();
        };
    }

    // Patrón 3: FunctionRegistration con tipo explícito
    // Necesario cuando la lambda anónima o el bean no tienen información de tipo en runtime
    @Bean
    public FunctionRegistration<Function<Order, Invoice>> explicitTypeFunction() {
        Function<Order, Invoice> fn = order -> new Invoice(
            order.id(),
            "Explicit: " + order.item(),
            order.price()
        );
        return new FunctionRegistration<>(fn, "explicitTypeFunction")
            .type(FunctionType.from(Order.class).to(Invoice.class).getType());
    }
}
```

Para invocar `processOrder`:
```bash
curl -X POST http://localhost:8080/processOrder \
  -H "Content-Type: application/json" \
  -d '{"id":1,"item":"book","price":10.00}'
# Respuesta: {"orderId":1,"description":"Factura por: book","total":12.1}
```

> **[CONCEPTO]** `FunctionType.from(Order.class).to(Invoice.class)` construye el `ParameterizedTypeReference` que Spring Cloud Function utiliza internamente para instruir al `MessageConverter` en la deserialización. Sin este registro explícito, una lambda sin información de tipo genérico en bytecode provocará que el framework deserialice el body como `LinkedHashMap` en lugar de `Order`.

## Tabla de elementos clave

La siguiente tabla recoge los mecanismos de resolución de tipos y serialización relevantes para entrevistas técnicas.

| Elemento | Tipo | Descripción |
|---|---|---|
| `FunctionTypeUtils` | Clase utilitaria SCF | Resuelve el tipo genérico de un bean funcional en runtime, incluyendo cadenas compuestas |
| `FunctionType` | Clase SCF | Builder para construir `ParameterizedTypeReference` con `from(A).to(B)` |
| `FunctionRegistration<T>` | Clase SCF | Registro programático que asocia bean + nombre + tipo explícito |
| `Message<T>` | `org.springframework.messaging.Message` | Wrapper de payload con headers; permite acceso a metadatos del transporte |
| `MessageBuilder` | `org.springframework.messaging.support` | Constructor fluido de `Message<T>` con headers adicionales |
| `Content-Type: application/json` | Header HTTP | Indica al converter que use Jackson para deserializar el body |
| `Content-Type: text/plain` | Header HTTP | Indica al converter que trate el body como `String` |
| `Accept: application/json` | Header HTTP | Negocia el formato de la respuesta serializada |
| Type erasure | Concepto Java | Los genéricos se borran en bytecode; causa que lambdas anónimas pierdan tipo en runtime |
| `spring.cloud.function.definition` | Propiedad | Cuando se componen funciones `a\|b`, el tipo de `b` debe ser compatible con el tipo de salida de `a` |

> **[ADVERTENCIA]** Usar tipos primitivos como `Function<String, String>` no presenta problemas de inferencia. El problema de type erasure aparece exclusivamente con POJOs genéricos en lambdas definidas fuera de un método `@Bean` con retorno tipado. Siempre declarar los beans como métodos `@Bean` con tipo de retorno explícito en lugar de asignarlos a variables `var`.

## Buenas y malas prácticas

**Hacer:**
- Declarar siempre los beans funcionales como métodos `@Bean` con tipo de retorno genérico explícito (`Function<Order,Invoice>`), no como `var` ni sin tipo de retorno: el compilador y Spring Cloud Function necesitan el tipo en la firma del método para la inferencia.
- Usar `Message<T>` solo cuando sea necesario acceder a headers del transporte; para lógica de negocio pura, usar directamente el POJO de dominio.
- Registrar con `FunctionRegistration` cuando se usen lambdas almacenadas en variables o beans cargados dinámicamente, donde la reflexión no puede obtener el tipo genérico.
- Incluir `Content-Type: application/json` explícitamente en las llamadas HTTP de prueba para evitar que el framework elija `text/plain` y deserialice el body como `String`.

**Evitar:**
- Evitar usar `LinkedHashMap` como tipo de entrada de la función para "ser flexible": obliga a castings manuales, pierde la validación de tipos y produce `ClassCastException` al acceder a campos de tipo no-String.
- Evitar funciones que retornan `Object` o `Void` en lugar del tipo concreto: el serializer no sabe cómo serializar la respuesta y el framework puede lanzar `MessageConversionException`.
- Evitar el patrón `Function<String, String>` cuando los datos son POJOs: fuerza serialización/deserialización manual con `ObjectMapper` dentro de la función, rompiendo la separación de responsabilidades y duplicando lógica de conversión en producción.

---

← [12.2 Configuración de funciones y exposición HTTP](sc-function-config.md) | [Índice (README.md)](README.md) | [12.4 Routing dinámico con RoutingFunction y expresiones SpEL](sc-function-routing.md) →
