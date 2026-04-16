# 3.3 Predicados de enrutamiento built-in

← [3.2 Definición y configuración de rutas](sc-gateway-rutas.md) | [Índice](README.md) | [3.4.1 Filtros de cabeceras del request](sc-gateway-filtros-request-headers.md) →

---

## Introducción

Un predicado en Spring Cloud Gateway es una función booleana que se evalúa contra el `ServerWebExchange` entrante: si se cumple, la ruta asociada es candidata al proxy; si no, se descarta. Sin predicados, todas las peticiones irían a la misma ruta, haciendo imposible el enrutamiento selectivo. El desarrollador necesita configurar predicados en las rutas para controlar qué peticiones son enrutadas a cada servicio, partiendo de los 12 predicados built-in (que cubren el 95% de los casos) y completando con predicados personalizados cuando los built-in no son suficientes.

Los predicados se combinan con AND implícito dentro de una misma ruta: si se declaran `Path` y `Method` juntos, ambas condiciones deben cumplirse. La negación con prefijo `!` permite invertir cualquier predicado. La composición OR se logra definiendo dos rutas separadas con el mismo `uri` y `order` equivalente.

> [PREREQUISITO] Este fichero asume que el lector conoce la estructura básica de una ruta (`id`, `uri`, `predicates`, `filters`). Ver [3.2 Definición y configuración de rutas](sc-gateway-rutas.md).

## Diagrama: evaluación de predicados en el ciclo de matching

El siguiente diagrama muestra cómo el `RoutePredicateHandlerMapping` evalúa los predicados de todas las rutas configuradas hasta encontrar la primera coincidencia.

```
Petición entrante
       │
       ▼
RoutePredicateHandlerMapping
  ┌────────────────────────────────────────────────────────┐
  │  Para cada Route en orden (order ASC):                 │
  │                                                        │
  │  ┌─────────────────────────────────────────────┐       │
  │  │  Evalúa predicado 1 (Path)   → true/false  │       │
  │  │  Evalúa predicado 2 (Method) → true/false  │       │
  │  │  ...                                        │       │
  │  │  AND de todos → match?                      │       │
  │  └─────────────────┬───────────────────────────┘       │
  │                    │ false → siguiente Route            │
  │                    │ true  → match encontrado           │
  └────────────────────┼───────────────────────────────────┘
                       │
                       ▼
              FilteringWebHandler
              (aplica filtros de la Route ganadora)
```

## Ejemplo central

