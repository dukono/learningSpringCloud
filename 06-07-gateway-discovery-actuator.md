# 6.1.7 Discovery Locator y endpoints de Actuator

← [6.1.6 WebSocket y SSE](./06-06-gateway-websocket.md) | [Índice](./README.md) | [6.2.1 Modelo de Route →](./06-08-gateway-route-modelo.md)

---

Spring Cloud Gateway ofrece dos herramientas complementarias que simplifican tanto la configuración inicial como la operación en tiempo de ejecución. El **Discovery Locator** permite que el Gateway descubra automáticamente los servicios registrados en Eureka y genere rutas sin necesidad de declarar ningún YAML por servicio. Los **endpoints de Actuator** permiten inspeccionar qué rutas están activas, qué filtros están registrados y, cuando es necesario, modificar la configuración de rutas sin reiniciar la aplicación. Ambas capacidades son especialmente valiosas en entornos con muchos microservicios, aunque cada una tiene un ámbito de uso recomendado diferente.

---

## Diagrama general

El siguiente diagrama muestra cómo el Discovery Locator transforma el registro de Eureka en rutas activas y cómo los endpoints de Actuator permiten observar y manipular esas rutas:

```
Eureka Registry
  ├── PRODUCTOS-SERVICE (3 instancias)
  ├── PEDIDOS-SERVICE (2 instancias)
  └── USUARIOS-SERVICE (1 instancia)
          │
          │ Discovery Locator (polling automático)
          ▼
Gateway routes (generadas automáticamente):
  ├── Path=/PRODUCTOS-SERVICE/**  → lb://PRODUCTOS-SERVICE
  ├── Path=/PEDIDOS-SERVICE/**    → lb://PEDIDOS-SERVICE
  └── Path=/USUARIOS-SERVICE/**  → lb://USUARIOS-SERVICE
          │
          │ Actuator endpoints
          ▼
  /actuator/gateway/routes        ← inspección
  /actuator/gateway/globalfilters ← inspección
  /actuator/gateway/refresh       ← recarga
  /actuator/gateway/routes/{id}   ← CRUD dinámico
```

---

## Parte 1: Discovery Locator

### Qué es y cómo funciona

El Discovery Locator es una funcionalidad del Gateway que, al activarse, consulta periódicamente el service registry (Eureka, Consul u otro DiscoveryClient compatible) y construye automáticamente una ruta por cada servicio registrado. No es necesario declarar ninguna ruta en YAML para los servicios: basta con que el servicio esté en Eureka y el Gateway lo expone de inmediato.

> [CONCEPTO] El Discovery Locator no crea rutas estáticas en memoria al arrancar y olvidarlas. Reacciona a los cambios en el registry: si un servicio nuevo se registra, la ruta aparece; si un servicio desaparece del registry, la ruta deja de estar disponible.

### Cómo se generan las rutas automáticas

Para cada servicio `PRODUCTOS-SERVICE` registrado en Eureka, el Discovery Locator construye internamente una definición de ruta con las siguientes características:

- **ID de ruta**: `ReactiveCompositeDiscoveryClient_PRODUCTOS-SERVICE`
- **Predicate**: `Path=/PRODUCTOS-SERVICE/**`
- **Filter**: `RewritePath=/PRODUCTOS-SERVICE/(?<remaining>.*), /${remaining}` — elimina el prefijo con el nombre del servicio antes de reenviar la petición
- **URI**: `lb://PRODUCTOS-SERVICE` — con balanceo de carga automático entre instancias

Cuando se activa `lower-case-service-id: true`, el predicate pasa a ser `Path=/productos-service/**`, lo que resulta más cómodo para los clientes HTTP convencionales.

### Configuración mínima

La activación del Discovery Locator requiere únicamente dos propiedades:

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true   # /productos-service/** en lugar de /PRODUCTOS-SERVICE/**
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

Con esta configuración, cualquier servicio que se registre en Eureka queda accesible a través del Gateway sin más cambios.

### Filtrado con include-expression

La propiedad `include-expression` acepta una expresión SpEL que se evalúa para cada servicio del registry. Solo los servicios para los que la expresión devuelva `true` se convierten en rutas. Esto permite excluir servicios internos que no deben exponerse al exterior:

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
          include-expression: "serviceId.startsWith('public-')"
