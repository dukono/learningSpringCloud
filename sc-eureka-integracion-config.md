# 2.10 Integración con Spring Cloud Config

← [2.9 Seguridad del Eureka Server](sc-eureka-seguridad.md) | [Índice (README.md)](README.md) | [2.11 Operación y troubleshooting de Eureka →](sc-eureka-troubleshooting.md)

---

Cuando se usan Spring Cloud Config y Eureka juntos, surge un problema de orden de arranque: el microservicio necesita conectar al Config Server para obtener su configuración, pero también necesita conocer la URL del Eureka Server para registrarse. Si la URL del Config Server se almacena en el propio Eureka, el servicio necesita consultar Eureka antes de tener configuración. Si la URL de Eureka se obtiene del Config Server, el servicio necesita configuración antes de poder registrarse. Spring Cloud ofrece dos estrategias explícitas para resolver este circular dependency: config-first bootstrap y eureka-first bootstrap.

> [ADVERTENCIA] Spring Cloud 2025.1.1 (Oakwood) con Spring Boot 4.0.x elimina el bootstrap context clásico (`spring-cloud-starter-bootstrap` y `bootstrap.yml`). Toda la integración con Config Server se hace exclusivamente via `spring.config.import` en `application.yml`. No uses `bootstrap.yml` ni `spring-cloud-starter-bootstrap` en proyectos nuevos con esta versión.

## Diagrama: config-first vs eureka-first bootstrap

El siguiente diagrama compara las dos estrategias de bootstrap y el orden de eventos en cada una.

```
CONFIG-FIRST BOOTSTRAP (más simple)
─────────────────────────────────────
1. Arranque del microservicio
2. spring.config.import → URL fija del Config Server
   (conocida en el momento del arranque, hardcoded o en env var)
3. Obtiene configuración del Config Server (incluyendo URL de Eureka)
4. Se registra en Eureka con la URL obtenida del Config Server
5. ✓ Config Server puede estar en una URL fija o en variables de entorno

EUREKA-FIRST BOOTSTRAP (más flexible)
─────────────────────────────────────
1. Arranque del microservicio
2. Necesita conocer solo la URL del Eureka Server (URL fija o env var)
3. Se registra en Eureka
4. Consulta Eureka para localizar el Config Server por nombre lógico
   (spring.cloud.config.discovery.service-id: config-server)
5. Obtiene configuración del Config Server vía su instancia Eureka
6. ✓ Config Server puede cambiar de IP sin reconfigurar todos los clientes
```

## Implementación: las dos estrategias con spring.config.import

El siguiente ejemplo muestra la configuración completa de cada estrategia para un microservicio cliente.

**Estrategia 1: Config-first (recomendada para la mayoría de casos)**

```yaml
# application.yml del microservicio
spring:
  application:
    name: servicio-pedidos

  config:
    # URL del Config Server conocida en el arranque (env var en Docker/K8s)
    # optional: evita que el arranque falle si el Config Server no está disponible.
    # Eliminar "optional:" para que el arranque falle si el Config Server no responde.
    import: "optional:configserver:${CONFIG_SERVER_URL:http://localhost:8888}"

eureka:
  client:
    service-url:
      # La URL de Eureka puede venir del Config Server (una vez cargada la config)
      # o configurarse aquí como fallback
      defaultZone: ${EUREKA_SERVER_URL:http://localhost:8761/eureka/}
  instance:
    prefer-ip-address: true
```

**Estrategia 2: Eureka-first (Config Server se localiza por nombre en Eureka)**

```yaml
# application.yml del microservicio — Eureka-first bootstrap
spring:
  application:
    name: servicio-pedidos

  config:
    # "optional:configserver:" sin URL: delega la resolución en Eureka
    import: "optional:configserver:"

  cloud:
    config:
      # Activar la localización del Config Server vía Eureka
      discovery:
        enabled: true
        # Nombre lógico con el que el Config Server está registrado en Eureka
        service-id: config-server
      # Fallback: si Eureka no tiene el Config Server, no fallar en el arranque
      fail-fast: false

eureka:
  client:
    service-url:
      # La URL de Eureka debe conocerse antes del arranque (env var o hardcoded)
      # porque el Config Server se localiza a través de ella
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

**Config Server con Eureka: configuración del servidor Config como cliente Eureka**

```yaml
# application.yml del Config Server
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          default-label: main

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    register-with-eureka: true
    fetch-registry: true
  instance:
    prefer-ip-address: true
```

> [EXAMEN] En una entrevista es frecuente preguntar: "¿Cuál es la diferencia entre config-first y eureka-first en Spring Cloud?". La respuesta: en config-first, la URL del Config Server es conocida (env var o fija) y Eureka se configura a través de la configuración obtenida del Config Server. En eureka-first, solo la URL de Eureka es conocida inicialmente; el Config Server se localiza por nombre lógico en el registro de Eureka. La estrategia config-first es más simple y es la recomendada; eureka-first añade flexibilidad pero introduce una dependencia circular en el arranque que debe resolverse con reintentos.

## Tabla de propiedades de integración

La siguiente tabla recoge las propiedades específicas de la integración entre Eureka y Config Server.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.config.import` | String | — | Fuente de configuración externa; `configserver:URL` para Config Server |
| `spring.cloud.config.discovery.enabled` | boolean | `false` | Activa la localización del Config Server vía Eureka (eureka-first) |
| `spring.cloud.config.discovery.service-id` | String | `configserver` | Nombre lógico del Config Server en el registro de Eureka |
| `spring.cloud.config.fail-fast` | boolean | `false` | Si `true`, el arranque falla inmediatamente si el Config Server no responde |

## Buenas y malas prácticas

Hacer:
- Usar la estrategia config-first en la mayoría de proyectos: es más simple, el orden de arranque es claro, y la URL del Config Server puede gestionarse como variable de entorno en la plataforma de despliegue.
- Añadir `optional:` al prefijo de `spring.config.import` en entornos donde el Config Server puede no estar disponible (desarrollo local sin Config Server levantado): sin `optional:`, el microservicio no arrancará si el Config Server no responde.
- En eureka-first, configurar `spring.cloud.config.fail-fast: false` y reintentos para tolerar el tiempo que tarda Eureka en tener al Config Server registrado y visible: el Config Server puede arrancar después del microservicio cliente en algunos entornos.

Evitar:
- Usar `bootstrap.yml` o la dependencia `spring-cloud-starter-bootstrap` en proyectos con Spring Boot 4.0.x: el bootstrap context fue eliminado; todo debe configurarse en `application.yml` con `spring.config.import`.
- Crear dependencias circulares de arranque sin reintentos: en eureka-first, si el microservicio arranca antes que el Config Server, la primera consulta al Config Server vía Eureka fallará; sin `fail-fast: false` y reintentos, el microservicio no arrancará aunque el Config Server se recupere en segundos.

---

← [2.9 Seguridad del Eureka Server](sc-eureka-seguridad.md) | [Índice (README.md)](README.md) | [2.11 Operación y troubleshooting de Eureka →](sc-eureka-troubleshooting.md)
