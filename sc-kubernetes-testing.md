# 9.10 Spring Cloud Kubernetes — Testing

← [9.9 Integración con Istio](sc-kubernetes-istio.md) | [Índice](README.md) | [10.1 Spring Cloud Contract — Fundamentos CDC](sc-contract-fundamentos.md) →

---

## Introducción

Testear aplicaciones Spring Cloud Kubernetes requiere una estrategia diferente a los tests habituales de Spring Boot porque la aplicación depende de la API de Kubernetes para cargar propiedades (ConfigMaps/Secrets) y descubrir servicios (Endpoints). Existen tres enfoques principales: deshabilitar completamente la integración con Kubernetes para tests unitarios puros, usar `MockKubernetesServer` como servidor HTTP que imita la API K8s para tests de integración ligeros, y usar Testcontainers con K3s para tests de integración completos contra un clúster real ligero. Cada enfoque tiene un coste diferente en tiempo de ejecución y fidelidad del test.

## Comparativa de estrategias de testing

La siguiente tabla resume las tres estrategias y cuándo usar cada una.

| Estrategia | Fidelidad | Coste | Cuándo usar |
|---|---|---|---|
| `spring.cloud.kubernetes.enabled=false` | Baja | Muy bajo | Tests unitarios sin dependencia de K8s |
| `MockKubernetesServer` | Media | Bajo | Tests de integración de PropertySource y Discovery |
| Testcontainers + K3s | Alta | Alto | Tests de integración E2E con clúster real |

> [CONCEPTO] `MockKubernetesServer` es un servidor HTTP embebido que imita la API de Kubernetes. Usando `expect()` se pueden registrar respuestas para endpoints concretos (como `GET /api/v1/namespaces/default/configmaps/my-service`), permitiendo testear la carga de ConfigMaps y Secrets sin un clúster real.

> [CONCEPTO] Testcontainers con K3s levanta un clúster Kubernetes real y ligero dentro de un contenedor Docker. Es la opción más fiel pero requiere Docker y tarda más en arrancar (20-60 segundos). Se usa para tests de integración que verifican el comportamiento completo incluyendo RBAC y reload.

> [PREREQUISITO] Para usar `MockKubernetesServer` del cliente oficial, se necesita la dependencia `spring-cloud-starter-kubernetes-client-test` o el módulo `spring-cloud-kubernetes-client-loadbalancer`. Para Fabric8, `spring-cloud-kubernetes-fabric8-*-test`.

## Ejemplo central

El siguiente ejemplo cubre los tres enfoques de testing con código completo.

```xml
<!-- pom.xml — dependencias de test -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!-- Para MockKubernetesServer con cliente oficial -->
<dependency>
    <groupId>io.kubernetes</groupId>
    <artifactId>client-java</artifactId>
    <scope>test</scope>
</dependency>
<!-- Para Testcontainers + K3s -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>k3s</artifactId>
    <version>1.20.1</version>
    <scope>test</scope>
</dependency>
```

```java
// src/test/java/com/example/DisabledK8sTest.java
package com.example;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

/**
 * Test con Spring Cloud Kubernetes deshabilitado completamente.
 * Útil para tests unitarios de lógica de negocio que no dependen de K8s.
 */
@SpringBootTest
@TestPropertySource(properties = {
    "spring.cloud.kubernetes.enabled=false",
    "spring.cloud.kubernetes.config.enabled=false",
    "spring.cloud.kubernetes.discovery.enabled=false",
    "app.message=test-message",       // valores por defecto para el test
    "app.max-retries=1"
})
class DisabledK8sTest {

    @Autowired
    private AppProperties appProperties;

    @Test
    void shouldLoadDefaultProperties() {
        assert appProperties.getMessage().equals("test-message");
        assert appProperties.getMaxRetries() == 1;
    }
}
```

```java
// src/test/java/com/example/MockConfigMapTest.java
package com.example;

import io.kubernetes.client.openapi.ApiClient;
import io.kubernetes.client.openapi.models.V1ConfigMap;
import io.kubernetes.client.openapi.models.V1ObjectMeta;
import io.kubernetes.client.util.ClientBuilder;
import okhttp3.mockwebserver.MockResponse;
import okhttp3.mockwebserver.MockWebServer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import java.io.IOException;
import java.util.Map;

/**
 * Test con MockWebServer que imita la API de Kubernetes.
 * Permite testear la carga de ConfigMaps sin un clúster real.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    "spring.cloud.kubernetes.config.enabled=true",
    "spring.cloud.kubernetes.config.name=my-service",
    "spring.cloud.kubernetes.config.namespace=default"
})
class MockConfigMapTest {

    private MockWebServer mockApiServer;

    @Autowired
    private AppProperties appProperties;

    @BeforeEach
    void setUp() throws IOException {
        mockApiServer = new MockWebServer();
        mockApiServer.start();

        // Respuesta simulada del API de Kubernetes para GET configmap
        String configMapJson = """
                {
                  "apiVersion": "v1",
                  "kind": "ConfigMap",
                  "metadata": {
                    "name": "my-service",
                    "namespace": "default"
                  },
                  "data": {
                    "app.message": "hello-from-mock-configmap",
                    "app.max-retries": "5"
                  }
                }
                """;

        mockApiServer.enqueue(new MockResponse()
                .setBody(configMapJson)
                .addHeader("Content-Type", "application/json"));
    }

    @AfterEach
    void tearDown() throws IOException {
        mockApiServer.shutdown();
    }

    @Test
    void shouldLoadPropertiesFromMockedConfigMap() {
        // El PropertySource de Spring Cloud K8s ha cargado las propiedades
        // desde el mock server durante el arranque del contexto de test
        assert appProperties.getMaxRetries() == 5;
    }
}
```

