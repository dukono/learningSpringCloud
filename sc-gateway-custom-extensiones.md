# 5.11 Extensiones custom: Custom Predicate Factory y Custom GatewayFilter Factory

← [5.10 Actuator y observabilidad del Gateway](sc-gateway-actuator-observabilidad.md) | [Índice](README.md) | [5.12 Testing / Verificación de Spring Cloud Gateway](sc-gateway-testing.md) →

---

## Introducción

Spring Cloud Gateway está diseñado para ser extensible: cuando las Predicate Factories y GatewayFilter Factories built-in no cubren un caso de uso específico, se pueden implementar factories custom siguiendo las mismas convenciones que las built-in. Una **Custom Predicate Factory** permite crear condiciones de matching personalizadas (p. ej. validar un claim JWT, comprobar una cabecera con lógica compleja). Una **Custom GatewayFilter Factory** permite crear transformaciones de request/response reutilizables que se configuran por ruta en YAML, exactamente como los filtros built-in. Ambas extensiones se implementan extendiendo clases abstractas del framework y registrándose como beans Spring.

> [CONCEPTO] Una **Custom Predicate Factory** extiende `AbstractRoutePredicateFactory<Config>` donde `Config` es una clase interna con los parámetros de configuración. El nombre del shortcut en YAML es el nombre de la clase menos el sufijo `RoutePredicateFactory`.

> [CONCEPTO] Una **Custom GatewayFilter Factory** extiende `AbstractGatewayFilterFactory<Config>`. El nombre del shortcut en YAML es el nombre de la clase menos el sufijo `GatewayFilterFactory`. El método `apply(Config config)` retorna una lambda `GatewayFilter`.

## Estructura de una Custom Predicate Factory

Una Predicate Factory custom sigue un patrón bien definido. La clase extiende `AbstractRoutePredicateFactory<Config>` con una clase interna `Config` que tiene los campos que el usuario puede configurar en YAML. El método `shortcutFieldOrder()` define el orden de los campos cuando se usa la sintaxis shortcut (`PredicateName=valor1,valor2`). El método `apply(Config config)` retorna el predicado real como lambda reactiva.

El nombre de la clase determina el shortcut YAML. Si la clase se llama `ClientVersionRoutePredicateFactory`, el shortcut en YAML es `ClientVersion=`.

## Estructura de una Custom GatewayFilter Factory

Una GatewayFilter Factory custom sigue el mismo patrón. La clase extiende `AbstractGatewayFilterFactory<Config>`. El método `apply(Config config)` retorna un `GatewayFilter` como lambda que recibe `(exchange, chain)` y puede ejecutar lógica PRE (antes de `chain.filter(exchange)`) y POST (en el `Mono` retornado).

El nombre de la clase determina el shortcut YAML. Si la clase se llama `RequestTimingGatewayFilterFactory`, el shortcut es `RequestTiming=`.

## Ejemplo central

El siguiente ejemplo implementa una Custom Predicate Factory que valida la versión mínima de cliente via header, y una Custom GatewayFilter Factory que añade timing de la petición en un header de respuesta:

