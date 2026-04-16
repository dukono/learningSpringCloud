# 1.3 Resolución de configuración por perfiles y labels

← [1.2 Config Client — configuración y arranque](sc-config-client.md) | [Índice (README.md)](README.md) | [1.4 Git backend — configuración y autenticación →](sc-config-git-backend.md)

---

## Introducción

El Config Server no devuelve un único fichero de propiedades: construye dinámicamente un `Environment` compuesto por múltiples `PropertySource` ordenados por precedencia. La regla que determina qué propiedades ve cada microservicio es la combinación de tres variables: el nombre de la aplicación (`{application}`), el perfil activo (`{profile}`) y la etiqueta del backend (`{label}`).

Entender esta resolución es crítico en entrevistas y en producción. Un error en el nombre del perfil o en el label puede hacer que un microservicio arranque con propiedades del entorno equivocado sin emitir ningún error: el servidor responde HTTP 200 con el fichero base, que puede no contener las propiedades de producción esperadas.

> [CONCEPTO] El mecanismo de resolución del Config Server sigue la convención de naming `{application}-{profile}.{yml|properties}` sobre el backend, con herencia desde un fichero base `application.{yml|properties}` y orden de precedencia estricto: las propiedades más específicas (con perfil) sobreescriben a las más generales (sin perfil).

> [PREREQUISITO] Para entender esta sección es necesario tener claro el concepto de `PropertySource` de Spring Framework: una fuente de propiedades ordenada en una lista donde las primeras entradas tienen mayor precedencia que las últimas.

## Diagrama: jerarquía de resolución de propiedades

El siguiente diagrama muestra qué ficheros consulta el servidor para construir la respuesta a una petición `GET /order-service/prod/main` y el orden de precedencia resultante.

```
Petición: GET /order-service/prod/main
         application=order-service  profile=prod  label=main

FICHEROS CONSULTADOS EN EL BACKEND GIT (rama main):
╔══════════════════════════════════╗  ← MAYOR PRECEDENCIA
║  order-service-prod.yml          ║  propiedades específicas de este servicio en prod
╠══════════════════════════════════╣
║  order-service.yml               ║  propiedades del servicio (todos los perfiles)
╠══════════════════════════════════╣
║  application-prod.yml            ║  propiedades globales para el perfil prod
╠══════════════════════════════════╣
║  application.yml                 ║  propiedades globales (todos los servicios)
╚══════════════════════════════════╝  ← MENOR PRECEDENCIA

RESULTADO: PropertySource[] con propiedades fusionadas.
Si la misma clave aparece en varios ficheros, gana la del fichero más arriba.
```

El servidor devuelve la lista completa de `PropertySource`; el cliente los aplica en ese mismo orden.

## Ejemplo central

A continuación se muestra una estructura de repositorio Git completa y la respuesta real que devuelve el Config Server para ilustrar la fusión de propiedades.

### Estructura del repositorio Git

```
config-repo/
├── application.yml               # base compartida por todos los servicios
├── application-prod.yml          # propiedades de producción globales
├── order-service.yml             # propiedades de order-service (todos los perfiles)
├── order-service-dev.yml         # propiedades de order-service en dev
└── order-service-prod.yml        # propiedades de order-service en prod
```

### Contenido de los ficheros

**application.yml** (base global):
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
logging:
  level:
    root: INFO
```

**application-prod.yml** (producción global):
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
logging:
  level:
    root: WARN
```

**order-service.yml** (base del servicio):
```yaml
order:
  payment-service-url: http://payment-service:8080
  max-retries: 3
  timeout-ms: 5000
```

**order-service-prod.yml** (servicio en producción):
```yaml
order:
  payment-service-url: https://payment-service.prod.internal:8443
  max-retries: 5
```

### Llamada HTTP al servidor y respuesta

```bash
# Petición directa al Config Server (útil para diagnóstico)
curl http://config-server:8888/order-service/prod/main
```

Respuesta JSON simplificada:

```json
{
  "name": "order-service",
  "profiles": ["prod"],
  "label": "main",
  "propertySources": [
    {
      "name": "git:…/order-service-prod.yml",
      "source": {
        "order.payment-service-url": "https://payment-service.prod.internal:8443",
        "order.max-retries": 5
      }
    },
    {
      "name": "git:…/order-service.yml",
      "source": {
        "order.payment-service-url": "http://payment-service:8080",
        "order.max-retries": 3,
        "order.timeout-ms": 5000
      }
    },
    {
      "name": "git:…/application-prod.yml",
      "source": {
        "management.endpoints.web.exposure.include": "health",
        "logging.level.root": "WARN"
      }
    },
    {
      "name": "git:…/application.yml",
      "source": {
        "management.endpoints.web.exposure.include": "health,info",
        "logging.level.root": "INFO"
      }
    }
  ]
}
```

