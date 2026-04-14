# 6.2 Rutas: Route, Predicates y YAML vs DSL

← [6.1 Concepto y Arquitectura](./06-01-gateway-concepto.md) | [Índice](./README.md) | [6.3 Filtros Predefinidos →](./06-03-gateway-filtros-predefinidos.md)

---

Sin rutas, el Gateway devuelve 404 para toda petición. Una **Route** une tres elementos: qué peticiones aplican (Predicates), qué transformaciones se hacen (Filters) y a dónde se enrutan (URI). Dominar la sintaxis de rutas y saber cuándo usar YAML frente al Java DSL es la base para gestionar un Gateway con decenas de servicios sin que la configuración se vuelva inmanejable.

> [PREREQUISITO] Tener `spring-cloud-starter-gateway` como dependencia y **sin** `spring-boot-starter-web` en el classpath. Ver [6.1](./06-01-gateway-concepto.md).

---

## 6.2.1 Árbol de componentes de una Route

```
Route
├── id          identificador único (string libre)
├── uri         destino: lb://servicio | http://host:puerto | forward:/path
├── order       prioridad de evaluación (menor número = se evalúa antes)
├── predicates  condiciones AND — todas deben cumplirse
│   ├── Path=/api/pedidos/**
│   ├── Method=GET,POST
│   └── Header=X-API-Version, v2
└── filters     transformaciones pre/post
    ├── StripPrefix=1
    ├── AddRequestHeader=X-Source, gateway
    └── CircuitBreaker(name=pedidosCB, fallbackUri=forward:/fallback/pedidos)
```

> [CONCEPTO] Varios predicates en la misma ruta se evalúan como **AND lógico**. Para lógica OR en YAML la única opción es declarar dos rutas separadas apuntando al mismo `uri`.

---

