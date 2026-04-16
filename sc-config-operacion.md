# 1.8 Operación del Config Server en producción (HA, Webhooks y Observabilidad)

← [1.7 Seguridad del Config Server](sc-config-security.md) | [Índice (README.md)](README.md) | [1.9 Testing / Verificación de Spring Cloud Config →](sc-config-testing.md)

---

## Introducción

Un Config Server que funciona en desarrollo o staging puede fallar silenciosamente en producción bajo tres tipos de presión: carga simultánea de decenas de microservicios al arrancar, falta de alta disponibilidad cuando el servidor cae, y ausencia de un mecanismo automático de propagación de cambios cuando el repositorio Git se actualiza.

Este fichero cubre los tres bloques operativos del "día 2" del Config Server: **alta disponibilidad** (múltiples instancias detrás de un balanceador o registro en Eureka), **webhooks** (propagación automática de cambios desde el repositorio Git) y **observabilidad** (endpoints Actuator, health indicator y logging de resolución). Los tres bloques son inseparables en la operación real de producción.

> [ADVERTENCIA] El Config Server es un prerequisito de arranque para todos los microservicios del ecosistema. Su indisponibilidad en producción tiene un radio de impacto igual al número de microservicios que lo usan. Planificar HA desde el diseño inicial, no como mejora posterior.

## Diagrama: arquitectura HA del Config Server

El siguiente diagrama muestra la topología recomendada para alta disponibilidad del Config Server con balanceador de carga y registro en Eureka.

```
┌──────────────────────────────────────────────────────────────┐
│                     Config Clients                           │
│  spring.cloud.config.discovery.enabled=true                  │
│  spring.cloud.config.discovery.service-id=config-server      │
└──────────────┬───────────────────────────────────────────────┘
               │  localiza instancias vía Eureka
               ▼
┌──────────────────────────────────────────────────────────────┐
│                   Eureka Server                              │
│   config-server → [10.0.0.1:8888, 10.0.0.2:8888]           │
└──────────────┬───────────────────────────────────────────────┘
               │  registra sus instancias
               ▼
┌─────────────────────┐       ┌─────────────────────┐
│  Config Server #1   │       │  Config Server #2   │
│  10.0.0.1:8888      │       │  10.0.0.2:8888      │
└──────────┬──────────┘       └──────────┬──────────┘
           │                             │
           └──────────┬──────────────────┘
                      │  comparten el mismo backend Git
                      ▼
              ┌───────────────┐
              │  Git Backend  │
              │  (repositorio │
              │   remoto)     │
              └───────────────┘
```

Cada instancia del Config Server es **stateless** respecto al estado de configuración (todo está en Git): no hay sincronización entre instancias, no hay líder/seguidor. El único estado local es el clon Git en `basedir`, que es un caché prescindible.

## Alta disponibilidad

### Opción 1: Múltiples instancias detrás de un load balancer

La opción más sencilla: desplegar N instancias del Config Server y poner un load balancer (Nginx, HAProxy, AWS ALB) delante.

```yaml
# application.yml del Config Client — apunta al VIP del balanceador
spring:
  config:
    import: "configserver:http://config-lb.internal:8888"
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.5
        max-interval: 2000
```

El balanceador (Nginx, HAProxy, ALB) enruta las peticiones entre las N instancias del Config Server. Spring Cloud Config no impone un balanceador específico.

### Opción 2: Descubrimiento dinámico vía Eureka

```yaml
# application.yml del Config Server — se registra en Eureka
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo

eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
  instance:
    prefer-ip-address: true
```

```yaml
# application.yml del Config Client — localiza el Config Server vía Eureka
spring:
  application:
    name: order-service
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server    # nombre con el que el Config Server se registra en Eureka
      fail-fast: true
      retry:
        max-attempts: 6

eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
```

> [ADVERTENCIA] El bootstrap con descubrimiento Eureka crea una dependencia de arranque circular: el Config Client necesita Eureka para encontrar al Config Server, y los microservicios necesitan al Config Server para su configuración. Resolver usando un `bootstrap.yml` (modo legacy) o configurando la URL de Eureka directamente en `application.yml` sin importarla del Config Server.

### refreshRate y comportamiento de caché

```yaml
spring:
  cloud:
    config:
      server:
        git:
          refresh-rate: 60       # segundos entre git fetch automáticos
          force-pull: true       # descarta cambios locales en el clon
          clone-on-start: true   # clona al arrancar, no en la primera petición
```

Con `refresh-rate: 0`, el servidor hace `git fetch` en cada petición, lo que puede ser un bottleneck bajo carga.

## Webhooks y notificaciones push

### Flujo webhook: GitHub/GitLab → Config Server → Bus

Cuando se hace un push al repositorio Git, el proveedor Git puede notificar al Config Server vía webhook HTTP. El Config Server entonces invalida su caché y publica un evento en Spring Cloud Bus para que todos los microservicios actualicen su configuración.

```xml
<!-- pom.xml del Config Server — añadir spring-cloud-config-monitor -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-monitor</artifactId>
</dependency>
<!-- Y el binder de bus: Kafka o AMQP (RabbitMQ) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

Con estas dependencias, el Config Server expone el endpoint `/monitor` que acepta los payloads webhook de GitHub, GitLab y Bitbucket.

```yaml
# application.yml del Config Server — configuración del bus
spring:
  cloud:
    bus:
      enabled: true
    stream:
      kafka:
        binder:
          brokers: kafka:9092
