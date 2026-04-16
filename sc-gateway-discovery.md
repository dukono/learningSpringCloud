# 3.6 Integración con Service Discovery y Load Balancer

← [3.5 GlobalFilters: filtros transversales y personalización](sc-gateway-globalfilters.md) | [Índice](README.md) | [3.7 Timeouts y configuración del cliente HTTP Netty](sc-gateway-timeouts.md) →

---

## Introducción

Hardcodear IPs o hostnames de servicios backend en la configuración del gateway produce un gateway frágil: cualquier cambio de instancia (restart, scaling, migración) requiere actualizar la configuración manualmente. Spring Cloud Gateway resuelve este problema integrándose con un `DiscoveryClient` (Eureka, Consul, Kubernetes, etc.) de dos formas: mediante el scheme `lb://` en rutas definidas manualmente (que `ReactiveLoadBalancerClientFilter` resuelve en tiempo de ejecución), y mediante el discovery locator que genera rutas automáticamente a partir de los servicios registrados. El desarrollador necesita integrar Gateway con el service discovery para enrutar peticiones por nombre lógico de servicio sin hardcodear IPs.

> [PREREQUISITO] Esta integración requiere un `DiscoveryClient` registrado en el contexto. La configuración de Eureka como registry se cubre en [2 Spring Cloud Netflix Eureka](sc-eureka-arquitectura.md). La política de balanceo (round-robin, random, weighted) se cubre en el módulo de Spring Cloud LoadBalancer.

## Diagrama: resolución de lb:// y discovery locator

El siguiente diagrama muestra los dos caminos de integración entre Gateway y el service discovery.

```
Camino 1: ruta manual con lb://
─────────────────────────────────────────────────────────────
Petición /api/orders/123
    │
    ▼
Gateway (ruta: uri: lb://order-service)
    │
    ▼
ReactiveLoadBalancerClientFilter
    │ consulta ServiceInstanceListSupplier
    ▼
DiscoveryClient (Eureka / Kubernetes / Consul)
    │ lista de instancias registradas como "order-service"
    ▼
ReactorLoadBalancer (round-robin por defecto)
    │ selecciona instancia: 10.0.1.15:8080
    ▼
NettyRoutingFilter → HTTP a http://10.0.1.15:8080/api/orders/123
                     (con StripPrefix si está configurado)

Camino 2: discovery locator automático
─────────────────────────────────────────────────────────────
spring.cloud.gateway.discovery.locator.enabled=true

DiscoveryClient registra: ORDER-SERVICE, CATALOG-SERVICE
    │
    ▼
Gateway genera automáticamente:
  Ruta: id=ORDER-SERVICE
        predicates: Path=/ORDER-SERVICE/**
        filters:    RewritePath=/ORDER-SERVICE/(?<remaining>.*), /${remaining}
        uri: lb://ORDER-SERVICE

  Ruta: id=CATALOG-SERVICE
        predicates: Path=/CATALOG-SERVICE/**
        uri: lb://CATALOG-SERVICE
```

## Ejemplo central

El siguiente ejemplo muestra la configuración completa de ambos modos de integración con service discovery.

### Dependencias Maven

```xml
<!-- Gateway -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!-- Eureka Client (DiscoveryClient) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!-- LoadBalancer (incluido transitivamente, pero explícito para claridad) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### Configuración YAML — rutas manuales con lb://

```yaml
# application.yml
spring:
  application:
    name: api-gateway

  cloud:
    gateway:
      routes:
        # Ruta manual: usa lb:// con nombre del servicio registrado en Eureka
        - id: order-service-route
          uri: lb://order-service           # nombre del servicio en Eureka (case-insensitive)
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

        - id: catalog-service-route
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - StripPrefix=1

        # Ruta con include-expression: solo enruta si el servicio cumple la condición
        # (útil para filtrar servicios internos del discovery)

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
```

### Configuración YAML — discovery locator automático

```yaml
# application.yml (modo discovery locator)
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          # lower-case-service-id: true convierte ORDER-SERVICE → order-service
          # Permite usar /order-service/** en lugar de /ORDER-SERVICE/**
          lower-case-service-id: true
          # include-expression: SpEL que se evalúa para cada servicio descubierto.
          # Solo se generan rutas para servicios donde la expresión es true.
          # metadata['gateway.enabled'] != 'false' → excluye servicios marcados como no-gateway
          include-expression: metadata['gateway.enabled'] != 'false'
          # url-expression: cómo construir la URI del servicio
          # Por defecto: 'lb://' + serviceId
          url-expression: "'lb://' + serviceId"
