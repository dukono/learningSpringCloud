# 3.6 Integración con Eureka y Spring Cloud LoadBalancer

← [3.5 Interceptores de petición — RequestInterceptor](sc-feign-interceptores.md) | [Índice](README.md) | [3.7 Integración con Resilience4j — Fallback y FallbackFactory](sc-feign-fallback.md) →

---

## Introducción

Cuando se usa `@FeignClient(name = "inventory-service")`, Feign no resuelve directamente esa cadena a una URL IP. En su lugar, genera una URL con el esquema `lb://inventory-service` que Spring Cloud LoadBalancer intercepta para resolver el nombre lógico a una instancia concreta registrada en Eureka. Esta integración es lo que convierte a Feign en un cliente HTTP "service-discovery aware": no necesitas hardcodear IPs ni puertos, y automáticamente balancea entre todas las instancias saludables del servicio. Comprender esta integración implica distinguir claramente qué hace Feign (construir y ejecutar la petición HTTP) de qué hacen Eureka (registro) y Spring Cloud LoadBalancer (resolución y balanceo).

> [PREREQUISITO] El módulo Eureka (2.x) es responsable del registro de instancias. Feign solo consume el serviceId ya registrado. Para que la integración funcione son necesarios tres starters: `spring-cloud-starter-openfeign`, `spring-cloud-starter-netflix-eureka-client` y (transitivamente) `spring-cloud-starter-loadbalancer`.

## Flujo de resolución de instancia

El flujo completo desde la llamada Java hasta la petición HTTP muestra cómo colaboran los tres componentes:

```
  @FeignClient(name = "inventory-service")
              │
              │ name → genera URL base: lb://inventory-service
              ▼
  ┌──────────────────────────────┐
  │  Feign Proxy                 │
  │  URL: lb://inventory-service │
  └────────────┬─────────────────┘
               │ delega resolución al LoadBalancerClient
               ▼
  ┌──────────────────────────────────────┐
  │  Spring Cloud LoadBalancer           │
  │  (ReactiveLoadBalancer /             │
  │   BlockingLoadBalancerClient)        │
  │                                      │
  │  1. Consulta ServiceInstanceList     │
  │     para "inventory-service"         │
  └──────────────┬───────────────────────┘
                 │ obtiene lista de instancias
                 ▼
  ┌──────────────────────────────────────┐
  │  Eureka Client (DiscoveryClient)     │
  │  Caché local del registro Eureka     │
  │  [                                   │
  │    {host: 10.0.1.5, port: 8081},     │
  │    {host: 10.0.1.6, port: 8081},     │
  │  ]                                   │
  └──────────────┬───────────────────────┘
                 │ RoundRobinLoadBalancer elige instancia
                 ▼
  ┌──────────────────────────────────────┐
  │  HTTP Client envía petición real a   │
  │  http://10.0.1.5:8081/api/v1/items/1 │
  └──────────────────────────────────────┘
```

## Ejemplo central

El siguiente ejemplo muestra la configuración completa de una aplicación Spring Boot que usa Feign con Eureka y LoadBalancer. Incluye las dependencias necesarias, configuración de la aplicación cliente, y ajuste del comportamiento del balanceo.

```xml
<!-- pom.xml — dependencias necesarias -->
<dependencies>
    <!-- Feign: cliente HTTP declarativo -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!-- Eureka Client: registro y discovery -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <!-- LoadBalancer: incluido transitivamente con Eureka, pero puede ser explícito -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml — cliente del servicio orders que llama a inventory-service
spring:
  application:
    name: orders-service         # este nombre se registra en Eureka
  cloud:
    openfeign:
      # Habilitación explícita de integración con Eureka (defecto: true si Eureka está en classpath)
      eureka:
        enabled: true

      # Configuración de timeouts para el cliente que usa lb://
      client:
        config:
          inventory-service:
            connectTimeout: 2000
            readTimeout: 5000

    # Configuración del LoadBalancer
    loadbalancer:
      retry:
        enabled: true            # reintentar en otra instancia si la actual falla
        max-retries-on-same-service-instance: 0
        max-retries-on-next-service-instance: 2
        retryable-status-codes: 503, 500

eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka-server:8761/eureka/
    register-with-eureka: true
    fetch-registry: true
    registry-fetch-interval-seconds: 30  # frecuencia de actualización de la caché local
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
```

```java
// Aplicación principal — habilitar Feign y Eureka Client
package com.example.orders;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.orders.clients")
// @EnableDiscoveryClient — ya no es necesario con Spring Cloud 2020+;
// el starter de Eureka auto-registra el cliente si está en classpath
public class OrdersApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApplication.class, args);
    }
}
```

