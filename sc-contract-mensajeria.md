# 10.4 Spring Cloud Contract — Contratos Mensajería

← [10.3 Contratos HTTP](sc-contract-http.md) | [Índice](README.md) | [10.5 Plugin Maven y Gradle](sc-contract-plugin-config.md) →

---

## Introducción

Los contratos de mensajería en Spring Cloud Contract permiten verificar la integración asíncrona entre microservicios que se comunican mediante brokers de mensajería (Kafka, RabbitMQ). A diferencia de los contratos HTTP, los contratos de mensajería trabajan con canales de entrada y salida, utilizando los bloques `input` y `outputMessage` del DSL. Spring Cloud Contract se integra con Spring Cloud Stream para la verificación automática en el lado del productor.

> [PREREQUISITO] Este nodo asume conocimiento del DSL Groovy/YAML de [10.2](sc-contract-dsl.md) y conceptos de mensajería asíncrona con Spring Cloud Stream.

## Dos patrones de contrato de mensajería

Spring Cloud Contract soporta dos patrones distintos para contratos de mensajería, según cómo se activa la producción del mensaje. Comprender la diferencia entre `triggeredBy` y `messageFrom` es fundamental para el examen.

El primer patrón cubre el caso en que el productor envía un mensaje como resultado de una llamada directa a un método (un comando interno). El segundo patrón cubre el caso reactivo: el productor recibe un mensaje y en respuesta envía otro.

```
Patrón 1: triggeredBy (método invocado)
────────────────────────────────────────
  Test generado
  ──────────────►  llama a confirmOrder()   ──►  mensaje enviado a "notifications"
  (clase base)      (método del productor)        (outputMessage.sentTo)


Patrón 2: messageFrom (mensaje entrante)
─────────────────────────────────────────
  Test generado
  ──────────────►  envía mensaje a "orders"  ──►  mensaje enviado a "notifications"
                   (input.messageFrom)              (outputMessage.sentTo)
```

> [CONCEPTO] `triggeredBy("methodName()")` hace que el test generado invoque el método `methodName()` definido en la **clase base del test del productor**. `messageFrom("channel")` hace que el test generado envíe un mensaje al canal de entrada especificado. Son mutuamente excluyentes dentro del bloque `input`.

## Ejemplo central: contrato con triggeredBy

El siguiente ejemplo completo muestra un contrato para el caso en que confirmar una orden dispara un mensaje de notificación. El contrato vive en el repositorio del productor.

```groovy
// src/test/resources/contracts/notification/orderConfirmedShouldSendNotification.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "when order is confirmed, a notification message is sent to notifications channel"

    input {
        // triggeredBy invoca un método en la clase base del test del productor
        triggeredBy("triggerOrderConfirmation()")
    }

    outputMessage {
        // sentTo: canal al que el productor debe enviar el mensaje
        sentTo("notifications")

        // messageHeaders: cabeceras del mensaje de salida
        messageHeaders {
            header("contentType", "application/json")
            header("eventType", "ORDER_CONFIRMED")
        }

        // messageBody: body del mensaje de salida
        messageBody([
            orderId  : 1,
            customerId: 42,
            status   : "CONFIRMED",
            timestamp: $(anyIsoDatetime())
        ])

        // bodyMatchers: validaciones adicionales sobre el body del mensaje
        bodyMatchers {
            jsonPath('$.orderId', byRegex("[0-9]+"))
            jsonPath('$.status', byRegex("CONFIRMED|CANCELLED|PENDING"))
        }
    }
}
```

```java
// Clase base del test del productor (src/test/java/.../BaseMessagingTest.java)
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.verifier.messaging.boot.AutoConfigureMessageVerifier;

@SpringBootTest
@AutoConfigureMessageVerifier  // activa la verificación de mensajería en el productor
public abstract class BaseMessagingTest {

    @Autowired
    private OrderService orderService;

    // El test generado llamará a este método cuando procese triggeredBy("triggerOrderConfirmation()")
    public void triggerOrderConfirmation() {
        // Crea una orden y la confirma, lo que dispara el envío del mensaje
        orderService.confirmOrder(1L);
    }
}
```

> [CONCEPTO] `@AutoConfigureMessageVerifier` es la anotación de Spring Cloud Contract que activa el soporte de verificación de mensajería en el lado del productor. Sin esta anotación, los tests generados para contratos de mensajería no pueden verificar los mensajes enviados.

## Ejemplo con messageFrom (mensaje entrante)

Este segundo ejemplo muestra el patrón reactivo: el productor reacciona a un mensaje entrante y como resultado envía un mensaje de salida.

