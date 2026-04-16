# 3.1 Arquitectura reactiva y modelo de ejecución

← [Índice (README.md)](README.md) | [3.2 Definición y configuración de rutas →](sc-gateway-rutas.md)

---

Spring Cloud Gateway no es un proxy HTTP clásico: es una capa de enrutamiento reactiva que modela cada petición como un flujo de datos no bloqueante a través de una cadena de componentes ordenados. Comprender ese modelo es el prerequisito para cualquier tarea de configuración: sin saber en qué punto del pipeline actúa cada predicado o filtro, y qué variables comparten entre sí, es imposible depurar comportamientos inesperados, controlar el orden de ejecución o razonar sobre el rendimiento. Gateway existe como módulo independiente de Spring Boot porque el routing de borde —predicados, reescritura de paths, rate limiting, circuit breaking— pertenece a la capa de infraestructura de microservicios, no al núcleo de cada aplicación.

> [PREREQUISITO] Gateway corre sobre Reactor Netty y el stack WebFlux de Spring Framework 7. `Mono<T>` y `Flux<T>` de Project Reactor son la unidad de trabajo de todos sus componentes internos. Consulta la documentación de WebFlux antes de continuar: https://docs.spring.io/spring-framework/reference/web/webflux.html

## Componentes core del modelo

Los cuatro tipos fundamentales que componen el modelo de ejecución de Gateway son los puntos de extensión sobre los que se construye cualquier configuración:

- **RouteLocator**: repositorio de `Route` disponibles en tiempo de ejecución. La implementación por defecto, `CachingRouteLocator`, resuelve y cachea las rutas al arrancar. Las rutas dinámicas invalidan esta caché mediante `RefreshRoutesEvent`.
- **RoutePredicateFactory**: fábrica de `AsyncPredicate<ServerWebExchange>`. Cada predicado evalúa si la petición entrante cumple una condición (path, método, cabecera, peso, etc.). El nombre del bean en Spring es el nombre de la clase sin el sufijo `RoutePredicateFactory`.
- **GatewayFilterFactory**: fábrica de `GatewayFilter` vinculado a una ruta concreta. Puede actuar antes del proxy (pre-filter), después (post-filter) o en ambas fases. El nombre del bean es el nombre de la clase sin el sufijo `GatewayFilterFactory`.
- **GlobalFilter**: filtro transversal aplicado a todas las rutas sin configuración explícita. Incluye los filtros de infraestructura del propio Gateway (`NettyRoutingFilter`, `ReactiveLoadBalancerClientFilter`, `GatewayMetricsFilter`) y los personalizados que implemente el desarrollador.

> [CONCEPTO] `GatewayFilter` y `GlobalFilter` son distintos en origen pero idénticos en ejecución: `FilteringWebHandler` los mezcla en una única lista ordenada por `getOrder()`. Un `GlobalFilter` con orden 5 se ejecutará entre un `GatewayFilter` de ruta con orden 4 y otro con orden 6.

## Flujo de procesamiento de una petición

El siguiente diagrama muestra el recorrido completo de una petición HTTP desde que entra en el proceso de Spring hasta que la respuesta vuelve al cliente. El camino crítico a memorizar es la secuencia `RoutePredicateHandlerMapping → FilteringWebHandler → NettyRoutingFilter`, porque determina dónde actúa cada tipo de componente.

```
Petición HTTP/HTTPS entrante
          │
          ▼
 HttpWebHandlerAdapter          [1] adapta HttpServerRequest → ServerWebExchange
          │                         (objeto que viaja por todo el pipeline)
          ▼
   DispatcherHandler             [2] delega en el handler adecuado
          │
          ▼
 RoutePredicateHandlerMapping    [3] evalúa predicados de cada Route
          │                         en orden ascendente de Route.order
          │                         → almacena Route ganadora en:
          │                           ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR
          ▼
   FilteringWebHandler           [4] construye cadena única de filtros:
          │                         GlobalFilters + GatewayFilters de la ruta,
          │                         mezclados por getOrder() de menor a mayor
          │
          ├── pre-filters  (modifican el request antes del proxy)
          │     ej: RewritePathGatewayFilter, AddRequestHeaderGatewayFilter
          │
          ├── ReactiveLoadBalancerClientFilter  [5] resuelve lb://service → IP:port
          │
          ├── NettyRoutingFilter               [6] envía la petición downstream
          │     (order = Integer.MAX_VALUE - 1)
          │
          └── post-filters (modifican la respuesta de vuelta al cliente)
                ej: AddResponseHeaderGatewayFilter
                    NettyWriteResponseFilter   [7] escribe la respuesta al cliente
          │
          ▼
  Respuesta HTTP al cliente
```

