# 5.7 Integración con Service Discovery (DiscoveryClientRouteDefinitionLocator)

← [5.6 GlobalFilter: interfaz, orden y filtros built-in](sc-gateway-global-filters.md) | [Índice](README.md) | [5.8 CORS en Spring Cloud Gateway](sc-gateway-cors.md) →

---

## Introducción

Spring Cloud Gateway puede generar rutas automáticamente para cada servicio registrado en un registro de servicios (Eureka, Consul) sin necesidad de definir cada ruta manualmente. Este modo de integración con Service Discovery se activa con una sola propiedad y es especialmente útil en entornos de desarrollo o cuando el catálogo de servicios cambia frecuentemente. Entender cómo se generan estas rutas automáticas, qué predicates y filters aplican, y sus limitaciones es clave para tomar la decisión correcta entre rutas automáticas y rutas explícitas.

> [PREREQUISITO] Para usar la integración con Service Discovery necesitas un starter de Discovery Client en el classpath (p. ej. `spring-cloud-starter-netflix-eureka-client`) y el gateway conectado al registro de servicios.

## Cómo funciona el DiscoveryClientRouteDefinitionLocator

El componente `DiscoveryClientRouteDefinitionLocator` es un `RouteDefinitionLocator` que consulta el `DiscoveryClient` al arrancar (y periódicamente) para obtener la lista de servicios registrados. Por cada servicio, genera dinámicamente una `RouteDefinition` con:

- **id**: el nombre del servicio (p. ej. `ReactiveCompositeDiscoveryClient_ORDER-SERVICE`)
- **uri**: `lb://ORDER-SERVICE` (load-balanced hacia Eureka)
- **predicate**: `Path=/ORDER-SERVICE/**` (por defecto en mayúsculas)
- **filter**: `RewritePath=/ORDER-SERVICE/(?<remaining>.*), /${remaining}` (elimina el prefijo del nombre del servicio)

> [CONCEPTO] Con `discovery.locator.enabled=true`, el Gateway crea automáticamente una ruta para CADA servicio en el registro. La ruta generada tiene el patrón `/{serviceId}/**` y stripea el prefijo antes de reenviar al upstream.

La propiedad `lower-case-service-id=true` convierte el ID del servicio a minúsculas tanto en el path del predicate como en el URI `lb://`, lo que resulta en rutas más amigables: `/order-service/**` en lugar de `/ORDER-SERVICE/**`.

## Diagrama de rutas automáticas

El siguiente diagrama muestra cómo el Gateway genera rutas desde el registro de Eureka:

```
Eureka Server
├── ORDER-SERVICE  (instancias: host1:8081, host2:8081)
├── PRODUCT-SERVICE (instancias: host3:8082)
└── USER-SERVICE   (instancias: host4:8083)

                    ↓ discovery.locator.enabled=true

Gateway genera automáticamente:

Ruta 1: id=ReactiveCompositeDiscoveryClient_ORDER-SERVICE
  predicate: Path=/ORDER-SERVICE/**
  filter: RewritePath=/ORDER-SERVICE/(?<r>.*), /${r}
  uri: lb://ORDER-SERVICE

Ruta 2: id=ReactiveCompositeDiscoveryClient_PRODUCT-SERVICE
  predicate: Path=/PRODUCT-SERVICE/**
  filter: RewritePath=/PRODUCT-SERVICE/(?<r>.*), /${r}
  uri: lb://PRODUCT-SERVICE

Ruta 3: id=ReactiveCompositeDiscoveryClient_USER-SERVICE
  predicate: Path=/USER-SERVICE/**
  filter: RewritePath=/USER-SERVICE/(?<r>.*), /${r}
  uri: lb://USER-SERVICE

Con lower-case-service-id=true:
  Path=/order-service/**, uri=lb://order-service, etc.
```

## Ejemplo central

El siguiente ejemplo muestra la configuración mínima para activar Discovery, combinada con rutas explícitas adicionales para servicios que necesitan configuración personalizada:

```yaml
# application.yml — Integración con Service Discovery
spring:
  application:
    name: api-gateway

  cloud:
    gateway:
      # Activar generación automática de rutas desde Discovery
      discovery:
        locator:
          enabled: true
          # Convierte el serviceId a minúsculas en path y uri lb://
          lower-case-service-id: true
          # Predicates adicionales para TODAS las rutas auto-generadas (opcional)
          # predicates:
          #   - name: Path
          #     args:
          #       pattern: "'/'+serviceId+'/**'"
          # Filters adicionales para TODAS las rutas auto-generadas (opcional)
          # filters:
          #   - name: RewritePath
          #     args:
          #       regexp: "'/'+serviceId+'/(?<remaining>.*)'"
          #       replacement: "'/${remaining}'"

      # Rutas explícitas (tienen PRIORIDAD sobre las auto-generadas)
      routes:
        # Ruta explícita para order-service con configuración adicional
        # Esta ruta SOBREESCRIBE la auto-generada para /order-service/**
        - id: order-service-custom
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=2
            - AddRequestHeader=X-Source, gateway
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/orders

  # Configuración del cliente Eureka
  eureka:
    client:
      service-url:
        defaultZone: http://localhost:8761/eureka/
    instance:
      prefer-ip-address: true
```

