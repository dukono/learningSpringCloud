# 2.6 Descubrimiento de servicios desde el cliente

← [2.5.2 Self-preservation mode](sc-eureka-self-preservation.md) | [Índice (README.md)](README.md) | [2.7 Dashboard y API REST del Eureka Server →](sc-eureka-api-rest.md)

---

Registrarse en Eureka es solo la mitad del ciclo: el otro rol del cliente Eureka es consultar el registro para localizar a los servicios que necesita llamar. Esta consulta no se hace en tiempo real para cada petición HTTP; el cliente mantiene una caché local del registro completo y la refresca periódicamente. Esto introduce un retraso de staleness (tipicamente 30 segundos) entre que una nueva instancia se registra y que otro servicio la ve disponible en su caché. Entender este modelo de caché y los mecanismos para interactuar con él —tanto a través de la abstracción `DiscoveryClient` como a través de `RestTemplate` o `WebClient` con `@LoadBalanced`— es esencial para diagnosticar problemas de routing en producción.

> [PREREQUISITO] El servicio consumidor debe tener configurado `eureka.client.fetch-registry: true` (valor por defecto) y la dependencia `spring-cloud-starter-netflix-eureka-client` para que `DiscoveryClient` tenga instancias disponibles.

## Diagrama: caché local del registro y resolución de nombres

El siguiente diagrama muestra cómo el cliente mantiene la caché local del registro y cómo Spring Cloud LoadBalancer la usa para resolver nombres lógicos antes de cada llamada HTTP.

```
Eureka Server
     │
     │  GET /eureka/apps (cada registry-fetch-interval-seconds = 30s)
     │
     ▼
┌────────────────────────────────────────────┐
│  Caché local del registro (EurekaClient)   │
│                                            │
│  servicio-catalogo → [inst1, inst2, inst3] │
│  servicio-pedidos  → [inst4, inst5]        │
│  servicio-pago     → [inst6]               │
└──────────────────┬─────────────────────────┘
                   │
         DiscoveryClient.getInstances("servicio-catalogo")
                   │
                   ▼
         [ServiceInstance inst1, inst2, inst3]
                   │
         Spring Cloud LoadBalancer elige inst2 (round-robin)
                   │
                   ▼
         HTTP GET http://10.0.2.4:8082/api/productos
```

## Implementación: patrones de descubrimiento desde el cliente

El siguiente ejemplo muestra los tres patrones principales de uso del descubrimiento: acceso directo a `DiscoveryClient`, `RestTemplate` con `@LoadBalanced` y `WebClient` con `@LoadBalanced`.

**Patrón 1: acceso directo a DiscoveryClient**

```java
package com.ejemplo.pedidos.service;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.List;

@Service
public class DescubrimientoService {

    private final DiscoveryClient discoveryClient;
    private final RestTemplate restTemplate;

    public DescubrimientoService(DiscoveryClient discoveryClient,
                                  RestTemplate restTemplate) {
        this.discoveryClient = discoveryClient;
        this.restTemplate = restTemplate;
    }

    /**
     * Lista todos los servicios registrados en Eureka.
     * Útil para health dashboards y administración.
     */
    public List<String> listarServicios() {
        return discoveryClient.getServices();
    }

    /**
     * Obtiene todas las instancias de un servicio por nombre lógico.
     * Permite implementar lógica de selección personalizada (ej: por zona).
     */
    public List<ServiceInstance> obtenerInstancias(String nombreServicio) {
        return discoveryClient.getInstances(nombreServicio);
    }

    /**
     * Llamada manual usando DiscoveryClient para seleccionar la instancia
     * e invocarla con RestTemplate sin @LoadBalanced.
     * Útil cuando se necesita lógica de selección personalizada.
     */
    public String llamarCatalogo(Long productoId) {
        List<ServiceInstance> instancias = discoveryClient.getInstances("servicio-catalogo");
        if (instancias.isEmpty()) {
            throw new RuntimeException("No hay instancias disponibles de servicio-catalogo");
        }
        // Selección simple: primera instancia (para producción usar LoadBalancer)
        ServiceInstance instancia = instancias.get(0);
        String url = instancia.getUri() + "/api/productos/" + productoId;
        return restTemplate.getForObject(url, String.class);
    }
}
```

**Patrón 2: RestTemplate con @LoadBalanced (recomendado para llamadas síncronas)**

```java
package com.ejemplo.pedidos.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    /**
     * @LoadBalanced hace que Spring Cloud LoadBalancer intercepte todas las
     * llamadas de este RestTemplate y resuelva nombres lógicos contra Eureka.
     * La URL http://servicio-catalogo/... se resuelve automáticamente.
     *
     * NOTA: @LoadBalanced delega en Spring Cloud LoadBalancer (spring-cloud-commons),
     * NO en Ribbon. Ribbon fue eliminado en Spring Cloud 2022.x+.
     */
    @Bean
    @LoadBalanced
    public RestTemplate loadBalancedRestTemplate() {
        return new RestTemplate();
    }
}
```

