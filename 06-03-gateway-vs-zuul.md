# 6.1.3 Spring Cloud Gateway vs Zuul 1 y Zuul 2

← [6.1.2 Arquitectura reactiva](./06-02-gateway-arquitectura.md) | [Índice](./README.md) | [6.1.4 Propiedades globales →](./06-04-gateway-propiedades-globales.md)

---

Cuando un equipo decide incorporar un API Gateway a su arquitectura de microservicios con Spring Cloud, la pregunta inevitable es: ¿qué solución elegir y por qué cambiar si ya se usa Zuul? Comprender las diferencias entre Zuul 1, Zuul 2 y Spring Cloud Gateway no es un ejercicio académico: es una decisión arquitectónica con impacto directo en el rendimiento bajo carga, la complejidad del código de filtros y el soporte a largo plazo. Los equipos que vienen de Zuul 1 necesitan saber con precisión qué ganan al migrar, qué tienen que cambiar y por qué Zuul 2, pese a existir, nunca es una opción real dentro del ecosistema Spring.

## Tabla comparativa

Las tres opciones difieren fundamentalmente en su modelo de ejecución, su integración con Spring y su estado actual de mantenimiento.

| Aspecto | Zuul 1 `[LEGACY]` | Zuul 2 `[LEGACY]` | Spring Cloud Gateway |
|---|---|---|---|
| Modelo de hilos | Bloqueante (un hilo por petición) | No bloqueante | No bloqueante |
| Stack | Servlet + Tomcat | Netty (no integrado en Spring) | WebFlux + Netty |
| Soporte Spring Cloud | Sí (legacy, en maintenance) | No (nunca integrado oficialmente) | Sí (activo) |
| Tipo de filtro | `ZuulFilter.run()` síncrono | Similar, pero no disponible en Spring | `GatewayFilter.filter()` → `Mono<Void>` |
| Configuración YAML | `zuul.routes.*` | N/A | `spring.cloud.gateway.routes.*` |
| Estado | Eliminado como default en 2020 | Nunca integrado | Activo y mantenido |
| Rendimiento bajo carga alta | Limitado por el tamaño del thread pool | Alto | Alto |

> [CONCEPTO] Zuul 1 implementa el patrón Servlet clásico: cada petición ocupa un hilo del pool durante toda su vida útil. Cuando el número de peticiones concurrentes supera el pool, las nuevas peticiones esperan o son rechazadas. Spring Cloud Gateway, al ser reactivo, gestiona miles de conexiones concurrentes con un número reducido de hilos del event loop.

Zuul 2 [LEGACY] fue la respuesta de Netflix al problema de rendimiento de Zuul 1: una reescritura completa basada en Netty para conseguir el mismo modelo no bloqueante que Spring Cloud Gateway. Sin embargo, Netflix lanzó Zuul 2 como librería interna de uso propio, sin publicar artefactos en Maven Central ni alinearlo con el modelo de integración de Spring Cloud. El equipo de Spring Cloud evaluó integrarlo y decidió construir Spring Cloud Gateway desde cero, con integración nativa en el ecosistema Spring (WebFlux, Security, Actuator, Config), en lugar de adaptar una librería diseñada para las necesidades específicas de la infraestructura de Netflix.

## Ejemplo funcional: la misma ruta en Zuul 1 y en Spring Cloud Gateway

Para ilustrar la equivalencia entre ambas configuraciones, se usa el caso más habitual en cualquier proyecto: enrutar todas las peticiones que lleguen a `/api/productos/**` hacia el servicio `productos-service` registrado en el servidor de descubrimiento, eliminando el prefijo de la ruta antes de reenviar la petición.

**Zuul 1 — configuración YAML:**

```yaml
zuul:
  routes:
    productos:
      path: /api/productos/**
      serviceId: productos-service
      stripPrefix: true
```

Con esta configuración, Zuul 1 intercepta cualquier petición cuya ruta comience por `/api/productos/`, resuelve `productos-service` a través de Ribbon (el balanceador de carga de Netflix OSS) y elimina el prefijo antes de reenviar. El hilo queda bloqueado hasta que el servicio de destino responde.

