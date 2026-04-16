# 9.7 Reactive DiscoveryClient y variantes de implementación

← [9.6 KubernetesDiscoveryClient — descubrimiento de servicios](sc-kubernetes-discovery.md) | [Índice (README.md)](README.md) | [9.8 Load Balancer con Kubernetes](sc-kubernetes-loadbalancer.md) →

---

## Introducción

Las aplicaciones Spring WebFlux operan en un modelo completamente no bloqueante, y llamar a un `DiscoveryClient` imperativo (que bloquea hasta obtener la lista de instancias) anularía los beneficios del modelo reactivo. `KubernetesReactiveDiscoveryClient` implementa `ReactiveDiscoveryClient` de Spring Cloud Commons y devuelve `Flux<ServiceInstance>`, permitiendo integrar el descubrimiento de servicios en cadenas reactivas de Project Reactor sin bloquear hilos del event loop. Este nodo también cubre `KubernetesInformerDiscoveryClient`, la variante basada en el cliente oficial de Kubernetes que usa informers nativos para mantener un cache local sincronizado, y explica cuándo elegir una implementación u otra.

> **[PREREQUISITO]** Para usar `KubernetesReactiveDiscoveryClient` es necesario que la aplicación use Spring WebFlux y tenga el starter `spring-cloud-starter-kubernetes-fabric8-all` o `spring-cloud-starter-kubernetes-client` (que incluye los beans reactivos). El RBAC descrito en [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) aplica igualmente.

## Representación visual

El diagrama compara el flujo de descubrimiento imperativo vs reactivo y las dos implementaciones disponibles en función del starter elegido.

```
Imperativo (WebMVC)                     Reactivo (WebFlux)
    │                                       │
    ▼                                       ▼
DiscoveryClient                     ReactiveDiscoveryClient
.getInstances("svc")                .getInstances("svc")
List<ServiceInstance>               Flux<ServiceInstance>
    │                                       │
    ▼                                       ▼
Bloquea hilo hasta                  No bloquea — la suscripción
obtener respuesta K8s               propaga el evento K8s
                                    al operador downstream

──────────────────────────────────────────────────────────
Implementaciones disponibles por starter:

Starter                          │ DiscoveryClient impl
─────────────────────────────────┼───────────────────────────────────
kubernetes-fabric8[-all]         │ KubernetesDiscoveryClient (fabric8)
                                 │ KubernetesReactiveDiscoveryClient
─────────────────────────────────┼───────────────────────────────────
kubernetes-client (oficial)      │ KubernetesInformerDiscoveryClient
                                 │   → usa informers nativos K8s
                                 │   → cache local sincronizado
                                 │   → menos llamadas al API server
```

> **[CONCEPTO]** `KubernetesInformerDiscoveryClient` (cliente oficial) mantiene un cache local de Services y Endpoints que se actualiza mediante informers de Kubernetes (watchers persistentes). Esto reduce la carga en el API server en clusters con muchos Services porque no hace una llamada HTTP por cada `getInstances()`, sino que sirve el resultado desde el cache. Contrapartida: el cache puede estar brevemente desactualizado (típicamente < 1 segundo).

## Ejemplo central

El siguiente ejemplo muestra una aplicación WebFlux que usa `KubernetesReactiveDiscoveryClient` directamente y un `WebClient` con balanceo reactivo integrado via `ReactorLoadBalancer`.

**pom.xml**:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: notificaciones-service

spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
    discovery:
      reactive:
        enabled: true    # Activa ReactiveDiscoveryClient
    loadbalancer:
      ribbon:
        enabled: false   # Deshabilita Ribbon si estuviera presente
```

**Uso directo de ReactiveDiscoveryClient**:

```java
package com.ejemplo.notificaciones.service;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.ReactiveDiscoveryClient;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Service
public class ServicioDescubrimientoReactivo {

    private final ReactiveDiscoveryClient reactiveDiscoveryClient;

    public ServicioDescubrimientoReactivo(ReactiveDiscoveryClient reactiveDiscoveryClient) {
        this.reactiveDiscoveryClient = reactiveDiscoveryClient;
    }

    public Flux<ServiceInstance> obtenerInstancias(String servicioId) {
        return reactiveDiscoveryClient.getInstances(servicioId);
    }

    public Mono<String> obtenerUrlPrimera(String servicioId) {
        return reactiveDiscoveryClient.getInstances(servicioId)
            .next()
            .map(inst -> "http://" + inst.getHost() + ":" + inst.getPort());
    }
}
```

**WebClient con @LoadBalanced y ReactorLoadBalancer** (integración con Spring Cloud LoadBalancer):

```java
package com.ejemplo.notificaciones.config;

