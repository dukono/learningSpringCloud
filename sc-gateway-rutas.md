# 3.2 Definición y configuración de rutas

← [3.1 Arquitectura reactiva y modelo de ejecución](sc-gateway-arquitectura.md) | [Índice](README.md) | [3.3 Predicados de enrutamiento built-in](sc-gateway-predicados.md) →

---

## Introducción

Una ruta en Spring Cloud Gateway es la unidad mínima de enrutamiento: decide qué peticiones se aceptan y adónde se envían. Sin rutas configuradas, el gateway no sabe a qué backend dirigir ninguna petición y devuelve 404 a todo. El desarrollador necesita definir rutas en Gateway para enrutar peticiones a los servicios backend correctos, ya sea mediante configuración YAML estática (la más habitual en proyectos que gestionan configuración como código) o mediante la DSL programática de `RouteLocatorBuilder` (útil cuando la lógica de construcción depende de condiciones de tiempo de arranque).

Cada ruta tiene exactamente cuatro campos obligatorios: `id` (identificador único de la ruta), `uri` (destino del proxy), `predicates` (condiciones de matching) y `filters` (transformaciones pre/post). Adicionalmente, `order` controla la prioridad cuando varias rutas compiten por la misma petición, y `metadata` permite adjuntar datos arbitrarios accesibles en los filtros.

## Diagrama: estructura de una ruta y su ciclo de vida

El siguiente diagrama muestra los campos que componen una `Route` y cómo el `RouteLocator` los expone al `RoutePredicateHandlerMapping` durante el procesamiento de cada petición entrante.

```
spring.cloud.gateway.routes[*]
  ├── id          → identificador único (string)
  ├── uri         → destino: http://, https://, lb://, forward://
  ├── predicates  → lista de RoutePredicateFactory (AND implícito)
  ├── filters     → lista de GatewayFilterFactory (cadena ordenada)
  ├── order       → int, menor = mayor prioridad (default: 0)
  └── metadata    → Map<String,Object> arbitrario por ruta

         ┌─────────────────────────┐
         │  RouteLocator           │
         │  (lista de Route)       │
         └────────────┬────────────┘
                      │  expone Flux<Route>
                      ▼
         ┌─────────────────────────┐
         │  RoutePredicateHandler  │
         │  Mapping                │  evalúa predicados en orden
         └────────────┬────────────┘
                      │  primera Route que hace match
                      ▼
         ┌─────────────────────────┐
         │  FilteringWebHandler    │  encadena GatewayFilters + GlobalFilters
         └────────────┬────────────┘
                      │
                      ▼  proxy al backend (uri)
```

## Ejemplo central

El siguiente ejemplo muestra las dos formas equivalentes de definir rutas: YAML (declarativa, recomendada para configuración externalizable) y programática con `RouteLocatorBuilder` (para lógica dinámica de construcción). Ambas variantes producen el mismo comportamiento en tiempo de ejecución.

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### Variante 1 — Configuración YAML (recomendada)

```yaml
# application.yml
spring:
  cloud:
    gateway:
      # Filtros aplicados por defecto a TODAS las rutas
      default-filters:
        - AddResponseHeader=X-Gateway-Version, 1.0

      routes:
        # Ruta 1: redirige /api/orders/** al servicio de órdenes
        - id: orders-route
          uri: lb://order-service          # lb:// usa Spring Cloud LoadBalancer
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1               # elimina /api del path antes de reenviar
          order: 1
          metadata:
            response-timeout: 5000        # ms, override del global
            connect-timeout: 1000

        # Ruta 2: redirige /api/catalog/** al servicio de catálogo con reescritura de path
        - id: catalog-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
            - Method=GET,HEAD
          filters:
            - RewritePath=/api/catalog/(?<segment>.*), /$\{segment}
          order: 2

        # Ruta 3: destino estático HTTP (sin service discovery)
        - id: legacy-route
          uri: https://legacy.internal.example.com
          predicates:
            - Path=/legacy/**
          order: 10
```

### Variante 2 — DSL programática con RouteLocatorBuilder

```java
package com.example.gateway;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayRoutesConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Ruta equivalente a la orders-route del YAML
            .route("orders-route", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addResponseHeader("X-Gateway-Version", "1.0"))
                .uri("lb://order-service"))
            // Ruta con condición dinámica en tiempo de arranque
            .route("catalog-route", r -> r
                .path("/api/catalog/**")
                .and()
                .method("GET", "HEAD")
                .filters(f -> f
                    .rewritePath("/api/catalog/(?<segment>.*)", "/${segment}"))
                .uri("lb://catalog-service"))
            .build();
    }
}
```

### Clase principal (sin cambios adicionales requeridos)

