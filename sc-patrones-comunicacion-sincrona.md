# 13.2 Comunicación síncrona: API Gateway y Backend for Frontend

← [13.1 Descomposición por capacidades de negocio](sc-patrones-descomposicion-dominio.md) | [Índice](README.md) | [13.3 Comunicación asíncrona y eventos](sc-patrones-comunicacion-asincrona-eventos.md) →

---

## Introducción

En una arquitectura de microservicios, los clientes externos no pueden ni deben conocer la topología interna del sistema. El patrón API Gateway actúa como punto de entrada único que encapsula esa complejidad, aplica políticas transversales y expone una superficie de API estable. Cuando diferentes tipos de clientes necesitan representaciones distintas de los mismos datos, el patrón Backend for Frontend (BFF) extiende este concepto creando gateways especializados por tipo de cliente.

## Diagrama: API Gateway como punto de entrada único

El siguiente diagrama muestra cómo el API Gateway actúa como fachada frente a múltiples microservicios internos, gestionando autenticación, enrutamiento y rate limiting de forma centralizada.

```
Cliente Web / Móvil / Externo
          │
          ▼
  ┌─────────────────┐
  │   API Gateway   │  ← autenticación, rate limiting, routing, SSL termination
  └────────┬────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
 Order  Inventory  User
Service  Service  Service
```

> [CONCEPTO] **API Gateway pattern**: el gateway es la única entrada al sistema de microservicios desde el exterior. Sus responsabilidades incluyen: enrutamiento de peticiones, autenticación y autorización centralizada, terminación TLS/SSL, rate limiting, logging y trazabilidad, y transformación de protocolos (REST → gRPC, por ejemplo). En el ecosistema Spring Cloud, la implementación de referencia es Spring Cloud Gateway.

## Backend for Frontend (BFF)

El patrón BFF reconoce que distintos tipos de clientes tienen necesidades diferentes: una app móvil necesita payloads compactos y optimizados para latencia en redes móviles, mientras que una SPA web puede manejar respuestas más ricas, y una API pública para terceros requiere contratos estables y versionados.

> [CONCEPTO] **Backend for Frontend (BFF)**: en lugar de un único API Gateway genérico, se crea un gateway especializado por tipo de cliente. Cada BFF agrega, filtra y transforma los datos del backend según las necesidades de su cliente específico. El BFF tiene su propio ciclo de vida y puede ser mantenido por el equipo de frontend del cliente correspondiente.

La elección entre un único API Gateway y múltiples BFF depende de la heterogeneidad de los clientes y la frecuencia de cambio de sus requisitos:

| Criterio | API Gateway único | BFF por tipo de cliente |
|---|---|---|
| Homogeneidad de clientes | Alta | Baja |
| Autonomía de equipos de frontend | No prioritaria | Priorizada |
| Mantenimiento | Un único punto | Múltiples equipos |
| Riesgo de acoplamiento | Bajo | Medio (riesgo de duplicación de lógica) |
| Optimización por cliente | Limitada | Alta |

## Ejemplo central: Spring Cloud Gateway como API Gateway

El siguiente ejemplo configura Spring Cloud Gateway para implementar el patrón API Gateway con enrutamiento a múltiples microservicios, autenticación JWT centralizada y rate limiting. El ejemplo incluye configuración YAML completa y un filtro global personalizado de trazabilidad.

```yaml
# application.yml — Spring Cloud Gateway como API Gateway

spring:
  cloud:
    gateway:
      routes:
        - id: orders-service
          uri: lb://orders-service          # lb:// usa Spring Cloud LoadBalancer + Eureka
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1                 # elimina /api del path antes de enrutar
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"

        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - StripPrefix=1

        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1

      default-filters:
        - TokenRelay               # propaga el JWT del usuario a los microservicios downstream
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://auth-server:9000
```

```java
// GlobalFilter de trazabilidad: añade traceId en todas las respuestas

package com.example.gateway.filters;

import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class TraceResponseFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            String traceId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-B3-TraceId");
            if (traceId != null) {
                exchange.getResponse()
                    .getHeaders()
                    .add("X-Trace-Id", traceId);
            }
        }));
    }

    @Override
    public int getOrder() {
        return -1; // ejecutar cerca del principio de la cadena
    }
}

// Bean para resolver la clave de rate limiting por usuario autenticado
@Component("userKeyResolver")
public class UserKeyResolver implements KeyResolver {

    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return exchange.getPrincipal()
            .map(Principal::getName)
            .defaultIfEmpty("anonymous");
    }
}
```

## Service Mesh vs Spring Cloud Gateway

> [CONCEPTO] **Service Mesh vs Spring Cloud Gateway**: un Service Mesh (Istio, Linkerd) gestiona la comunicación **este-oeste** (service-to-service dentro del clúster), mientras que Spring Cloud Gateway gestiona la comunicación **norte-sur** (cliente externo → clúster). El Service Mesh opera a nivel de red (sidecar proxy) sin modificar el código de la aplicación, mientras que el Gateway opera a nivel de aplicación. Son complementarios: el Gateway para el acceso externo, el Service Mesh para las políticas de red interna.

> [ADVERTENCIA] Un API Gateway mal diseñado puede convertirse en un cuello de botella o un punto único de fallo (SPOF). En producción debe desplegarse en alta disponibilidad con al menos 2 réplicas y un circuit breaker para los servicios downstream.

## Buenas y malas prácticas

**Buenas prácticas:**
- El Gateway gestiona únicamente responsabilidades transversales (authn, authz, routing, observabilidad). La lógica de negocio pertenece a los microservicios.
- Implementar circuit breakers en el gateway para los servicios downstream.
- Versionar las rutas del gateway (`/api/v1/...`, `/api/v2/...`) para mantener compatibilidad con clientes existentes.
- En BFF, cada BFF es mantenido por el equipo del cliente que sirve (team ownership).

**Malas prácticas:**
- Poner lógica de negocio en el API Gateway (transformaciones complejas, validaciones de dominio).
- Crear un BFF por cada microservicio del backend (eso es el antipatrón de re-crear el monolito).
- Compartir el mismo BFF entre tipos de cliente muy diferentes (móvil y web con necesidades divergentes).

## Verificación y práctica

> [EXAMEN] 1. ¿Por qué se usa un API Gateway en lugar de exponer directamente cada microservicio al cliente, y qué responsabilidades asume el gateway?

> [EXAMEN] 2. ¿En qué caso creas un BFF separado por tipo de cliente en lugar de un único API Gateway, y cuál es el trade-off principal?

> [EXAMEN] 3. ¿Qué responsabilidades de red gestiona un Service Mesh que Spring Cloud Gateway no cubre, y cuándo son complementarios?

> [EXAMEN] 4. ¿Qué filtro de Spring Cloud Gateway se usa para propagar el token JWT del usuario autenticado a los microservicios downstream, y por qué es necesario?

---

← [13.1 Descomposición por capacidades de negocio](sc-patrones-descomposicion-dominio.md) | [Índice](README.md) | [13.3 Comunicación asíncrona y eventos](sc-patrones-comunicacion-asincrona-eventos.md) →
