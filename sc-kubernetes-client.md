# 9.9 Kubernetes Java Client: fabric8 vs cliente oficial

← [9.8 Load Balancer con Kubernetes](sc-kubernetes-loadbalancer.md) | [Índice (README.md)](README.md) | [9.10 Health Indicators de Spring Cloud Kubernetes](sc-kubernetes-health.md) →

---

## Introducción

Spring Cloud Kubernetes abstrae la interacción con la Kubernetes API, pero lo hace usando uno de dos clientes Java de Kubernetes: el cliente de Fabric8 (`io.fabric8:kubernetes-client`) o el cliente oficial de Kubernetes (`io.kubernetes:client-java`). La elección del cliente determina qué starter se usa (ver [9.1 Setup y starters](sc-kubernetes-setup.md)), qué `DiscoveryClient` se autoconfigurar, y qué propiedades de conexión son relevantes. En producción, ambos clientes detectan automáticamente si la aplicación corre dentro del cluster (usando la `in-cluster config` del ServiceAccount inyectado por Kubernetes) o fuera (leyendo `kubeconfig` del fichero `$HOME/.kube/config`). La elección entre ambos clientes no es trivial: fabric8 ofrece un DSL fluente más familiar para developers de Spring, mientras que el cliente oficial ofrece informers nativos y mejor rendimiento en clusters grandes.

> **[ADVERTENCIA]** El `kubeconfig` en `$HOME/.kube/config` es solo para entornos locales de desarrollo. En producción (dentro del cluster), Kubernetes inyecta automáticamente las credenciales del ServiceAccount en `/var/run/secrets/kubernetes.io/serviceaccount/`. SCK detecta en qué entorno se ejecuta y usa la configuración adecuada; no es necesario (ni recomendable) copiar un kubeconfig dentro de la imagen Docker.

## Representación visual

El diagrama muestra la jerarquía de decisión para la configuración del cliente y los mecanismos de detección de entorno.

```
¿Dónde se ejecuta la aplicación?
              │
    ┌─────────┴──────────┐
    │                    │
  Dentro del cluster   Fuera del cluster (local/CI)
    │                    │
    ▼                    ▼
In-cluster config     kubeconfig
/var/run/secrets/     $HOME/.kube/config
kubernetes.io/        o KUBECONFIG env var
serviceaccount/
  ├── token           SCK lo detecta automáticamente
  ├── ca.crt          leyendo el ServiceAccount token
  └── namespace       primero (si existe) y kubeconfig
                      como fallback

┌─────────────────────────────────────────────────────────┐
│              Fabric8 KubernetesClient                   │
│  DSL fluente: client.configMaps().inNamespace("ns").get()
│  Beans: KubernetesClient (autoconfigurado)              │
│  Timeouts: connectionTimeout, requestTimeout            │
│  Starter: kubernetes-fabric8[-all]                      │
├─────────────────────────────────────────────────────────┤
│           Official Kubernetes Java Client               │
│  ApiClient + CoreV1Api                                  │
│  Informers nativos: SharedInformerFactory               │
│  Bean: ApiClient (autoconfigurado)                      │
│  Starter: kubernetes-client                             │
└─────────────────────────────────────────────────────────┘
```

> **[CONCEPTO]** `kubeconfig` es el fichero de configuración de acceso al cluster de Kubernetes, típicamente ubicado en `$HOME/.kube/config` en entornos locales. Contiene: endpoints del API server, certificados de autenticación, tokens, y contextos (cluster + usuario + namespace). SCK detecta su existencia automáticamente y lo usa cuando no hay in-cluster config disponible. La estructura interna del fichero es responsabilidad del equipo de plataforma; los developers solo deben saber dónde vive y que SCK lo lee sin configuración adicional.

## Ejemplo central

El siguiente ejemplo muestra cómo configurar las propiedades de conexión para ambos clientes y cómo acceder al bean del cliente Java directamente para operaciones personalizadas sobre la API de Kubernetes.

**pom.xml — Opción Fabric8**:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
</dependency>
```

**pom.xml — Opción Cliente Oficial**:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
</dependency>
```

**application.yml — Configuración Fabric8**:

```yaml
spring:
  application:
    name: admin-service
  config:
    import: "kubernetes:configmap/admin-config"

spring:
  cloud:
    kubernetes:
      client:
        namespace: produccion
        connection-timeout: 5000       # ms — timeout de conexión al API server
        request-timeout: 10000         # ms — timeout de cada request HTTP
        trust-certs: false             # true solo en desarrollo con certs auto-firmados
```

**application.yml — Configuración Cliente Oficial**:

```yaml
spring:
  cloud:
    kubernetes:
      client:
        namespace: produccion
```

**Uso directo del KubernetesClient (Fabric8)** para operaciones avanzadas:

