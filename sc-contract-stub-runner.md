# 10.7 Stub Runner en el consumer

← [10.6 Stub JAR y modos de StubRunner](sc-contract-stubs.md) | [Índice](README.md) | [10.8 Contract Messaging — contratos de eventos](sc-contract-messaging.md) →

---

## Introducción

El consumer de un contrato CDC no puede depender del producer real para ejecutar sus tests: el producer puede estar en desarrollo, en un entorno inaccesible o en una versión diferente. El Stub Runner resuelve este problema arrancando automáticamente un servidor WireMock local cargado con los stubs del producer, de modo que el consumer puede ejecutar sus tests con total independencia. La configuración del Stub Runner en el consumer es la pieza que cierra el ciclo CDC: sin ella, el consumer tiene contratos pero no puede verificar que su código los consume correctamente. Este fichero cubre todos los parámetros de `@AutoConfigureStubRunner`, las integraciones con Feign/RestTemplate/WebClient, el binding de puertos y la configuración programática.

> [TAREA] El desarrollador necesita configurar el Stub Runner en el consumer para que los tests de integración usen stubs en lugar de servicios reales.

> [RESULTADO] Después de leer este fichero el desarrollador puede anotar un test con `@AutoConfigureStubRunner`, configurar `stubrunner.ids` con el formato completo, inyectar el puerto con `@StubRunnerPort` y verificar que su cliente Feign/RestTemplate llama al stub correcto.

> [PREREQUISITO] Stub JAR generado y disponible (10.6); cliente HTTP del consumer (Feign, RestTemplate o WebClient) configurado.

## Representación visual

El siguiente diagrama muestra cómo el Stub Runner intercepta las llamadas del consumer y las redirige al servidor WireMock local.

```
[Test del Consumer]
  @AutoConfigureStubRunner(ids="com.example:order-service:1.0.0:stubs:8090")
         │
         ▼
[StubRunner arranca WireMock en :8090]
  carga mappings desde stub JAR
  └─ META-INF/com.example/order-service/1.0.0/mappings/
         │
         ▼
[OrderClient (@FeignClient o RestTemplate)]
  apuntado a http://localhost:8090
         │
         ▼
[WireMock responde con el stub del contrato]
         │
         ▼
[Test verifica comportamiento del consumer]
```

**Formato del parámetro `ids`:**

```
groupId : artifactId : version : classifier : port
  com.example : order-service : 1.0.0 : stubs : 8090

versión + = latest disponible
puerto 0 = dinámico (usar @StubRunnerPort para inyectarlo)
```

## Ejemplo central

Los siguientes ejemplos muestran la configuración completa del Stub Runner con los tres clientes HTTP más usados.

**Con @FeignClient — configuración más común:**

```java
// application.yml del consumer (test)
// src/test/resources/application.yml
```

```yaml
# apuntar el FeignClient al stub WireMock
order-service:
  url: http://localhost:8090

spring:
  cloud:
    openfeign:
      client:
        config:
          order-service:
            url: http://localhost:8090
```

```java
@FeignClient(name = "order-service", url = "${order-service.url}")
public interface OrderClient {
    @GetMapping("/orders/{id}")
    OrderDTO getOrder(@PathVariable Long id);
}
```

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:1.0.0:stubs:8090",
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderConsumerFeignTest {

    @Autowired
    private OrderClient orderClient;

    @Test
    void shouldFetchOrderFromStub() {
        OrderDTO order = orderClient.getOrder(42L);
        assertThat(order.getId()).isEqualTo(42L);
        assertThat(order.getNif()).isNotBlank();
    }
}
```

**Con RestTemplate y puerto dinámico:**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs",  // puerto dinámico
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderConsumerRestTemplateTest {

    @StubRunnerPort("order-service")  // inyecta el puerto asignado dinámicamente
    private int stubPort;

    @Autowired
    private RestTemplate restTemplate;

    @Test
    void shouldFetchOrderDynamicPort() {
        String url = "http://localhost:" + stubPort + "/orders/42";
        OrderDTO order = restTemplate.getForObject(url, OrderDTO.class);
        assertThat(order).isNotNull();
        assertThat(order.getAmount()).isPositive();
    }
}
```

**Con WebClient (API reactiva):**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs",
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderConsumerWebClientTest {

    @StubRunnerPort("order-service")
    private int stubPort;

    private WebClient webClient;

    @BeforeEach
    void setup() {
        webClient = WebClient.builder()
            .baseUrl("http://localhost:" + stubPort)
            .build();
    }

    @Test
    void shouldFetchOrderReactive() {
        OrderDTO order = webClient.get()
            .uri("/orders/42")
            .retrieve()
            .bodyToMono(OrderDTO.class)
            .block(Duration.ofSeconds(5));
        assertThat(order.getId()).isEqualTo(42L);
    }
}
```

**Múltiples stubs en un mismo test:**

```java
@AutoConfigureStubRunner(
    ids = {
        "com.example:order-service:+:stubs:8090",
        "com.example:payment-service:+:stubs:8091",
        "com.example:inventory-service:+:stubs:8092"
    },
    stubsMode = StubRunnerProperties.StubsMode.CLASSPATH
)
class OrderFlowConsumerTest { ... }
```

**Configuración programática con StubRunnerOptions:**

```java
@Bean
public StubRunnerOptions stubRunnerOptions() {
    return StubRunnerOptionsBuilder.newInstance()
        .withStubsMode(StubRunnerProperties.StubsMode.CLASSPATH)
        .withStubs("com.example:order-service:+:stubs:8090")
        .build();
}
```

**application.yml — mover configuración fuera del código:**

```yaml
stubrunner:
  stubs-mode: classpath
  ids:
    - com.example:order-service:+:stubs:8090
  # para modo REMOTE:
  # repository-root: http://nexus:8081/repository/maven-releases/
  # stubs-mode: remote