```java
package com.ejemplo.pedidos.service;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class CatalogoService {

    private final RestTemplate restTemplate;

    public CatalogoService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public String obtenerProducto(Long id) {
        // Spring Cloud LoadBalancer resuelve "servicio-catalogo"
        // contra la caché local del registro de Eureka antes de la llamada HTTP
        return restTemplate.getForObject(
            "http://servicio-catalogo/api/productos/" + id,
            String.class
        );
    }
}
```

**Patrón 3: WebClient con @LoadBalanced (para servicios reactivos)**

```java
package com.ejemplo.pedidos.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}
```

```java
package com.ejemplo.pedidos.service;

import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@Service
public class CatalogoReactivoService {

    private final WebClient.Builder webClientBuilder;

    public CatalogoReactivoService(WebClient.Builder webClientBuilder) {
        this.webClientBuilder = webClientBuilder;
    }

    public Mono<String> obtenerProducto(Long id) {
        // Mismo mecanismo que RestTemplate @LoadBalanced pero reactivo
        return webClientBuilder.build()
            .get()
            .uri("http://servicio-catalogo/api/productos/" + id)
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

**Patrón 4: OpenFeign con resolución Eureka**

```java
package com.ejemplo.pedidos.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

/**
 * El atributo name se resuelve contra el registro de Eureka vía DiscoveryClient
 * en cada llamada. No es necesario especificar la URL base.
 * Configuración completa del cliente Feign en el módulo Spring Cloud OpenFeign.
 * Ver: https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/
 */
@FeignClient(name = "servicio-catalogo")
public interface CatalogoFeignClient {

    @GetMapping("/api/productos/{id}")
    String obtenerProducto(@PathVariable Long id);
}
```

> [ADVERTENCIA] El `DiscoveryClient` devuelve instancias desde la caché local del cliente, no consultando el servidor en tiempo real. Con `registry-fetch-interval-seconds: 30` (default), una instancia recién registrada puede tardar hasta 30 segundos en aparecer en la caché de un consumidor. Este retraso es inherente al modelo de Eureka y debe tenerse en cuenta en estrategias de despliegue.

## Tabla de propiedades del descubrimiento

La siguiente tabla recoge los parámetros que controlan la caché local y el comportamiento del descubrimiento del cliente.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `eureka.client.fetch-registry` | boolean | `true` | Activa la descarga del registro al cliente; debe ser `true` para descubrimiento |
| `eureka.client.registry-fetch-interval-seconds` | int | `30` | Frecuencia de refresco de la caché local del registro en segundos |
| `eureka.client.cache-refresh-executor-thread-pool-size` | int | `2` | Tamaño del pool de hilos para el refresco de caché |
| `eureka.client.cache-refresh-executor-exponential-back-off-bound` | int | `10` | Factor máximo de backoff exponencial si el refresco falla |

## Buenas y malas prácticas

Hacer:
- Usar `DiscoveryClient` (la interfaz de Spring Cloud Commons) en el código de aplicación en lugar de `EurekaDiscoveryClient` (la implementación concreta): el código que depende de la interfaz puede cambiar de backend (Eureka, Consul, Kubernetes) sin modificar la lógica de negocio.
- Combinar `@LoadBalanced RestTemplate` o `@LoadBalanced WebClient` con Resilience4j Circuit Breaker: si Spring Cloud LoadBalancer elige una instancia que no responde, el circuit breaker detecta el fallo y evita llamadas repetidas a esa instancia sin esperar a que Eureka la retire del registro.
- Reducir `registry-fetch-interval-seconds` a 10 segundos en entornos con deployments frecuentes: los 30 segundos por defecto generan una ventana de routing hacia instancias desaparecidas que puede resultar en 503s visibles durante el despliegue.

Evitar:
- Llamar a `discoveryClient.getInstances()` en el path caliente de cada petición HTTP: la operación lee desde la caché local (sin red), pero la llamada tiene overhead de sincronización; lo correcto es delegar en `@LoadBalanced` que ya gestiona el balanceo de forma eficiente.
- Cachear manualmente la lista de instancias devuelta por `getInstances()` en una variable de clase sin TTL: la caché local de Eureka ya se refresca automáticamente; una caché manual más antigua desincronizaría ambas.

---

← [2.5.2 Self-preservation mode](sc-eureka-self-preservation.md) | [Índice (README.md)](README.md) | [2.7 Dashboard y API REST del Eureka Server →](sc-eureka-api-rest.md)