**Spring Cloud Gateway — configuración YAML equivalente:**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1
```

El resultado funcional es idéntico: las peticiones a `/api/productos/listado` llegan al servicio como `/listado`. La diferencia está en el interior: SCG ejecuta el enrutamiento de forma reactiva a través de WebFlux, sin bloquear ningún hilo. El prefijo `lb://` indica a SCG que debe resolver el nombre del servicio mediante el cliente de descubrimiento configurado (Eureka, Consul, etc.).

> [EXAMEN] La propiedad `stripPrefix: true` de Zuul 1 equivale al filtro `StripPrefix=1` de SCG. El número `1` indica cuántos segmentos de ruta se eliminan. Si la ruta fuera `/api/v1/productos/**`, se necesitaría `StripPrefix=2` para eliminar tanto `/api` como `/v1`.

## Buenas y malas prácticas

Hacer:
- Usar Spring Cloud Gateway para cualquier proyecto nuevo con Spring Boot 2.4+ o superior.
- Al migrar desde Zuul 1, eliminar explícitamente `spring-boot-starter-web` del classpath para evitar conflictos entre el servidor Servlet y WebFlux.
- Tratar los filtros de SCG como código reactivo desde el principio: diseñarlos para devolver `Mono<Void>` y encadenar operaciones con operadores reactivos.
- Documentar en el equipo que Zuul 2 no es una opción dentro de Spring Cloud, para evitar confusiones cuando alguien lee documentación de Netflix directamente.
- Mantener los tests de integración con `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)` al migrar; el mecanismo sigue siendo compatible.

Evitar:
- Añadir Zuul 1 `[LEGACY]` a proyectos nuevos aunque aparezca en documentación o tutoriales antiguos; el módulo `spring-cloud-starter-netflix-zuul` fue eliminado como dependencia recomendada en Spring Cloud 2020.0 (Ilford).
- Llamar a código bloqueante (JDBC, llamadas HTTP síncronas con RestTemplate, etc.) dentro de un `GatewayFilter` sin envolver la llamada en `Schedulers.boundedElastic()`.
- Asumir que la configuración de Zuul 2 `[LEGACY]` es accesible desde Spring Cloud; Zuul 2 es una librería interna de Netflix y no tiene artefactos publicados para uso general en Spring.
- Mezclar `spring-boot-starter-web` (stack Servlet) con `spring-cloud-starter-gateway` (stack WebFlux) en el mismo módulo Maven/Gradle.
- Usar `zuul.routes.*` en nuevos ficheros de configuración pensando que SCG los leerá; son namespaces completamente distintos.

## Guía de migración de Zuul 1 a Spring Cloud Gateway

Migrar desde Zuul 1 `[LEGACY]` a Spring Cloud Gateway es una tarea estructurada que implica cambios en las dependencias, en la configuración YAML y en el código de los filtros. Los pasos siguientes cubren todos los puntos de cambio habituales.

> [ADVERTENCIA] La migración no es incremental en el mismo módulo. Zuul 1 requiere el stack Servlet (Tomcat/Jetty) y SCG requiere el stack reactivo (Netty/WebFlux). Ambos no pueden coexistir en el mismo proceso. La migración implica reemplazar, no añadir.

### Paso 1: reemplazar dependencias

En el `pom.xml` o `build.gradle`, sustituir la dependencia de Zuul por la de Gateway y eliminar el starter web de Servlet:

```xml
<!-- Eliminar -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Añadir -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

Spring Cloud Gateway trae transitivamente `spring-boot-starter-webflux` y Netty. No es necesario añadirlos explícitamente.

### Paso 2: convertir rutas YAML

Cada entrada bajo `zuul.routes.*` tiene su equivalente directo en `spring.cloud.gateway.routes.*`. La conversión sigue siempre el mismo patrón:

| Zuul 1 | Spring Cloud Gateway |
|---|---|
| `zuul.routes.<nombre>.path` | `predicates: - Path=<valor>` |
| `zuul.routes.<nombre>.serviceId` | `uri: lb://<serviceId>` |
| `zuul.routes.<nombre>.url` | `uri: http://<url>` |
| `zuul.routes.<nombre>.stripPrefix: true` | `filters: - StripPrefix=1` |
| `zuul.routes.<nombre>.stripPrefix: false` | (omitir el filtro StripPrefix) |

