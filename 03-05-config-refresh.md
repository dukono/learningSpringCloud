# Parte 3.5 — Spring Cloud Config: Refresco de Configuración

← [Config Client y Perfiles](./03-04-config-client.md) | [Volver al índice](./README.md) | Siguiente: [Cifrado y HA →](./03-06-config-avanzado.md)

---

## 3.7 Refresco de configuración: @RefreshScope y Spring Cloud Bus

Por defecto, los microservicios leen la configuración **una sola vez al arrancar**. Si se cambia una propiedad en el repo Git, el servicio no lo verá hasta que se reinicie.

---

### Solución 1: Refresco manual con @RefreshScope

> **[CONCEPTO]** **`@RefreshScope`** es un scope de Spring que destruye y recrea un bean cuando se invoca `/actuator/refresh`. El bean recibe los nuevos valores de propiedades en la recreación. Solo los beans anotados con `@RefreshScope` se recargan — el resto del contexto permanece intacto.

`@RefreshScope` marca un bean para que sus propiedades se recarguen cuando se llame al endpoint `/actuator/refresh`.

```java
@RestController
@RefreshScope   // el bean se destruye y se recrea al llamar a /actuator/refresh
                // AVISO: si el bean tiene estado interno o conexiones abiertas,
                // se perderán en cada refresco. Usar solo en beans de configuración.
public class FeatureController {

    @Value("${feature.nueva-ui:false}")
    private boolean nuevaUiEnabled;

    @GetMapping("/feature-flag")
    public boolean isEnabled() {
        return nuevaUiEnabled;
    }
}
```

```xml
<!-- Necesario para exponer /actuator/refresh -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh, health, info
```

```bash
# Trigger del refresco (POST sin body)
curl -X POST http://localhost:8083/actuator/refresh
```

**Problema de escala:** Si hay 20 instancias del servicio, hay que llamar a los 20 endpoints individualmente.

#### Respuesta de `/actuator/refresh`

El endpoint devuelve la lista de propiedades que cambiaron. Una lista vacía indica que no hubo cambios:

```bash
curl -X POST http://localhost:8083/actuator/refresh
# → ["pedidos.max-items", "feature.nueva-ui"]   (propiedades que cambiaron)
# → []                                           (sin cambios desde el último refresco)
```

Útil para confirmar que el refresco aplicó los cambios esperados y detectar si una propiedad no está siendo recogida por `@RefreshScope`.

#### Comportamiento cuando Git no está disponible en el momento del refresco

```
Cliente llama a POST /actuator/refresh
         ↓
Config Client contacta al Config Server
         ↓
Config Server intenta fetch/pull del repositorio Git
         ↓ (Git no responde)
Config Server devuelve error 500 al cliente
         ↓
/actuator/refresh devuelve error al caller
         ↓
Los beans @RefreshScope NO se recargan
Los valores anteriores se mantienen intactos
```

> **El cliente no queda en estado inconsistente:** si el refresco falla, los beans `@RefreshScope` conservan los valores que tenían. El servicio sigue funcionando con la configuración anterior. El fallo se registra en los logs del Config Server con el error Git.
>
> Para detectar este escenario en producción: monitorizar el código de respuesta de las llamadas a `/actuator/refresh` (debería ser 200 con lista de propiedades cambiadas, no 500).

#### Lógica personalizada tras un refresco: `EnvironmentChangeEvent`

Cuando el contexto se refresca, Spring publica un `EnvironmentChangeEvent` con las claves que cambiaron. Se puede escuchar para ejecutar lógica propia:

```java
@Component
public class ConfigChangeListener implements ApplicationListener<EnvironmentChangeEvent> {

    @Override
    public void onApplicationEvent(EnvironmentChangeEvent event) {
        Set<String> changedKeys = event.getKeys();
        log.info("Propiedades cambiadas: {}", changedKeys);

        if (changedKeys.contains("feature.nueva-ui")) {
            // invalidar caché de la feature, notificar a otros componentes, etc.
            featureCacheService.invalidate("nueva-ui");
        }

        if (changedKeys.stream().anyMatch(k -> k.startsWith("pedidos."))) {
            // recalcular configuración derivada
            pedidosService.recargarConfiguracion();
        }
    }
}
```

> `EnvironmentChangeEvent` se dispara tanto con refresco manual (`/actuator/refresh`) como con refresco vía Spring Cloud Bus. El bean listener **no necesita** `@RefreshScope`.

#### Beans que NO deben usar @RefreshScope

`@RefreshScope` destruye y recrea el bean completo. Algunos tipos de beans no soportan eso:

| Bean | Motivo | Alternativa |
|---|---|---|
| `DataSource` / HikariCP | El pool de conexiones no se cierra limpiamente; las conexiones activas quedan huérfanas | Reiniciar el servicio si cambia la URL de BD |
| `@Scheduled` | Los timers se cancelan al destruir el bean y no se reprograman solos | Reiniciar, o usar un `SchedulingConfigurer` externo |
| Beans con estado acumulado | El estado en memoria se pierde en cada refresco | Externalizar el estado (BD, caché distribuida) |
| `EntityManagerFactory` / Hibernate | La inicialización del schema y los metamodelos se pierde | Reiniciar el servicio |

> **[ADVERTENCIA]** Aplicar `@RefreshScope` a un `@Configuration` que define un `DataSource` puede causar errores de transacción o pérdida de conexiones activas en producción, sin ningún mensaje de error claro. Es uno de los errores más difíciles de diagnosticar en este contexto.

---

### Solución 2: Spring Cloud Bus (refresco masivo)

> **[CONCEPTO]** **Spring Cloud Bus** conecta todos los microservicios a través de un broker de mensajes (Kafka o RabbitMQ). Un único evento publicado en el bus se propaga simultáneamente a todas las instancias suscritas, eliminando la necesidad de llamar a `/actuator/refresh` en cada instancia individualmente.

El refresco manual con `/actuator/refresh` funciona bien con 3 o 4 instancias. Con 50 instancias de 10 servicios distintos, hacer 50 llamadas POST manuales o en un script no es viable ni en tiempo ni en fiabilidad. La solución es invertir la dirección del flujo: en lugar de que alguien llame a cada instancia, publicar un único evento en un canal de mensajería al que todas las instancias están suscritas. Todas reciben el mensaje simultáneamente y se refrescan solas. Kafka y RabbitMQ actúan como ese canal de difusión — por eso son un prerequisito.

> **[PREREQUISITO]** Spring Cloud Bus requiere un broker de mensajes externo en ejecución: **Kafka** o **RabbitMQ**. Sin el broker levantado, la aplicación no arranca al incluir la dependencia.

Spring Cloud Bus conecta todos los microservicios a través del broker para **propagar el evento de refresco a todas las instancias** con un solo POST.

```xml
<!-- En todos los microservicios y en el Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    <!-- o spring-cloud-starter-bus-amqp para RabbitMQ -->
</dependency>
```

```yaml
# Configuración del broker (en cada microservicio)
spring:
  kafka:
    bootstrap-servers: localhost:9092
  # o para RabbitMQ:
  # rabbitmq:
  #   host: localhost
  #   port: 5672
  cloud:
    bus:
      enabled: true
      id: ${spring.application.name}:${server.port}:${random.uuid}

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv, health
```

```bash
# Un solo POST al Config Server o a cualquier instancia
# refresca TODOS los servicios conectados al Bus
curl -X POST http://localhost:8888/actuator/busrefresh

# Refrescar solo instancias específicas — patrón: {service-name}:{port}:{uuid}
curl -X POST http://localhost:8888/actuator/busrefresh/pedidos-service:**
# "pedidos-service:**" → todas las instancias de pedidos-service (cualquier puerto y uuid)

curl -X POST http://localhost:8888/actuator/busrefresh/pedidos-service:8083:*
# Solo instancias de pedidos-service en el puerto 8083

curl -X POST http://localhost:8888/actuator/busrefresh/pedidos-service:8083:abc-uuid-123
# Solo la instancia exacta con ese puerto y uuid
```

> El ID de cada instancia en el Bus se configura con `spring.cloud.bus.id=${spring.application.name}:${server.port}:${random.uuid}`. El patrón de `busrefresh` usa `*` como comodín para cada segmento.

```
Config Server recibe POST /actuator/busrefresh
         ↓
  publica RefreshRemoteApplicationEvent en Kafka topic "springCloudBus"
         ↓
  todos los microservicios suscritos al topic
  reciben el evento y refrescan sus @RefreshScope beans
```

### `spring.cloud.bus.destination` — cambiar el nombre del topic

Por defecto Spring Cloud Bus usa el topic `springCloudBus` en Kafka o la exchange `springCloudBus` en RabbitMQ. Si este nombre colisiona con otro sistema existente, o si se quieren entornos completamente aislados (dev/staging/prod con el mismo broker), se puede cambiar:

```yaml
spring:
  cloud:
    bus:
      destination: mi-empresa-config-bus   # nombre del topic/exchange en el broker
      # Todos los servicios conectados al Bus deben usar el mismo destination
      # Un servicio con destination distinto no recibirá los eventos
```

