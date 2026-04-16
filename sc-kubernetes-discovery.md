# 9.6 KubernetesDiscoveryClient — descubrimiento de servicios

← [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) | [Índice (README.md)](README.md) | [9.7 Reactive DiscoveryClient y variantes de implementación](sc-kubernetes-discovery-reactive.md) →

---

## Introducción

En un cluster Kubernetes, los servicios ya son descubribles vía DNS (el CoreDNS del cluster resuelve `nombre-servicio.namespace.svc.cluster.local`). Sin embargo, Spring Cloud Gateway, OpenFeign y Spring Cloud LoadBalancer trabajan con la abstracción `DiscoveryClient` de Spring Cloud Commons, que devuelve una lista de `ServiceInstance` con metadatos ricos (puertos, namespace, etiquetas, estado de readiness). `KubernetesDiscoveryClient` implementa esa abstracción consultando la Kubernetes Services API en lugar de Eureka o Consul, permitiendo que el resto del ecosistema Spring Cloud funcione sin cambios de código al ejecutarse en Kubernetes. Sin este bridge, un microservicio desplegado en Kubernetes que usa `@LoadBalanced` o `@FeignClient` tendría que usar DNS directamente y perdería la integración con Spring Cloud LoadBalancer.

> **[PREREQUISITO]** Configurar el RBAC con permisos `get`/`list`/`watch` sobre `services`, `endpoints` y `pods` antes de activar el DiscoveryClient. Ver [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md). Sin esos permisos, el DiscoveryClient devuelve una lista vacía o lanza 403 en el arranque.

## Representación visual

El diagrama muestra cómo `KubernetesDiscoveryClient` consulta la Kubernetes API para construir la lista de `ServiceInstance` que Spring Cloud LoadBalancer y otros clientes consumen.

```
Aplicación Spring Cloud
    │
    ├── OpenFeign / @LoadBalanced RestClient
    │         │
    │         ▼
    │   Spring Cloud LoadBalancer
    │         │  getInstances("nombre-servicio")
    │         ▼
    │   KubernetesDiscoveryClient
    │         │
    │         ▼
    │   Kubernetes API Server
    │   GET /api/v1/namespaces/default/services/nombre-servicio
    │   GET /api/v1/namespaces/default/endpoints/nombre-servicio
    │         │
    │         ▼
    │   ServiceInstance {
    │     serviceId: "nombre-servicio"
    │     host: "10.96.0.5"  ← ClusterIP del Service K8s
    │     port: 8080
    │     metadata: {namespace: "default", label.app: "nombre-servicio"}
    │   }
    │
    └── Kubernetes Service (ClusterIP) ─► Pod(s) destino
```

> **[CONCEPTO]** Un Kubernetes `Service` de tipo `ClusterIP` es el recurso K8s que SCK expone como `ServiceInstance`. SCK resuelve la IP del Service (ClusterIP) o los IPs de los pods (via Endpoints API en modo POD del LoadBalancer). La definición del Service es responsabilidad del equipo de plataforma; SCK solo lo consulta.

> **[CONCEPTO]** `KubernetesInformerDiscoveryClient` (disponible con el starter `kubernetes-client` oficial) usa informers nativos de Kubernetes para mantener un cache local sincronizado con el API server, reduciendo la carga en el API server en clusters con muchos servicios. Ver [9.7 Reactive DiscoveryClient y variantes de implementación](sc-kubernetes-discovery-reactive.md) para comparación.

## Ejemplo central

El siguiente ejemplo muestra una aplicación que usa `KubernetesDiscoveryClient` para listar servicios y un cliente `@FeignClient` que resuelve instancias vía el DiscoveryClient de Kubernetes.

**pom.xml**:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: inventario-service
  config:
    import: "kubernetes:configmap/inventario-config"

spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        all-namespaces: false
        namespaces:
          - produccion
        service-labels:
          tier: backend              # Solo descubre servicios con esta etiqueta
        include-not-ready-addresses: false
    openfeign:
      client:
        config:
          default:
            connect-timeout: 2000
            read-timeout: 5000
```

**Clase principal con @EnableDiscoveryClient y @EnableFeignClients**:

```java
package com.ejemplo.inventario;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class InventarioServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(InventarioServiceApplication.class, args);
    }
}
```

**FeignClient que usa KubernetesDiscoveryClient via Spring Cloud LoadBalancer**:

```java
package com.ejemplo.inventario.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import java.util.List;

// "catalogo-service" es el nombre del Kubernetes Service en el namespace
@FeignClient(name = "catalogo-service")
public interface CatalogoClient {

