# 9.4 Spring Cloud Kubernetes — Service Discovery con KubernetesDiscoveryClient

← [9.3 Secrets como PropertySource](sc-kubernetes-secrets.md) | [Índice](README.md) | [9.5 Starters: Fabric8 vs Cliente Oficial](sc-kubernetes-starters.md) →

---

## Introducción

`KubernetesDiscoveryClient` es la implementación de la interfaz `DiscoveryClient` de Spring Cloud que descubre instancias de servicios usando la API de Kubernetes en lugar de un registro externo como Eureka. En lugar de consultar a un servidor Eureka, el cliente interroga directamente los recursos `Service` y `Endpoints` del clúster para obtener la lista de instancias disponibles. Esta integración permite que el código de aplicación que usa `@LoadBalanced RestTemplate`, `WebClient` o `FeignClient` funcione sin modificaciones al migrar de Eureka a Kubernetes.

## Diagrama de descubrimiento

El siguiente diagrama muestra cómo `KubernetesDiscoveryClient` resuelve un nombre de servicio lógico en instancias concretas comparado con Eureka.

```
Eureka (pila clásica):
  FeignClient("order-service")
       │
       ▼ DiscoveryClient.getInstances("order-service")
  Eureka Server → [10.0.0.1:8080, 10.0.0.2:8080]

Kubernetes:
  FeignClient("order-service")
       │
       ▼ KubernetesDiscoveryClient.getInstances("order-service")
  K8s API Server → GET /api/v1/namespaces/default/endpoints/order-service
                 → [Pod IP 1:8080, Pod IP 2:8080]
```

> [CONCEPTO] `KubernetesDiscoveryClient` implementa exactamente la misma interfaz `DiscoveryClient` que `EurekaDiscoveryClient`. Esto significa que toda la lógica de balanceo de carga de Spring Cloud LoadBalancer, que consume `DiscoveryClient`, funciona sin cambios en el código de aplicación.

> [CONCEPTO] Cuando se trabaja con headless services (ClusterIP: None en el manifiesto del Service K8s), `KubernetesDiscoveryClient` devuelve las IPs individuales de cada pod en lugar de la IP del Service. Esto es útil para bases de datos distribuidas o caches que requieren conexión directa a pods específicos.

> [PREREQUISITO] El ServiceAccount del pod necesita los verbos `get`, `watch` y `list` sobre los recursos `services` y `endpoints` en el namespace correspondiente.

## Ejemplo central

El siguiente ejemplo muestra la configuración completa para usar `KubernetesDiscoveryClient` con un `FeignClient` y filtrado por labels, incluyendo el descubrimiento en todos los namespaces.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: api-gateway
  cloud:
    kubernetes:
      discovery:
        enabled: true
        all-namespaces: false           # true para buscar en todos los namespaces
        service-labels:
          monitored: "true"             # solo servicios con este label
        include-not-ready-addresses: false
    loadbalancer:
      enabled: true
```

```java
// src/main/java/com/example/ApiGatewayApplication.java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/client/OrderClient.java
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import java.util.List;

// "order-service" es el nombre del Service de Kubernetes
@FeignClient(name = "order-service")
public interface OrderClient {

    @GetMapping("/orders")
    List<String> getOrders();

    @GetMapping("/orders/{id}")
    String getOrder(@PathVariable("id") Long id);
}
```

```java
// src/main/java/com/example/DiscoveryController.java
package com.example;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;

@RestController
public class DiscoveryController {

    private final DiscoveryClient discoveryClient;

    public DiscoveryController(DiscoveryClient discoveryClient) {
        this.discoveryClient = discoveryClient;
    }

    @GetMapping("/services")
    public List<String> services() {
        return discoveryClient.getServices();
    }

    @GetMapping("/services/{name}")
    public List<ServiceInstance> instances(@PathVariable String name) {
        return discoveryClient.getInstances(name);
    }
}
```

## Tabla de propiedades de Discovery

La siguiente tabla resume las propiedades más relevantes para configurar el descubrimiento de servicios.

| Propiedad | Valor por defecto | Descripción |
|---|---|---|
| `spring.cloud.kubernetes.discovery.enabled` | `true` | Activa `KubernetesDiscoveryClient` |
| `spring.cloud.kubernetes.discovery.all-namespaces` | `false` | Descubre servicios en todos los namespaces; requiere ClusterRole |
| `spring.cloud.kubernetes.discovery.service-labels` | — | Mapa de etiquetas para filtrar servicios |
| `spring.cloud.kubernetes.discovery.include-not-ready-addresses` | `false` | Incluye pods no listos en la lista de instancias |
| `spring.cloud.kubernetes.discovery.namespaces` | — | Lista explícita de namespaces donde buscar (alternativa a `all-namespaces`) |

## Headless Services vs ClusterIP

Cuando un Service de Kubernetes tiene `clusterIP: None` (headless), el DNS devuelve directamente los registros A de los pods individuales en lugar de la IP del Service. `KubernetesDiscoveryClient` detecta este tipo de service y devuelve las IPs individuales de los pods, lo que permite que Spring Cloud LoadBalancer balancee directamente a nivel de pod. Con un Service de ClusterIP normal, la lista de instancias contiene una única entrada con la IP del Service, y el balanceo ocurre dentro de kube-proxy.

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `service-labels` para filtrar qué servicios son visibles para el DiscoveryClient, evitando que la aplicación intente balancear hacia servicios de infraestructura no aptos.
- Activar `all-namespaces: true` solo cuando realmente se necesita descubrimiento cross-namespace; requiere ClusterRole que amplía los permisos del pod.
- Usar `@EnableDiscoveryClient` aunque la auto-configuración lo registre automáticamente: mejora la legibilidad y documenta la intención.
- Ver la integración con Istio antes de activar discovery en entornos con service mesh (ver sc-kubernetes-istio.md).

**Malas prácticas:**
- Usar `all-namespaces: true` con un ClusterRole irrestricto: un pod comprometido podría descubrir todos los servicios del clúster, incluidos los de administración.
- Incluir `include-not-ready-addresses: true` en producción sin lógica de retry: puede enviar tráfico a pods en estado de inicialización.
- Depender de la IP del pod (headless) sin considerar que las IPs cambian cuando K8s recrea el pod.

> [ADVERTENCIA] `spring.cloud.kubernetes.discovery.all-namespaces=true` requiere un ClusterRole (no Role con scope de namespace) con verbos `list` y `watch` sobre `services` y `endpoints` en todo el clúster. Verificar con el equipo de plataforma antes de usar esta opción en producción.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué interfaz implementa `KubernetesDiscoveryClient` y cuál es la implicación para el código que ya usa `@FeignClient` con Eureka?

> [EXAMEN] 2. ¿Qué ocurre cuando `KubernetesDiscoveryClient` consulta un headless Service (clusterIP: None) comparado con un Service de ClusterIP normal?

> [EXAMEN] 3. ¿Qué permisos RBAC adicionales requiere activar `spring.cloud.kubernetes.discovery.all-namespaces=true` y por qué?

> [EXAMEN] 4. ¿Cómo se filtra `KubernetesDiscoveryClient` para que solo descubra servicios que tengan el label `monitored=true`?

> [EXAMEN] 5. ¿En qué escenario conviene desactivar `KubernetesDiscoveryClient` completamente y delegar todo el routing a otro componente del stack?

---

← [9.3 Secrets como PropertySource](sc-kubernetes-secrets.md) | [Índice](README.md) | [9.5 Starters: Fabric8 vs Cliente Oficial](sc-kubernetes-starters.md) →