```java
package com.example.gateway.predicate;

import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.server.ServerWebExchange;

import jakarta.validation.constraints.NotEmpty;
import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;

/**
 * Custom Predicate Factory: valida que el header X-Client-Version cumpla la versión mínima.
 *
 * Uso en YAML:
 *   predicates:
 *     - ClientVersion=1.5.0   (shortcut: versión mínima requerida)
 *     # O forma expandida:
 *     - name: ClientVersion
 *       args:
 *         minVersion: 1.5.0
 *
 * Shortcut YAML: "ClientVersion" (nombre de clase sin sufijo RoutePredicateFactory)
 */
@Component
public class ClientVersionRoutePredicateFactory
        extends AbstractRoutePredicateFactory<ClientVersionRoutePredicateFactory.Config> {

    public ClientVersionRoutePredicateFactory() {
        super(Config.class);
    }

    /**
     * Define el orden de los campos en el shortcut YAML.
     * "ClientVersion=1.5.0" → config.setMinVersion("1.5.0")
     */
    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("minVersion");
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            HttpHeaders headers = exchange.getRequest().getHeaders();
            String clientVersion = headers.getFirst("X-Client-Version");

            if (clientVersion == null || clientVersion.isBlank()) {
                // Si no se envía el header, no coincide (ruta no activa)
                return false;
            }

            // Comparación semántica simple: major.minor.patch
            return isVersionSufficient(clientVersion, config.getMinVersion());
        };
    }

    /**
     * Compara versiones en formato major.minor.patch.
     * Retorna true si actual >= required.
     */
    private boolean isVersionSufficient(String actual, String required) {
        try {
            int[] actualParts = parseVersion(actual);
            int[] requiredParts = parseVersion(required);

            for (int i = 0; i < 3; i++) {
                if (actualParts[i] > requiredParts[i]) return true;
                if (actualParts[i] < requiredParts[i]) return false;
            }
            return true; // versiones iguales
        } catch (Exception e) {
            return false; // formato inválido → no coincide
        }
    }

    private int[] parseVersion(String version) {
        String[] parts = version.split("\\.");
        return new int[]{
            Integer.parseInt(parts[0]),
            parts.length > 1 ? Integer.parseInt(parts[1]) : 0,
            parts.length > 2 ? Integer.parseInt(parts[2]) : 0
        };
    }

    @Validated
    public static class Config {
        @NotEmpty
        private String minVersion;

        public String getMinVersion() {
            return minVersion;
        }

        public void setMinVersion(String minVersion) {
            this.minVersion = minVersion;
        }
    }
}
```

```java
package com.example.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.Arrays;
import java.util.List;

/**
 * Custom GatewayFilter Factory: mide el tiempo de procesamiento de la petición
 * y añade el resultado en el header de respuesta X-Response-Time.
 *
 * Uso en YAML:
 *   filters:
 *     - RequestTiming=X-Response-Time   (shortcut: nombre del header)
 *     # O forma expandida:
 *     - name: RequestTiming
 *       args:
 *         headerName: X-Response-Time
 *
 * Shortcut YAML: "RequestTiming" (nombre sin sufijo GatewayFilterFactory)
 */
@Component
public class RequestTimingGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RequestTimingGatewayFilterFactory.Config> {

    public RequestTimingGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("headerName");
    }

    @Override
    public GatewayFilter apply(Config config) {
        // La lambda GatewayFilter contiene la lógica PRE y POST
        return (ServerWebExchange exchange, GatewayFilterChain chain) -> {
            // === FASE PRE: capturar tiempo de inicio ===
            long startTime = Instant.now().toEpochMilli();

            return chain.filter(exchange)
                // === FASE POST: calcular duración y añadir header ===
                .then(Mono.fromRunnable(() -> {
                    ServerHttpResponse response = exchange.getResponse();
                    long duration = Instant.now().toEpochMilli() - startTime;
                    String headerName = config.getHeaderName() != null
                        ? config.getHeaderName()
                        : "X-Response-Time";
                    // Añadir header de timing a la respuesta
                    response.getHeaders().add(headerName, duration + "ms");
                }));
        };
    }

    public static class Config {
        private String headerName = "X-Response-Time"; // valor por defecto

        public String getHeaderName() {
            return headerName;
        }

        public void setHeaderName(String headerName) {
            this.headerName = headerName;
        }
    }
}
```

```yaml
# application.yml — Uso de las factories custom
spring:
  cloud:
    gateway:
      routes:
        - id: versioned-api
          uri: lb://api-service
          predicates:
            - Path=/api/**
            # Custom Predicate: solo acepta clientes con versión >= 1.5.0
            - ClientVersion=1.5.0
          filters:
            - StripPrefix=1
            # Custom Filter: añade X-Response-Time a la respuesta
            - RequestTiming=X-Response-Time

        - id: timing-default-header
          uri: lb://other-service
          predicates:
            - Path=/other/**
          filters:
            - StripPrefix=1
            # Sin parámetro → usa el valor por defecto "X-Response-Time"
            - RequestTiming
```

