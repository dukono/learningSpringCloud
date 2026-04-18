# 5.10 Actuator y observabilidad del Gateway

← [5.9 Seguridad y OAuth2](sc-gateway-seguridad-oauth2.md) | [Índice](README.md) | [5.11 Extensiones custom](sc-gateway-custom-extensiones.md) →

---

## Introducción

La observabilidad de un API Gateway es crítica: es el punto de entrada de todo el tráfico y sus fallos impactan a todos los servicios downstream. Spring Cloud Gateway expone endpoints Actuator específicos para inspeccionar y manipular rutas en tiempo de ejecución, y se integra con Micrometer para emitir métricas por ruta. Este nodo cubre los endpoints Actuator específicos del Gateway (G1), el logging de filtros y el sistema de métricas, dejando los exporters de Micrometer y la infraestructura de trazas distribuidas (Prometheus, Zipkin, OTEL) como G2.

> [PREREQUISITO] Para los endpoints Actuator del Gateway necesitas `spring-boot-starter-actuator` y exponer los endpoints en `management.endpoints.web.exposure.include`.

## Endpoints Actuator específicos del Gateway

Spring Cloud Gateway añade al contexto Actuator un conjunto de endpoints bajo `/actuator/gateway` que permiten inspeccionar el estado del gateway y modificar rutas en caliente. Estos endpoints son los más frecuentes en preguntas de certificación relacionadas con operación del Gateway.

> [CONCEPTO] El endpoint `POST /actuator/gateway/refresh` recarga las definiciones de rutas desde todos los `RouteDefinitionLocator` (YAML, Java DSL, Discovery) sin reiniciar el proceso. Es la operación de operación más importante del Gateway en producción.

Los endpoints de administración de rutas (`POST /routes/{id}`, `DELETE /routes/{id}`) permiten añadir y eliminar rutas dinámicamente en tiempo de ejecución. Estas operaciones usan el `InMemoryRouteDefinitionRepository` por defecto, que no persiste entre reinicios.

## Configuración de observabilidad

La siguiente configuración muestra todos los niveles de observabilidad disponibles en el Gateway: endpoints Actuator, logging de filtros y métricas Micrometer.

```yaml
# application.yml — Configuración completa de observabilidad del Gateway
management:
  endpoints:
    web:
      exposure:
        # Exponer los endpoints de Gateway y los estándar
        include:
          - gateway
          - health
          - info
          - metrics
          - prometheus
  endpoint:
    gateway:
      enabled: true
    health:
      show-details: always

spring:
  cloud:
    gateway:
      # ===== MÉTRICAS MICROMETER =====
      metrics:
        enabled: true
        # Tags adicionales por ruta (nombre de ruta, uri destino, etc.)
        tags:
          application: ${spring.application.name}

      # ===== LOGGING DE RED (solo para debug/desarrollo) =====
      # Activa Netty wire-level logging de peticiones/respuestas HTTP
      httpclient:
        wiretap: false  # true solo para debug (log muy verbose - nivel TRACE)
      httpserver:
        wiretap: false  # true solo para debug

# ===== LOGGING DE FILTROS =====
logging:
  level:
    # Log de toda la infraestructura de rutas del Gateway
    org.springframework.cloud.gateway: DEBUG
    # Log de decisiones de matching de rutas
    org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping: DEBUG
    # Log de la cadena de filtros (filter names y orden)
    org.springframework.cloud.gateway.filter.factory: DEBUG
    # Wire logging (requiere httpclient.wiretap=true o httpserver.wiretap=true)
    reactor.netty.http.client: DEBUG
    reactor.netty.http.server: DEBUG
```

## Ejemplo central

El siguiente ejemplo incluye la configuración Actuator del Gateway con todos los endpoints activos y un ejemplo de interacción con la API de administración de rutas:

```yaml
# application.yml — Configuración Actuator del Gateway
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      metrics:
        enabled: true
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

management:
  endpoints:
    web:
      exposure:
        include: "gateway,health,metrics,prometheus"
      base-path: /management   # cambiar base path (opcional, por defecto /actuator)
  endpoint:
    gateway:
      enabled: true
```

```bash
# ===== Comandos para interactuar con los endpoints Actuator del Gateway =====

# Listar todas las rutas activas
curl -s http://localhost:8080/actuator/gateway/routes | jq .

# Ver una ruta específica por ID
curl -s http://localhost:8080/actuator/gateway/routes/order-service | jq .

# Listar todos los GlobalFilters registrados (con su orden)
curl -s http://localhost:8080/actuator/gateway/globalfilters | jq .

# Listar todas las GatewayFilter Factories disponibles
curl -s http://localhost:8080/actuator/gateway/routefilters | jq .

# Refrescar rutas en caliente (útil tras cambios en Config Server)
curl -X POST http://localhost:8080/actuator/gateway/refresh

# Añadir una ruta dinámicamente (InMemoryRouteDefinitionRepository — no persiste)
curl -X POST http://localhost:8080/actuator/gateway/routes/new-route \
  -H "Content-Type: application/json" \
  -d '{
    "id": "new-route",
    "predicates": [{"name": "Path", "args": {"pattern": "/new/**"}}],
    "filters": [{"name": "StripPrefix", "args": {"parts": "1"}}],
    "uri": "lb://new-service",
    "order": 0
  }'

# Eliminar una ruta dinámica
curl -X DELETE http://localhost:8080/actuator/gateway/routes/new-route

# Ver métricas específicas del Gateway
curl -s "http://localhost:8080/actuator/metrics/spring.cloud.gateway.requests" | jq .

# Ver métricas filtradas por ruta
curl -s "http://localhost:8080/actuator/metrics/spring.cloud.gateway.requests?tag=routeId:order-service" | jq .
```

