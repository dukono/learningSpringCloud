# 9.9 Spring Cloud Kubernetes — Integración con Istio

← [9.8 Leader Election](sc-kubernetes-leader.md) | [Índice](README.md) | [9.10 Testing](sc-kubernetes-testing.md) →

---

## Introducción

Istio es un service mesh que gestiona el tráfico entre microservicios a nivel de red mediante proxies Envoy (sidecar) inyectados en cada pod. Cuando Istio está activo en el clúster, existe un riesgo de doble balanceo: Spring Cloud LoadBalancer balancea desde el lado del cliente usando `KubernetesDiscoveryClient`, y simultáneamente el sidecar Envoy de Istio aplica sus propias reglas de routing y balanceo. Para evitar este conflicto, Spring Cloud Kubernetes ofrece el modo `SERVICE` en el load balancer, que delega completamente el routing a Istio y elimina el balanceo del lado del cliente.

## Diagrama: POD mode vs SERVICE mode

La siguiente comparativa ilustra el flujo de tráfico con cada modo de load balancer.

```
MODO POD (sin Istio o sin delegar):
  Llamada a "order-service"
        │
  Spring Cloud LB resuelve IPs de pods vía KubernetesDiscoveryClient
        │ elige Pod-IP-1 o Pod-IP-2
        ▼
  Envoy sidecar intercepta el tráfico (doble balanceo) ← PROBLEMA
        │
        ▼
  Pod destino

MODO SERVICE (con Istio):
  Llamada a "order-service"
        │
  Spring Cloud LB usa DNS del Service K8s (no resuelve pods individuales)
        │ → http://order-service.default.svc.cluster.local:8080
        ▼
  Envoy sidecar de Istio intercepta y aplica VirtualService/DestinationRule
        │ Istio gestiona routing, retry, mTLS, circuit breaker
        ▼
  Pod destino (seleccionado por Istio)
```

> [CONCEPTO] Con `spring.cloud.kubernetes.loadbalancer.mode=SERVICE`, Spring Cloud LoadBalancer no consulta `KubernetesDiscoveryClient` para obtener IPs de pods. En su lugar, construye la URL usando el nombre DNS del Service de Kubernetes (`{service-name}.{namespace}.svc.cluster.local`). Istio intercepta esa petición y aplica sus políticas de routing.

> [CONCEPTO] Desactivar `KubernetesDiscoveryClient` (`spring.cloud.kubernetes.discovery.enabled=false`) cuando Istio gestiona el service mesh es opcional pero recomendable. Si se mantiene activo en modo SERVICE, el DiscoveryClient sigue descubriendo servicios pero el load balancer los ignora para el routing.

> [ADVERTENCIA] Con `mode=POD`, Spring Cloud LoadBalancer resuelve las IPs de los pods directamente y balancea entre ellas antes de que el tráfico llegue al Envoy sidecar. Esto evita que Istio aplique sus VirtualServices, DestinationRules, mTLS y políticas de fault injection, rompiendo la gobernanza del service mesh.

## Ejemplo central

El siguiente ejemplo muestra la configuración completa para una aplicación que coexiste con Istio: modo SERVICE en el load balancer, desactivación de discovery y configuración Istio de ejemplo.

```yaml
# src/main/resources/application.yml — configuración para coexistir con Istio
spring:
  application:
    name: api-gateway
  cloud:
    kubernetes:
      loadbalancer:
        mode: SERVICE              # delegar routing a Istio
      discovery:
        enabled: false             # no necesario cuando Istio gestiona routing
      config:
        enabled: true              # ConfigMap sigue siendo útil
```

```yaml
# kubernetes/istio-virtualservice.yaml — VirtualService de Istio (gestiona el routing)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
  namespace: default
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
          weight: 100
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 100
```

```java
// src/main/java/com/example/OrderServiceClient.java
package com.example;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import java.util.List;

/**
 * Con mode=SERVICE, Feign construye la URL como
 * http://order-service.default.svc.cluster.local/orders
 * y el Envoy sidecar de Istio intercepta el tráfico aplicando
 * las reglas del VirtualService.
 * No se necesita @LoadBalanced ni KubernetesDiscoveryClient.
 */
@FeignClient(name = "order-service", url = "http://order-service:8080")
public interface OrderServiceClient {

    @GetMapping("/orders")
    List<String> getOrders();
}
```