    @GetMapping("/api/productos/{id}")
    ProductoDto getProducto(@PathVariable("id") Long id);

    @GetMapping("/api/productos")
    List<ProductoDto> listarProductos();
}
```

**Uso directo del DiscoveryClient** (para diagnóstico o lógica de routing):

```java
package com.ejemplo.inventario.service;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class ServicioDiagnostico {

    private final DiscoveryClient discoveryClient;

    public ServicioDiagnostico(DiscoveryClient discoveryClient) {
        this.discoveryClient = discoveryClient;
    }

    public List<String> listarServiciosDescubiertos() {
        return discoveryClient.getServices();
    }

    public List<ServiceInstance> obtenerInstancias(String servicioId) {
        List<ServiceInstance> instancias = discoveryClient.getInstances(servicioId);
        instancias.forEach(inst ->
            System.out.printf("Servicio: %s, Host: %s, Puerto: %d, Metadata: %s%n",
                inst.getServiceId(), inst.getHost(), inst.getPort(), inst.getMetadata())
        );
        return instancias;
    }
}
```

**Metadatos DTO**:

```java
package com.ejemplo.inventario.client;

public record ProductoDto(Long id, String nombre, Double precio) {}
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué contiene el `metadata` de un `ServiceInstance` devuelto por `KubernetesDiscoveryClient`?" Contiene las etiquetas (`labels`) del Kubernetes Service, el namespace, los puertos nombrados del Service, y opcionalmente las anotaciones. Estos metadatos permiten implementar estrategias de routing avanzadas (selección por versión, canary deployments) sin modificar el código de la aplicación.

## Tabla de elementos clave

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.discovery.enabled` | boolean | `true` | Activa KubernetesDiscoveryClient |
| `spring.cloud.kubernetes.discovery.service-name` | String | — | Nombre del Service K8s a usar como serviceId en Spring Cloud |
| `spring.cloud.kubernetes.discovery.all-namespaces` | boolean | `false` | Descubre servicios en todos los namespaces (requiere ClusterRole) |
| `spring.cloud.kubernetes.discovery.namespaces` | List | — | Lista explícita de namespaces donde buscar servicios |
| `spring.cloud.kubernetes.discovery.service-labels` | Map | — | Filtra servicios por etiquetas K8s (solo devuelve los que coincidan) |
| `spring.cloud.kubernetes.discovery.include-not-ready-addresses` | boolean | `false` | Incluye pods no ready como instancias disponibles |
| `spring.cloud.kubernetes.discovery.metadata.add-labels` | boolean | `true` | Incluye labels del Service en el metadata del ServiceInstance |
| `spring.cloud.kubernetes.discovery.metadata.add-annotations` | boolean | `false` | Incluye anotaciones del Service en el metadata |

## Buenas y malas prácticas

**Hacer:**

- Usar `service-labels` para filtrar el conjunto de servicios descubiertos en clusters grandes: sin filtrado, el DiscoveryClient carga todos los Services del namespace en memoria, lo que puede ser costoso.
- Mantener `include-not-ready-addresses=false` (default) en producción: incluir pods no-ready en el pool de balanceo provoca errores 503 cuando el LoadBalancer selecciona esas instancias.
- Asegurarse de que el nombre del `@FeignClient` coincide exactamente con el nombre del Kubernetes Service: el nombre se usa como serviceId para consultar el DiscoveryClient y luego el LoadBalancer.
- Activar `metadata.add-labels=true` para habilitar routing por versión o canary: las etiquetas `version: v2` en el Service son accesibles como metadatos del ServiceInstance.

**Evitar:**

- Usar `all-namespaces=true` sin necesidad: requiere ClusterRoleBinding (scope de cluster) y carga todos los Services de todos los namespaces, incluyendo los de sistema (`kube-system`).
- Confundir el KubernetesDiscoveryClient con el DNS de Kubernetes: SCK no reemplaza el DNS; lo complementa. Un acceso directo `http://catalogo-service/api` funciona via DNS incluso sin SCK. SCK añade la integración con Spring Cloud LoadBalancer y metadatos de instancia.
- Usar Project Reactor (Flux/Mono) con el DiscoveryClient imperativo en aplicaciones WebFlux: para entornos reactivos usar `KubernetesReactiveDiscoveryClient` que devuelve `Flux<ServiceInstance>` (ver [9.7](sc-kubernetes-discovery-reactive.md)).

---

← [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) | [Índice (README.md)](README.md) | [9.7 Reactive DiscoveryClient y variantes de implementación](sc-kubernetes-discovery-reactive.md) →