> [ADVERTENCIA] Los post-filters se ejecutan en orden **inverso** al de los pre-filters. Un filtro con `getOrder()` = 1 actúa primero en la fase pre (antes del proxy) y último en la fase post (después del proxy). Este comportamiento simula una pila: el último filtro en entrar es el primero en salir con la respuesta.

## La Route: unidad de enrutamiento

Una `Route` encapsula todo lo que el gateway necesita para enrutar una petición: condición de activación, destino y transformaciones. Se evalúa de forma completa —todos sus predicados deben ser `true` simultáneamente— o no activa.

Los URI schemes son la parte más relevante de `Route.uri` porque determinan qué `GlobalFilter` asume el control del proxy tras el routing:

| Scheme    | Ejemplo                        | GlobalFilter encargado del reenvío                          |
|-----------|-------------------------------|-------------------------------------------------------------|
| `lb://`   | `lb://orders-service`         | `ReactiveLoadBalancerClientFilter` → resuelve con Spring Cloud LoadBalancer |
| `http://` | `http://host:8080`            | `NettyRoutingFilter` → proxy HTTP directo                   |
| `https://`| `https://api.externa.com`     | `NettyRoutingFilter` → proxy HTTPS directo                  |
| `forward://` | `forward:///local/fallback` | `ForwardRoutingFilter` → dispatch interno en el mismo proceso |

## Ejemplo central

El siguiente ejemplo configura una aplicación Gateway mínima con dos rutas: una con balanceo de carga vía service discovery y otra con URI fija. Se muestra primero la API programática con `RouteLocatorBuilder` y después el equivalente exacto en YAML; ambas producen el mismo resultado en tiempo de ejecución.

### Dependencias Maven (Spring Cloud 2025.1.1 / Spring Boot 4.0.x)

Declara el BOM de Spring Cloud en `dependencyManagement` antes de las dependencias individuales para garantizar la compatibilidad de versiones entre módulos.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2025.1.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- Starter Gateway reactivo: incluye spring-boot-starter-webflux -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
  </dependency>
  <!-- Necesario para resolver lb:// cuando el registry es Eureka -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
</dependencies>
```

### Configuración programática con RouteLocatorBuilder

La API fluent de `RouteLocatorBuilder` permite inyectar beans y aplicar lógica condicional en tiempo de construcción de rutas, algo que la configuración YAML no soporta.

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
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            // Ruta 1: /api/orders/** → servicio "orders" descubierto por Eureka
            // RewritePath elimina el prefijo /api antes de enviar al downstream
            .route("orders-route", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<segment>.*)", "/${segment}"))
                .uri("lb://orders-service"))           // scheme lb://
            // Ruta 2: /legacy/** → URI fija, evaluada después (order = 10)
            .route("legacy-route", r -> r
                .path("/legacy/**")
                .order(10)
                .uri("https://legacy.internal.corp"))  // scheme https://
            .build();
    }
}
```

### Equivalente en application.yml