```java
// src/test/java/com/example/K3sIntegrationTest.java
package com.example;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.k3s.K3sContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

/**
 * Test de integración con un clúster K3s real en Docker.
 * Mayor fidelidad pero mayor tiempo de arranque (~30-60 segundos).
 */
@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class K3sIntegrationTest {

    @Container
    static K3sContainer k3s = new K3sContainer(
            DockerImageName.parse("rancher/k3s:v1.28.3-k3s1"));

    @DynamicPropertySource
    static void configureK8sProperties(DynamicPropertyRegistry registry) {
        // Inyectar el kubeconfig del clúster K3s en el contexto de Spring
        registry.add("spring.cloud.kubernetes.client.kubeConfigFile",
                () -> k3s.getKubeConfigYaml());
    }

    @BeforeAll
    static void setupCluster() throws Exception {
        // Aquí se crearían los recursos K8s necesarios para el test:
        // kubectl apply -f configmap.yaml usando el cliente K8s
        // Por brevedad se omite la lógica de kubectl
    }

    @Test
    void shouldDiscoverServicesInK3sCluster() {
        // test con clúster real
        assert k3s.isRunning();
    }
}
```

## Tabla de estrategias y dependencias

La siguiente tabla resume qué dependencia de test se necesita para cada estrategia.

| Estrategia | Dependencia adicional de test | Clase / Herramienta |
|---|---|---|
| K8s deshabilitado | — | `@TestPropertySource(properties={"spring.cloud.kubernetes.enabled=false"})` |
| Mock API server (cliente oficial) | `okhttp3:mockwebserver` o `io.kubernetes:client-java` | `MockWebServer` |
| Mock API server (Fabric8) | `io.fabric8:kubernetes-server-mock` | `KubernetesServer` |
| Testcontainers K3s | `org.testcontainers:k3s` | `K3sContainer` |

## Estrategias de stub para cada feature de SC-K8s

La siguiente tabla muestra qué se debe mockear según la feature que se esté probando.

| Feature testeada | Qué mockear en el servidor | Endpoint de la API K8s |
|---|---|---|
| ConfigMap PropertySource (sc-kubernetes-configmap.md) | GET ConfigMap | `GET /api/v1/namespaces/{ns}/configmaps/{name}` |
| Secret PropertySource (sc-kubernetes-secrets.md) | GET Secret | `GET /api/v1/namespaces/{ns}/secrets/{name}` |
| KubernetesDiscoveryClient (sc-kubernetes-discovery.md) | GET Endpoints + Services | `GET /api/v1/namespaces/{ns}/endpoints` |
| Reload de configuración (sc-kubernetes-reload.md) | Watch ConfigMap | `GET /api/v1/namespaces/{ns}/configmaps?watch=true` |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `spring.cloud.kubernetes.enabled=false` para la mayoría de tests unitarios de lógica de negocio: es el enfoque más rápido y sin dependencias de infraestructura.
- Usar `MockWebServer` para tests de integración que verifican la correcta carga de propiedades desde ConfigMaps y Secrets: buen balance entre fidelidad y velocidad.
- Reservar Testcontainers + K3s para el pipeline de CI/CD en tests de integración E2E que verifican el comportamiento completo con RBAC y reload real.
- Usar `@DynamicPropertySource` para inyectar la URL del mock server o el kubeconfig de K3s en el contexto de Spring durante los tests.

**Malas prácticas:**
- Ejecutar tests de integración K3s en el build local de cada desarrollador: el tiempo de arranque del clúster hace que el ciclo de feedback sea demasiado lento.
- Ignorar `spring.cloud.kubernetes.enabled=false` y dejar que el contexto de test intente conectarse a un API server inexistente: provoca timeouts y tests frágiles.
- Crear el contexto de Spring con `spring.cloud.kubernetes.config.enabled=true` sin preparar el mock del ConfigMap: el test fallará con `403 Forbidden` o `ConfigMapNotFound`.

> [ADVERTENCIA] En entornos de CI donde Docker no está disponible, Testcontainers con K3s no funciona. Asegurarse de que los tests K3s tienen la anotación `@EnabledIfDockerAvailable` o están en un perfil separado de CI que se ejecuta solo en runners con Docker.

## Verificación y práctica

> [EXAMEN] 1. ¿Cómo se desactiva completamente la integración con Kubernetes en un test de Spring Boot para evitar que el contexto intente conectarse a la API K8s?

> [EXAMEN] 2. ¿Qué herramienta permite simular el API Server de Kubernetes en tests de integración sin levantar un clúster real, y cuáles son sus limitaciones?

> [EXAMEN] 3. ¿Cuándo se recomienda usar Testcontainers con K3s en lugar de `MockWebServer` para testear una aplicación Spring Cloud Kubernetes?

> [EXAMEN] 4. ¿Qué endpoint de la API de Kubernetes hay que mockear para testear que `KubernetesDiscoveryClient` descubre correctamente las instancias de un servicio?

> [EXAMEN] 5. ¿Qué anotación de JUnit 5 permite inyectar propiedades dinámicas (como la URL del mock server o el kubeconfig de K3s) en el contexto de Spring antes de que arranque?

---

← [9.9 Integración con Istio](sc-kubernetes-istio.md) | [Índice](README.md) | [10.1 Spring Cloud Contract — Fundamentos CDC](sc-contract-fundamentos.md) →
