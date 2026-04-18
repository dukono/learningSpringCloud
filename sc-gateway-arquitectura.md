# 5.1 Arquitectura y ciclo de vida de Spring Cloud Gateway

← [4.13 Testing con Resilience4j](sc-circuitbreaker-testing.md) | [Índice](README.md) | [5.2 Configuración de rutas](sc-gateway-configuracion-rutas.md) →

---

## Introducción

Spring Cloud Gateway es el API Gateway oficial del ecosistema Spring Cloud para aplicaciones reactivas. Resuelve el problema de enrutar, filtrar y securizar tráfico HTTP/HTTPS hacia microservicios downstream sin el coste de bloqueo de hilos que caracterizaba a Zuul 1. Su existencia se justifica porque un API Gateway centraliza corte transversal (autenticación, rate limiting, logging, retries) en un único punto, evitando que cada microservicio implemente estas preocupaciones de forma independiente. Se necesita cuando hay múltiples microservicios que deben exponerse como una API coherente al exterior.

> [PREREQUISITO] Spring Cloud Gateway requiere Spring Boot 3.x y WebFlux en el classpath. No puede desplegarse como WAR en un contenedor de servlets como Tomcat.

## Arquitectura: motor reactivo WebFlux/Netty

Spring Cloud Gateway está construido sobre Project Reactor y Netty, lo que significa que cada petición HTTP se procesa en un loop de eventos no bloqueante. A diferencia de Zuul 1, que asigna un hilo del pool por cada petición entrante y lo bloquea mientras espera respuesta del upstream, Gateway maneja miles de peticiones concurrentes con un número reducido de hilos del EventLoop.

> [CONCEPTO] Los tres bloques centrales de Gateway son: **Route** (unidad de enrutamiento con id, uri destino, predicates y filters), **Predicate** (condición booleana sobre la petición) y **Filter** (transformación pre o post del request/response).

El motor reactivo implica que cualquier lógica custom dentro de un filtro debe ser no bloqueante. Llamadas bloqueantes dentro de un `GlobalFilter` o `GatewayFilter` degradan el rendimiento del servidor completo.

> [ADVERTENCIA] Nunca uses operaciones bloqueantes (JDBC, llamadas HTTP síncronas, `Thread.sleep`) dentro de un filtro de Gateway. Usa `Mono`/`Flux` con operadores reactivos o delega en un scheduler separado con `subscribeOn(Schedulers.boundedElastic())`.

## Ciclo de vida de una petición

El siguiente diagrama muestra el flujo completo desde que llega una petición HTTP hasta que se envía la respuesta al cliente:

```
Cliente HTTP
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│               Spring Cloud Gateway (Netty)               │
│                                                         │
│  1. HttpServerRequest                                   │
│       │                                                 │
│       ▼                                                 │
│  2. RoutePredicateHandlerMapping                        │
│     ┌──────────────────────────────────┐               │
│     │  Para cada Route definida:       │               │
│     │  ¿Predicate(request) == true?    │               │
│     │  → Primera coincidencia gana     │               │
│     └──────────────────────────────────┘               │
│       │ Route seleccionada                              │
│       ▼                                                 │
│  3. FilteringWebHandler                                 │
│     ┌──────────────────────────────────┐               │
│     │  Combina:                        │               │
│     │  - GlobalFilters (todas rutas)   │               │
│     │  - GatewayFilters (esta ruta)    │               │
│     │  Ordena por Ordered.getOrder()   │               │
│     └──────────────────────────────────┘               │
│       │                                                 │
│       ▼                                                 │
│  4. GatewayFilterChain.filter(exchange)                 │
│     PRE-filters → proxy request → POST-filters          │
│       │                                                 │
│       ▼                                                 │
│  5. NettyRoutingFilter / ForwardRoutingFilter           │
│     (envía petición al upstream)                        │
│       │                                                 │
│       ▼                                                 │
│  6. Respuesta upstream → POST-filters → Cliente         │
└─────────────────────────────────────────────────────────┘
```

Cada filtro en la cadena puede ejecutar lógica **antes** de llamar a `chain.filter(exchange)` (fase PRE) y **después** de que el `Mono` retornado se complete (fase POST, usando `.then()` o `.doFinally()`).

## Ejemplo central

El siguiente ejemplo muestra una aplicación Gateway mínima con una ruta que proxea peticiones a un servicio downstream, incluyendo un filtro que añade un header personalizado:

```java
package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Ruta 1: proxear /api/orders/** hacia el servicio de pedidos
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .addRequestHeader("X-Gateway-Source", "spring-cloud-gateway")
                    .stripPrefix(1))   // elimina /api del path enviado al upstream
                .uri("lb://order-service"))  // lb:// = load-balanced via Eureka
            // Ruta 2: redirección simple
            .route("legacy-redirect", r -> r
                .path("/old-api/**")
                .filters(f -> f.redirectTo(301, "https://api.example.com"))
                .uri("no://op"))
            .build();
    }
}
```

