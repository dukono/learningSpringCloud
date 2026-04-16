# 3.11 Filtros y predicados personalizados (extension points)

← [3.10 WebSockets y Server-Sent Events](sc-gateway-websockets.md) | [Índice](README.md) | [3.12 Spring Cloud Gateway MVC](sc-gateway-mvc.md) →

---

## Introducción

Los 12 predicados y los más de 20 filtros built-in cubren la mayoría de los casos de uso estándar, pero hay escenarios que requieren lógica propia: validar un API key contra un vault, añadir un header firmado con HMAC, o enrutar según una versión de esquema en el body del request. Spring Cloud Gateway proporciona dos puntos de extensión bien definidos para estos casos: `AbstractGatewayFilterFactory` para filtros por ruta y `AbstractRoutePredicateFactory` para predicados personalizados. Ambos siguen el mismo patrón de diseño, comparten el mecanismo de configuración YAML con `shortcutFieldOrder`, y se registran automáticamente por autoconfiguración sin necesidad de `@Bean` explícito adicional. El desarrollador necesita implementar factories de filtros y predicados personalizados para cubrir requisitos de enrutamiento no soportados por los built-in.

## Diagrama: estructura de una factory personalizada

El siguiente diagrama muestra el patrón de implementación compartido por `AbstractGatewayFilterFactory` y `AbstractRoutePredicateFactory`.

```
AbstractGatewayFilterFactory<Config>          AbstractRoutePredicateFactory<Config>
  │                                              │
  ├── Config (clase interna)                    ├── Config (clase interna)
  │    └── campos con getters/setters            │    └── campos con getters/setters
  │                                              │
  ├── shortcutFieldOrder()                      ├── shortcutFieldOrder()
  │    └── orden de campos en sintaxis corta     │    └── orden de campos en sintaxis corta
  │         YAML: - NombreFilter=val1,val2       │         YAML: - NombrePredicado=val1,val2
  │                                              │
  └── apply(Config config)                      └── apply(Config config)
       └── devuelve GatewayFilter                    └── devuelve Predicate<ServerWebExchange>
            (lambda o clase anónima)

Registro automático:
  @Component en la factory → Spring detecta el bean
  autoconfiguración de Gateway lo incluye en el contexto
  Nombre de la factory determina el nombre en YAML:
    NombreFactory: "XxxxGatewayFilterFactory" → filtro "Xxxx"
    NombreFactory: "XxxxRoutePredicateFactory" → predicado "Xxxx"
```

## Ejemplo central

El siguiente ejemplo implementa un filtro personalizado que valida un HMAC en el request y un predicado personalizado que evalúa metadata del servicio registrado.

### AbstractGatewayFilterFactory — filtro de validación HMAC

```java
package com.example.gateway;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.util.Arrays;
import java.util.Base64;
import java.util.List;

/**
 * Filtro personalizado que valida una firma HMAC-SHA256 en el header X-Signature.
 *
 * Uso en YAML (sintaxis corta gracias a shortcutFieldOrder):
 *   filters:
 *     - HmacValidation=my-secret-key,X-Signature
 *
 * Uso en YAML (sintaxis completa):
 *   filters:
 *     - name: HmacValidation
 *       args:
 *         secretKey: my-secret-key
 *         signatureHeader: X-Signature
 *
 * El nombre "HmacValidation" viene de "HmacValidationGatewayFilterFactory"
 * menos el sufijo "GatewayFilterFactory".
 */
@Component
public class HmacValidationGatewayFilterFactory
        extends AbstractGatewayFilterFactory<HmacValidationGatewayFilterFactory.Config> {

    public HmacValidationGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        // Define el orden de los campos en la sintaxis corta YAML
        return Arrays.asList("secretKey", "signatureHeader");
    }

    @Override
    public GatewayFilter apply(Config config) {
        // La lambda es el filtro real: recibe el exchange y la cadena
        return (exchange, chain) -> {
            String signature = exchange.getRequest().getHeaders()
                .getFirst(config.getSignatureHeader());

            if (signature == null || signature.isBlank()) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            // Calcular HMAC del path + query string
            String data = exchange.getRequest().getPath().value()
                + "?" + exchange.getRequest().getQueryParams().toSingleValueMap();

            if (!isValidSignature(data, config.getSecretKey(), signature)) {
                exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                return exchange.getResponse().setComplete();
            }

            return chain.filter(exchange);
        };
    }

    private boolean isValidSignature(String data, String secretKey, String signature) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secretKey.getBytes(), "HmacSHA256"));
            String expectedSignature = Base64.getEncoder().encodeToString(
                mac.doFinal(data.getBytes()));
            return expectedSignature.equals(signature);
        } catch (Exception e) {
            return false;
        }
    }

    public static class Config {
        private String secretKey;
        private String signatureHeader;

        public String getSecretKey() { return secretKey; }
        public void setSecretKey(String secretKey) { this.secretKey = secretKey; }
        public String getSignatureHeader() { return signatureHeader; }
        public void setSignatureHeader(String signatureHeader) { this.signatureHeader = signatureHeader; }
    }
}
```