```

Con la expresión anterior, solo los servicios cuyo nombre comience por `public-` generarán rutas automáticas. El resto permanece invisible para el Gateway.

> [ADVERTENCIA] En producción, usar Discovery Locator sin configurar `include-expression` expone automáticamente cualquier servicio que se registre en Eureka, incluyendo servicios internos que no deberían ser accesibles desde el exterior. Un servicio de administración, un job de batch o un servicio de soporte interno quedarían accesibles si se registran en el mismo registry.

### Limitaciones del Discovery Locator

Las rutas generadas automáticamente siguen siempre el mismo patrón: el nombre del servicio como prefijo del path. Esto implica que los clientes deben conocer los nombres de los servicios internos para construir sus URLs. Además, no es posible añadir predicates adicionales (por ejemplo, restringir por cabecera o por método HTTP) ni aplicar filtros específicos a una ruta generada automáticamente. Todas las rutas heredan los mismos filtros globales, pero no se puede personalizar cada una de forma individual.

---

## Parte 2: Endpoints de Actuator del Gateway

### Habilitación

Spring Boot Actuator expone un conjunto de endpoints específicos para el Gateway bajo la ruta `/actuator/gateway/`. Para activarlos, hay que incluir `gateway` en la lista de endpoints expuestos:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: gateway,health,info
  endpoint:
    gateway:
      enabled: true
```

> [EXAMEN] Sin `management.endpoints.web.exposure.include: gateway`, los endpoints del Gateway no son accesibles aunque Actuator esté en el classpath. Este es un error frecuente en pruebas técnicas.

### Endpoints disponibles

Los siguientes endpoints están disponibles una vez habilitado el módulo `gateway` de Actuator:

| Endpoint | Método | Descripción |
|---|---|---|
| `/actuator/gateway/routes` | GET | Lista todas las rutas activas con sus predicates y filtros |
| `/actuator/gateway/routes/{id}` | GET | Detalle completo de una ruta específica |
| `/actuator/gateway/globalfilters` | GET | Lista los filtros globales registrados con su orden de ejecución |
| `/actuator/gateway/routefilters` | GET | Lista las fábricas de filtros de ruta disponibles |
| `/actuator/gateway/refresh` | POST | Recarga la configuración de rutas sin reiniciar la aplicación |
| `/actuator/gateway/routes/{id}` | POST | Crea o actualiza una ruta en tiempo de ejecución |
| `/actuator/gateway/routes/{id}` | DELETE | Elimina una ruta en tiempo de ejecución |

### Ejemplos de uso con curl

Los siguientes ejemplos muestran las operaciones más habituales con los endpoints de Actuator del Gateway. Todos asumen que el Gateway está ejecutándose en `localhost:8080`.

```bash
# Ver todas las rutas activas
curl http://localhost:8080/actuator/gateway/routes | jq .

# Ver el detalle de una ruta concreta
curl http://localhost:8080/actuator/gateway/routes/productos-route | jq .

# Ver los filtros globales registrados y su orden de ejecución
curl http://localhost:8080/actuator/gateway/globalfilters | jq .

# Ver las fábricas de filtros de ruta disponibles
curl http://localhost:8080/actuator/gateway/routefilters | jq .

# Forzar la recarga de rutas (útil tras cambios en Spring Cloud Config)
curl -X POST http://localhost:8080/actuator/gateway/refresh

# Crear una ruta en tiempo de ejecución sin reiniciar el Gateway
curl -X POST http://localhost:8080/actuator/gateway/routes/nueva-ruta \
  -H "Content-Type: application/json" \
  -d '{
    "id": "nueva-ruta",
    "uri": "lb://nuevo-servicio",
    "predicates": [{"name": "Path", "args": {"pattern": "/api/nuevo/**"}}],
    "filters": [{"name": "StripPrefix", "args": {"parts": "2"}}]
  }'

# Recargar para que la nueva ruta quede activa
curl -X POST http://localhost:8080/actuator/gateway/refresh

# Eliminar una ruta en tiempo de ejecución
curl -X DELETE http://localhost:8080/actuator/gateway/routes/nueva-ruta

# Recargar para que la eliminación tenga efecto
curl -X POST http://localhost:8080/actuator/gateway/refresh
```

### Rutas dinámicas y persistencia

Las rutas creadas o modificadas mediante los endpoints de Actuator se almacenan en memoria dentro del `InMemoryRouteDefinitionRepository` que usa el Gateway por defecto. Esto significa que no persisten tras un reinicio de la aplicación.

Para conseguir persistencia real, es necesario implementar o configurar un `RouteDefinitionRepository` con almacenamiento externo. Spring Cloud Gateway incluye soporte para Redis a través de `RedisRouteDefinitionRepository`. En la práctica, el patrón más habitual en producción es gestionar las definiciones de rutas en Spring Cloud Config Server y usar el endpoint `/actuator/gateway/refresh` para aplicar los cambios cuando se actualiza la configuración remota.