### Paso 3: convertir ZuulFilter a GatewayFilter o GlobalFilter

Un filtro de Zuul 1 tiene esta estructura:

```java
// Zuul 1 [LEGACY] — filtro para añadir una cabecera de correlación
public class CorrelationIdFilter extends ZuulFilter {

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.addZuulRequestHeader("X-Correlation-Id", UUID.randomUUID().toString());
        return null;
    }
}
```

El equivalente en Spring Cloud Gateway como `GlobalFilter` (se aplica a todas las rutas, igual que el filtro `pre` de Zuul con `shouldFilter()` siempre verdadero):

```java
// Spring Cloud Gateway — filtro global equivalente
@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest().mutate()
                .header("X-Correlation-Id", UUID.randomUUID().toString())
                .build();
        return chain.filter(exchange.mutate().request(request).build());
    }

    @Override
    public int getOrder() {
        return 1;
    }
}
```

Los cambios estructurales son tres: el método `run()` desaparece y es reemplazado por `filter(exchange, chain)`, el valor de retorno es `Mono<Void>` en lugar de `Object`, y el acceso a la petición se hace a través del `ServerWebExchange` inmutable (con `mutate()`) en lugar del `RequestContext` estático de Zuul.

### Paso 4: revisar llamadas bloqueantes dentro de los filtros

Si algún filtro de Zuul 1 realizaba operaciones bloqueantes (consulta a base de datos, llamada HTTP con `RestTemplate`, lectura de fichero), esas operaciones deben adaptarse. La opción más segura durante una migración es envolverlas temporalmente en `Schedulers.boundedElastic()` hasta poder reescribirlas con clientes reactivos:

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return Mono.fromCallable(() -> servicioLegacy.obtenerDato())  // llamada bloqueante
               .subscribeOn(Schedulers.boundedElastic())
               .flatMap(dato -> {
                   ServerHttpRequest request = exchange.getRequest().mutate()
                           .header("X-Dato-Legado", dato)
                           .build();
                   return chain.filter(exchange.mutate().request(request).build());
               });
}
```

> [ADVERTENCIA] Ejecutar código bloqueante directamente en el event loop de Netty (sin `subscribeOn`) degrada el rendimiento de todo el gateway al bloquear los hilos que gestionan todas las conexiones concurrentes.

### Paso 5: ajustar los tests

Los tests de integración con `@SpringBootTest` siguen funcionando sin cambios en la anotación. Lo que cambia es cómo se realizan las llamadas HTTP en el test: `TestRestTemplate` se reemplaza por `WebTestClient`, que viene incluido en `spring-boot-starter-test` cuando el contexto es reactivo.

```java
// Antes — Zuul 1 con TestRestTemplate [LEGACY]
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GatewayIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void deberia_enrutar_a_productos() {
        ResponseEntity<String> response = restTemplate.getForEntity("/api/productos/listado", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

```java
// Después — Spring Cloud Gateway con WebTestClient
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GatewayIntegrationTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void deberia_enrutar_a_productos() {
        webTestClient.get()
                .uri("/api/productos/listado")
                .exchange()
                .expectStatus().isOk();
    }
}
```

> [EXAMEN] En los exámenes de certificación de Spring, es frecuente preguntar por las diferencias entre `ZuulFilter` y `GatewayFilter`. El punto clave es el modelo de retorno: `run()` devuelve `Object` (normalmente `null`) en Zuul 1, mientras que `filter()` devuelve `Mono<Void>` en SCG, reflejo directo del modelo reactivo frente al bloqueante.

---

← [6.1.2 Arquitectura reactiva](./06-02-gateway-arquitectura.md) | [Índice](./README.md) | [6.1.4 Propiedades globales →](./06-04-gateway-propiedades-globales.md)