### AbstractRoutePredicateFactory — predicado personalizado por versión de API

```java
package com.example.gateway;

import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;

/**
 * Predicado personalizado que verifica la versión de API en el header X-API-Version.
 *
 * Uso en YAML (sintaxis corta):
 *   predicates:
 *     - ApiVersion=v2
 *
 * Uso en YAML (sintaxis completa):
 *   predicates:
 *     - name: ApiVersion
 *       args:
 *         version: v2
 */
@Component
public class ApiVersionRoutePredicateFactory
        extends AbstractRoutePredicateFactory<ApiVersionRoutePredicateFactory.Config> {

    public ApiVersionRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("version");
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String apiVersion = exchange.getRequest().getHeaders()
                .getFirst("X-API-Version");
            // Si no hay header, evalúa solo si la versión esperada es "any"
            if (apiVersion == null) {
                return "any".equals(config.getVersion());
            }
            return config.getVersion().equals(apiVersion);
        };
    }

    public static class Config {
        private String version;
        public String getVersion() { return version; }
        public void setVersion(String version) { this.version = version; }
    }
}
```

### Uso combinado en YAML

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: v2-api-route
          uri: lb://api-v2-service
          predicates:
            - Path=/api/**
            - ApiVersion=v2                        # predicado personalizado
          filters:
            - HmacValidation=my-secret,X-Signature # filtro personalizado (sintaxis corta)
            - StripPrefix=1
          order: 1

        - id: v1-api-route
          uri: lb://api-v1-service
          predicates:
            - Path=/api/**
            - ApiVersion=v1
          filters:
            - StripPrefix=1
          order: 2
```

## Tabla de elementos clave

La siguiente tabla resume los conceptos de implementación que deben dominarse para entrevistas técnicas.

| Concepto | AbstractGatewayFilterFactory | AbstractRoutePredicateFactory |
|----------|------------------------------|-------------------------------|
| Tipo retornado por `apply()` | `GatewayFilter` (interfaz funcional) | `Predicate<ServerWebExchange>` |
| Clase base | `AbstractGatewayFilterFactory<C>` | `AbstractRoutePredicateFactory<C>` |
| Naming convention | `XxxxGatewayFilterFactory` → `Xxxx` en YAML | `XxxxRoutePredicateFactory` → `Xxxx` en YAML |
| Registro | `@Component` + autoconfiguración | `@Component` + autoconfiguración |
| `shortcutFieldOrder()` | Orden de campos para sintaxis `- Name=val1,val2` | Igual |
| Clase `Config` | Interna, con getters/setters | Interna, con getters/setters |
| Acceso a la `Route` | `exchange.getAttribute(GATEWAY_ROUTE_ATTR)` | No disponible (predicado evalúa antes del routing) |
| `getOrder()` | Implementar `Ordered` en el `GatewayFilter` retornado | N/A (los predicados no tienen orden entre sí) |

> [EXAMEN] El naming convention es crítico: el nombre de la clase factory **debe** terminar en `GatewayFilterFactory` o `RoutePredicateFactory`. Spring Cloud Gateway usa `BeanFactoryUtils` para descubrir los beans por este sufijo. Si la clase se llama `HmacFilter` en lugar de `HmacGatewayFilterFactory`, no se registra automáticamente y el YAML falla con `Unable to find GatewayFilterFactory with name HmacFilter`.

> [ADVERTENCIA] Los filtros personalizados con `@Component` se registran automáticamente y están disponibles para **todas** las rutas. Si el filtro tiene efectos secundarios (escritura en base de datos, llamadas externas), asegurarse de que es stateless o que su estado es thread-safe: el mismo bean es reutilizado para todas las peticiones concurrentes.

## Buenas y malas prácticas

**Hacer:**
- Implementar `shortcutFieldOrder()` en todas las factories: permite usar la sintaxis corta en YAML (`- NombreFilter=val1,val2`), que es más legible y mantiene coherencia con los filtros built-in.
- Hacer la clase `Config` inmutable si es posible (o al menos thread-safe): la instancia de `Config` puede ser compartida entre múltiples instancias del filtro para la misma ruta.
- Retornar `Mono.error()` en lugar de lanzar excepciones en el GatewayFilter: en el modelo reactivo, las excepciones sincrónicas en una lambda pueden no ser capturadas por los manejadores de error del framework.

**Evitar:**
- Guardar estado mutable en campos de la factory (contadores, caches no thread-safe): la factory es un singleton Spring y sus campos son compartidos entre todas las peticiones concurrentes; usar el `ServerWebExchange` para estado por petición.
- Hacer llamadas bloqueantes (JDBC, cliente REST síncrono) dentro del `GatewayFilter`: bloquea el event loop de Reactor Netty para todas las peticiones en ese worker; usar `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` si es inevitable.

---

← [3.10 WebSockets y Server-Sent Events](sc-gateway-websockets.md) | [Índice](README.md) | [3.12 Spring Cloud Gateway MVC](sc-gateway-mvc.md) →