El cliente fusiona esta lista de arriba a abajo: `order.payment-service-url` vale `https://payment-service.prod.internal:8443` (del primer `PropertySource`), y `order.timeout-ms` vale `5000` (solo aparece en el segundo).

### Configuración del cliente para este ejemplo

```yaml
# application.yml del microservicio order-service
spring:
  application:
    name: order-service         # determina {application}
  config:
    import: "configserver:http://config-server:8888"
  profiles:
    active: prod                # determina {profile}
  cloud:
    config:
      label: main               # determina {label} — rama Git
```

## Tabla: endpoint HTTP del Config Server

La siguiente tabla describe los patrones de URL que expone el servidor y qué devuelve cada uno.

| Endpoint | Ejemplo | Descripción |
|---|---|---|
| `/{app}/{profile}` | `/order-service/prod` | Resolución estándar. Label por defecto del servidor. |
| `/{app}/{profile}/{label}` | `/order-service/prod/main` | Especifica rama/tag/commit explícitamente. |
| `/{app}/{profile}/{label}/{path}` | `/order-service/prod/main/order-service.yml` | Devuelve el fichero crudo (sin fusión). |
| `/{label}/{app}-{profile}.yml` | `/main/order-service-prod.yml` | Fichero crudo con label antes del nombre. |
| `/{app}-{profile}.properties` | `/order-service-prod.properties` | Propiedades fusionadas en formato `.properties`. |

> [EXAMEN] La diferencia entre `/{app}/{profile}` (devuelve `PropertySource[]` fusionados) y `/{app}-{profile}.yml` (devuelve el fichero crudo sin fusión) es una pregunta frecuente. El cliente usa el primer formato; el segundo es útil solo para depuración o para ver el contenido de un fichero específico.

## Buenas y malas prácticas

**Hacer:**
- Definir explícitamente `spring.cloud.config.label` en el cliente para cada entorno. Confiar en el label por defecto del servidor puede causar que un despliegue en producción use la rama `develop` si el servidor tiene `default-label: develop` como valor por defecto.
- Usar `application.yml` (sin perfil) para propiedades verdaderamente globales e invariantes: timeouts de heartbeat, configuración de Actuator, nivel de log base. Cuantas menos propiedades en el fichero base, más predecible es la resolución.
- Verificar la resolución de configuración de un servicio en un entorno concreto con `curl` al endpoint del servidor antes de arrancar la instancia. Es la forma más rápida de confirmar que las propiedades correctas están disponibles.

**Evitar:**
- Nombrar perfiles con guiones bajos (`order_service`): el separador canónico en Spring es el guion (`order-service`). Los guiones bajos en el nombre de perfil pueden provocar fallos de resolución de ficheros según la versión del backend.
- Sobrecargar `application-prod.yml` con propiedades específicas de un servicio concreto. Si esa propiedad cambia, afecta a todos los servicios que usen ese perfil.
- Usar labels dinámicos que apunten a commits específicos en producción sin un proceso de promoción controlado: facilita la configuración de cada instancia en un commit diferente, rompiendo la consistencia.

## Comparación: tres variables de resolución

La siguiente tabla resume qué controla cada variable de resolución y dónde se configura.

| Variable | Corresponde a | Se configura en el cliente | Efecto en el backend Git |
|---|---|---|---|
| `{application}` | `spring.application.name` | `spring.application.name` | Nombre del fichero YAML: `order-service.yml` |
| `{profile}` | Perfil activo de Spring | `spring.profiles.active` o `spring.cloud.config.profile` | Sufijo del fichero: `order-service-prod.yml` |
| `{label}` | Rama / tag / commit | `spring.cloud.config.label` | Referencia Git que el servidor hace checkout al leer |

> [ADVERTENCIA] `{label}` en el backend Git puede ser una rama (`main`), un tag (`v1.2.0`) o incluso un hash de commit corto. Sin embargo, los hashes de commit no son mutables: si el backend tiene `refreshRate` activo, apuntar a un commit concreto bloquea las actualizaciones. Usar ramas en producción y tags para rollbacks controlados.

---

← [1.2 Config Client — configuración y arranque](sc-config-client.md) | [Índice (README.md)](README.md) | [1.4 Git backend — configuración y autenticación →](sc-config-git-backend.md)
