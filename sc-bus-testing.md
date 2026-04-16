# 7.9 Testing de Spring Cloud Bus

← [7.8 Patrones de fallo y troubleshooting de Spring Cloud Bus](sc-bus-troubleshooting.md) | [Índice (README.md)](README.md) | [8.1 Fundamentos OAuth2 y JWT para microservicios](sc-security-oauth2-fundamentals.md) →

---

Spring Cloud Bus no expone un puerto ni una API propia: su comportamiento se manifiesta como efectos secundarios —actualizaciones de `@RefreshScope`, eventos recibidos en listeners— que solo son observables en un contexto con broker real o simulado. Testear Bus correctamente significa verificar que los eventos se propagan desde el origen hasta los suscriptores y que el estado de los beans se actualiza como se espera; hacerlo mal (solo testear el listener de forma unitaria sin trigger de Bus) da una falsa sensación de cobertura. Existen tres niveles: test unitario del listener de evento en aislamiento, test de integración con `TestChannelBinderConfiguration` (broker simulado), y test de integración con Testcontainers (broker real).

> [PREREQUISITO] Requiere `spring-cloud-starter-bus-kafka` o `spring-cloud-starter-bus-amqp` en producción. Para tests de integración con broker simulado se necesita `spring-cloud-stream-test-binder`. Para tests con broker real se necesita `org.testcontainers:kafka` o `org.testcontainers:rabbitmq`.

## Estrategias de testing de Spring Cloud Bus

Las tres estrategias cubren distintos niveles de fidelidad y velocidad. La elección depende de qué se quiere verificar: la lógica del listener, la propagación del mensaje, o el comportamiento end-to-end con un broker real.

| Estrategia | Broker | Velocidad | Fidelidad | Cuándo usar |
|---|---|---|---|---|
| 1 — Test unitario del listener | Sin broker | Muy rápida | Baja | Validar lógica del handler de evento |
| 2 — Test con TestChannelBinder | Simulado | Rápida | Media | Verificar wiring de Bus sin infraestructura |
| 3 — Test con Testcontainers | Real (Docker) | Lenta | Alta | Verificar propagación real y efectos en @RefreshScope |

## Estrategia 1: Test unitario del listener de `RemoteApplicationEvent`

Un `RemoteApplicationEvent` (como `RefreshRemoteApplicationEvent`) es un POJO: puede instanciarse y publicarse en un `ApplicationEventPublisher` local sin necesidad de broker. Este test verifica que el listener reacciona correctamente al evento.

```xml
<!-- pom.xml — dependencias de test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.bus;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent;
import org.springframework.context.ApplicationEventPublisher;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(properties = {
    "spring.cloud.bus.enabled=false",   // Bus deshabilitado: no hay broker
    "spring.cloud.config.enabled=false" // Config Server no necesario aquí
})
class RefreshListenerUnitTest {

    @Autowired
    private ApplicationEventPublisher publisher;

    @Autowired
    private MyRefreshAwareService service;

    @Test
    void whenRefreshEventPublished_thenServiceUpdatesState() {
        // El ID "test-origin" es arbitrario para tests unitarios
        RefreshRemoteApplicationEvent event =
            new RefreshRemoteApplicationEvent(this, "test-origin", null);

        publisher.publishEvent(event);

        assertThat(service.getLastRefreshTimestamp()).isNotNull();
    }
}
```

> [ADVERTENCIA] `spring.cloud.bus.enabled=false` desactiva el bus completo, por lo que este test no prueba la propagación real — solo la lógica del listener al recibir el evento ya instanciado localmente.

## Estrategia 2: Test de integración con `TestChannelBinderConfiguration`

`TestChannelBinderConfiguration` (módulo `spring-cloud-stream-test-binder`) reemplaza el binder real (Kafka/RabbitMQ) por un binder en memoria. Permite verificar que el mensaje de Bus se produce y consume correctamente sin levantar infraestructura externa.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-binder</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

```java
package com.example.bus;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.binder.test.InputDestination;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Import;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
class BusIntegrationTest {

    @Autowired
    private InputDestination inputDestination;

    @Autowired
    private OutputDestination outputDestination;

    @Autowired
    private MyRefreshAwareService service;

    @Test
    void whenRefreshEventSentToInputChannel_thenServiceRefreshes() {
        // Construir el payload del RefreshRemoteApplicationEvent serializado como JSON
        String payload = """
            {
              "type": "RefreshRemoteApplicationEvent",
              "originService": "test-service:8080",
              "destinationService": "**"
            }
            """;

        Message<byte[]> message = MessageBuilder
            .withPayload(payload.getBytes())
            .build();

        // Enviar al canal de entrada del bus (destino por defecto: "springCloudBus")
        inputDestination.send(message, "springCloudBus");

        // Verificar que el servicio procesó el refresh
        assertThat(service.getLastRefreshTimestamp()).isNotNull();
    }
}
```

