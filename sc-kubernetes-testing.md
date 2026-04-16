# 9.12 Testing de Spring Cloud Kubernetes

← [9.11 Leader Election con Spring Cloud Kubernetes](sc-kubernetes-leader.md) | [Índice (README.md)](README.md) | [10.1 Arquitectura CDC y modelo Consumer-Driven Contracts](sc-contract-fundamentos.md) →

---

## Introducción

Testear una aplicación Spring Cloud Kubernetes presenta un desafío específico: el código depende de la Kubernetes API para cargar configuración y descubrir servicios, pero los tests unitarios y de integración estándar no tienen acceso a un cluster real. Sin una estrategia de testing adecuada, los tests fallan al arrancar porque SCK intenta conectarse a un API server que no existe, o la aplicación carga sin ConfigMaps y los tests no verifican el comportamiento real. Spring Cloud Kubernetes ofrece dos estrategias complementarias: `@EnableKubernetesMockClient` para tests unitarios y de integración rápidos (basado en WireMock para simular el API server), y Testcontainers con K3s o kind para tests de integración de alta fidelidad sobre un cluster Kubernetes real (pero efímero). Este nodo explica cuándo usar cada estrategia y proporciona ejemplos completos de ambas.

> **[PREREQUISITO]** Familiaridad con `@SpringBootTest`, JUnit 5 y `@MockBean`. Para la estrategia con Testcontainers, Docker debe estar disponible en el entorno de CI. Ver la documentación de Testcontainers para la configuración de `DockerImageName`.

## Representación visual

Las dos estrategias cubren niveles de test distintos con distintos compromisos entre velocidad y fidelidad.

```
Nivel de test        │ Estrategia SCK           │ Velocidad │ Fidelidad
─────────────────────┼──────────────────────────┼───────────┼──────────
Unitario             │ @MockBean DiscoveryClient │ Muy alta  │ Baja
                     │ @Value con @TestPropertySource│ Muy alta │ Media
─────────────────────┼──────────────────────────┼───────────┼──────────
Integración          │ @EnableKubernetesMock     │ Alta      │ Media
(mock API server)    │ Client + KubernetesMock   │           │
                     │ Server (WireMock)         │           │
─────────────────────┼──────────────────────────┼───────────┼──────────
Integración real     │ Testcontainers K3s/kind   │ Media     │ Alta
(cluster K8s efímero)│ + cliente real SCK        │           │

Recomendación: la pirámide de tests aplicada a SCK:
  - 70% unitarios con @MockBean / @TestPropertySource
  - 25% integración con @EnableKubernetesMockClient
  -  5% integración real con Testcontainers K3s
```

> **[CONCEPTO]** `@EnableKubernetesMockClient` activa un servidor WireMock embebido que simula las respuestas del Kubernetes API server. Los tests pueden registrar ConfigMaps y Services en este servidor mock y verificar que la aplicación los carga correctamente sin necesidad de un cluster real.

## Ejemplo central

El ejemplo cubre las tres estrategias: test unitario con `@MockBean`, test de integración con `@EnableKubernetesMockClient`, y test de integración real con Testcontainers K3s.

**pom.xml** (dependencias de test):

```xml
<dependencies>
  <!-- Dependencias principales de la aplicación -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- Dependencias de test -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <!-- Mock del cliente K8s para @EnableKubernetesMockClient -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-kubernetes-fabric8-client-test</artifactId>
    <scope>test</scope>
  </dependency>
  <!-- Testcontainers para tests de integración reales -->
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>k3s</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

**Estrategia 1 — Test unitario con @MockBean** (sin contexto K8s):

```java
package com.ejemplo.catalogo;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.test.context.TestPropertySource;
import java.util.List;
import static org.mockito.Mockito.when;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    properties = {
        "spring.cloud.kubernetes.enabled=false",   // Desactiva SCK completamente
        "spring.config.import="                    // Evita importación K8s
    }
)
@TestPropertySource(properties = {
    "catalogo.max-resultados=50",
    "catalogo.cache-ttl-segundos=600"
})
class CatalogoServiceTest {

    @MockBean
    private DiscoveryClient discoveryClient;

    @Autowired
    private CatalogoService catalogoService;

