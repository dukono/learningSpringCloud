# 2.12 Testing / Verificación de Spring Cloud Netflix Eureka

← [2.11 Operación y troubleshooting de Eureka](sc-eureka-troubleshooting.md) | [Índice (README.md)](README.md) | [3.1 Arquitectura reactiva y modelo de ejecución →](sc-gateway-arquitectura.md)

---

Verificar el comportamiento de Eureka en los tests requiere estrategias diferentes según lo que se quiere comprobar: que el código de negocio funciona independientemente del registro (tests unitarios), que el servicio se integra correctamente con el mecanismo de discovery (tests de integración), o que el registro y la detección de instancias funcionan de extremo a extremo (tests de integración con servidor real). Usar siempre un Eureka Server real en todos los tests es lento y frágil; usar siempre mocks hace que los tests no detecten problemas de configuración reales. La clave es elegir la estrategia adecuada a cada nivel.

## Estrategia 1: desactivar Eureka completamente en tests unitarios y de integración de negocio

La estrategia más rápida y robusta para tests que no verifican el comportamiento de discovery es desactivar Eureka por completo con propiedades de test. El servicio arranca sin intentar conectar al servidor Eureka ni registrarse.

```java
package com.ejemplo.pedidos.service;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test de integración de la lógica de negocio del servicio.
 * Eureka está desactivado: el test no necesita servidor Eureka y arranca rápido.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    // Desactiva el cliente Eureka completamente: no intenta registrarse
    // ni descargar el registro. El bean EurekaClient no se crea.
    "eureka.client.enabled=false"
})
class PedidoServiceTest {

    @Autowired
    private PedidoService pedidoService;

    @Test
    void crearPedido_devuelveIdValido() {
        Long id = pedidoService.crear("producto-1", 3);
        assertThat(id).isPositive();
    }
}
```

> [CONCEPTO] `eureka.client.enabled=false` es la propiedad canónica para desactivar Eureka en tests. Es más completa que `spring.cloud.discovery.enabled=false` porque también desactiva el heartbeat, el fetch del registro y la autoconfiguración de `EurekaClient`.

## Estrategia 2: mock de DiscoveryClient con @MockBean

Para tests que verifican código que usa `DiscoveryClient` (por ejemplo, un servicio que filtra instancias por zona o que implementa lógica de fallback), la estrategia correcta es mockear la interfaz con `@MockBean`. Esto evita la necesidad de un servidor Eureka real y permite controlar exactamente qué instancias "están disponibles".

```java
package com.ejemplo.pedidos.service;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.test.context.TestPropertySource;

import java.net.URI;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {"eureka.client.enabled=false"})
class CatalogoLocalizadorTest {

    @MockBean
    private DiscoveryClient discoveryClient;

    @Autowired
    private CatalogoLocalizador catalogoLocalizador;

    @Test
    void obtenerInstanciaEnZona_devuelveInstanciaCorrecta() {
        // Arrange: preparar una instancia mock con metadata de zona
        ServiceInstance instanciaMock = mock(ServiceInstance.class);
        when(instanciaMock.getInstanceId()).thenReturn("catalogo:10.0.1.5:8082");
        when(instanciaMock.getUri()).thenReturn(URI.create("http://10.0.1.5:8082"));
        when(instanciaMock.getMetadata()).thenReturn(Map.of("zona", "eu-west-1a"));

        when(discoveryClient.getInstances("servicio-catalogo"))
            .thenReturn(List.of(instanciaMock));

        // Act
        ServiceInstance resultado = catalogoLocalizador.obtenerInstanciaEnZona("eu-west-1a");

        // Assert
        assertThat(resultado.getMetadata().get("zona")).isEqualTo("eu-west-1a");
        assertThat(resultado.getUri().toString()).isEqualTo("http://10.0.1.5:8082");
    }

    @Test
    void obtenerInstanciaEnZona_lanzaExcepcionSiNoHayInstancias() {
        when(discoveryClient.getInstances("servicio-catalogo"))
            .thenReturn(List.of());

        org.junit.jupiter.api.Assertions.assertThrows(
            RuntimeException.class,
            () -> catalogoLocalizador.obtenerInstanciaEnZona("eu-west-1a")
        );
    }
}
```

## Estrategia 3: test de registro efectivo con Eureka Server embebido

Para verificar que un microservicio se registra correctamente en Eureka, incluyendo su metadata y estado, se usa un test con `@SpringBootTest` que levanta el Eureka Server en el mismo contexto de Spring. Esta estrategia es más costosa en tiempo pero verifica el comportamiento real de registro end-to-end.