```yaml
# application.yml equivalente para la primera ruta
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway
            - StripPrefix=1
```

La dependencia Maven necesaria es:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

> [ADVERTENCIA] `spring-cloud-starter-gateway` incluye WebFlux. Si el proyecto tiene `spring-boot-starter-web` (servlet stack) en el classpath al mismo tiempo, la aplicación fallará al arrancar con un error de contexto incompatible.

## Tabla de elementos clave

La siguiente tabla resume los componentes del ciclo de vida y su responsabilidad dentro del motor de Gateway:

| Componente | Responsabilidad |
|---|---|
| `RoutePredicateHandlerMapping` | Evalúa predicates en orden; selecciona la primera ruta que coincide |
| `FilteringWebHandler` | Construye la cadena de filtros (GlobalFilter + GatewayFilter de la ruta) y la ejecuta |
| `GatewayFilterChain` | Interfaz reactiva que propaga el `ServerWebExchange` a lo largo de la cadena |
| `ServerWebExchange` | Contexto mutable de la petición: request, response, attributes |
| `NettyRoutingFilter` | GlobalFilter que realiza la llamada HTTP al upstream mediante Netty |
| `ReactiveLoadBalancerClientFilter` | GlobalFilter que resuelve `lb://service-id` a una instancia concreta |
| `RouteLocator` | Fuente de definiciones de rutas (YAML, Java DSL, Discovery) |

## Comparación: Spring Cloud Gateway vs Zuul 1

Spring Cloud Gateway y Zuul 1 cumplen el mismo rol de API Gateway pero con modelos de ejecución fundamentalmente distintos. Zuul 1 usa un modelo bloqueante basado en Servlet 2.5: cada petición ocupa un hilo del pool durante toda su duración. Bajo carga alta, este modelo limita la concurrencia al tamaño del thread pool (típicamente 200 hilos).

Gateway usa un modelo no bloqueante (EventLoop): decenas de miles de peticiones concurrentes se multiplexan sobre unos pocos hilos. El coste por petición inactiva (esperando respuesta upstream) es mínimo. Zuul 2 también adoptó el modelo reactivo, pero Spring Cloud oficial dejó de mantenerlo; Gateway es el sucesor recomendado.

> [EXAMEN] Pregunta frecuente: "¿Por qué Spring Cloud Gateway no puede desplegarse como WAR en Tomcat?" Respuesta: porque requiere el servidor reactivo Netty incluido en WebFlux. Los contenedores servlet tradicionales no soportan el modelo de I/O no bloqueante que necesita el motor de Gateway.

## Buenas y malas prácticas

**Buenas prácticas:**
- Definir rutas con IDs únicos y descriptivos para facilitar diagnóstico via Actuator.
- Usar `lb://` solo cuando hay un `LoadBalancer` configurado (Eureka + Spring Cloud LoadBalancer en el classpath).
- Mantener los filtros sin estado para facilitar escalado horizontal del Gateway.
- Habilitar `spring.cloud.gateway.httpclient.wiretap=true` solo en desarrollo/debug, nunca en producción.

**Malas prácticas:**
- Incluir lógica de negocio dentro de filtros Gateway (viola la separación de responsabilidades).
- Usar `spring-boot-starter-web` junto con `spring-cloud-starter-gateway` en el mismo proyecto.
- Ignorar el orden de los predicates: una ruta con `Path=/**` al inicio captura todas las peticiones y ninguna ruta posterior se evaluará.

## Verificación y práctica

1. ¿Por qué Spring Cloud Gateway no puede desplegarse como WAR en un servidor de aplicaciones basado en Servlet? ¿Qué modelo de I/O utiliza internamente?

2. ¿Cuál es la diferencia entre un `GlobalFilter` y un `GatewayFilter`? ¿Cuándo se aplica cada uno?

3. En el ciclo de vida de una petición, ¿qué componente es responsable de seleccionar la ruta correcta entre varias definidas? ¿Qué ocurre si ninguna ruta coincide?

4. ¿Qué significa `uri: lb://order-service` y qué componentes del ecosistema Spring Cloud deben estar presentes en el classpath para que funcione?

5. Un desarrollador coloca lógica bloqueante (consulta JDBC) dentro de un `GlobalFilter`. ¿Qué impacto tiene esto en el rendimiento del gateway y cómo debería corregirse?

---

← [4.13 Testing con Resilience4j](sc-circuitbreaker-testing.md) | [Índice](README.md) | [5.2 Configuración de rutas](sc-gateway-configuracion-rutas.md) →