```java
package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

// @EnableDiscoveryClient es opcional con Spring Boot 3.x (auto-configurado)
// pero documentar la intención es buena práctica
@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

Para verificar las rutas auto-generadas en tiempo de ejecución, usa el endpoint Actuator:

```bash
# Ver todas las rutas activas (incluyendo las auto-generadas)
curl http://localhost:8080/actuator/gateway/routes | jq .

# Respuesta esperada para ORDER-SERVICE:
# {
#   "route_id": "ReactiveCompositeDiscoveryClient_order-service",
#   "route_definition": {
#     "id": "ReactiveCompositeDiscoveryClient_order-service",
#     "predicates": [{"name": "Path", "args": {"pattern": "/order-service/**"}}],
#     "filters": [{"name": "RewritePath", "args": {...}}],
#     "uri": "lb://order-service"
#   }
# }
```

## Tabla de propiedades de Discovery Locator

| Propiedad | Por defecto | Descripción |
|---|---|---|
| `spring.cloud.gateway.discovery.locator.enabled` | `false` | Activa la generación automática de rutas |
| `spring.cloud.gateway.discovery.locator.lower-case-service-id` | `false` | Convierte serviceId a minúsculas en path y uri |
| `spring.cloud.gateway.discovery.locator.predicates` | Path por serviceId | Predicates adicionales para rutas auto-generadas |
| `spring.cloud.gateway.discovery.locator.filters` | RewritePath de serviceId | Filters adicionales para rutas auto-generadas |
| `spring.cloud.gateway.discovery.locator.include-expression` | `true` | SpEL para filtrar qué servicios generan ruta |

## Rutas explícitas vs rutas automáticas

Las rutas explícitas (definidas en `routes[]` del YAML o Java DSL) y las rutas auto-generadas por Discovery coexisten, pero las **explícitas tienen prioridad** porque se registran con un orden menor. Si defines una ruta explícita para `/order-service/**` y también tienes `discovery.locator.enabled=true`, la ruta explícita ganará para ese path.

> [EXAMEN] Con `spring.cloud.gateway.discovery.locator.enabled=true` y un servicio registrado como `ORDER-SERVICE`, la ruta auto-generada tiene el predicate `Path=/ORDER-SERVICE/**` (sin `lower-case-service-id`) o `Path=/order-service/**` (con `lower-case-service-id=true`).

> [ADVERTENCIA] Las rutas auto-generadas no tienen CircuitBreaker, Retry ni seguridad configurados. Para producción, las rutas auto-generadas son un buen punto de partida pero deben complementarse con rutas explícitas que añadan resiliencia y autenticación a los servicios críticos.

## Buenas y malas prácticas

**Buenas prácticas:**
- Activar `lower-case-service-id=true` siempre: los paths con mayúsculas en URL son no convencionales y confusos para los consumidores.
- Usar las rutas auto-generadas en desarrollo para iterar rápido; migrar a rutas explícitas en producción para control total.
- Combinar `discovery.locator.enabled=true` con rutas explícitas para servicios críticos que necesitan CircuitBreaker o seguridad adicional.
- Verificar las rutas generadas via `/actuator/gateway/routes` tras cada cambio en el registro de servicios.

**Malas prácticas:**
- Usar rutas auto-generadas en producción sin añadir resiliencia (CircuitBreaker, Retry): un servicio caído hará que todas las peticiones fallen con 503 sin fallback.
- Esperar que las rutas auto-generadas reflejen cambios en Eureka instantáneamente: hay un delay de polling y los timeouts de Eureka.
- Activar `discovery.locator.enabled=true` sin conocer todos los servicios registrados en Eureka: puedes exponer inadvertidamente servicios internos.

## Verificación y práctica

1. ¿Qué propiedad activa la generación automática de rutas desde Service Discovery? ¿Qué ruta se genera para un servicio registrado como `ORDER-SERVICE`?

2. ¿Qué efecto tiene `spring.cloud.gateway.discovery.locator.lower-case-service-id=true`? ¿Por qué es recomendable activarlo?

3. Si tienes `discovery.locator.enabled=true` y además defines una ruta explícita para `/order-service/**`, ¿cuál tiene prioridad y por qué?

4. ¿Qué predicate y qué filter incluye por defecto una ruta auto-generada para el servicio `PRODUCT-SERVICE`?

5. ¿Por qué las rutas auto-generadas no son apropiadas para producción sin configuración adicional?

---

← [5.6 GlobalFilter: interfaz, orden y filtros built-in](sc-gateway-global-filters.md) | [Índice](README.md) | [5.8 CORS en Spring Cloud Gateway](sc-gateway-cors.md) →