```java
package com.ejemplo.eureka;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.test.context.TestPropertySource;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test que verifica el registro efectivo en un Eureka Server embebido.
 * El contexto de Spring incluye tanto el servidor como el cliente.
 * El webEnvironment con RANDOM_PORT hace que el cliente se registre
 * con el puerto aleatorio asignado.
 */
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    classes = {
        com.ejemplo.eurekaserver.EurekaServerApplication.class,
        com.ejemplo.pedidos.ServicioPedidosApplication.class
    }
)
@TestPropertySource(properties = {
    "spring.application.name=servicio-pedidos-test",
    "eureka.client.enabled=true",
    "eureka.client.service-url.defaultZone=http://localhost:${local.server.port}/eureka/",
    "eureka.client.register-with-eureka=true",
    "eureka.client.fetch-registry=true",
    "eureka.instance.lease-renewal-interval-in-seconds=1",
    "eureka.client.registry-fetch-interval-seconds=1",
    "eureka.server.enable-self-preservation=false"
})
class EurekaRegistroIntegrationTest {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Test
    void servicioRegistrado_esDescubrible() throws InterruptedException {
        // Dar tiempo a que el registro se propague entre el cliente y el servidor
        // Con intervalos de 1s configurados arriba, 3s son suficientes
        Thread.sleep(3000);

        List<ServiceInstance> instancias = discoveryClient.getInstances("servicio-pedidos-test");

        assertThat(instancias)
            .as("El servicio debe estar registrado en Eureka")
            .isNotEmpty();

        ServiceInstance instancia = instancias.get(0);
        assertThat(instancia.getServiceId())
            .isEqualToIgnoringCase("servicio-pedidos-test");
    }

    @Test
    void servicioRegistrado_tieneMetadataCorrecta() throws InterruptedException {
        Thread.sleep(3000);

        List<ServiceInstance> instancias = discoveryClient.getInstances("servicio-pedidos-test");
        assertThat(instancias).isNotEmpty();

        ServiceInstance instancia = instancias.get(0);
        // Verificar que la URI del servicio es accesible (no un hostname no resolvible)
        assertThat(instancia.getUri()).isNotNull();
        assertThat(instancia.getPort()).isPositive();
    }
}
```

> [ADVERTENCIA] Los tests con Eureka Server embebido son inherentemente lentos porque deben esperar a que los intervalos de heartbeat y fetch del registro se completen. Reducir los intervalos (`lease-renewal-interval-in-seconds: 1`, `registry-fetch-interval-seconds: 1`) en las propiedades del test acelera los tiempos de propagación. No usar estos valores reducidos en producción.

## Tabla de estrategias: situación y recomendación

La siguiente tabla guía la elección de estrategia de verificación según el objetivo del test.

| Situación | Estrategia recomendada | Velocidad | Cobertura |
|---|---|---|---|
| Test unitario de lógica de negocio sin discovery | `eureka.client.enabled=false` | Muy rápida | Lógica de negocio pura |
| Test de código que usa `DiscoveryClient` | `@MockBean DiscoveryClient` + `eureka.client.enabled=false` | Rápida | Lógica de selección de instancias |
| Verificar que el servicio se registra en Eureka | Eureka Server embebido + `@SpringBootTest` | Lenta | Registro end-to-end |
| Verificar propagación de estado entre peers del cluster | Test de integración con dos servidores Eureka levantados | Muy lenta | Alta disponibilidad |
| Verificar que el health check de Actuator se propaga al estado Eureka | Eureka Server embebido + forzar health DOWN via `TestHealthIndicator` | Lenta | Integración health-Eureka |

**Preguntas de entrevista sobre testing de Eureka**

Una pregunta frecuente es: "¿Cómo testeas que tu microservicio se registra correctamente en Eureka sin un servidor real?". La respuesta correcta distingue entre dos objetivos: si lo que se quiere verificar es la lógica que usa `DiscoveryClient`, un `@MockBean` es suficiente y más rápido. Si se quiere verificar que la configuración de registro es correcta (metadata, intervalo de heartbeat, prefer-ip-address), es necesario un Eureka Server embebido o un servidor de test levantado con Docker (Testcontainers).

Otra pregunta frecuente es: "¿Por qué mis tests de integración con Eureka son no deterministas (a veces pasan, a veces fallan)?". La causa más común es no esperar suficiente tiempo a que el registro se propague antes de hacer aserciones sobre `discoveryClient.getInstances()`. La solución es reducir todos los intervalos de Eureka en las propiedades del test y añadir un `Thread.sleep()` calibrado, o usar Awaitility para una espera basada en condición.

```java
// Uso de Awaitility para esperas basadas en condición (más robusto que Thread.sleep)
import static org.awaitility.Awaitility.await;
import java.util.concurrent.TimeUnit;

@Test
void servicioRegistrado_esDescubribleConAwaitility() {
    await()
        .atMost(10, TimeUnit.SECONDS)
        .pollInterval(500, TimeUnit.MILLISECONDS)
        .until(() -> !discoveryClient.getInstances("servicio-pedidos-test").isEmpty());

    List<ServiceInstance> instancias = discoveryClient.getInstances("servicio-pedidos-test");
    assertThat(instancias).isNotEmpty();
}
```

---

← [2.11 Operación y troubleshooting de Eureka](sc-eureka-troubleshooting.md) | [Índice (README.md)](README.md) | [3.1 Arquitectura reactiva y modelo de ejecución →](sc-gateway-arquitectura.md)
