# 3.13 Actuator, administración y observabilidad

← [3.12 Spring Cloud Gateway MVC](sc-gateway-mvc.md) | [Índice](README.md) | [3.14 Manejo de errores y respuestas de error](sc-gateway-error-handling.md) →

---

## Introducción

Sin los endpoints de Actuator de Gateway, operar el gateway en producción es operar a ciegas: no se puede ver qué rutas están activas, qué filtros están registrados, ni cuánto tráfico ha procesado cada ruta. Spring Cloud Gateway expone un endpoint `gateway` dedicado que permite inspeccionar y modificar la configuración en tiempo de ejecución, complementado con métricas Micrometer que alimentan dashboards de observabilidad. El desarrollador necesita operar los endpoints Actuator y métricas de Gateway para administrar rutas en producción y monitorizar el tráfico. Este fichero cubre tanto la administración (endpoints `GET`/`POST` de `/actuator/gateway/*`) como las métricas (`GatewayMetricsFilter` y la métrica `spring.cloud.gateway.requests`), que se fusionan porque comparten la misma configuración de `management.*`.

## Diagrama: endpoints Actuator de Gateway y su función

El siguiente diagrama muestra la estructura de endpoints disponibles y qué información exponen.

```
/actuator/gateway/
  │
  ├── GET  /routes              → lista todas las rutas resueltas (id, predicates, filters, uri, order)
  ├── GET  /routes/{id}         → detalle de una ruta por ID
  ├── POST /refresh             → publica RefreshRoutesEvent (recarga CachingRouteLocator)
  ├── GET  /globalfilters       → GlobalFilters registrados con su orden (int)
  ├── GET  /routefilters        → GatewayFilterFactory disponibles (beans registrados)
  └── GET  /routepredicates     → RoutePredicateFactory disponibles

/actuator/gateway/routes — respuesta con verbose.enabled=true:
  [
    {
      "route_id": "order-service-route",
      "route_definition": {
        "id": "order-service-route",
        "predicates": [{"name": "Path", "args": {"pattern": "/api/orders/**"}}],
        "filters": [{"name": "StripPrefix", "args": {"parts": "1"}}],
        "uri": "lb://order-service",
        "order": 0
      },
      "order": 0
    }
  ]

/actuator/prometheus — métricas de Gateway:
  # HELP spring_cloud_gateway_requests_seconds_count Tiempo de peticiones a través del Gateway
  spring_cloud_gateway_requests_seconds_count{routeId="order-service-route",
    routeUri="lb://order-service",outcome="SUCCESSFUL",status="200",httpMethod="GET",...}
```

## Ejemplo central

El siguiente ejemplo muestra la configuración completa para exponer los endpoints de administración y métricas Prometheus.

### Dependencias Maven

```xml
<!-- Actuator (necesario para los endpoints de Gateway) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Micrometer con exportador Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Configuración YAML

```yaml
# application.yml
management:
  endpoints:
    web:
      # Exponer los endpoints de Gateway + health + prometheus
      exposure:
        include: gateway, health, info, prometheus, metrics

  endpoint:
    gateway:
      enabled: true     # activa el endpoint /actuator/gateway

  # Opciones de seguridad del Actuator (producción)
  server:
    port: 8081          # separar el puerto de administración del de API
    # En producción: configurar Spring Security en el puerto de management

spring:
  cloud:
    gateway:
      # verbose.enabled: incluye predicados y filtros resueltos en GET /actuator/gateway/routes
      # Con false (default): solo id, uri y order — más ligero para gateways con cientos de rutas
      actuator:
        verbose:
          enabled: true

      # Métricas
      metrics:
        enabled: true               # activar GatewayMetricsFilter (default: true si Actuator presente)
        tags:
          path:
            enabled: false          # ADVERTENCIA: añadir el tag path genera alta cardinalidad
                                    # cada URL única crea una serie de métricas distinta
```

### Consultas operativas habituales con curl / jq

```bash
# ── Ver todas las rutas activas ──────────────────────────────────────────
curl -s http://localhost:8081/actuator/gateway/routes | \
  jq '.[] | {id: .route_id, uri: .route_definition.uri, order: .order}'

# ── Ver detalle de una ruta específica ──────────────────────────────────
curl -s http://localhost:8081/actuator/gateway/routes/order-service-route | jq .

# ── Recargar rutas tras cambio en Config Server ─────────────────────────
curl -X POST http://localhost:8081/actuator/gateway/refresh

# ── Ver GlobalFilters con su orden ──────────────────────────────────────
curl -s http://localhost:8081/actuator/gateway/globalfilters | jq .

# ── Ver métricas de Gateway (Prometheus) ────────────────────────────────
curl -s http://localhost:8081/actuator/prometheus | \
  grep spring_cloud_gateway_requests