```yaml
# Dependencia necesaria para persistencia en Redis
# En pom.xml:
# <dependency>
#   <groupId>org.springframework.boot</groupId>
#   <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
# </dependency>

spring:
  data:
    redis:
      host: localhost
      port: 6379
  cloud:
    gateway:
      redis-route-definition-repository:
        enabled: true
```

---

## Tabla de propiedades

Las siguientes propiedades controlan el comportamiento del Discovery Locator y la exposición de los endpoints de Actuator:

| Propiedad | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `spring.cloud.gateway.discovery.locator.enabled` | boolean | `false` | Activa la generación automática de rutas a partir del service registry |
| `spring.cloud.gateway.discovery.locator.lower-case-service-id` | boolean | `false` | Convierte los nombres de servicio a minúsculas en los paths generados |
| `spring.cloud.gateway.discovery.locator.include-expression` | String | `true` | Expresión SpEL para filtrar qué servicios del registry generan ruta. Por defecto incluye todos |
| `spring.cloud.gateway.discovery.locator.url-expression` | String | `'lb://'+serviceId` | Expresión SpEL que determina la URI de destino de cada ruta generada |
| `management.endpoints.web.exposure.include` | list | `health` | Lista de endpoints de Actuator a exponer. Incluir `gateway` para los endpoints del Gateway |
| `management.endpoint.gateway.enabled` | boolean | `true` | Habilita o deshabilita específicamente el endpoint `gateway` de Actuator |

---

## Buenas y malas prácticas

**Buenas prácticas:**

- Usar Discovery Locator en entornos de desarrollo y prototipos para agilizar la integración de nuevos servicios sin tocar la configuración del Gateway.
- Configurar siempre `include-expression` cuando se usa Discovery Locator en entornos compartidos o de preproducción, para limitar qué servicios quedan expuestos.
- Usar `lower-case-service-id: true` para que las URLs generadas sean más legibles y compatibles con las convenciones REST habituales.
- Proteger los endpoints de Actuator con Spring Security en cualquier entorno que no sea estrictamente local. Los endpoints `POST` y `DELETE` permiten modificar la configuración del Gateway en tiempo de ejecución.
- Combinar el endpoint `/actuator/gateway/refresh` con Spring Cloud Config para aplicar cambios de rutas sin reiniciar.

**Malas prácticas:**

- Activar Discovery Locator en producción sin `include-expression`: cualquier servicio que se registre en Eureka queda automáticamente accesible desde el exterior.
- Crear rutas vía Actuator en producción sin un `RouteDefinitionRepository` persistente: las rutas desaparecen en el siguiente reinicio sin dejar rastro en el control de versiones.
- Exponer todos los endpoints de Actuator con `include: "*"` en producción sin restricciones de acceso.
- Confiar en las rutas generadas por el Discovery Locator para servicios que requieren predicates o filtros específicos: en esos casos, es necesario definir la ruta explícitamente en YAML.

---

## Comparación: Discovery Locator vs rutas explícitas

Ambas estrategias son compatibles y pueden coexistir en el mismo Gateway, pero cada una tiene su ámbito de uso natural. La siguiente tabla resume los aspectos clave para elegir entre ellas:

| Aspecto | Discovery Locator | Rutas explícitas en YAML |
|---|---|---|
| Configuración inicial | Cero YAML por servicio | Una entrada por servicio |
| Exposición automática | Todo servicio en Eureka queda expuesto (salvo filtro) | Solo los servicios declarados |
| Personalización de predicates | No disponible | Total |
| Personalización de filtros por ruta | No disponible | Total |
| Auditoría de seguridad | Difícil (rutas implícitas) | Sencilla (todo declarado explícitamente) |
| Tiempo de onboarding de un nuevo servicio | Inmediato (se registra en Eureka) | Requiere cambio en configuración del Gateway |
| Caso de uso recomendado | Desarrollo / prototipos | Producción |

> [ADVERTENCIA] En producción, las rutas implícitas del Discovery Locator dificultan la auditoría de seguridad. Un revisor que inspeccione únicamente la configuración YAML del Gateway no verá las rutas generadas automáticamente. Para entornos productivos, el enfoque recomendado es desactivar el Discovery Locator y declarar todas las rutas de forma explícita.

En la práctica, un patrón habitual es usar Discovery Locator durante el desarrollo para no tener que actualizar la configuración del Gateway cada vez que se crea un nuevo microservicio, y migrar a rutas explícitas antes de pasar a producción, cuando la lista de servicios expuestos es estable y los requisitos de seguridad están definidos.

---

← [6.1.6 WebSocket y SSE](./06-06-gateway-websocket.md) | [Índice](./README.md) | [6.2.1 Modelo de Route →](./06-08-gateway-route-modelo.md)