> [EXAMEN] `TestChannelBinderConfiguration` importa un binder de test que sustituye TODOS los binders configurados en `spring.cloud.stream.bindings`. Si la app tiene bindings de Stream y Bus simultáneamente, todos quedan sustituidos en el test.

## Estrategia 3: Test con Testcontainers (broker real)

Este nivel verifica el comportamiento end-to-end: el evento viaja por un broker Kafka o RabbitMQ real, llega a los suscriptores, y los beans `@RefreshScope` se re-instancian. Requiere Docker disponible en el entorno de CI.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.bus;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.cloud.context.refresh.ContextRefresher;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class BusE2ETest {

    @Container
    @ServiceConnection
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @Autowired
    private ContextRefresher contextRefresher;

    @Autowired
    private MyConfigurableBean configurableBean;

    @Test
    void whenRefreshTriggered_thenRefreshScopeBeanReinitialized() {
        // Estado inicial
        String valueBefore = configurableBean.getConfigValue();

        // Disparar refresh (Bus propaga el evento a todos los nodos del topic)
        contextRefresher.refresh();

        // Esperar propagación asíncrona (hasta 5 segundos)
        await()
            .atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() ->
                assertThat(configurableBean.getConfigValue())
                    .isNotEqualTo(valueBefore)
            );
    }
}
```

### Tabla resumen: cuándo usar cada estrategia

| Situación | Estrategia recomendada |
|---|---|
| Verificar lógica del listener ante un evento concreto | 1 — Test unitario local |
| Verificar wiring Bus sin levantar broker en CI | 2 — TestChannelBinderConfiguration |
| Verificar propagación real y re-instanciación de @RefreshScope | 3 — Testcontainers |
| Detectar incompatibilidades de serialización de eventos | 3 — Testcontainers |
| Test rápido en pipeline sin Docker | 2 — TestChannelBinderConfiguration |

## Tabla de elementos clave

Los componentes y propiedades que un desarrollador senior necesita reconocer al testear Spring Cloud Bus:

| Componente / Propiedad | Tipo | Descripción |
|---|---|---|
| `TestChannelBinderConfiguration` | clase | Binder en memoria de spring-cloud-stream-test-binder; reemplaza Kafka/Rabbit en tests |
| `InputDestination` | bean | Canal de entrada en memoria; permite enviar mensajes al bus simulado |
| `OutputDestination` | bean | Canal de salida en memoria; permite leer mensajes producidos por el bus |
| `spring.cloud.bus.enabled` | boolean (true) | Desactivar en tests unitarios que no necesitan broker |
| `ContextRefresher.refresh()` | método | Dispara un `RefreshRemoteApplicationEvent` y propaga por Bus |
| `@RefreshScope` | anotación | Marca beans que se re-instancian al recibir `RefreshRemoteApplicationEvent` |
| `RemoteApplicationEvent` | clase base | Evento de Bus; subclases: `RefreshRemoteApplicationEvent`, `EnvironmentChangeRemoteApplicationEvent` |

## Buenas y malas prácticas

**Hacer:**
- Usar `TestChannelBinderConfiguration` para tests de integración rápidos que no requieren Docker; verifica el wiring completo sin coste de infraestructura.
- Usar Testcontainers para los tests que verifican efectos en `@RefreshScope`: son los únicos que garantizan que la propagación y la re-instanciación ocurren de verdad.
- Añadir `awaitility` para esperar la propagación asíncrona en tests con broker real; los asserts síncronos inmediatamente después del trigger siempre fallan.
- Nombrar explícitamente el `destinationService` en los tests para verificar el enrutamiento selectivo (no siempre `"**"`).

**Evitar:**
- Testear solo el listener en aislamiento y declarar que el módulo Bus está "testeado": los tests unitarios no detectan problemas de serialización, wiring o configuración del broker.
- Usar `@SpringBootTest` con Kafka/RabbitMQ real en los tests de PR sin Testcontainers: requiere infraestructura externa en el agente de CI, lo que produce tests frágiles y difíciles de replicar localmente.
- Mezclar `TestChannelBinderConfiguration` con tests que también usen `spring-cloud-stream` para mensajería de negocio: el binder de test sustituye todos los canales, lo que puede interferir con la validación de otros bindings.

---

← [7.8 Patrones de fallo y troubleshooting de Spring Cloud Bus](sc-bus-troubleshooting.md) | [Índice (README.md)](README.md) | [8.1 Fundamentos OAuth2 y JWT para microservicios](sc-security-oauth2-fundamentals.md) →