# ── Ver métricas estructuradas (JSON) ────────────────────────────────────
curl -s "http://localhost:8081/actuator/metrics/spring.cloud.gateway.requests" | jq .
```

### Consulta de métricas por ruta (Prometheus PromQL)

```promql
# Tasa de peticiones por ruta (req/s últimos 5 min)
rate(spring_cloud_gateway_requests_seconds_count{routeId="order-service-route"}[5m])

# Latencia P99 por ruta
histogram_quantile(0.99,
  rate(spring_cloud_gateway_requests_seconds_bucket{routeId="order-service-route"}[5m]))

# Tasa de errores 5xx
rate(spring_cloud_gateway_requests_seconds_count{outcome="SERVER_ERROR"}[5m])

# Distribución por outcome (SUCCESSFUL, CLIENT_ERROR, SERVER_ERROR, REDIRECTION)
sum by (routeId, outcome) (
  rate(spring_cloud_gateway_requests_seconds_count[5m]))
```

## Tabla de elementos clave

La siguiente tabla recoge los endpoints de Actuator y las propiedades de métricas.

| Endpoint / Propiedad | Tipo | Descripción |
|---------------------|------|-------------|
| `GET /actuator/gateway/routes` | endpoint | Lista todas las rutas resueltas |
| `GET /actuator/gateway/routes/{id}` | endpoint | Detalle de una ruta por ID |
| `POST /actuator/gateway/refresh` | endpoint | Publica `RefreshRoutesEvent`; recarga rutas |
| `GET /actuator/gateway/globalfilters` | endpoint | GlobalFilters registrados con su orden |
| `GET /actuator/gateway/routefilters` | endpoint | GatewayFilterFactory beans disponibles |
| `GET /actuator/gateway/routepredicates` | endpoint | RoutePredicateFactory beans disponibles |
| `management.endpoint.gateway.enabled` | `boolean` | Activa/desactiva el endpoint gateway |
| `gateway.actuator.verbose.enabled` | `boolean` | Incluye predicados/filtros en el listado de rutas |
| `gateway.metrics.enabled` | `boolean` | Activa `GatewayMetricsFilter` |
| `gateway.metrics.tags.path.enabled` | `boolean` | Añade tag `path` a la métrica (alta cardinalidad) |

Tags por defecto de la métrica `spring.cloud.gateway.requests`:

| Tag | Valores posibles | Descripción |
|-----|-----------------|-------------|
| `routeId` | ID de la ruta | Ruta que procesó la petición |
| `routeUri` | URI del backend | Destino de la petición (ej: `lb://order-service`) |
| `outcome` | `SUCCESSFUL`, `REDIRECTION`, `CLIENT_ERROR`, `SERVER_ERROR` | Categoría del resultado |
| `status` | Código HTTP (200, 404, 500...) | Status HTTP devuelto al cliente |
| `httpMethod` | GET, POST, etc. | Método HTTP de la petición |
| `httpStatusCode` | Código HTTP numérico | Alias de `status` |

> [ADVERTENCIA] `gateway.metrics.tags.path.enabled=true` añade el path de la petición como tag. En una API con IDs en el path (`/api/orders/123`, `/api/orders/456`...) cada ID crea una serie de métricas distinta. Con millones de peticiones esto puede causar "cardinality explosion" que satura el almacenamiento de Prometheus y degrada su rendimiento. Mantener `false` (default) en producción.

> [EXAMEN] La diferencia entre `GET /actuator/gateway/routefilters` y `GET /actuator/gateway/globalfilters`: `routefilters` lista las `GatewayFilterFactory` beans disponibles (factories de filtros por ruta), que se instancian por ruta; `globalfilters` lista los beans `GlobalFilter` registrados con su orden de ejecución, que se aplican a todas las rutas.

## Buenas y malas prácticas

**Hacer:**
- Separar el puerto de management del de API en producción (`management.server.port=8081`): los endpoints de Actuator no deben ser accesibles desde internet; el puerto separado permite configurar reglas de firewall/security group distintas.
- Proteger `POST /actuator/gateway/refresh` con autenticación: este endpoint modifica el comportamiento del gateway en tiempo de ejecución; sin protección cualquier petición puede recargar las rutas.
- Usar `verbose.enabled=false` en gateways con decenas o cientos de rutas: la respuesta verbosa serializa todos los predicados y filtros de cada ruta, lo que puede producir respuestas de varios MB que saturan el panel de monitorización.

**Evitar:**
- Exponer `/actuator/gateway/*` en el mismo puerto que la API en producción sin protección: estos endpoints revelan la configuración interna completa del gateway (rutas, URIs de backend, filtros), información valiosa para un atacante.
- Activar `tags.path.enabled=true` sin comprometer el presupuesto de almacenamiento de Prometheus: en APIs REST con parámetros de ruta variables la cardinalidad crece linealmente con el número de recursos únicos accedidos.

---

← [3.12 Spring Cloud Gateway MVC](sc-gateway-mvc.md) | [Índice](README.md) | [3.14 Manejo de errores y respuestas de error](sc-gateway-error-handling.md) →