> **[ADVERTENCIA]** Si se cambia `destination` en el Config Server pero no en los clientes (o viceversa), el `busrefresh` no llegará a los clientes. Es el primer parámetro que revisar si los eventos de Bus no se propagan.

### Tipos de eventos del Bus: `RefreshRemoteApplicationEvent` y `EnvironmentChangeEvent`

Spring Cloud Bus maneja dos eventos distintos que es importante no confundir:

| Evento | Quién lo publica | Quién lo escucha | Qué hace |
|---|---|---|---|
| `RefreshRemoteApplicationEvent` | Config Server (vía `busrefresh` o `/monitor`) | Todos los microservicios suscritos | Dispara `ContextRefresher.refresh()` en cada instancia |
| `EnvironmentChangeEvent` | Cada microservicio (localmente) | Solo el microservicio que lo recibe | Notifica las keys que cambiaron tras aplicar el refresco |

```java
// Escuchar EnvironmentChangeEvent para lógica propia tras el refresco:
@EventListener
public void onEnvironmentChange(EnvironmentChangeEvent event) {
    // event.getKeys() devuelve las keys que cambiaron
    // Este evento es LOCAL — se dispara en el servicio donde se aplicó el refresco
}

// Escuchar RefreshRemoteApplicationEvent para detectar que el Bus envió un refresco:
@EventListener
public void onRemoteRefresh(RefreshRemoteApplicationEvent event) {
    // event.getDestinationService() — a qué servicio iba dirigido el evento
    // Este evento es el que viaja por el Bus (Kafka/RabbitMQ)
}
```

---

### Solución 3: Refresco automático por Webhook (CI/CD)

Permite que el propio repositorio Git dispare el refresco automáticamente cada vez que se hace un commit, sin intervención manual.

```
Developer hace commit en el repo de configuración
        ↓
GitHub/GitLab ejecuta el webhook configurado
        ↓
POST automático a /actuator/busrefresh del Config Server
        ↓
Spring Cloud Bus propaga el evento a todos los servicios
```

#### Configuración del webhook en GitHub

En el repositorio Git de configuración: **Settings → Webhooks → Add webhook**

```
Payload URL:  https://config-server.miempresa.com/actuator/busrefresh
Content type: application/json
Secret:       [token secreto para verificar la procedencia]
Events:       Just the push event
```

#### Proteger el endpoint del webhook con secreto

```yaml
# application.yml del Config Server
management:
  endpoints:
    web:
      exposure:
        include: busrefresh
  endpoint:
    busrefresh:
      # Spring verifica la cabecera X-Hub-Signature que envía GitHub
      enabled: true
```

> **[ADVERTENCIA]** El endpoint `/actuator/busrefresh` debe estar protegido. En producción, el Config Server no debe ser accesible públicamente — usar un API Gateway o un túnel seguro como intermediario entre GitHub y el Config Server interno.

---

### Solución 4: Refresco selectivo con `/monitor` (Spring Cloud Config Monitor)

El webhook al endpoint `busrefresh` refresca **todos** los servicios (o los especificados manualmente). El problema: si solo cambió la configuración de `pedidos-service`, todos los demás servicios también reciben el evento de refresco innecesariamente. Con `spring-cloud-config-monitor`, el Config Server recibe el webhook nativo de GitHub/GitLab/Bitbucket, analiza qué ficheros cambiaron en el commit, y publica el evento de refresco **solo para los servicios cuya configuración fue modificada**.

```xml
<!-- En el Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-monitor</artifactId>
</dependency>
```

```
GitHub hace commit en el repo de configuración
        ↓
GitHub ejecuta el webhook
        ↓
POST automático a /monitor del Config Server
  con el payload nativo de GitHub (lista de ficheros cambiados)
        ↓
Config Monitor analiza los ficheros:
  pedidos-service-prod.yml → afecta a "pedidos-service" en perfil "prod"
  application.yml → afecta a TODOS los servicios
        ↓
Publica RefreshRemoteApplicationEvent
  solo para los servicios afectados (via Spring Cloud Bus)
```

```yaml
# Webhook en GitHub apuntando a /monitor (no a /actuator/busrefresh)
# Payload URL: https://config-server.miempresa.com/monitor
# Content-Type: application/json
# Events: push

# El Config Server detecta automáticamente la plataforma por las cabeceras:
# X-Github-Event → GitHub
# X-Gitlab-Event → GitLab
# X-Event-Key: repo:push → Bitbucket
```

**Diferencia entre `/monitor` y `/actuator/busrefresh`:**