```java
package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

> [ADVERTENCIA] Si defines rutas tanto en YAML como con `@Bean RouteLocator`, ambas coexisten. Las rutas del `@Bean` se mezclan con las de `PropertiesRouteDefinitionLocator`. El campo `order` controla la prioridad entre todas ellas. Si no lo defines explícitamente, el orden de evaluación depende del orden de inserción, lo que puede producir comportamientos inesperados en producción.

## Tabla de elementos clave

La siguiente tabla recoge los campos de configuración de una `Route` y las propiedades globales más relevantes que un profesional senior debe conocer de memoria.

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.gateway.routes[*].id` | `String` | requerido | Identificador único de la ruta; aparece en logs y métricas como `routeId` |
| `spring.cloud.gateway.routes[*].uri` | `URI` | requerido | Destino del proxy: `lb://`, `http://`, `https://`, `forward://` |
| `spring.cloud.gateway.routes[*].predicates` | `List<String>` | `[]` | Condiciones de matching; todas deben cumplirse (AND implícito) |
| `spring.cloud.gateway.routes[*].filters` | `List<String>` | `[]` | Filtros por ruta aplicados en orden de declaración |
| `spring.cloud.gateway.routes[*].order` | `int` | `0` | Prioridad: número menor = mayor prioridad en el matching |
| `spring.cloud.gateway.routes[*].metadata` | `Map<String,Object>` | `{}` | Datos arbitrarios accesibles en filtros vía `route.getMetadata()` |
| `spring.cloud.gateway.default-filters` | `List<String>` | `[]` | Filtros aplicados a todas las rutas; equivale a añadirlos en cada ruta |
| `spring.cloud.gateway.routes[*].metadata.connect-timeout` | `int` (ms) | global | Override del connect-timeout para esta ruta específica |
| `spring.cloud.gateway.routes[*].metadata.response-timeout` | `long` (ms) | global | Override del response-timeout para esta ruta específica |

> [EXAMEN] La diferencia entre `spring.cloud.gateway.default-filters` y un `GlobalFilter` es frecuente en entrevistas. `default-filters` son `GatewayFilter` instanciados para cada ruta (pueden acceder al `Route` en contexto); los `GlobalFilter` son beans únicos que se aplican transversalmente sin pertenecer a ninguna ruta concreta.

## Buenas y malas prácticas

**Hacer:**
- Usar el scheme `lb://` con el nombre lógico del servicio en lugar de IPs o URLs hardcodeadas: si una instancia cambia de dirección, el Load Balancer resuelve la nueva sin tocar la configuración de Gateway.
- Asignar `id` descriptivos y estables (`orders-route`, no `route-1`): los logs de métricas incluyen `routeId` como tag; un ID legible simplifica la correlación en Grafana/Kibana.
- Usar `order` explícito cuando varias rutas tienen prefijos solapados (ej: `/api/**` y `/api/orders/**`): sin `order`, la ruta más genérica puede capturar antes que la específica dependiendo del orden de carga del contexto.
- Externalizar la configuración de rutas a un Config Server (Spring Cloud Config): permite cambiar rutas sin redeployar el gateway en entornos con muchos microservicios.
- Añadir `metadata.response-timeout` por ruta para servicios con SLAs distintos: un timeout global demasiado alto mantiene conexiones Netty ocupadas ante servicios lentos.

**Evitar:**
- Definir URI con `http://` fijo a una sola instancia en producción: elimina el balanceo y crea un punto único de fallo; usar siempre `lb://` con el nombre del servicio registrado.
- Mezclar YAML y `@Bean RouteLocator` para las mismas rutas: genera duplicados difíciles de detectar; `actuator/gateway/routes` mostrará la ruta dos veces con el mismo `id`, lo que provoca comportamiento no determinista.
- Definir rutas con `Path=/**` sin `order=LOWEST_PRECEDENCE`: una ruta catch-all sin orden bajo captura peticiones destinadas a rutas más específicas, produciendo 502 o respuestas inesperadas.
- Usar `default-filters` para filtros que solo aplican a algunas rutas: añade overhead de procesamiento en rutas que no lo necesitan y dificulta el razonamiento sobre el comportamiento de cada ruta.

## Comparación: YAML vs RouteLocatorBuilder

La elección entre ambas formas depende del contexto del proyecto, no de preferencias estilísticas.

| Criterio | YAML (`application.yml` / Config Server) | `RouteLocatorBuilder` (`@Bean`) |
|----------|------------------------------------------|----------------------------------|
| Externalizable sin recompilación | Sí (con Config Server) | No (requiere recompilación) |
| Condiciones dinámicas en tiempo de arranque | No | Sí (lógica Java completa) |
| Soporte para rutas dinámicas en tiempo de ejecución | No (estático al arranque) | No (también estático al arranque) |
| Legibilidad para equipos con perfil DevOps | Alta | Media (requiere conocer la API Java) |
| Compatible con `RouteDefinitionRepository` (rutas dinámicas) | Sí (usa `PropertiesRouteDefinitionLocator`) | Parcialmente (requiere integración manual) |
| Recomendado para | Producción, Config Server, GitOps | Casos con lógica compleja de construcción |

> [CONCEPTO] Las rutas definidas tanto en YAML como con `@Bean RouteLocator` son estáticas: se resuelven en el arranque y no cambian sin reiniciar el proceso. Las rutas verdaderamente dinámicas (modificables en tiempo de ejecución) requieren implementar `RouteDefinitionRepository`, tema cubierto en [3.9 Rutas dinámicas y recarga en caliente](sc-gateway-rutas-dinamicas.md).

---

← [3.1 Arquitectura reactiva y modelo de ejecución](sc-gateway-arquitectura.md) | [Índice](README.md) | [3.3 Predicados de enrutamiento built-in](sc-gateway-predicados.md) →