```java
// src/main/java/com/example/WebClientConfig.java
package com.example;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

/**
 * Con Istio en modo SERVICE, WebClient usa el nombre DNS del Service
 * directamente. El load balancing y el mTLS los gestiona el sidecar Envoy.
 * No se usa @LoadBalanced porque el balanceo lo hace Istio.
 */
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient orderServiceWebClient() {
        return WebClient.builder()
                .baseUrl("http://order-service:8080")
                .build();
    }
}
```

## Tabla de propiedades relacionadas con Istio

La siguiente tabla resume las propiedades relevantes para la integración con Istio.

| Propiedad | Valor para Istio | Descripción |
|---|---|---|
| `spring.cloud.kubernetes.loadbalancer.mode` | `SERVICE` | Delega el routing al DNS del Service K8s (Istio intercepta) |
| `spring.cloud.kubernetes.discovery.enabled` | `false` | Desactiva el DiscoveryClient cuando Istio gestiona el service mesh |
| `spring.cloud.kubernetes.loadbalancer.mode` | `POD` (default) | Modo sin Istio: SC LoadBalancer resuelve IPs de pods individuales |

## Comparativa: con y sin Istio

La siguiente tabla resume cuándo usar cada modo y qué funcionalidades quedan activas.

| Escenario | `loadbalancer.mode` | `discovery.enabled` | Quién balancea |
|---|---|---|---|
| Sin service mesh | `POD` | `true` | Spring Cloud LoadBalancer |
| Con Istio (recomendado) | `SERVICE` | `false` | Istio Envoy sidecar |
| Con Istio (parcial) | `SERVICE` | `true` | Istio (discovery activo pero no usado para routing) |
| Migración gradual | `SERVICE` | `true` | Istio (DiscoveryClient disponible para consultas manuales) |

## Buenas y malas prácticas

**Buenas prácticas:**
- Configurar `mode=SERVICE` y `discovery.enabled=false` cuando el clúster usa Istio: evita doble balanceo y delega toda la gobernanza de tráfico al service mesh.
- Eliminar la anotación `@LoadBalanced` de los beans `RestTemplate` o `WebClient` cuando se usa `mode=SERVICE`: con Istio, el load balancing lo realiza el sidecar.
- Mantener `config.enabled=true` y usar ConfigMaps: la configuración vía API de Kubernetes sigue siendo válida independientemente de si Istio gestiona el routing.
- Coordinar con el equipo de plataforma la configuración de `DestinationRule` con `trafficPolicy.tls.mode=ISTIO_MUTUAL` para activar mTLS automático entre servicios.

**Malas prácticas:**
- Mantener `mode=POD` cuando Istio está activo: Spring Cloud LoadBalancer balancea a nivel de pod, el tráfico nunca pasa por el Envoy sidecar y las políticas de Istio (retry, circuit breaker, mTLS, observabilidad) no se aplican.
- [LEGACY] Usar configuración de Ribbon (eliminado en Spring Cloud 3.x) en entornos con Istio: Ribbon ya no forma parte de Spring Cloud 2022.x/2025.x.
- Activar `discovery.enabled=true` y `mode=SERVICE` simultáneamente sin documentarlo: genera confusión porque el DiscoveryClient está activo pero no influye en el routing.

> [ADVERTENCIA] Si el cluster tiene habilitado `PeerAuthentication` con `STRICT` mTLS en Istio y la aplicación Spring Boot se comunica directamente sin pasar por el sidecar (por usar `mode=POD`), las conexiones serán rechazadas por Istio. Verificar siempre el modo del load balancer cuando se activa mTLS estricto en el namespace.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué propiedad cambia el modo del load balancer de Spring Cloud Kubernetes para delegar el routing a Istio, y cuál es su valor correcto?

> [EXAMEN] 2. ¿Por qué produce problemas usar `mode=POD` cuando Istio está activo en el clúster?

> [EXAMEN] 3. ¿Qué ocurre con las políticas de VirtualService y DestinationRule de Istio cuando Spring Cloud LoadBalancer usa `mode=POD` y resuelve las IPs de pods directamente?

> [EXAMEN] 4. ¿Es necesario desactivar `spring.cloud.kubernetes.discovery.enabled` cuando se usa `mode=SERVICE` con Istio? ¿Qué impacto tiene dejarlo activo?

> [EXAMEN] 5. Un servicio configurado con `mode=SERVICE` y `discovery.enabled=false` necesita conocer programáticamente qué instancias de un servicio están disponibles. ¿Cómo puede hacerlo sin activar de nuevo el DiscoveryClient?

---

← [9.8 Leader Election](sc-kubernetes-leader.md) | [Índice](README.md) | [9.10 Testing](sc-kubernetes-testing.md) →