## 6.2.3 Configuración en YAML

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      routes:

        # Ruta básica con StripPrefix
        - id: pedidos-route
          uri: lb://pedidos-service        # lb:// delega la resolución al LoadBalancer (Eureka/Consul)
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1                # /api/pedidos/42  →  /pedidos/42

        # Múltiples predicates (todos deben cumplirse — AND)
        - id: productos-v2-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
            - Method=GET
            - Header=X-API-Version, v2
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway

        # URL externa fija (sin LoadBalancer)
        - id: pagos-externos-route
          uri: https://api.pagos-externos.com
          predicates:
            - Path=/pagos/**
          filters:
            - RewritePath=/pagos/(?<segmento>.*), /v3/${segmento}

        # Canary deployment — 80 % v1, 20 % v2
        - id: catalogo-v1
          uri: lb://catalogo-service-v1
          predicates:
            - Path=/api/catalogo/**
            - Weight=catalogo-group, 80

        - id: catalogo-v2
          uri: lb://catalogo-service-v2
          predicates:
            - Path=/api/catalogo/**
            - Weight=catalogo-group, 20

        # Restringida por IP — solo red interna
        - id: admin-route
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            - RemoteAddr=10.0.0.0/8
          order: 1                         # se evalúa antes que rutas genéricas

        # Ventana temporal (campaña)
        - id: promo-navidad
          uri: lb://promo-service
          predicates:
            - Path=/promo/**
            - Between=2025-12-01T00:00:00Z, 2025-12-31T23:59:59Z
```

---

## 6.2.4 Configuración en Java DSL

```java
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import reactor.core.publisher.Mono;

@Configuration
public class GatewayRoutesConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()

            // Equivalente al YAML anterior
            .route("pedidos-route", r -> r
                .path("/api/pedidos/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway-Source", "spring-cloud-gateway")
                    .circuitBreaker(c -> c
                        .setName("pedidosCB")
                        .setFallbackUri("forward:/fallback/pedidos"))
                )
                .uri("lb://pedidos-service")
            )

            // OR lógico entre métodos — solo posible en DSL
            .route("productos-get-o-head", r -> r
                .path("/api/productos/**")
                .and()
                .method(HttpMethod.GET, HttpMethod.HEAD)
                .filters(f -> f.stripPrefix(1))
                .uri("lb://productos-service")
            )

            // ModifyResponseBody — solo disponible en DSL, no en YAML
            .route("legacy-xml-route", r -> r
                .path("/legacy/**")
                .filters(f -> f
                    .modifyResponseBody(String.class, String.class,
                        (exchange, body) -> Mono.just(transformarXmlAJson(body)))
                )
                .uri("http://sistema-legacy:8080")
            )

            .build();
    }

    private String transformarXmlAJson(String xml) {
        // Elimina tags XML y envuelve en JSON; en producción usar Jackson-dataformat-xml
        return "{\"data\": \"" + xml.replaceAll("<[^>]+>", "").trim() + "\"}";
    }
}
```

> [ADVERTENCIA] `ModifyRequestBody` y `ModifyResponseBody` **solo existen en Java DSL**, no en YAML. Para modificar el body desde YAML hay que implementar un `GatewayFilter` personalizado (ver [6.4](./06-04-gateway-filtros-custom.md)).

---

## 6.2.2 Todos los Predicates disponibles

| Predicate | Ejemplo YAML | Descripción |
|---|---|---|
| `Path` | `Path=/pedidos/**` | Patrón de ruta URL (ant-style) |
| `Method` | `Method=GET,POST` | Método HTTP |
| `Header` | `Header=X-Version, v2` | Header presente con valor (regex) |
| `Query` | `Query=format, json` | Query param con valor (regex) |
| `Host` | `Host=**.miempresa.com` | Valor del header `Host` |
| `After` | `After=2025-01-01T00:00:00Z` | Solo tras fecha ISO-8601 |
| `Before` | `Before=2025-12-31T23:59:59Z` | Solo antes de fecha ISO-8601 |
| `Between` | `Between=2025-01-01T00:00:00Z, 2025-06-30T23:59:59Z` | Ventana temporal |
| `Cookie` | `Cookie=sessionId, abc\d+` | Cookie con valor (regex) |
| `RemoteAddr` | `RemoteAddr=192.168.1.0/24` | IP del cliente o rango CIDR |
| `XForwardedRemoteAddr` | `XForwardedRemoteAddr=10.0.0.0/8` | Como `RemoteAddr` pero lee `X-Forwarded-For` (cuando hay proxy delante) |
| `Weight` | `Weight=grupo1, 80` | Porcentaje de tráfico del grupo para A/B o canary |

---

## 6.2.5 Orden de evaluación y campo `order`

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `id` | String | Autogenerado (UUID) | Identificador único de la ruta |
| `uri` | String | — (obligatorio) | Destino: `lb://`, `http://`, `https://`, `forward:/` |
| `order` | int | 0 | Prioridad de evaluación; menor número se evalúa antes |
| `predicates` | List | `[]` | Condiciones AND que debe cumplir la petición |
| `filters` | List | `[]` | Transformaciones pre/post aplicadas a la petición/respuesta |

---

## 6.2.6 Canary deployment y A/B testing con Weight

El predicate `Weight` distribuye el tráfico entre grupos de rutas según un porcentaje. Es la forma estándar de hacer canary deployment o A/B testing en el Gateway sin infraestructura adicional.

```yaml
spring:
  cloud:
    gateway:
      routes:
        # 90 % del tráfico va a la versión estable
        - id: checkout-v1
          uri: lb://checkout-service-v1
          predicates:
            - Path=/api/checkout/**
            - Weight=checkout-group, 90

        # 10 % va a la nueva versión bajo prueba
        - id: checkout-v2
          uri: lb://checkout-service-v2
          predicates:
            - Path=/api/checkout/**
            - Weight=checkout-group, 10
```

El Gateway asigna cada petición a uno de los dos grupos de forma aleatoria pero manteniendo la distribución porcentual. No hay afinidad de sesión por defecto: el mismo usuario puede ser dirigido a v1 en una petición y a v2 en la siguiente.

> [EXAMEN] Los pesos son relativos dentro del grupo, no porcentajes absolutos. `Weight=grupo, 1` y `Weight=grupo, 9` distribuyen el tráfico en proporción 10 %/90 %, igual que `Weight=grupo, 10` y `Weight=grupo, 90`. Los valores absolutos no importan, solo la proporción entre ellos.

---

## 6.2.7 Rutas con `forward:` — fallbacks internos

Cuando el Circuit Breaker detecta que un microservicio está fallando, redirige la petición a un endpoint del propio Gateway en lugar de devolver un error genérico al cliente. `forward:` es la URI especial que indica que el destino es un controller en el mismo proceso del Gateway.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Map;

@RestController
public class FallbackController {

    // Un único endpoint genérico para todos los servicios
    @RequestMapping("/fallback/{servicio}")
    public ResponseEntity<Map<String, String>> fallback(@PathVariable String servicio) {
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "service_unavailable",
                "servicio", servicio,
                "mensaje", "El servicio no está disponible temporalmente. Inténtelo de nuevo en unos segundos."
            ));
    }
}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: pedidosCB
                fallbackUri: forward:/fallback/pedidos
```

El flujo cuando el Circuit Breaker está abierto:
```
Cliente → Gateway → CB abierto → forward:/fallback/pedidos → FallbackController → 503 al cliente
                               (nunca llega al microservicio)
```

---

← [6.1 Concepto y Arquitectura](./06-01-gateway-concepto.md) | [Índice](./README.md) | [6.3 Filtros Predefinidos →](./06-03-gateway-filtros-predefinidos.md)