```java
// Cliente Feign con resolución por serviceId en Eureka
package com.example.orders.clients;

import com.example.orders.dto.InventoryResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;

// name="inventory-service" → Spring Cloud LoadBalancer busca instancias de ese serviceId
// No se especifica url → se usa lb://inventory-service automáticamente
@FeignClient(
    name = "inventory-service",
    path = "/api/v1"
)
public interface InventoryClient {

    @GetMapping("/items/{id}")
    InventoryResponse getItem(@PathVariable("id") Long id);

    @GetMapping("/items/availability")
    boolean checkAvailability(@RequestParam("sku") String sku, @RequestParam("qty") int quantity);
}
```

```java
// Configuración de una política de balanceo personalizada (no Round Robin)
// Se puede registrar un LoadBalancer personalizado para un servicio específico
package com.example.orders.feign.config;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.core.env.Environment;

// Esta clase SÍ puede llevar @Configuration porque se usa en @LoadBalancerClient,
// no en @FeignClient directamente
public class RandomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
        Environment environment,
        LoadBalancerClientFactory clientFactory
    ) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            clientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}
```

```java
// Aplicación con política Round Robin (defecto) para inventory y Random para payment
package com.example.orders;

import com.example.orders.feign.config.RandomLoadBalancerConfig;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.orders.clients")
@LoadBalancerClient(
    name = "payment-service",
    configuration = RandomLoadBalancerConfig.class
)
public class OrdersApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApplication.class, args);
    }
}
```

## Tabla de propiedades de integración

| Propiedad | Descripción | Defecto |
|---|---|---|
| `spring.cloud.openfeign.eureka.enabled` | Habilitar resolución de nombre vía Eureka | `true` si Eureka está en classpath |
| `eureka.client.fetch-registry` | Si el cliente descarga el registro de Eureka | `true` |
| `eureka.client.registry-fetch-interval-seconds` | Frecuencia de actualización de caché local | 30 |
| `spring.cloud.loadbalancer.retry.enabled` | Reintentar en otra instancia ante fallo | `false` |
| `spring.cloud.loadbalancer.retry.max-retries-on-next-service-instance` | Reintentos en instancias distintas | 1 |

## Límites de responsabilidad (G1 vs G2)

Feign pertenece al Grupo 1 (módulo sc-feign). La lógica de balanceo pertenece al Grupo 2 (sc-loadbalancer y sc-eureka). Feign solo sabe que debe usar `lb://serviceId`; todo lo demás es responsabilidad de los otros módulos. Este límite es clave para la certificación:

- Feign resuelve: declaración de la interfaz, construcción de `RequestTemplate`, ejecución HTTP.
- LoadBalancer resuelve: selección de instancia (Round Robin, Random, etc.), reintentos inter-instancia.
- Eureka resuelve: registro, discovery, caché de instancias.

## Buenas y malas prácticas

**Buenas prácticas:**
- Nunca hardcodear `url` en `@FeignClient` en producción si Eureka está disponible.
- Configurar `spring.cloud.loadbalancer.retry.enabled=true` con un circuit breaker como respaldo ante instancias caídas.
- Usar `eureka.client.registry-fetch-interval-seconds` con un valor razonable (30s): muy bajo aumenta carga en Eureka, muy alto puede usar instancias desregistradas.

**Malas prácticas:**
- Omitir el starter `spring-cloud-starter-loadbalancer` esperando que solo con Feign y Eureka funcione el `lb://` — sin LoadBalancer la resolución falla.
- Asumir que el balanceo es perfecto sin circuit breaker: si una instancia registrada no responde, el LoadBalancer puede seguir enviándole tráfico hasta que expire su lease.

> [ADVERTENCIA] La caché local de Eureka en el cliente se actualiza cada 30 segundos por defecto. Si una instancia es dada de baja, el cliente Feign puede seguir intentando llamarla hasta que expire el TTL de la caché. Configura `spring.cloud.loadbalancer.retry` y un circuit breaker para manejar este escenario.

## Verificación y práctica

> [EXAMEN] **1.** ¿Qué URL interna genera Feign cuando se configura `@FeignClient(name = "inventory-service")`? ¿Qué componente resuelve esa URL a una IP:puerto real?

> [EXAMEN] **2.** ¿Qué tres dependencias (starters) son necesarias para que Feign resuelva nombres de servicio usando Eureka?

> [EXAMEN] **3.** ¿Cuál es el algoritmo de balanceo por defecto de Spring Cloud LoadBalancer y cómo se cambia a Random para un servicio específico?

> [EXAMEN] **4.** Si `spring.cloud.openfeign.eureka.enabled=false`, ¿qué ocurre cuando Feign intenta llamar a `@FeignClient(name = "inventory-service")` sin `url`?

> [EXAMEN] **5.** ¿Por qué Feign puede enviar una petición a una instancia ya dada de baja en Eureka? ¿Cuánto tiempo puede durar esta situación por defecto?

---

← [3.5 Interceptores de petición — RequestInterceptor](sc-feign-interceptores.md) | [Índice](README.md) | [3.7 Integración con Resilience4j — Fallback y FallbackFactory](sc-feign-fallback.md) →