| | `/actuator/busrefresh` | `/monitor` |
|---|---|---|
| **Requiere Bus** | Sí | Sí |
| **Qué refresca** | Todos (o los especificados manualmente) | Solo los servicios con ficheros modificados |
| **Payload del webhook** | No interpreta el payload | Interpreta el formato nativo de GitHub/GitLab/Bitbucket |
| **Dependencia extra** | No | `spring-cloud-config-monitor` |
| **Selectividad** | Manual | Automática por fichero |

---

### Comparativa de opciones de refresco

| Opción | Cuándo usarla | Requiere infraestructura extra |
|---|---|---|
| Reiniciar el servicio | Cambios poco frecuentes, entorno simple | No |
| `@RefreshScope` + `/actuator/refresh` manual | Pocas instancias, refresco ocasional | No |
| Spring Cloud Bus + POST manual | Múltiples instancias, refresco bajo demanda | Sí (Kafka o RabbitMQ) |
| Spring Cloud Bus + Webhook a `busrefresh` | CI/CD automatizado, refresca todos los servicios | Sí (Kafka o RabbitMQ + webhook) |
| Spring Cloud Config Monitor + `/monitor` | CI/CD automatizado, refresca solo los servicios afectados | Sí (Kafka o RabbitMQ + `spring-cloud-config-monitor`) |

---

---

## 3.7.1 Testing del mecanismo de refresco

### Testear que `@RefreshScope` recarga los valores

El test simula un cambio de propiedad en el entorno y verifica que el bean se recarga con el nuevo valor. No requiere un Config Server real — `TestPropertyValues` inyecta directamente en el contexto.

```java
@SpringBootTest
@ActiveProfiles("test")
class RefreshScopeTest {

    @Autowired
    private FeatureController featureController;   // bean con @RefreshScope

    @Autowired
    private ContextRefresher contextRefresher;

    @Autowired
    private ConfigurableApplicationContext context;

    @Test
    void beanSeRecargaTrasRefresh() {
        assertThat(featureController.isEnabled()).isFalse();

        // Simular cambio de propiedad sin tocar el Config Server
        TestPropertyValues.of("feature.nueva-ui=true")
            .applyTo(context);

        contextRefresher.refresh();   // equivale a POST /actuator/refresh

        assertThat(featureController.isEnabled()).isTrue();
    }
}
```

```yaml
# src/test/resources/application-test.yml
spring:
  cloud:
    config:
      enabled: false   # el test no necesita Config Server
    bus:
      enabled: false   # deshabilitar el Bus — no hay Kafka/RabbitMQ en tests unitarios
```

---

### Deshabilitar Spring Cloud Bus en tests

Si la aplicación usa Spring Cloud Bus, la dependencia `spring-cloud-starter-bus-kafka` intenta conectar al broker al arrancar el contexto de test. Sin broker disponible, el test falla.

```yaml
# src/test/resources/application-test.yml
spring:
  cloud:
    bus:
      enabled: false          # deshabilita el Bus completamente en tests
  autoconfigure:
    exclude:
      - org.springframework.cloud.stream.binder.kafka.config.KafkaBinderConfiguration
```

> **[ADVERTENCIA]** Deshabilitar el Bus en tests es correcto para tests de unidad e integración local. En tests de integración que verifican el comportamiento del refresco masivo (end-to-end), usar Testcontainers para levantar un broker real:

```java
@SpringBootTest
@Testcontainers
class BusRefreshIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private ApplicationEventPublisher publisher;

    @Test
    void busRefreshPropagaElEventoATodasLasInstancias() {
        // publicar el evento y verificar que los beans @RefreshScope se recargan
        publisher.publishEvent(new RefreshRemoteApplicationEvent(
            this, "test-origin", null));

        // verificar el estado esperado tras el refresco
    }
}
```

---

### Resumen: qué estrategia usar

| Caso | Estrategia |
|---|---|
| Test de `@RefreshScope` sin servidor | `TestPropertyValues` + `ContextRefresher` + `enabled: false` |
| Test con Bus sin broker real | `spring.cloud.bus.enabled: false` |
| Test de integración del Bus end-to-end | Testcontainers con Kafka o RabbitMQ |
| Verificar `EnvironmentChangeEvent` | `@EventListener` en el test + `ContextRefresher.refresh()` |

---

→ **Extensión programática:** [03-05-extension-programatica.md](./03-05-extension-programatica.md) — `ContextRefresher`, `RefreshScope.refresh()`, publicar `RefreshRemoteApplicationEvent` sin HTTP

← [Config Client y Perfiles](./03-04-config-client.md) | [Volver al índice](./README.md) | Siguiente: [Cifrado y HA →](./03-06-config-avanzado.md)