El siguiente ejemplo cubre los predicados más utilizados en producción, con su sintaxis YAML exacta y notas sobre comportamiento no obvio.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:

        # ── PathRoutePredicateFactory ──────────────────────────────────
        # Patrones glob: **, *, ? — captura variables con {segment}
        - id: path-example
          uri: lb://catalog-service
          predicates:
            - Path=/catalog/{category}/items/**
          # {category} queda disponible en ServerWebExchange como uri template variable

        # ── MethodRoutePredicateFactory ────────────────────────────────
        - id: method-example
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=POST,PUT,PATCH   # AND implícito con Path

        # ── HeaderRoutePredicateFactory ────────────────────────────────
        # Header nombre regexp — el valor del header debe hacer match con la regexp
        - id: header-example
          uri: lb://v2-service
          predicates:
            - Path=/api/**
            - Header=X-API-Version, v2.*

        # ── QueryRoutePredicateFactory ─────────────────────────────────
        # Parámetro de query + regexp opcional
        - id: query-example
          uri: lb://search-service
          predicates:
            - Path=/search
            - Query=q              # solo comprueba existencia del parámetro
            - Query=format, json   # comprueba existencia Y valor exacto

        # ── HostRoutePredicateFactory ──────────────────────────────────
        # Patrones Ant: **, *
        - id: host-example
          uri: lb://tenant-service
          predicates:
            - Host=**.acme.com,**.acme.io   # OR implícito entre hosts

        # ── CookieRoutePredicateFactory ────────────────────────────────
        - id: cookie-example
          uri: lb://ab-service
          predicates:
            - Path=/checkout/**
            - Cookie=session, [a-f0-9]{32}   # nombre + regexp del valor

        # ── AfterRoutePredicateFactory / BeforeRoutePredicateFactory ───
        # ZonedDateTime en formato ISO-8601 con offset
        - id: after-example
          uri: lb://promo-service
          predicates:
            - Path=/promo/**
            - After=2025-12-01T00:00:00+01:00[Europe/Madrid]

        - id: between-example
          uri: lb://flash-sale-service
          predicates:
            - Path=/flash-sale/**
            - Between=2025-12-24T00:00:00+01:00[Europe/Madrid],2025-12-26T00:00:00+01:00[Europe/Madrid]

        # ── RemoteAddrRoutePredicateFactory ───────────────────────────
        # CIDR IPv4/IPv6; útil para rutas de administración internas
        - id: remote-addr-example
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            - RemoteAddr=10.0.0.0/8,172.16.0.0/12

        # ── XForwardedRemoteAddrRoutePredicateFactory ──────────────────
        # Igual que RemoteAddr pero lee X-Forwarded-For (para proxies inversos)
        - id: x-forwarded-example
          uri: lb://internal-service
          predicates:
            - Path=/internal/**
            - XForwardedRemoteAddr=192.168.0.0/16

        # ── WeightRoutePredicateFactory (canary / A-B routing) ─────────
        # group + weight; las rutas del mismo grupo se evalúan por peso relativo
        - id: weight-v2
          uri: lb://catalog-service-v2
          predicates:
            - Path=/api/catalog/**
            - Weight=catalog-group, 20     # 20% del tráfico
          order: 1

        - id: weight-v1
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
            - Weight=catalog-group, 80     # 80% del tráfico
          order: 1

        # ── Negación de predicados con prefijo ! ───────────────────────
        # Enruta todo lo que NO sea peticiones de móviles (según User-Agent)
        - id: non-mobile-route
          uri: lb://web-service
          predicates:
            - Path=/app/**
            - "!Header=User-Agent, .*(Mobile|Android|iPhone).*"
```

### Predicado personalizado (AbstractRoutePredicateFactory)

Cuando los built-in no son suficientes, se extiende `AbstractRoutePredicateFactory`. El detalle completo de los puntos de extensión está en [3.11 Filtros y predicados personalizados](sc-gateway-extension-points.md). Aquí se muestra la estructura mínima:

```java
package com.example.gateway;

import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;

@Component
public class TenantRoutePredicateFactory
        extends AbstractRoutePredicateFactory<TenantRoutePredicateFactory.Config> {

    public TenantRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("tenantId");
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String header = exchange.getRequest()
                    .getHeaders().getFirst("X-Tenant-Id");
            return config.getTenantId().equals(header);
        };
    }

    public static class Config {
        private String tenantId;
        public String getTenantId() { return tenantId; }
        public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    }
}
```

Uso en YAML (gracias a `shortcutFieldOrder`, una sola línea):

```yaml
predicates:
  - Tenant=acme-corp
```

## Tabla de elementos clave

La siguiente tabla resume todos los predicados built-in con sus parámetros obligatorios y comportamiento relevante para entrevistas.

| Predicado | Parámetros | Comportamiento notable |
|-----------|-----------|----------------------|
| `Path` | patrones glob separados por coma | Variables `{name}` disponibles como URI template vars en filtros |
| `Method` | verbos HTTP separados por coma | Case-insensitive |
| `Header` | nombre, regexp | Hace match si el header existe Y su valor cumple la regexp |
| `Query` | nombre [, regexp] | Sin regexp: solo comprueba existencia; con regexp: comprueba valor |
| `Host` | patrones Ant separados por coma | OR implícito entre los patrones; `**.example.com` incluye subdomains |
| `Cookie` | nombre, regexp | Igual que Header pero sobre cookies; útil para A/B con cookie de sesión |
| `After` | ZonedDateTime ISO-8601 | Activa la ruta solo después de esa fecha/hora |
| `Before` | ZonedDateTime ISO-8601 | Activa la ruta solo antes de esa fecha/hora |
| `Between` | ZonedDateTime, ZonedDateTime | Combinación de After y Before |
| `RemoteAddr` | CIDR IPv4/IPv6 separados por coma | Compara la IP real de la conexión TCP, no headers |
| `XForwardedRemoteAddr` | CIDR IPv4/IPv6 separados por coma | Lee el primer valor de `X-Forwarded-For`; requiere confiar en el proxy upstream |
| `Weight` | grupo, peso | Distribución proporcional dentro del grupo; sin estado externo |

> [EXAMEN] `RemoteAddr` vs `XForwardedRemoteAddr`: en un entorno con un load balancer delante del gateway, la IP real del cliente solo aparece en `X-Forwarded-For`. Usar `RemoteAddr` en ese caso siempre matcheará la IP del load balancer, no la del cliente.

> [ADVERTENCIA] `WeightRoutePredicateFactory` no requiere un store externo: distribuye el tráfico de forma estadística en memoria por instancia del gateway. En un cluster de varias instancias de Gateway, la distribución es aproximada pero no exacta a nivel global. Para distribución exacta se necesita un mecanismo externo (Redis, etc.).

## Buenas y malas prácticas

**Hacer:**
- Usar `Path` como predicado principal en todas las rutas: es el más eficiente (evaluación por árbol de prefijos) y el más predecible para equipos de operaciones.
- Añadir `Method` junto a `Path` en rutas de escritura (POST, PUT, DELETE): evita que peticiones GET inesperadas alcancen endpoints que modifican estado.
- Usar `XForwardedRemoteAddr` en lugar de `RemoteAddr` cuando el gateway está detrás de un reverse proxy o un cloud load balancer: `RemoteAddr` en ese contexto evalúa siempre la IP del proxy, nunca la del cliente real.
- Usar `Weight` para canary releases graduales: permite aumentar el porcentaje de v2 de forma incremental sin downtime y sin cambiar la URL del cliente.
- Usar `Between` para ventanas de mantenimiento programadas: la ruta al servicio de mantenimiento se activa automáticamente en el rango horario sin intervención manual.

**Evitar:**
- Combinar más de 4 predicados en la misma ruta sin documentar la intención: la AND implícita acumulada es difícil de razonar y de depurar cuando la ruta no hace match.
- Usar `Header=Authorization, .*` como predicado de autorización real: los predicados se evalúan antes que los filtros de seguridad y no deben sustituir a Spring Security; un atacante que inyecta el header pasa el predicado.
- Usar `!` (negación) en predicados costosos como `Header` con regexp complejas sin medir el impacto: la negación fuerza la evaluación completa del predicado para cada petición, incluidas las que debería ignorar la ruta.
- Definir rutas `Weight` con el mismo `order` pero sin el mismo `Path`: si los paths difieren, `Weight` no tiene efecto porque las rutas no compiten por las mismas peticiones.

## Comparación: composición de predicados

La siguiente tabla muestra cómo lograr distintos tipos de lógica booleana combinando predicados built-in.

| Lógica deseada | Implementación en Gateway |
|----------------|--------------------------|
| A AND B | Dos predicados en la misma ruta |
| A OR B | Dos rutas separadas con el mismo `uri` y `order` |
| NOT A | Prefijo `!` en el predicado: `- "!Method=POST"` |
| A AND (B OR C) | Dos rutas: `[A, B]` y `[A, C]` con mismo `uri` y `order` |
| Distribución porcentual | `Weight` con grupo compartido |

> [CONCEPTO] La composición AND implícita es la regla más importante a recordar: dentro de una sola ruta, todos los predicados deben cumplirse. La OR requiere múltiples rutas. No existe un operador OR sintáctico en la DSL YAML de predicados.

---

← [3.2 Definición y configuración de rutas](sc-gateway-rutas.md) | [Índice](README.md) | [3.4.1 Filtros de cabeceras del request](sc-gateway-filtros-request-headers.md) →