## Tabla de convenciones de extensión

| Aspecto | Custom Predicate Factory | Custom GatewayFilter Factory |
|---|---|---|
| Clase base | `AbstractRoutePredicateFactory<Config>` | `AbstractGatewayFilterFactory<Config>` |
| Shortcut YAML | Nombre clase - `RoutePredicateFactory` | Nombre clase - `GatewayFilterFactory` |
| Método principal | `apply(Config config)` → `Predicate<ServerWebExchange>` | `apply(Config config)` → `GatewayFilter` |
| Shortcut order | `shortcutFieldOrder()` → `List<String>` | `shortcutFieldOrder()` → `List<String>` |
| Registro | `@Component` o `@Bean` | `@Component` o `@Bean` |
| Fase de ejecución | Evaluación en matching de ruta | PRE y/o POST en la cadena de filtros |

## Detalles sobre la clase Config interna

La clase `Config` interna es un simple POJO con campos, getters y setters. Spring Cloud Gateway usa esta clase para deserializar los argumentos de configuración del YAML a un objeto tipado que se pasa al método `apply()`. Los campos deben tener setters con el mismo nombre que las claves en YAML.

> [EXAMEN] Para que el shortcut YAML funcione (`FilterName=valor`), debes implementar `shortcutFieldOrder()` retornando la lista de nombres de campo de `Config` en el orden en que se mapean los valores del shortcut. Si no implementas `shortcutFieldOrder()`, solo funciona la forma expandida con `name:` y `args:`.

> [ADVERTENCIA] El nombre de la clase es exactamente el shortcut YAML menos el sufijo. `MyCustomGatewayFilterFactory` → shortcut `MyCustom=`. Si la clase se llama `MyCustomFilterFactory` (sin "Gateway"), el shortcut sería `MyCustomFilter=`. La convención es siempre incluir "Gateway" en el nombre de las GatewayFilter Factories.

## Buenas y malas prácticas

**Buenas prácticas:**
- Seguir exactamente la convención de nomenclatura: `NombreRoutePredicateFactory` / `NombreGatewayFilterFactory`.
- Usar `@Validated` y `@NotEmpty` en la clase `Config` para validación temprana en el arranque.
- Implementar `shortcutFieldOrder()` siempre para compatibilidad con sintaxis shortcut en YAML.
- Usar `.then(Mono.fromRunnable(...))` para lógica POST no bloqueante en GatewayFilterFactory.

**Malas prácticas:**
- Poner lógica bloqueante en el `apply()` o en el `GatewayFilter` retornado: bloquea el EventLoop.
- Omitir `shortcutFieldOrder()`: la factory solo funciona con la sintaxis expandida `name: / args:`, confundiendo a quien la configura.
- Modificar directamente los headers del `ServerHttpRequest` sin `mutate()`: lanza `UnsupportedOperationException`.
- Crear una GatewayFilter Factory sin estado mutable en `Config`: el estado global en la factory es problemático bajo carga concurrente.

## Verificación y práctica

1. ¿Qué clase base debes extender para crear un GatewayFilter Factory custom? ¿Cómo se llama el método que retorna el filtro real?

2. Si creas una clase llamada `RateLimitByUserGatewayFilterFactory`, ¿cuál será el shortcut en YAML?

3. ¿Qué hace `shortcutFieldOrder()` y qué ocurre si no lo implementas?

4. ¿Cómo ejecutas lógica POST (después de que el upstream responde) en una Custom GatewayFilter Factory? ¿Qué operador reactivo usas?

5. ¿Por qué debes usar `exchange.mutate().request(...)` para modificar headers en un filtro en lugar de modificar directamente `exchange.getRequest().getHeaders()`?

---

← [5.10 Actuator y observabilidad del Gateway](sc-gateway-actuator-observabilidad.md) | [Índice](README.md) | [5.12 Testing / Verificación de Spring Cloud Gateway](sc-gateway-testing.md) →
