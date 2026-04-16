# 12.4 Routing dinámico con RoutingFunction y expresiones SpEL

← [12.3 Manejo de tipos, inferencia y serialización](sc-function-tipos.md) | [Índice (README.md)](README.md) | [12.5.1 Despliegue en AWS Lambda](sc-function-aws.md) →

---

## Introducción

El problema que resuelve `RoutingFunction` es la necesidad de un único punto de entrada que despache la invocación a distintas funciones según el contenido del mensaje o de sus headers, sin duplicar código de routing en cada función. En arquitecturas event-driven, distintos tipos de evento llegan al mismo canal pero deben procesarse con lógica diferente: un `OrderCreatedEvent` va a `orderProcessor` y un `OrderCancelledEvent` va a `cancellationHandler`. Sin `RoutingFunction`, el desarrollador implementa un switch manual dentro de una función monolítica que viola el principio de responsabilidad única y dificulta el testing.

`RoutingFunction` es una función especial registrada por Spring Cloud Function que evalúa en runtime la función destino mediante: (1) el header `spring.cloud.function.definition` del mensaje entrante, o (2) una expresión SpEL configurada en `spring.cloud.function.routing-expression`. Se activa automáticamente cuando `spring.cloud.function.routing-expression` está definido en las propiedades o cuando el mensaje entrante contiene el header `spring.cloud.function.definition`.

> **[PREREQUISITO]** Requiere `sc-function-modelo.md` (12.1), `sc-function-config.md` (12.2) y `sc-function-tipos.md` (12.3). El lector debe entender `FunctionCatalog` y `Message<T>` antes de estudiar el routing.

## Representación visual

El flujo de routing dinámico parte del mensaje entrante y evalúa la función destino antes de delegar la invocación. La siguiente figura muestra los dos mecanismos de decisión.

```
  Mensaje entrante
  ┌─────────────────────────────────────────────────────┐
  │  Headers: spring.cloud.function.definition=orderProc│
  │  Body: {"id":1,"item":"book"}                       │
  └──────────────────────┬──────────────────────────────┘
                         │
                         ▼
               RoutingFunction
          (registro automático SCF)
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
  Mecanismo 1:               Mecanismo 2:
  Header                     SpEL expression
  spring.cloud.function      spring.cloud.function
  .definition                .routing-expression
  "orderProcessor"           headers['type']=='ORDER'
                             ?'orderProcessor'
                             :'cancellationHandler'
          │                             │
          └──────────────┬──────────────┘
                         │
                         ▼
              FunctionCatalog.lookup(destino)
                         │
               ┌─────────┴────────────┐
               ▼                      ▼
        orderProcessor      cancellationHandler
        Function<Order,      Function<Order,
         Invoice>              Void>
```

## Ejemplo central

El ejemplo implementa un router que despacha a tres funciones distintas según el header `eventType`. Se muestra tanto la configuración via SpEL como el uso del header directo en el mensaje.

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
      # Activa RoutingFunction como función por defecto
      # La expresión SpEL evalúa el header 'eventType' para elegir la función destino
      routing-expression: >
        headers['eventType'] == 'ORDER_CREATED' ? 'processOrder' :
        headers['eventType'] == 'ORDER_CANCELLED' ? 'cancelOrder' :
        'defaultHandler'
```

### Código Java completo

```java
package com.example.scfunction.routing;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.function.Function;

@SpringBootApplication
public class FunctionRoutingApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionRoutingApplication.class, args);
    }

    public record OrderEvent(Long id, String item, String status) {}
    public record ProcessResult(Long orderId, String outcome) {}

    // Función destino 1: procesa pedidos creados
    @Bean
    public Function<Message<OrderEvent>, Message<ProcessResult>> processOrder() {
        return message -> {
            OrderEvent event = message.getPayload();
            ProcessResult result = new ProcessResult(event.id(), "PROCESSED: " + event.item());
            return MessageBuilder.withPayload(result)
                .setHeader("X-Handler", "processOrder")
                .build();
        };
    }

    // Función destino 2: cancela pedidos
    @Bean
    public Function<Message<OrderEvent>, Message<ProcessResult>> cancelOrder() {
        return message -> {
            OrderEvent event = message.getPayload();
            ProcessResult result = new ProcessResult(event.id(), "CANCELLED: " + event.item());
            return MessageBuilder.withPayload(result)
                .setHeader("X-Handler", "cancelOrder")
                .build();
        };
    }

    // Función destino 3: manejador por defecto
    @Bean
    public Function<Message<OrderEvent>, Message<ProcessResult>> defaultHandler() {
        return message -> {
            OrderEvent event = message.getPayload();
            ProcessResult result = new ProcessResult(event.id(), "DEFAULT: " + event.item());
            return MessageBuilder.withPayload(result)
                .setHeader("X-Handler", "defaultHandler")
                .build();
        };
    }
}
```

Invocación con routing vía SpEL (header `eventType` evaluado por la expresión):
```bash
# Ruta a processOrder
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -H "eventType: ORDER_CREATED" \
  -d '{"id":1,"item":"book","status":"new"}'
