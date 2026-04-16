# 9.8 Load Balancer con Kubernetes

← [9.7 Reactive DiscoveryClient y variantes de implementación](sc-kubernetes-discovery-reactive.md) | [Índice (README.md)](README.md) | [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md) →

---

## Introducción

Kubernetes ya incluye balanceo de carga a nivel de plataforma: el kube-proxy distribuye el tráfico entre los pods detrás de un `Service ClusterIP`. Sin embargo, este balanceo opera a nivel de red (iptables o ipvs) y es opaco para la aplicación: no aplica políticas de retry, no tiene en cuenta el estado de salud de los pods desde la perspectiva de la aplicación, y no permite estrategias de routing avanzadas (canary, versioned routing). Spring Cloud LoadBalancer integrado con `KubernetesDiscoveryClient` opera a nivel de cliente (client-side load balancing): la aplicación obtiene la lista de instancias disponibles y elige una según el algoritmo configurado, pudiendo excluir instancias no-ready, aplicar circuit breakers, y usar metadatos del Service para routing inteligente. `spring.cloud.kubernetes.loadbalancer.mode` permite elegir entre resolver al Service ClusterIP (delegando el routing final a kube-proxy) o directamente a los IPs de los pods (bypasando kube-proxy).

> **[PREREQUISITO]** El LoadBalancer de SCK se construye sobre `KubernetesDiscoveryClient`. Asegurarse de que el RBAC incluye `get`/`list`/`watch` sobre `services`, `endpoints` y `pods` (ver [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md)).

## Representación visual

Los dos modos de balanceo tienen implicaciones operativas distintas. El diagrama compara el flujo de resolución en cada modo.

```
Modo SERVICE (default)
┌──────────────────────────────────────────────────────────┐
│  Aplicación                                              │
│    │  getInstances("catalogo-service")                   │
│    ▼                                                     │
│  KubernetesServiceInstanceListSupplier                   │
│    │  Consulta Kubernetes Services API                   │
│    │  Retorna: ClusterIP del Service (ej. 10.96.5.10)    │
│    ▼                                                     │
│  LoadBalancer selecciona instancia (RoundRobin)          │
│    │  HTTP GET http://10.96.5.10:8080/api/productos      │
│    ▼                                                     │
│  kube-proxy distribuye a uno de los pods backend        │
└──────────────────────────────────────────────────────────┘

Modo POD
┌──────────────────────────────────────────────────────────┐
│  Aplicación                                              │
│    │  getInstances("catalogo-service")                   │
│    ▼                                                     │
│  KubernetesServiceInstanceListSupplier (modo POD)        │
│    │  Consulta Kubernetes Endpoints API                  │
│    │  Retorna: IPs individuales de cada pod              │
│    │  (ej. 172.17.0.5, 172.17.0.6, 172.17.0.7)          │
│    ▼                                                     │
│  LoadBalancer selecciona pod directamente                │
│    │  HTTP GET http://172.17.0.5:8080/api/productos      │
│    ▼                                                     │
│  Pod destino (sin pasar por kube-proxy)                  │
└──────────────────────────────────────────────────────────┘
```

> **[CONCEPTO]** En modo POD, SCK consulta la `Endpoints API` de Kubernetes (no solo `Service ClusterIP`) para obtener los IPs individuales de los pods que forman parte del Service. Esto permite que el LoadBalancer aplique algoritmos de selección más ricos (weighted, zone-aware) y excluya pods no-ready sin depender de kube-proxy. La Endpoints API es el recurso K8s que registra las IPs de los pods activos detrás de un Service.

> **[CONCEPTO]** `KubernetesServiceInstanceListSupplier` es el componente de SCK que implementa `ServiceInstanceListSupplier` de Spring Cloud LoadBalancer. Es el punto de extensión para personalizar cómo se obtiene y filtra la lista de instancias antes de aplicar el algoritmo de selección.

## Ejemplo central

El siguiente ejemplo configura Spring Cloud LoadBalancer con SCK en modo POD, con un `ServiceInstanceListSupplier` personalizado que filtra instancias por versión usando etiquetas del Service.

**pom.xml**:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: gateway-interno
  config:
    import: "kubernetes:configmap/gateway-interno-config"

spring:
  cloud:
    kubernetes:
      loadbalancer:
        mode: POD               # SERVICE (default) | POD
      discovery:
        enabled: true
    loadbalancer:
      configurations: health-check   # Activa health-check aware LB