```java
package com.ejemplo.admin.service;

import io.fabric8.kubernetes.api.model.ConfigMap;
import io.fabric8.kubernetes.api.model.ConfigMapBuilder;
import io.fabric8.kubernetes.client.KubernetesClient;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
public class ConfigMapAdminService {

    private final KubernetesClient kubernetesClient;

    public ConfigMapAdminService(KubernetesClient kubernetesClient) {
        this.kubernetesClient = kubernetesClient;
    }

    public ConfigMap leerConfigMap(String nombre, String namespace) {
        // DSL fluente de Fabric8
        return kubernetesClient.configMaps()
            .inNamespace(namespace)
            .withName(nombre)
            .get();
    }

    public void crearOActualizarConfigMap(String nombre, String namespace, Map<String, String> datos) {
        ConfigMap configMap = new ConfigMapBuilder()
            .withNewMetadata()
                .withName(nombre)
                .withNamespace(namespace)
            .endMetadata()
            .withData(datos)
            .build();

        kubernetesClient.configMaps()
            .inNamespace(namespace)
            .createOrReplace(configMap);
    }

    public void listarPodsDeployment(String namespace, String appLabel) {
        kubernetesClient.pods()
            .inNamespace(namespace)
            .withLabel("app", appLabel)
            .list()
            .getItems()
            .forEach(pod -> System.out.println(
                pod.getMetadata().getName() + " — " + pod.getStatus().getPhase()
            ));
    }
}
```

**Uso directo del ApiClient (Cliente Oficial)** para operaciones avanzadas:

```java
package com.ejemplo.admin.service;

import io.kubernetes.client.openapi.ApiClient;
import io.kubernetes.client.openapi.ApiException;
import io.kubernetes.client.openapi.apis.CoreV1Api;
import io.kubernetes.client.openapi.models.V1ConfigMap;
import org.springframework.stereotype.Service;

@Service
public class ConfigMapOfficialClientService {

    private final ApiClient apiClient;

    public ConfigMapOfficialClientService(ApiClient apiClient) {
        this.apiClient = apiClient;
    }

    public V1ConfigMap leerConfigMap(String nombre, String namespace) throws ApiException {
        CoreV1Api api = new CoreV1Api(apiClient);
        return api.readNamespacedConfigMap(nombre, namespace).execute();
    }
}
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué ventaja tiene el cliente oficial de Kubernetes frente a Fabric8 para aplicaciones SCK en producción?" La principal ventaja es el uso de informers nativos (`SharedInformerFactory`), que mantienen un cache local sincronizado con el API server mediante conexiones WebSocket persistentes. Esto reduce la latencia de `getInstances()` de ~100ms (llamada HTTP síncrona) a ~1ms (lectura de cache), y reduce la carga en el API server en clusters grandes. Fabric8 también tiene watchers, pero su integración con Spring Cloud es menos directa.

## Tabla de elementos clave

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.client.namespace` | String | autodetectado | Override del namespace de operación |
| `spring.cloud.kubernetes.client.connection-timeout` | int (ms) | `10000` | Timeout de conexión al API server (fabric8) |
| `spring.cloud.kubernetes.client.request-timeout` | int (ms) | `10000` | Timeout de cada request HTTP al API server (fabric8) |
| `spring.cloud.kubernetes.client.trust-certs` | boolean | `false` | Confiar en certificados auto-firmados del API server (solo dev) |
| `KUBECONFIG` | env var | `$HOME/.kube/config` | Ruta al fichero kubeconfig para entornos locales |

## Comparación detallada: fabric8 vs cliente oficial

La tabla siguiente resume las diferencias operativas más relevantes para la selección del starter en un proyecto Spring Cloud Kubernetes.

| Aspecto | Fabric8 | Cliente Oficial |
|---|---|---|
| Starter | `kubernetes-fabric8[-all]` | `kubernetes-client` |
| Bean autoconfigurado | `KubernetesClient` | `ApiClient` |
| DSL | Fluente (builders, lambdas) | Generado de OpenAPI spec |
| DiscoveryClient | `KubernetesDiscoveryClient` | `KubernetesInformerDiscoveryClient` |
| Cache local de servicios | No (llamada directa) | Sí (informers nativos) |
| Madurez con Spring Cloud | Alta (integración histórica) | Media (más reciente) |
| Rendimiento clusters grandes | Moderado | Alto (informers reducen carga API) |
| Impersonación | Sí (`withRequestConfig`) | Sí (via `ApiClient`) |

## Buenas y malas prácticas

**Hacer:**

- Elegir un único starter por proyecto: mezclar `kubernetes-fabric8` y `kubernetes-client` en el mismo classpath causa conflictos de autoconfiguración.
- Usar `trust-certs=false` en producción y proporcionar el CA bundle correcto via montaje de volumen o en-cluster config: confiar en todos los certificados abre la posibilidad de ataques MITM contra el API server.
- Para entornos locales de desarrollo, asegurarse de que `KUBECONFIG` o `$HOME/.kube/config` apunta al cluster correcto antes de arrancar la aplicación: SCK autodetecta el kubeconfig y podría conectarse al cluster equivocado si hay múltiples contextos configurados.

**Evitar:**

- Inyectar `KubernetesClient` o `ApiClient` en beans de dominio (servicios de negocio): crear un servicio de infraestructura dedicado que encapsule las operaciones sobre la Kubernetes API, facilitando el mocking en tests.
- Copiar el fichero kubeconfig dentro de la imagen Docker para "facilitar la configuración": es una práctica insegura que expone credenciales del cluster en la imagen. En producción siempre usar in-cluster config.

---

← [9.8 Load Balancer con Kubernetes](sc-kubernetes-loadbalancer.md) | [Índice (README.md)](README.md) | [9.10 Health Indicators de Spring Cloud Kubernetes](sc-kubernetes-health.md) →
