# 7.9 Spring Cloud Bus — Testing y Troubleshooting

← [7.8 Spring Cloud Bus — Endpoint busenv y cambio en caliente](sc-bus-busenv.md) | [Índice](README.md) | [8.1 OAuth2 en microservicios — Roles y flujos de autorización](sc-security-oauth2-conceptos-flujos.md) →

---

## Introducción

Testear Spring Cloud Bus implica verificar que los eventos `RemoteApplicationEvent` se publican correctamente en el canal de salida y que los nodos receptores los procesan como se espera. El enfoque recomendado es usar el `TestChannelBinder` de Spring Cloud Stream Test, que simula el broker sin necesitar RabbitMQ ni Kafka reales en el entorno de test. El troubleshooting aborda los problemas más comunes que aparecen en producción y en el examen de certificación.

> [PREREQUISITO] Para usar `TestChannelBinder` es necesario añadir `spring-cloud-stream-test-binder` al classpath de test. Este binder reemplaza automáticamente a los binders reales (AMQP o Kafka) en el contexto de test.

## Testing con TestChannelBinder

`TestChannelBinder` es un binder de prueba de Spring Cloud Stream que permite capturar y verificar mensajes sin necesidad de un broker externo. En el contexto del Bus, permite interceptar los eventos `RemoteApplicationEvent` que se publican en el canal `springCloudBusOutput`.

Los pasos para testear el Bus con `TestChannelBinder` son:

1. Añadir `spring-cloud-stream-test-binder` a las dependencias de test.
2. Usar `@SpringBootTest` para cargar el contexto completo.
3. Inyectar `OutputDestination` para capturar mensajes publicados.
4. Inyectar `InputDestination` para simular mensajes entrantes.
5. Usar `ObjectMapper` para deserializar y verificar el contenido del mensaje.

## Ejemplo central — Tests completos de Bus

El siguiente ejemplo muestra tests para verificar la publicación de `BusRefreshEvent`, la recepción de eventos y el comportamiento del refresh usando `TestChannelBinder`.

```xml
<!-- pom.xml — dependencias de test -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-binder</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// BusRefreshEventPublicationTest.java — verifica que bus-refresh publica el evento
package com.example.orderservice;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.messaging.Message;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class BusRefreshEventPublicationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private OutputDestination outputDestination;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void whenBusRefreshEndpointCalled_thenEventIsPublishedToBroker() throws Exception {
        // 1. Invocar el endpoint bus-refresh
        mockMvc.perform(post("/actuator/bus-refresh"))
                .andExpect(status().isOk());

        // 2. Capturar el mensaje publicado en el canal de salida del Bus
        Message<byte[]> message = outputDestination.receive(1000, "springCloudBus");

        // 3. Verificar que el mensaje no es nulo (se publicó algo)
        assertThat(message).isNotNull();

        // 4. Deserializar y verificar el tipo del evento
        String payload = new String(message.getPayload());
        Map<?, ?> eventMap = objectMapper.readValue(payload, Map.class);

        assertThat(eventMap).containsKey("type");
        assertThat(eventMap.get("type").toString()).contains("RefreshRemoteApplicationEvent");
    }
}
```

```java
// BusCustomEventTest.java — verifica eventos personalizados
package com.example.orderservice;

import com.example.shared.events.CacheInvalidationEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.binder.test.InputDestination;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class BusCustomEventTest {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Autowired
    private OutputDestination outputDestination;

    @Autowired
    private InputDestination inputDestination;

    @Autowired
    private ObjectMapper objectMapper;

    @Value("${spring.cloud.bus.id:test-service:default:8080}")
    private String busId;

    @Test
    void whenCacheInvalidationEventPublished_thenMessageArrivesOnBus() throws Exception {
        // 1. Publicar un evento personalizado
        CacheInvalidationEvent event = new CacheInvalidationEvent(
                this, busId, "**", "products", "test invalidation"
        );
        eventPublisher.publishEvent(event);

        // 2. Capturar el mensaje publicado
        Message<byte[]> message = outputDestination.receive(1000, "springCloudBus");
        assertThat(message).isNotNull();

        // 3. Verificar el contenido del mensaje
        String payload = new String(message.getPayload());
        assertThat(payload).contains("CacheInvalidationEvent");
        assertThat(payload).contains("products");
    }

    @Test
    void whenMessageReceivedOnBusInput_thenEventIsProcessed() throws Exception {
        // 1. Crear un BusRefreshEvent simulado como JSON
        String busRefreshEventJson = """
                {
                  "type": "RefreshRemoteApplicationEvent",
                  "destinationService": "**",
                  "originService": "config-server:default:8888",
                  "id": "test-event-id-123"
                }
                """;

        // 2. Publicar el mensaje en el canal de entrada del Bus
        inputDestination.send(
                MessageBuilder.withPayload(busRefreshEventJson.getBytes())
                              .setHeader("contentType", "application/json")
                              .build(),
                "springCloudBus"
        );

        // El procesamiento es asíncrono; verificar efectos secundarios
        // en un test real se usaría un @EventListener capturador
    }
}
```

```yaml
# application-test.yml — configuración para tests del Bus
spring:
  application:
    name: order-service
  cloud:
    bus:
      enabled: true
      id: order-service:default:8080
    # TestChannelBinder se activa automáticamente con spring-cloud-stream-test-binder
    # No es necesario configurar RabbitMQ ni Kafka para tests

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env, health
  endpoint:
    bus-refresh:
      enabled: true
```