    @Test
    void debeUsarConfiguracionDeTestProperties() {
        assertThat(catalogoService.getMaxResultados()).isEqualTo(50);
    }

    @Test
    void debeDelegarDescubrimientoAlDiscoveryClient() {
        ServiceInstance instanciaMock = mockServiceInstance("inventario-service", "10.0.0.1", 8080);
        when(discoveryClient.getInstances("inventario-service"))
            .thenReturn(List.of(instanciaMock));

        List<ServiceInstance> instancias = catalogoService.obtenerInstanciasInventario();

        assertThat(instancias).hasSize(1);
        assertThat(instancias.get(0).getHost()).isEqualTo("10.0.0.1");
    }

    private ServiceInstance mockServiceInstance(String serviceId, String host, int port) {
        ServiceInstance mock = org.mockito.Mockito.mock(ServiceInstance.class);
        when(mock.getServiceId()).thenReturn(serviceId);
        when(mock.getHost()).thenReturn(host);
        when(mock.getPort()).thenReturn(port);
        return mock;
    }
}
```

**Estrategia 2 — Test de integración con @EnableKubernetesMockClient**:

```java
package com.ejemplo.catalogo;

import io.fabric8.kubernetes.api.model.ConfigMapBuilder;
import io.fabric8.kubernetes.client.KubernetesClient;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.kubernetes.fabric8.test.EnableKubernetesMockClient;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@EnableKubernetesMockClient(crud = true)   // WireMock con soporte CRUD sobre K8s API
class ConfigMapPropertySourceTest {

    // KubernetesClient conectado al servidor WireMock embebido
    static KubernetesClient mockClient;

    @Value("${catalogo.max-resultados:0}")
    private int maxResultados;

    @Test
    void debeLeerPropiedadesDelConfigMapMock() {
        // Registrar un ConfigMap en el servidor WireMock embebido
        mockClient.configMaps()
            .inNamespace("default")
            .create(new ConfigMapBuilder()
                .withNewMetadata()
                    .withName("catalogo-service")
                    .withNamespace("default")
                .endMetadata()
                .withData(Map.of("application.properties",
                    "catalogo.max-resultados=75\ncatalogo.cache-ttl-segundos=900"))
                .build()
            );

        // La propiedad debe ser accesible via Spring Environment
        assertThat(maxResultados).isEqualTo(75);
    }
}
```

**Estrategia 3 — Test de integración real con Testcontainers K3s**:

```java
package com.ejemplo.catalogo;

import io.fabric8.kubernetes.api.model.ConfigMapBuilder;
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.KubernetesClientBuilder;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.testcontainers.k3s.K3sContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
class K3sIntegrationTest {

    @Container
    static K3sContainer k3s = new K3sContainer(
        DockerImageName.parse("rancher/k3s:v1.27.4-k3s1")
    );

    static KubernetesClient client;

    @BeforeAll
    static void inicializarCliente() {
        // Obtener kubeconfig del contenedor K3s y crear cliente real
        client = new KubernetesClientBuilder()
            .withConfig(io.fabric8.kubernetes.client.Config.fromKubeconfig(
                k3s.getKubeConfigYaml()
            ))
            .build();
    }

    @Test
    void debeCrearYLeerConfigMapEnK3s() {
        // Crear un ConfigMap real en el cluster K3s efímero
        client.configMaps()
            .inNamespace("default")
            .create(new ConfigMapBuilder()
                .withNewMetadata()
                    .withName("test-config")
                    .withNamespace("default")
                .endMetadata()
                .withData(Map.of("clave", "valor-de-prueba"))
                .build()
            );

        // Leer el ConfigMap de vuelta
        var configMap = client.configMaps()
            .inNamespace("default")
            .withName("test-config")
            .get();

        assertThat(configMap).isNotNull();
        assertThat(configMap.getData()).containsEntry("clave", "valor-de-prueba");
    }