```

Configuración del webhook en GitHub:
- URL: `https://config-server:8888/monitor`
- Content type: `application/json`
- Events: `push`

Cuando se recibe el webhook, el Config Server publica un `RefreshRemoteApplicationEvent` en el bus. Todos los microservicios suscritos al bus reciben el evento y ejecutan `ContextRefresher.refresh()` automáticamente.

`spring-cloud-config-monitor` publica un `RefreshRemoteApplicationEvent` en el bus; la propagación la gestiona Spring Cloud Bus. Ver módulo Spring Cloud Bus.

> [EXAMEN] El endpoint `/monitor` no es parte del Actuator estándar: es específico de `spring-cloud-config-monitor`. Su diferencia con `POST /actuator/refresh` es que `/monitor` acepta el formato nativo de los webhooks de Git (GitHub, GitLab, Bitbucket) y determina automáticamente qué aplicaciones afecta el push, publicando un evento selectivo (solo las aplicaciones cuyos ficheros de configuración cambiaron).

## Observabilidad

### Health indicator del Config Server

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,env,refresh
  endpoint:
    health:
      show-details: when-authorized
```

```bash
# Health del Config Server — incluye el estado del backend Git
curl http://config-server:8888/actuator/health

{
  "status": "UP",
  "components": {
    "configServer": {
      "status": "UP",
      "details": {
        "repositories": [
          {
            "name": "default",
            "profiles": ["*"],
            "label": null,
            "uri": "https://github.com/mi-org/config-repo",
            "info": { "deleteUntrackedBranches": false, "forcePull": true }
          }
        ]
      }
    }
  }
}
```

El `HealthIndicator` del Config Server verifica la conectividad con el backend (Git, Vault, etc.) en cada llamada a `/actuator/health`. Si el backend no es accesible, el estado pasa a `DOWN`.

### Endpoint /actuator/env

```bash
# Ver todas las propiedades resueltas para una aplicación concreta (desde el servidor)
curl http://config-server:8888/order-service/prod

# Ver el Environment interno del propio Config Server
curl http://config-server:8888/actuator/env
```

### Logging de resolución de configuración

```yaml
# application.yml del Config Server
logging:
  level:
    org.springframework.cloud.config.server: DEBUG
    org.eclipse.jgit: WARN    # reducir el ruido de JGit en DEBUG
```

Con `DEBUG`, el log muestra cada petición de resolución: qué ficheros se buscan, cuáles se encuentran y en qué rama/directorio.

```
DEBUG o.s.c.c.s.e.JGitEnvironmentRepository - Fetching for remote branches...
DEBUG o.s.c.c.s.e.JGitEnvironmentRepository - Found 4 resource(s) for config
DEBUG o.s.c.c.s.EnvironmentController - Returning environment for 'order-service/prod'
```

## Tabla de parámetros operativos

La siguiente tabla resume los parámetros más relevantes para la operación en producción.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.config.discovery.enabled` | `boolean` | `false` | Activa el descubrimiento del servidor vía Eureka en el cliente. |
| `spring.cloud.config.discovery.service-id` | `String` | `configserver` | Nombre del servicio Config Server en Eureka. |
| `spring.cloud.config.server.git.refresh-rate` | `int` | `0` | Segundos entre git fetch automáticos. `0` = fetch en cada petición. |
| `spring.cloud.config.server.health.enabled` | `boolean` | `true` | Activa el health indicator del backend de configuración. |
| `spring.cloud.config.server.health.repositories.*` | `Map` | — | Repositorios a verificar en el health check. |
| `management.endpoints.web.exposure.include` | `String` | `health` | Endpoints Actuator expuestos. Para debug añadir `env`. |

## Buenas y malas prácticas

**Hacer:**
- Monitorear el health indicator del Config Server en el sistema de alertas. Un estado `DOWN` en el Config Server significa que el siguiente arranque de cualquier microservicio fallará si `fail-fast: true` está activado.
- Usar `refresh-rate` entre 30 y 120 segundos en producción para balancear entre latencia de propagación de cambios y carga sobre el servidor Git.
- Configurar la URL del webhook en Git como HTTPS con autenticación (secret token de GitHub/GitLab) para evitar que actores externos puedan disparar refrescos masivos en el ecosistema.

**Evitar:**
- Exponer `/actuator/env` sin autenticación: este endpoint devuelve todas las propiedades resueltas del Config Server, incluyendo las no cifradas.
- Operar el Config Server con una sola instancia sin plan de HA si el ecosistema tiene más de 5 microservicios dependientes: una caída del servidor en producción bloquea todos los reinicios y despliegues hasta que el servidor vuelva.
- Ignorar los tiempos de arranque bajo carga: con 50 microservicios arrancando simultáneamente (rolling deploy), el Config Server recibe 50 peticiones concurrentes en segundos. Dimensionar el servidor y el backend Git para este pico de carga.

---

← [1.7 Seguridad del Config Server](sc-config-security.md) | [Índice (README.md)](README.md) | [1.9 Testing / Verificación de Spring Cloud Config →](sc-config-testing.md)