La configuración YAML usa los mismos nombres de predicado y filtro, sin los sufijos `RoutePredicateFactory` ni `GatewayFilterFactory`.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders-route
          uri: lb://orders-service
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /${segment}
        - id: legacy-route
          uri: https://legacy.internal.corp
          order: 10
          predicates:
            - Path=/legacy/**
```

> [ADVERTENCIA] No incluyas `spring-boot-starter-web` (spring-webmvc) en el classpath junto con `spring-cloud-starter-gateway`. Gateway requiere `DispatcherHandler` de WebFlux; spring-webmvc registra `DispatcherServlet`, produciendo un conflicto de autoconfiguración que impide el arranque: `Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway`.

## Tabla de elementos clave

Los componentes e identificadores que un desarrollador senior debe reconocer de inmediato en un diagnóstico de producción o revisión de código:

| Componente / Campo | Tipo | Descripción |
|---|---|---|
| `RouteLocator` | interfaz | Repositorio de rutas; implementación activa en runtime: `CachingRouteLocator` |
| `Route.id` | String | Identificador único; aparece en logs, `/actuator/gateway/routes` y tag `routeId` de métricas |
| `Route.uri` | URI | Destino del proxy; el scheme determina el `GlobalFilter` encargado del reenvío |
| `Route.predicates` | List | Condiciones AND implícitas; todas deben ser `true` para activar la ruta |
| `Route.filters` | List | Filtros de ruta; mezclados con `GlobalFilter` en cadena ordenada por `getOrder()` |
| `Route.order` | int | Prioridad de evaluación entre rutas (menor = mayor prioridad); defecto 0 |
| `Route.metadata` | Map\<String,Object\> | Datos arbitrarios accesibles en filtros; soporta `connect-timeout`, `response-timeout` por ruta |
| `ServerWebExchange` | objeto central | Contenedor de request + response + atributos que recorre todo el pipeline |
| `GATEWAY_ROUTE_ATTR` | atributo de exchange | Clave donde `RoutePredicateHandlerMapping` almacena la `Route` ganadora |

## Buenas y malas prácticas

**Hacer:**
- Asignar `id` explícito a cada ruta. Spring genera un UUID aleatorio si se omite, lo que hace ininterpretables los logs de Actuator y los tags de métricas.
- Usar `lb://nombre-servicio` en lugar de `http://host:puerto` en entornos con service discovery. El scheme `lb://` habilita balanceo automático y failover sin hardcodear IPs.
- Acceder a la ruta activa dentro de filtros con `ServerWebExchangeUtils.getAttribute(exchange, GATEWAY_ROUTE_ATTR)` en lugar de resolver la ruta manualmente.
- Establecer `order` explícito cuando dos rutas tienen predicados de path solapados. Sin orden, la ruta ganadora es la primera registrada en memoria, y ese orden puede variar entre reinicios.

**Evitar:**
- Introducir operaciones bloqueantes (`block()`, JDBC síncrono, `Thread.sleep()`) dentro de `GatewayFilter` o `GlobalFilter`. Gateway ejecuta todos los filtros en el event loop de Reactor Netty; un bloqueo paraliza el thread y degrada el throughput de todas las rutas activas.
- Duplicar la misma ruta en YAML y en `RouteLocatorBuilder`. Ambas fuentes son aditivas: la ruta se evaluará dos veces con comportamiento no determinista.
- Usar `forward://` como destino de producción para lógica de negocio. Está diseñado para fallbacks locales (ej. respuesta de circuit breaker), no para routing ordinario.

## Comparación: Gateway reactivo vs Gateway MVC

Desde Spring Cloud 2023.0.x existe una variante basada en el stack servlet (`spring-cloud-starter-gateway-mvc`). La elección afecta al modelo de threading, al subset de filtros disponibles y a la compatibilidad con aplicaciones existentes. El stack reactivo sigue siendo la implementación de referencia y es el que cubre el resto de secciones de este módulo; Gateway MVC se trata en detalle en [3.12](sc-gateway-mvc.md).

| Criterio | Gateway reactivo (WebFlux) | Gateway MVC (servlet) |
|---|---|---|
| Dependencia | `spring-cloud-starter-gateway` | `spring-cloud-starter-gateway-mvc` |
| Stack base | WebFlux + Reactor Netty | Spring MVC + Tomcat / Jetty |
| Modelo de threading | Event loop (no bloqueante) | Thread-per-request (virtual threads con Java 21) |
| Subset de GatewayFilter | Completo | Reducido |
| Compatible con `spring-webmvc` | No — conflicto irreconciliable | Sí |
| Cuándo elegir | Proyecto nuevo o máximo throughput | Migración incremental desde stack servlet |

> [EXAMEN] La pregunta más habitual sobre arquitectura de Gateway en entrevistas de nivel senior es: "¿Por qué no puede coexistir `spring-webmvc` con `spring-cloud-starter-gateway`?" Respuesta: Gateway requiere `DispatcherHandler` de WebFlux como bean principal; `spring-webmvc` registra `DispatcherServlet`, y Spring Boot no puede reconciliar ambos en el mismo contexto de aplicación.

---

← [Índice (README.md)](README.md) | [3.2 Definición y configuración de rutas →](sc-gateway-rutas.md)