```

**@LoadBalanced RestClient** (integración automática con KubernetesDiscoveryClient):

```java
package com.ejemplo.gateway.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;

@Configuration
public class LoadBalancerConfig {

    @Bean
    @LoadBalanced
    public RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }
}
```

**Servicio que usa el RestClient con balanceo**:

```java
package com.ejemplo.gateway.service;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class CatalogoProxy {

    private final RestClient restClient;

    public CatalogoProxy(RestClient.Builder restClientBuilder) {
        this.restClient = restClientBuilder.build();
    }

    public String obtenerProducto(Long id) {
        // "catalogo-service" es el nombre del Kubernetes Service
        // KubernetesDiscoveryClient (modo POD) resuelve al IP del pod
        return restClient.get()
            .uri("http://catalogo-service/api/productos/{id}", id)
            .retrieve()
            .body(String.class);
    }
}
```

**ServiceInstanceListSupplier personalizado** (filtrado por versión via etiquetas K8s):

```java
package com.ejemplo.gateway.config;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.core.DelegatingServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import reactor.core.publisher.Flux;
import java.util.List;
import java.util.stream.Collectors;

public class VersionAwareSupplier extends DelegatingServiceInstanceListSupplier {

    private final String targetVersion;

    public VersionAwareSupplier(ServiceInstanceListSupplier delegate, String targetVersion) {
        super(delegate);
        this.targetVersion = targetVersion;
    }

    @Override
    public String getServiceId() {
        return getDelegate().getServiceId();
    }

    @Override
    public Flux<List<ServiceInstance>> get() {
        return getDelegate().get()
            .map(instancias -> instancias.stream()
                .filter(inst -> targetVersion.equals(inst.getMetadata().get("version")))
                .collect(Collectors.toList()));
    }
}
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué diferencia hay entre el balanceo que hace kube-proxy en modo SERVICE y el balanceo client-side en modo POD?" En modo SERVICE, la aplicación envía a la ClusterIP del Service y kube-proxy (iptables) distribuye el tráfico; la aplicación no conoce los pods individuales. En modo POD, la aplicación obtiene los IPs reales de los pods via Endpoints API y elige directamente; puede excluir pods no-ready o aplicar algoritmos personalizados. La diferencia práctica: en modo POD, un pod que falla health checks pero sigue en la Endpoints list brevemente puede ser excluido más rápido por el LoadBalancer client-side. Para Spring Cloud LoadBalancer — algoritmos de balanceo (RoundRobin, WeightedRandom), ver la documentación del módulo sc-loadbalancer.

## Tabla de elementos clave

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.loadbalancer.mode` | enum | `SERVICE` | `SERVICE` (ClusterIP) o `POD` (IPs de pods via Endpoints API) |
| `spring.cloud.loadbalancer.configurations` | String | — | Activa configuraciones adicionales: `health-check`, `zone-preference` |
| `KubernetesServiceInstanceListSupplier` | Bean | — | Implementación de ServiceInstanceListSupplier para Kubernetes |

## Buenas y malas prácticas

**Hacer:**

- Usar modo `SERVICE` (default) para la mayoría de casos: el Service ClusterIP de Kubernetes ya gestiona la disponibilidad de pods y simplifica la configuración RBAC (no necesita permisos sobre `endpoints`).
- Usar modo `POD` cuando se necesite routing por versión, canary deployments, o cuando el circuit breaker de Spring Cloud necesite excluir pods específicos con mayor velocidad de respuesta.
- Combinar con `spring.cloud.loadbalancer.configurations=health-check` para que el LoadBalancer excluya instancias que fallen el health check de Actuator antes de enviar tráfico.

**Evitar:**

- Usar modo `POD` en clusters con kube-proxy en modo iptables sin `PodDisruptionBudget`: si un pod termina, puede pasar hasta varios segundos hasta que la Endpoints API se actualice y el LoadBalancer client-side deje de enviarle tráfico.
- Implementar un `ServiceInstanceListSupplier` personalizado que haga llamadas bloqueantes: el LoadBalancer de Spring Cloud es reactivo internamente; las operaciones bloqueantes deben ejecutarse en un scheduler separado.

---

← [9.7 Reactive DiscoveryClient y variantes de implementación](sc-kubernetes-discovery-reactive.md) | [Índice (README.md)](README.md) | [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md) →