    @Test
    void debeVerificarRecargarConfigMapEnCaliente() throws InterruptedException {
        // Test de recarga: crear ConfigMap, verificar valor inicial,
        // actualizar ConfigMap, verificar nuevo valor propagado
        client.configMaps()
            .inNamespace("default")
            .create(new ConfigMapBuilder()
                .withNewMetadata()
                    .withName("reload-test")
                    .withNamespace("default")
                .endMetadata()
                .withData(Map.of("application.properties", "mi.valor=inicial"))
                .build()
            );

        // Actualizar el ConfigMap
        Thread.sleep(2000);  // Esperar propagación
        client.configMaps()
            .inNamespace("default")
            .withName("reload-test")
            .edit(cm -> {
                cm.getData().put("application.properties", "mi.valor=actualizado");
                return cm;
            });

        // Verificar que el valor se propagó (en un test real, comprobar via HTTP endpoint)
        var updated = client.configMaps()
            .inNamespace("default")
            .withName("reload-test")
            .get();

        assertThat(updated.getData().get("application.properties"))
            .contains("mi.valor=actualizado");
    }
}
```

> **[EXAMEN]** Pregunta frecuente: "¿Cuándo usar `@EnableKubernetesMockClient` vs Testcontainers K3s para tests de SCK?" `@EnableKubernetesMockClient` es adecuado para tests de integración rápidos que verifican que la aplicación carga propiedades de un ConfigMap correctamente. Testcontainers K3s es necesario cuando se necesita verificar comportamientos que dependen de internals de Kubernetes: la propagación real del watch de ConfigMaps, el comportamiento del RBAC, o la integración con el ServiceAccount. La regla práctica: si el test puede verificarse con un mock del API server, usar `@EnableKubernetesMockClient`; si necesita el comportamiento real de Kubernetes, usar K3s.

## Tabla de elementos clave

| Componente | Estrategia | Descripción |
|---|---|---|
| `@EnableKubernetesMockClient` | Mock API server | Activa WireMock embebido simulando la Kubernetes API |
| `KubernetesMockServer` | Mock API server | Servidor WireMock accesible desde el test para registrar respuestas |
| `spring.cloud.kubernetes.enabled=false` | Unit test | Desactiva SCK completamente para tests sin contexto K8s |
| `K3sContainer` (Testcontainers) | Integración real | Cluster K3s efímero en Docker para tests de alta fidelidad |
| `@MockBean DiscoveryClient` | Unit test | Mock del DiscoveryClient para tests de lógica de servicio |
| `@TestPropertySource` | Unit test | Proporciona propiedades de test sin necesitar ConfigMap |

## Buenas y malas prácticas

**Hacer:**

- Desactivar SCK con `spring.cloud.kubernetes.enabled=false` en todos los tests unitarios que no necesitan la integración K8s: evita que el contexto de Spring intente conectarse a un cluster inexistente y acelera el arranque del test en un factor de 3-5x.
- Usar `@TestPropertySource` o `application-test.properties` para proporcionar los valores de configuración en tests unitarios: es la forma más simple y directa, sin necesidad de mocks del API server.
- Incluir al menos un test de integración con `@EnableKubernetesMockClient` que verifique la carga del ConfigMap y la resolución de propiedades end-to-end: detecta errores de configuración (namespace incorrecto, nombre de clave erróneo) antes de llegar a producción.
- Reservar Testcontainers K3s para los tests que verifican comportamientos específicos de Kubernetes (recarga de ConfigMap vía watch, RBAC, leader election): son más lentos (arranque del cluster ~30s) y consumen más recursos.

**Evitar:**

- Usar `@SpringBootTest` sin `spring.cloud.kubernetes.enabled=false` en tests unitarios que no necesitan SCK: el contexto intentará conectarse al API server y fallará con `ConnectException` en entornos sin Kubernetes (CI, máquinas locales sin kubeconfig).
- Copiar el kubeconfig de producción en el entorno de CI para tests de integración: usar Testcontainers K3s o kind para crear un cluster efímero. El kubeconfig de producción en CI es un riesgo de seguridad grave.
- Omitir tests de recarga dinámica de ConfigMap: es uno de los comportamientos más frecuentemente rotos por upgrades de versión de SCK o cambios de RBAC, y solo puede verificarse con un cluster Kubernetes real o un mock de alta fidelidad.

---

← [9.11 Leader Election con Spring Cloud Kubernetes](sc-kubernetes-leader.md) | [Índice (README.md)](README.md) | [10.1 Arquitectura CDC y modelo Consumer-Driven Contracts](sc-contract-fundamentos.md) →