```groovy
// src/test/resources/contracts/shipping/orderPaidShouldTriggerShipment.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "when payment confirmed message is received, shipment notification is sent"

    input {
        // messageFrom: canal desde el que llega el mensaje de entrada
        messageFrom("payments")

        messageHeaders {
            header("contentType", "application/json")
        }

        messageBody([
            orderId   : 1,
            amountPaid: 150.00
        ])

        // messageBodyMatchers: matchers sobre el body del mensaje entrante
        messageBodyMatchers {
            jsonPath('$.orderId', byRegex("[0-9]+"))
        }
    }

    outputMessage {
        sentTo("shipments")
        messageHeaders {
            header("contentType", "application/json")
        }
        messageBody([
            orderId: 1,
            carrier: "DHL",
            eta    : $(anyIsoDate())
        ])
    }
}
```

## Equivalente YAML de contrato de mensajería

El mismo contrato con `triggeredBy` expresado en formato YAML para equipos que prefieren no usar Groovy.

```yaml
# src/test/resources/contracts/notification/orderConfirmedShouldSendNotification.yml
description: "when order is confirmed, a notification message is sent"
input:
  triggeredBy: triggerOrderConfirmation()
outputMessage:
  sentTo: notifications
  headers:
    contentType: application/json
    eventType: ORDER_CONFIRMED
  body:
    orderId: 1
    customerId: 42
    status: CONFIRMED
  matchers:
    body:
      - path: $.orderId
        type: by_regex
        value: "[0-9]+"
      - path: $.status
        type: by_regex
        value: "CONFIRMED|CANCELLED|PENDING"
```

## Tabla de bloques de mensajería

Los bloques disponibles en contratos de mensajería cubren todos los aspectos del flujo asíncrono.

| Bloque | Subbloques | Descripción |
|---|---|---|
| `input` | `triggeredBy` | Invoca un método en la clase base del productor |
| `input` | `messageFrom` | Canal de entrada del mensaje que activa al productor |
| `input` | `messageHeaders` | Headers del mensaje de entrada |
| `input` | `messageBody` | Body del mensaje de entrada |
| `input` | `messageBodyMatchers` | Matchers sobre el body entrante |
| `outputMessage` | `sentTo` | Canal destino del mensaje de salida |
| `outputMessage` | `messageHeaders` | Headers del mensaje de salida |
| `outputMessage` | `messageBody` | Body del mensaje de salida |
| `outputMessage` | `bodyMatchers` | Matchers sobre el body de salida |

> [ADVERTENCIA] Los contratos de mensajería **no pueden mezclarse** con bloques HTTP (`request`/`response`) en el mismo fichero de contrato. Un contrato es exclusivamente HTTP o exclusivamente de mensajería.

## Configuración de dependencias para mensajería

Para que la verificación de mensajería funcione en el productor, se necesitan las dependencias correctas además de `@AutoConfigureMessageVerifier`.

```xml
<!-- pom.xml del PRODUCTOR — dependencias para mensajería -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>
<!-- Para mensajería con Spring Cloud Stream -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-support</artifactId>
    <scope>test</scope>
</dependency>
```

```xml
<!-- pom.xml del CONSUMIDOR — necesita el stub runner -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

## Buenas y malas prácticas

**Buenas prácticas**:
- Usar `triggeredBy` cuando el mensaje se genera como resultado de una acción del dominio (comando explícito).
- Usar `messageFrom` cuando el productor reacciona a un evento externo (patrón event-driven).
- Mantener los métodos invocados por `triggeredBy` en la clase base con nombres descriptivos del dominio.
- Añadir `bodyMatchers` para validar que los campos dinámicos del mensaje cumplen el formato esperado.

**Malas prácticas**:
- Omitir `@AutoConfigureMessageVerifier` en la clase base del productor — los tests de mensajería se generan pero no pueden verificar mensajes.
- Confundir `sentTo` (destino del mensaje de salida) con `messageFrom` (canal de entrada) — son bloques diferentes con semántica opuesta.
- Usar valores fijos para timestamps o IDs en el messageBody sin matchers — los tests del productor fallarán con valores reales.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia entre `triggeredBy` y `messageFrom` en el bloque `input` de un contrato de mensajería?

> [EXAMEN] 2. ¿Qué anotación debe incluirse en la clase base del productor para habilitar la verificación de contratos de mensajería?

> [EXAMEN] 3. En un contrato de mensajería, ¿qué define el bloque `sentTo` y en qué bloque del DSL se encuentra?

> [EXAMEN] 4. ¿Pueden coexistir bloques `request`/`response` y `input`/`outputMessage` en el mismo fichero de contrato?

> [EXAMEN] 5. ¿Qué método de la clase base del productor invoca el test generado cuando el contrato usa `triggeredBy("confirmOrder()")`?

---

← [10.3 Contratos HTTP](sc-contract-http.md) | [Índice](README.md) | [10.5 Plugin Maven y Gradle](sc-contract-plugin-config.md) →