import org.springframework.cloud.client.loadbalancer.reactive.ReactorLoadBalancerExchangeFilterFunction;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(ReactorLoadBalancerExchangeFilterFunction lbFunction) {
        // El nombre del servicio en la URL es resuelto por KubernetesDiscoveryClient
        return WebClient.builder()
            .filter(lbFunction)
            .build();
    }
}
```

**Controlador reactivo que consume el WebClient con balanceo**:

```java
package com.ejemplo.notificaciones.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@RestController
public class NotificacionesController {

    private final WebClient webClient;

    public NotificacionesController(WebClient webClient) {
        this.webClient = webClient;
    }

    @GetMapping("/notificaciones/usuario/{id}")
    public Mono<String> obtenerNotificaciones(@PathVariable Long id) {
        // "usuarios-service" es el nombre del Kubernetes Service
        // ReactorLoadBalancer usa KubernetesReactiveDiscoveryClient internamente
        return webClient.get()
            .uri("http://usuarios-service/api/usuarios/{id}", id)
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué diferencia hay entre `KubernetesDiscoveryClient` (fabric8) y `KubernetesInformerDiscoveryClient` (cliente oficial)?" La clave está en el mecanismo de sincronización: fabric8 hace llamadas al API server en cada `getInstances()`, mientras que el InformerDiscoveryClient mantiene un cache local actualizado via informers K8s nativos. En clusters con alta frecuencia de llamadas de descubrimiento, el InformerDiscoveryClient reduce la carga en el API server significativamente.

## Tabla de elementos clave

| Propiedad / Componente | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.discovery.reactive.enabled` | boolean | `true` (si WebFlux presente) | Activa el bean `ReactiveDiscoveryClient` |
| `KubernetesReactiveDiscoveryClient` | Bean | — | Implementación reactiva de fabric8 |
| `KubernetesInformerDiscoveryClient` | Bean | — | Implementación con cache (starter oficial) |
| `ReactorLoadBalancerExchangeFilterFunction` | Bean | — | Filter de WebClient que integra Spring Cloud LoadBalancer reactivo |
| `ReactorLoadBalancer` | Bean | — | Algoritmo de balanceo reactivo (RoundRobin por defecto) |

## Buenas y malas prácticas

**Hacer:**

- Usar `KubernetesReactiveDiscoveryClient` con `WebClient` en aplicaciones WebFlux: es la única forma de mantener el pipeline completamente no bloqueante desde el descubrimiento hasta la llamada HTTP.
- Preferir `KubernetesInformerDiscoveryClient` (starter oficial) en clusters con más de 50 servicios activos: el cache local reduce la latencia de `getInstances()` de ~100ms (llamada al API) a ~1ms (cache en memoria).
- Combinar `ReactorLoadBalancerExchangeFilterFunction` con el `WebClient` en lugar de gestionar manualmente la URL del servicio: el LoadBalancer aplica el algoritmo de selección (RoundRobin, etc.) de forma transparente.

**Evitar:**

- Mezclar `KubernetesReactiveDiscoveryClient` y `KubernetesDiscoveryClient` en la misma aplicación sin necesidad: añaden beans adicionales y pueden causar ambigüedad en la inyección si no se cualifican con `@Qualifier`.
- Llamar a `reactiveDiscoveryClient.getInstances().blockFirst()` en una aplicación WebFlux: bloquea el event loop de Netty, causando degradación de rendimiento severa bajo carga. Para Project Reactor, para uso avanzado de Flux/Mono ver la documentación de Project Reactor.
- Usar `spring.cloud.discovery.reactive.enabled=false` en aplicaciones WebFlux si se usa `@LoadBalanced` `WebClient`: el filtro de LoadBalancer reactivo depende de que el `ReactiveDiscoveryClient` esté registrado.

## Comparación de variantes de DiscoveryClient

La elección de implementación depende del starter seleccionado en 9.1 y del modelo de programación de la aplicación.

| Variante | Starter | Modelo | Cache local | Carga API server |
|---|---|---|---|---|
| `KubernetesDiscoveryClient` | fabric8[-all] | Imperativo | No (llamada directa) | Alta frecuencia |
| `KubernetesReactiveDiscoveryClient` | fabric8[-all] | Reactivo | No (llamada directa) | Alta frecuencia |
| `KubernetesInformerDiscoveryClient` | kubernetes-client | Imperativo | Sí (informers) | Baja (eventos K8s) |

---

← [9.6 KubernetesDiscoveryClient — descubrimiento de servicios](sc-kubernetes-discovery.md) | [Índice (README.md)](README.md) | [9.8 Load Balancer con Kubernetes](sc-kubernetes-loadbalancer.md) →