## Troubleshooting — Problemas comunes

Los problemas más frecuentes de Spring Cloud Bus en producción y sus soluciones se presentan a continuación. Conocer estos escenarios es relevante para el examen de certificación.

**Problema 1: El refresh no se propaga a todos los nodos**

La causa más común es que el consumer group del binding `springCloudBusInput` está configurado incorrectamente en Kafka. Si varias instancias del mismo servicio comparten el mismo consumer group, Kafka solo entrega el mensaje a una de ellas.

```yaml
# INCORRECTO — mismo group para todas las instancias en Kafka
spring:
  cloud:
    stream:
      bindings:
        springCloudBusInput:
          group: order-service    # Kafka solo entrega a UNA instancia del grupo

# CORRECTO — group único por instancia
spring:
  cloud:
    stream:
      bindings:
        springCloudBusInput:
          group: ${spring.application.name}-${random.uuid}
```

**Problema 2: Mensajes duplicados en las instancias**

Ocurre cuando en RabbitMQ se configura erróneamente un consumer group compartido, o cuando `spring.cloud.bus.destination` apunta al mismo exchange/topic en múltiples entornos (dev y prod comparten broker).

```yaml
# CORRECTO — destination diferente por entorno
spring:
  cloud:
    bus:
      destination: springCloudBus-${spring.profiles.active}
      # dev  → springCloudBus-dev
      # prod → springCloudBus-prod
```

**Problema 3: bus.id mal configurado — instancias no se reconocen como destino**

Si el `spring.cloud.bus.id` tiene el formato incorrecto o se duplica entre instancias, `ServiceMatcher` puede ignorar eventos dirigidos a la instancia o procesarlos cuando no debería.

```bash
# Verificar el bus.id actual de una instancia
curl http://localhost:8080/actuator/env | jq '.["spring.cloud.bus.id"]'

# O usar el endpoint de env para ver todas las propiedades del Bus
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.name | contains("bus"))'
```

## Tabla de diagnóstico de problemas comunes

Los síntomas, causas y soluciones de los problemas más frecuentes del Bus son:

| Síntoma | Causa probable | Solución |
|---------|---------------|----------|
| Refresh no llega a algunos nodos | Consumer group incorrecto en Kafka | `group: ${app.name}-${random.uuid}` |
| Mensajes duplicados | Consumer group compartido en Kafka / exchange compartido entre entornos | Group único / destination por entorno |
| Evento ignorado por el receptor | `@AcceptRemoteApplicationEvent` faltante | Añadir anotación en la clase del evento |
| `bus-refresh` → 404 Not Found | Endpoint no expuesto en Actuator | Añadir `bus-refresh` a `exposure.include` |
| `bus-refresh` → 401 Unauthorized | Spring Security protege el endpoint | Autenticar la llamada |
| Evento publicado pero beans no actualizados | Beans sin `@RefreshScope` | Añadir `@RefreshScope` a los beans |
| bus.id duplicado entre instancias | Configuración estática del bus.id | Usar `${server.port}` o `${random.uuid}` |

## Buenas y malas prácticas

**Buenas prácticas:**

- Usar `TestChannelBinder` en todos los tests que involucren el Bus. Nunca usar un broker real en tests unitarios o de integración si puede evitarse.
- Separar los tests de publicación (verificar que el evento se publica en el canal de salida) de los tests de recepción (verificar que el evento entrante produce el efecto esperado).
- Incluir tests de troubleshooting en los tests de integración para verificar que la configuración del consumer group es correcta.

**Malas prácticas:**

- Testear el Bus con un broker real (Embedded RabbitMQ o Embedded Kafka) cuando `TestChannelBinder` es suficiente. Aumenta la complejidad y el tiempo de ejecución de los tests sin beneficio para verificar la lógica del Bus.
- No verificar los mensajes en el `OutputDestination` tras invocar los endpoints del Bus. Es el único modo de confirmar que el Bus publicó el evento correctamente.
- Ignorar el log de WARN de Spring Cloud Bus durante el arranque. Muchos problemas de configuración se manifiestan como warnings en el log antes de manifestarse como fallos en producción.

## Verificación y práctica

> [EXAMEN] **1.** ¿Qué dependencia de test debe añadirse para usar `TestChannelBinder` y testear el Bus sin broker real?

> [EXAMEN] **2.** ¿Qué clase de Spring Cloud Stream Test se usa para capturar mensajes publicados por el Bus en los tests?

> [EXAMEN] **3.** ¿Por qué múltiples instancias del mismo servicio reciben mensajes duplicados del Bus cuando se usa Kafka con un consumer group compartido?

> [EXAMEN] **4.** ¿Qué anotación falta en la clase de un evento personalizado si el receptor lo ignora sin lanzar excepción?

> [EXAMEN] **5.** Si `POST /actuator/bus-refresh` devuelve 404, ¿cuál es la causa más probable y cómo se soluciona?

---

← [7.8 Spring Cloud Bus — Endpoint busenv y cambio en caliente](sc-bus-busenv.md) | [Índice](README.md) | [8.1 OAuth2 en microservicios — Roles y flujos de autorización](sc-security-oauth2-conceptos-flujos.md) →