```

## Tabla de elementos clave

Los parámetros de `@AutoConfigureStubRunner` y `stubrunner.*` que un profesional senior debe dominar.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `ids` | String[] | — | Formato: `groupId:artifactId:version:classifier:port`; `+` = latest |
| `stubsMode` | enum | `CLASSPATH` | Origen de los stubs; sobreescribe `stubrunner.stubs-mode` |
| `repositoryRoot` | String | — | URL Nexus/Artifactory para `StubsMode.REMOTE` |
| `@StubRunnerPort(name)` | Anotación | — | Inyecta el puerto asignado dinámicamente al stub identificado por `name` |
| `stubrunner.ids` | String | — | Alternativa YAML a `ids` en la anotación |
| `stubrunner.stubs-mode` | enum | `CLASSPATH` | Alternativa YAML a `stubsMode` |
| `stubrunner.repository-root` | String | — | Alternativa YAML a `repositoryRoot` |
| `classifier` | String | `stubs` | Classifier del JAR de stubs |
| `port=0` | int | — | Asignación dinámica de puerto; requiere `@StubRunnerPort` para inyectarlo |
| `StubRunnerOptions` | Clase | — | Configuración programática; útil para beans condicionales por entorno |

## Buenas y malas prácticas

**Hacer:**

- Usar puerto fijo (ej: `8090`) solo en tests de integración con docker-compose donde el puerto está declarado; en todos los demás casos usar puerto dinámico (`ids=...::0`) e inyectar con `@StubRunnerPort` para evitar conflictos de puerto en CI cuando varios tests arrancan en paralelo.
- Externalizar `stubrunner.stubs-mode` y `stubrunner.repository-root` en `src/test/resources/application.yml` en lugar de codificarlos en la anotación; así se pueden cambiar por perfil (`-Dspring.profiles.active=ci`) sin recompilar.
- Añadir `@StubRunnerPort("artifactId")` usando el `artifactId` sin sufijo `-service`; si el ID completo del stub es `com.example:order-service`, la anotación es `@StubRunnerPort("order-service")`.
- Verificar que el FeignClient, RestTemplate o WebClient apunta a `localhost:${stubPort}` en el perfil de test; un cliente apuntando al host de producción ignora el stub y falla con `Connection refused` o, peor, llama al servicio real.

**Evitar:**

- No usar `stubsMode = StubsMode.LOCAL` en el pipeline CI si el producer no ha ejecutado `mvn install` previamente en el mismo agente: el stub JAR no estará en `~/.m2` y el test fallará con `StubNotFoundException`, camuflando el error como un problema del consumer.
- No declarar más de 5-6 stubs simultáneos en un test si cada uno usa puerto fijo numerado secuencialmente (8090, 8091, ...): en entornos CI con alta concurrencia, los puertos entran en conflicto entre builds paralelas. Usar puertos dinámicos.
- No olvidar `<exclusions>*:*</exclusions>` al añadir el stub JAR como dependencia en `pom.xml`: sin exclusiones, las dependencias transitivas del producer pueden entrar en conflicto con las del consumer, causando `NoSuchMethodError` difíciles de diagnosticar.
- No usar `@AutoConfigureStubRunner` en tests de producción que acaban en el classpath de producción (fuera de `src/test`): el starter `spring-cloud-starter-contract-stub-runner` tiene scope `test` por diseño; usarlo en producción añade WireMock al runtime innecesariamente.

## Comparación: configuración via anotación vs YAML

Ambas formas son funcionalmente equivalentes; la elección afecta la mantenibilidad del proyecto.

| Criterio | `@AutoConfigureStubRunner(...)` | `application.yml` (stubrunner.*) |
|---|---|---|
| Visibilidad en código | Alta (en el test) | Baja (en fichero externo) |
| Reutilización entre tests | Baja (repetición en cada test) | Alta (hereda todos los tests) |
| Override por perfil CI | No sin recompilar | Sí con `-Dspring.profiles.active=ci` |
| Ideal para | Tests únicos con config específica | Suite de tests con config compartida |

---

← [10.6 Stub JAR y modos de StubRunner](sc-contract-stubs.md) | [Índice](README.md) | [10.8 Contract Messaging — contratos de eventos](sc-contract-messaging.md) →