# Respuesta: {"orderId":1,"outcome":"PROCESSED: book"}

# Ruta a cancelOrder
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -H "eventType: ORDER_CANCELLED" \
  -d '{"id":2,"item":"pen","status":"active"}'
# Respuesta: {"orderId":2,"outcome":"CANCELLED: pen"}
```

Invocación con routing vía header `spring.cloud.function.definition` (sin SpEL configurado):
```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -H "spring.cloud.function.definition: processOrder" \
  -d '{"id":3,"item":"notebook","status":"new"}'
```

> **[CONCEPTO]** Cuando `spring.cloud.function.routing-expression` está configurado, Spring Cloud Function registra automáticamente un bean llamado `functionRouter` en `FunctionCatalog`. Este bean es el que recibe todas las invocaciones al path raíz `/` y evalúa la expresión SpEL con el contexto del mensaje para determinar la función destino.

> **[ADVERTENCIA]** Si la expresión SpEL retorna un nombre de función que no existe en `FunctionCatalog`, `RoutingFunction` lanza `FunctionNotFoundException` en runtime. Es obligatorio incluir siempre un caso por defecto en la expresión (`? 'fn1' : 'defaultHandler'`) o manejar la excepción.

## Tabla de elementos clave

La siguiente tabla resume los mecanismos de routing dinámico y sus propiedades de configuración.

| Elemento | Tipo | Descripción |
|---|---|---|
| `RoutingFunction` | Clase interna SCF | Función especial que evalúa destino en runtime; se registra como `functionRouter` |
| `spring.cloud.function.routing-expression` | Propiedad String (SpEL) | Expresión evaluada con el `Message` como contexto; retorna nombre de función destino |
| `spring.cloud.function.definition` (header) | Header del mensaje | Header del mensaje entrante que especifica la función destino; tiene prioridad sobre SpEL |
| `headers['key']` | SpEL en routing | Acceso a headers del mensaje en la expresión de routing |
| `payload['field']` | SpEL en routing | Acceso a campos del payload en la expresión de routing (requiere tipo resuelto) |
| `FunctionNotFoundException` | Excepción SCF | Lanzada cuando el nombre resuelto no existe en `FunctionCatalog` |
| Path raíz `/` | Endpoint HTTP | `RoutingFunction` se mapea al path raíz cuando está activa |

## Buenas y malas prácticas

**Hacer:**
- Incluir siempre un caso `else` o ternario por defecto en la expresión SpEL para evitar `FunctionNotFoundException` con eventos no reconocidos en producción.
- Usar el header `spring.cloud.function.definition` en integraciones con Spring Cloud Stream cuando el routing debe ser por mensaje (cada mensaje decide su función destino) y la expresión SpEL global no es suficientemente específica.
- Mantener las expresiones SpEL simples y delegables en un solo header de tipo: expresiones complejas que acceden al `payload` requieren que el tipo esté resuelto previamente, lo que introduce dependencia con la configuración de tipos.
- Testear la expresión SpEL con `SpelExpressionParser` de forma aislada antes de integrarla en la propiedad, para verificar que todos los casos posibles retornan un nombre de función válido.

**Evitar:**
- Evitar acceder al payload en expresiones SpEL (`payload['field']`) con POJOs complejos: si el tipo no se ha resuelto aún en el ciclo de evaluación, el payload puede llegar como `LinkedHashMap` y la expresión produce `NullPointerException`.
- Evitar mezclar routing por header y routing por SpEL sin documentar la precedencia: el header `spring.cloud.function.definition` tiene prioridad sobre `routing-expression`, lo que puede producir comportamiento inesperado si ambos están activos.
- Evitar usar `RoutingFunction` cuando todas las rutas son estáticas y conocidas en compilación: en ese caso, la composición declarativa con `spring.cloud.function.definition=a|b` es más eficiente y predecible.

---

← [12.3 Manejo de tipos, inferencia y serialización](sc-function-tipos.md) | [Índice (README.md)](README.md) | [12.5.1 Despliegue en AWS Lambda](sc-function-aws.md) →
