# 13.5 Strangler Fig con Spring Cloud Gateway

← [13.4 Outbox Pattern con Spring Cloud Stream](sc-patrones-outbox.md) | [Índice (README.md)](README.md) | [13.6 API Composition con Spring Cloud Gateway y OpenFeign](sc-patrones-api-composition.md) →

---

El patrón Strangler Fig es la estrategia para migrar de forma incremental un sistema monolítico a microservicios: en lugar de una migración big-bang que requiere desplegar todo el nuevo sistema de golpe, se extrae funcionalidad del monolito en nuevos microservicios uno por uno, mientras el monolito sigue operativo. Spring Cloud Gateway actúa como el "strangler" — intercepta todas las peticiones entrantes y las dirige al nuevo microservicio para las funcionalidades ya migradas, o las redirecciona al monolito para las que aún no lo están. Desde el cliente, la API no cambia; solo cambia quién responde internamente. La migración puede pausarse y reanudarse sin impacto en los clientes.

> [PREREQUISITO] Requiere `spring-cloud-starter-gateway`. El monolito y los nuevos microservicios deben ser accesibles desde el Gateway (misma red o DNS resoluble).

## Flujo del Strangler Fig con Gateway

```
Clientes (sin cambios en URL)
         │
         ▼
┌──────────────────────────────────────┐
│  Spring Cloud Gateway (Strangler)    │
│                                      │
│  GET /api/orders/**  → OrderService  │ ← ya migrado
│  GET /api/products/** → Monolith     │ ← pendiente migración
│  POST /api/auth/**   → AuthService   │ ← ya migrado
│  /* (default)        → Monolith      │ ← resto del monolito
└──────────────────────────────────────┘
         │                    │
         ▼                    ▼
  Nuevo OrderService    Monolito legacy
  (microservicio)        (en vivo)
```

## Ejemplo central: Gateway como Strangler Fig

### Dependencias Maven del Gateway Strangler

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
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!-- Para routing basado en feature flags -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

### application.yml — routing progresivo

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Funcionalidades ya migradas a microservicios
        - id: orders-migrated
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - TokenRelay=

        - id: auth-migrated
          uri: lb://auth-service
          predicates:
            - Path=/api/auth/**
          filters:
            - StripPrefix=1

        - id: users-migrated
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - TokenRelay=

        # Fallback: todo lo que no esté migrado va al monolito
        # DEBE ser la última ruta (menor orden = mayor prioridad)
        - id: monolith-fallback
          uri: http://legacy-monolith:8080
          predicates:
            - Path=/**
          # Sin StripPrefix: el monolito espera las URLs originales
```

### Routing con feature flags (control dinámico de la migración)

```java
package com.example.strangler;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

// Filter que puede redirigir al monolito si el feature flag está desactivado
// Útil para rollback rápido durante la migración
@Component
public class FeatureFlagRoutingFilter implements GlobalFilter, Ordered {

    private final FeatureFlagService featureFlagService;

    public FeatureFlagRoutingFilter(FeatureFlagService featureFlagService) {
        this.featureFlagService = featureFlagService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // Si el feature flag "orders-microservice" está desactivado,
        // redirigir al monolito aunque la ruta de orders esté configurada
        if (path.startsWith("/api/orders") && !featureFlagService.isEnabled("orders-microservice")) {
            // Modificar la URI de routing para ir al monolito
            exchange.getAttributes().put(
                org.springframework.cloud.gateway.support.ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR,
                null  // null fuerza re-matching de rutas; ajustar según la implementación
            );
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

### Compatibilidad de respuesta — transformación de headers

```java
package com.example.strangler;

import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;

// Filter para añadir headers que el monolito incluía pero el nuevo servicio no
// Necesario durante la migración para mantener compatibilidad con clientes
@Component
public class LegacyCompatibilityFilterFactory
        extends AbstractGatewayFilterFactory<LegacyCompatibilityFilterFactory.Config> {

    public LegacyCompatibilityFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // Añadir headers que el monolito incluía en todas las respuestas
            HttpHeaders headers = exchange.getResponse().getHeaders();
            if (!headers.containsKey("X-Legacy-Version")) {
                headers.add("X-Legacy-Version", "migrated");
            }
            if (!headers.containsKey("X-Powered-By")) {
                headers.add("X-Powered-By", "Spring");
            }
        }));
    }

    public static class Config {}
}
```

```yaml
# Aplicar el filter de compatibilidad a las rutas migradas
spring:
  cloud:
    gateway:
      routes:
        - id: orders-migrated
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - TokenRelay=
            - LegacyCompatibility=   # añade headers de compatibilidad
```

> [CONCEPTO] El orden de las rutas en el Gateway importa: el Gateway evalúa las rutas en orden de configuración y usa la primera que coincide. Las rutas específicas (paths largos) deben ir antes que las genéricas; la ruta fallback del monolito (`Path=/**`) debe ser siempre la última. Una ruta fallback mal posicionada puede capturar peticiones destinadas a microservicios.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| Ruta de fallback `Path=/**` | Ruta de último recurso que dirige al monolito todo lo no capturado por rutas más específicas |
| Orden de rutas | Gateway evalúa rutas en orden de configuración; rutas específicas antes que genéricas |
| `StripPrefix=1` | Elimina el primer segmento de path (`/api/`) antes de routing; necesario si el monolito y microservicios tienen path bases distintos |
| Feature flags | Permiten activar/desactivar el routing a un microservicio sin redespliegue; implementar con Spring Cloud Config o un servicio de feature flags |
| Transformación de respuesta | Filters que adaptan headers o cuerpos de respuesta para mantener compatibilidad con clientes durante la migración |
| `lb://service-name` | Routing mediante Service Discovery (Eureka); permite actualizar la URL del microservicio sin cambiar el Gateway |

## Buenas y malas prácticas

**Hacer:**
- Implementar feature flags en la configuración del Gateway para poder revertir el routing al monolito en tiempo real (sin redespliegue) ante problemas con el nuevo microservicio; es el mecanismo de rollback más rápido.
- Mantener los tests de contrato del monolito activos durante la migración; son la red de seguridad que confirma que el nuevo microservicio implementa exactamente el mismo comportamiento que el monolito para la funcionalidad migrada.
- Migrar funcionalidades de más pequeñas a más grandes; las funcionalidades más acotadas (ej: servicio de catálogo de productos de solo lectura) validan el proceso de migración con menor riesgo antes de abordar funcionalidades críticas.

**Evitar:**
- Migrar y el monolito simultáneamente sin un plan de rollback; si el nuevo microservicio falla en producción y el monolito ya fue modificado para no incluir esa funcionalidad, no hay rollback posible.
- Olvidar la ruta fallback del monolito; sin ella, cualquier path no cubierto por las rutas de microservicios devuelve 404 en lugar de ser atendido por el monolito.
- Hacer cambios de schema de base de datos durante la fase Strangler Fig; compartir base de datos entre monolito y microservicio es aceptable temporalmente, pero los cambios de schema deben coordinarse para no romper al monolito que aún usa esa tabla.

---

← [13.4 Outbox Pattern con Spring Cloud Stream](sc-patrones-outbox.md) | [Índice (README.md)](README.md) | [13.6 API Composition con Spring Cloud Gateway y OpenFeign](sc-patrones-api-composition.md) →