```

### Personalización de predicados y filtros del discovery locator

Los predicados y filtros que genera el discovery locator son configurables mediante propiedades, aunque para control granular es preferible definir rutas manuales.

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
          # predicates personalizados para las rutas auto-generadas
          predicates:
            - name: Path
              args:
                pattern: "'/'+serviceId+'/**'"
          # filtros personalizados
          filters:
            - name: RewritePath
              args:
                regexp: "'/' + serviceId + '/(?<remaining>.*)'"
                replacement: "'/${remaining}'"
```

### Verificación: consultar rutas generadas vía Actuator

```bash
# Verificar qué rutas generó el discovery locator
curl http://localhost:8080/actuator/gateway/routes | jq '.[] | {id: .route_id, uri: .uri}'

# Respuesta esperada con lower-case-service-id=true:
# { "id": "order-service", "uri": "lb://order-service" }
# { "id": "catalog-service", "uri": "lb://catalog-service" }
```

## Tabla de elementos clave

La siguiente tabla recoge las propiedades de integración con service discovery más relevantes.

| Propiedad | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.gateway.discovery.locator.enabled` | `boolean` | `false` | Activa la generación automática de rutas desde el DiscoveryClient |
| `spring.cloud.gateway.discovery.locator.lower-case-service-id` | `boolean` | `false` | Convierte el serviceId a minúsculas en el path y en el id de la ruta |
| `spring.cloud.gateway.discovery.locator.include-expression` | `String` (SpEL) | `true` | Expresión SpEL para filtrar servicios; si es `false`, el servicio no genera ruta |
| `spring.cloud.gateway.discovery.locator.url-expression` | `String` (SpEL) | `'lb://' + serviceId` | Expresión para construir la URI del servicio |
| `spring.cloud.loadbalancer.ribbon.enabled` | `boolean` | `false` | [LEGACY] Deshabilita Ribbon; en Spring Boot 4 Ribbon no está disponible |
| `spring.cloud.loadbalancer.strategy` | `String` | `round-robin` | Estrategia de balanceo: `round-robin`, `random` |

> [EXAMEN] Con `discovery.locator.enabled=true` y rutas manuales coexistentes, las rutas manuales tienen prioridad si tienen `order` menor que las rutas auto-generadas. Las rutas del locator tienen `order=0` por defecto. Si una ruta manual no define `order`, también es `0` y el comportamiento depende del orden de inserción.

> [ADVERTENCIA] El discovery locator expone todos los servicios registrados en Eureka con una ruta pública. Si el gateway es accesible desde internet, esto puede exponer servicios internos de infraestructura (ej: Eureka, Config Server, bases de datos con cliente HTTP). Usar `include-expression` para filtrar servicios que no deben ser accesibles externamente.

## Buenas y malas prácticas

**Hacer:**
- Usar `lower-case-service-id=true` en todos los entornos: los servicios Eureka registran su nombre en mayúsculas por convención Java, pero las URLs con mayúsculas son propensas a errores en clientes; el lowercase las normaliza.
- Usar rutas manuales con `lb://` en producción para servicios con lógica de enrutamiento específica (predicados personalizados, filtros de autenticación, rate limiting): el discovery locator genera rutas genéricas sin filtros de seguridad.
- Añadir `include-expression: metadata['gateway.enabled'] != 'false'` al discovery locator: permite a cada servicio declarar si quiere ser expuesto por el gateway añadiendo metadatos en su configuración de Eureka.

**Evitar:**
- Usar discovery locator como única estrategia de enrutamiento en APIs públicas: expone automáticamente todos los servicios sin distinción de visibilidad, creando superficie de ataque.
- Depender del orden de registro en Eureka para resolver conflictos de rutas: el orden es no determinista entre reinicios; usar `order` explícito en rutas manuales.
- Mezclar nombres de servicios con mayúsculas y minúsculas en rutas manuales vs. discovery locator: si `lower-case-service-id=true` y una ruta manual usa `lb://ORDER-SERVICE` (mayúsculas), la ruta manual puede no coincidir con las instancias que el locator registra como `order-service`.

---

← [3.5 GlobalFilters: filtros transversales y personalización](sc-gateway-globalfilters.md) | [Índice](README.md) | [3.7 Timeouts y configuración del cliente HTTP Netty](sc-gateway-timeouts.md) →