```java
package com.example.gateway.filter;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Instant;

/**
 * GlobalFilter custom para métricas adicionales por ruta.
 * Complementa las métricas automáticas de GatewayMetricsFilter.
 */
@Component
public class CustomMetricsGlobalFilter implements GlobalFilter, Ordered {

    private final MeterRegistry meterRegistry;

    public CustomMetricsGlobalFilter(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public int getOrder() {
        return 10; // Después de los built-in de resolución de ruta
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = Instant.now().toEpochMilli();

        return chain.filter(exchange)
            .doFinally(signalType -> {
                Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
                String routeId = route != null ? route.getId() : "unknown";
                HttpStatus status = exchange.getResponse().getStatusCode() != null
                    ? HttpStatus.resolve(exchange.getResponse().getStatusCode().value())
                    : HttpStatus.INTERNAL_SERVER_ERROR;

                long duration = Instant.now().toEpochMilli() - startTime;

                meterRegistry.timer("gateway.request.duration",
                    "routeId", routeId,
                    "status", status != null ? String.valueOf(status.value()) : "unknown",
                    "method", exchange.getRequest().getMethod().name()
                ).record(java.time.Duration.ofMillis(duration));
            });
    }
}
```

## Tabla de endpoints Actuator del Gateway

| Endpoint | Método | Descripción |
|---|---|---|
| `/actuator/gateway/routes` | GET | Lista todas las rutas activas con su definición completa |
| `/actuator/gateway/routes/{id}` | GET | Obtiene la definición de una ruta por ID |
| `/actuator/gateway/routes/{id}` | POST | Añade una ruta dinámica (no persiste en reinicio) |
| `/actuator/gateway/routes/{id}` | DELETE | Elimina una ruta dinámica |
| `/actuator/gateway/refresh` | POST | Recarga las definiciones de rutas sin reiniciar |
| `/actuator/gateway/globalfilters` | GET | Lista GlobalFilters con su orden de ejecución |
| `/actuator/gateway/routefilters` | GET | Lista GatewayFilter Factories disponibles |

## Métricas automáticas de Gateway

Cuando `spring.cloud.gateway.metrics.enabled=true`, el `GatewayMetricsFilter` emite automáticamente la métrica `spring.cloud.gateway.requests` con los siguientes tags:

`routeId` identifica la ruta que manejó la petición. `routeUri` contiene el URI destino de la ruta. `outcome` clasifica la respuesta (SUCCESSFUL, CLIENT_ERROR, SERVER_ERROR). `status` es el código HTTP de la respuesta. `httpMethod` es el método HTTP de la petición.

> [EXAMEN] La métrica automática del Gateway es `spring.cloud.gateway.requests` (tipo Timer/Counter). Para verla via Actuator: `GET /actuator/metrics/spring.cloud.gateway.requests`. Para filtrar por ruta: añadir `?tag=routeId:order-service`.

## Frontera G1/G2

Los endpoints `/actuator/gateway/*`, las propiedades `spring.cloud.gateway.metrics.*` y el logging `logging.level.org.springframework.cloud.gateway` son G1 y se cubren aquí. La propiedad base de Actuator `management.endpoints.web.exposure.include`, los exporters de métricas (Prometheus, Grafana), la configuración de trazas distribuidas (Zipkin, OpenTelemetry) y el `TraceContext` de Micrometer son G2.

## Buenas y malas prácticas

**Buenas prácticas:**
- Activar `spring.cloud.gateway.metrics.enabled=true` en producción para visibilidad de SLA por ruta.
- Usar `POST /actuator/gateway/refresh` en el webhook del Config Server para actualizar rutas sin reiniciar.
- Proteger los endpoints `/actuator/gateway/routes` con autenticación: exponen la topología completa del sistema.
- Usar `logging.level.org.springframework.cloud.gateway=DEBUG` solo en desarrollo/staging; en producción afecta rendimiento.

**Malas prácticas:**
- Activar `httpclient.wiretap=true` en producción: registra el cuerpo completo de cada petición/respuesta en nivel TRACE, impactando rendimiento y exponiendo datos sensibles en logs.
- Exponer `/actuator/gateway/routes` sin autenticación en producción: expone la arquitectura interna.
- Añadir rutas dinámicamente via Actuator en producción sin un mecanismo de persistencia: se pierden en el próximo reinicio.

## Verificación y práctica

1. ¿Qué endpoint Actuator te permite ver las rutas activas en tiempo de ejecución? ¿Y cuál refresca las rutas sin reiniciar el gateway?

2. ¿Qué propiedad activa las métricas automáticas del Gateway? ¿Cuál es el nombre de la métrica emitida y qué tags incluye?

3. Un operador añade una ruta via `POST /actuator/gateway/routes/new-route`. ¿Qué ocurre con esa ruta cuando el Gateway se reinicia?

4. ¿Qué diferencia hay entre `spring.cloud.gateway.httpclient.wiretap=true` y `logging.level.org.springframework.cloud.gateway=DEBUG`?

5. ¿Por qué debes proteger los endpoints `/actuator/gateway/*` con autenticación en producción?

---

← [5.9 Seguridad y OAuth2](sc-gateway-seguridad-oauth2.md) | [Índice](README.md) | [5.11 Extensiones custom](sc-gateway-custom-extensiones.md) →
