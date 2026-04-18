# 1.4 Resolución de configuración — perfiles y overrides

← [1.3.2 Backends alternativos](sc-config-backends.md) | [Índice](README.md) | [1.5 Refresh de configuración](sc-config-refresh.md) →

---

## Introducción

La resolución de configuración en Spring Cloud Config no es una simple lectura de un fichero: es un proceso de fusión multicapa donde múltiples ficheros se combinan siguiendo reglas de precedencia bien definidas. Entender este proceso es fundamental para predecir qué valor recibirá un microservicio cuando tiene configuración en múltiples fuentes — el repositorio remoto, el perfil de entorno, y sus propias propiedades locales. Las preguntas del examen VMware sobre este tema evalúan específicamente la capacidad de predecir el valor final de una propiedad cuando hay conflictos.

> [CONCEPTO] El Config Server construye una lista ordenada de PropertySources para cada petición `{application}/{profile}/{label}`. El primer PropertySource (más específico) tiene mayor precedencia. Las propiedades locales del cliente pueden tener menor o mayor precedencia que las del servidor, dependiendo de la configuración de `allow-override`.

## Diagrama de precedencia de PropertySources

El proceso de resolución sigue una jerarquía estricta. Comprender este diagrama es la clave para responder las preguntas de precedencia del examen.

```
JERARQUÍA DE PRECEDENCIA (de mayor a menor)
═══════════════════════════════════════════
  1. {application}-{profile}.yml          ← MÁS ESPECÍFICO (mayor prioridad)
     (ej: order-service-prod.yml)

  2. {application}.yml
     (ej: order-service.yml)

  3. application-{profile}.yml            ← Configuración global por entorno
     (ej: application-prod.yml)

  4. application.yml                      ← Base global (menor prioridad en servidor)

  5. Propiedades locales del cliente      ← Depende de allow-override
     (application.properties/yml local)

  6. Overrides del servidor               ← Pueden tener mayor prioridad que todo
     (spring.cloud.config.server.overrides)
═══════════════════════════════════════════
NOTA: En el servidor, mayor posición en la lista = menor prioridad
Los PropertySources se fusionan: las claves del primero ganan
```

Cuando un cliente `order-service` con perfil `prod` solicita configuración, el servidor devuelve hasta cuatro PropertySources en este orden. La clave `server.port` que aparece en `order-service-prod.yml` con valor `8081` sobreescribe cualquier `server.port` de `order-service.yml` o `application.yml`.

> [EXAMEN] `application.yml` (sin perfil) siempre se incluye como base y se fusiona con todos los demás. Sus valores tienen la menor precedencia dentro del servidor. Una clave definida en `application.yml` y redefinida en `order-service-prod.yml` siempre tendrá el valor de `order-service-prod.yml`.

## Perfiles — segmentación por entorno

Los perfiles de Spring son el mecanismo principal para segmentar configuración por entorno (dev, test, staging, prod). Un cliente puede activar múltiples perfiles simultáneamente; el servidor entrega PropertySources para cada perfil activo, y la precedencia sigue el orden en que se declararon los perfiles.

```yaml
# En el cliente: activa dos perfiles simultáneamente
spring:
  profiles:
    active: prod,monitoring

# El servidor devolverá PropertySources para AMBOS perfiles:
# 1. order-service-monitoring.yml   (último perfil activo = mayor prioridad)
# 2. order-service-prod.yml
# 3. application-monitoring.yml
# 4. application-prod.yml
# 5. order-service.yml
# 6. application.yml
```

La convención de nomenclatura es `{application}-{profile}.yml` o `{application}-{profile}.properties`. El servidor reconoce ambos formatos y los devuelve con la misma precedencia.

## Ejemplo central

El siguiente ejemplo muestra el escenario completo de resolución con conflicto de claves para demostrar exactamente qué valor recibe el cliente.

**Ficheros en el repositorio Git**:

```yaml
# application.yml (base global)
app:
  name: "My Application"
  timeout: 30
  feature-flag: false

server:
  port: 8080
```

```yaml
# application-prod.yml (global para producción)
app:
  timeout: 10
  feature-flag: true

logging:
  level:
    root: WARN
```

```yaml
# order-service.yml (base del servicio)
app:
  name: "Order Service"
  max-orders: 100

server:
  port: 8081
```

```yaml
# order-service-prod.yml (servicio en producción)
app:
  max-orders: 500
  payment-url: "https://payment.prod.internal"

server:
  port: 9091
```

**Resultado de resolución para `order-service/prod/main`**:

El cliente `order-service` con perfil `prod` recibe las siguientes propiedades fusionadas (la clave con mayor precedencia gana):

```
server.port           = 9091        ← de order-service-prod.yml
app.name              = "Order Service" ← de order-service.yml
app.timeout           = 10          ← de application-prod.yml
app.feature-flag      = true        ← de application-prod.yml
app.max-orders        = 500         ← de order-service-prod.yml
app.payment-url       = "https://payment.prod.internal" ← de order-service-prod.yml
logging.level.root    = WARN        ← de application-prod.yml
```

## Overrides — propiedades no sobreescribibles

> [ADVERTENCIA] Esta sección es contenido avanzado (Grupo 2). Los overrides son un mecanismo para forzar valores desde el servidor que el cliente no puede sobreescribir con sus propiedades locales.

El mecanismo de overrides permite al Config Server inyectar propiedades que tienen mayor precedencia que cualquier propiedad local del cliente. Es útil para forzar configuración de seguridad o compliance que los equipos de servicio no deben poder ignorar.

```yaml
# application.yml del Config Server
spring:
  cloud:
    config:
      server:
        overrides:
          security.require-ssl: true
          logging.level.org.springframework.security: DEBUG
          management.endpoints.web.exposure.include: "health,info"
```

Estas propiedades llegan al cliente como un PropertySource con la máxima precedencia, por encima de `application.yml` local y de las propiedades del sistema operativo. El comportamiento se puede modificar en el cliente:

```yaml
# En el cliente: controla si los overrides del servidor pueden ser sobreescritos localmente
spring:
  cloud:
    config:
      allow-override: true          # true = los overrides NO son inviolables (default)
      override-none: false          # false = comportamiento estándar (default)
      override-system-properties: true  # true = overrides tienen precedencia sobre system props
```

> [EXAMEN] La semántica de `allow-override` es contraintuitiva. `allow-override: true` significa que el cliente PUEDE sobreescribir los overrides del servidor con propiedades locales. `allow-override: false` significa que los overrides del servidor son INVIOLABLES.

## Buenas y malas prácticas

Hacer:
- Usar `application.yml` (sin perfil) en el repositorio para propiedades que aplican a todos los servicios y entornos (ej. configuración de logging base, timeouts de red comunes).
- Nombrar los ficheros con la convención `{application}-{profile}.yml` exactamente como el `spring.application.name` del cliente; cualquier diferencia de mayúsculas/minúsculas puede hacer que el servidor no encuentre el fichero en sistemas de archivos case-sensitive.
- Documentar en el repositorio de configuración la precedencia esperada para evitar confusión en el equipo.
- Usar overrides del servidor solo para propiedades de compliance o seguridad obligatorias; no abusar del mecanismo para configuración normal.

Evitar:
- Definir la misma clave en múltiples niveles sin documentar cuál es el valor esperado; genera confusión cuando el valor final es diferente al que alguien editó.
- Asumir que las propiedades locales del cliente siempre tienen menor precedencia que las del Config Server; depende de `allow-override` y `override-none`.
- Confundir `spring.profiles.active` del cliente (determina el perfil para resolución de configuración) con `spring.profiles.active` del servidor (activa el backend).
- Mezclar YAML y properties para el mismo servicio en el repositorio; el servidor fusiona ambos pero el resultado puede ser sorprendente si los formatos coexisten.

## Verificación y práctica

Para verificar la resolución y precedencia de propiedades, el endpoint `env` de Actuator muestra todas las fuentes y sus valores:

```bash
# Ver todas las propiedades y sus PropertySources (ordenados por precedencia)
curl http://localhost:8080/actuator/env | python3 -m json.tool

# Ver el valor y la fuente de una propiedad específica
curl http://localhost:8080/actuator/env/app.max-orders
# Respuesta:
# {
#   "property": {
#     "source": "configserver:https://github.com/myorg/config-repo/order-service/order-service-prod.yml",
#     "value": "500"
#   },
#   "activeProfiles": ["prod"]
# }

# Ver la configuración que devuelve el servidor directamente (sin cliente)
curl http://localhost:8888/order-service/prod/main
```

**Preguntas estilo examen VMware Spring Professional:**

1. Un cliente `payment-service` tiene perfil `prod` activo. El repositorio contiene: `application.yml`, `application-prod.yml`, `payment-service.yml`, `payment-service-prod.yml`. ¿En qué orden fusiona el Config Server los PropertySources?
2. La clave `app.timeout` está definida como `30` en `application.yml` y como `10` en `application-prod.yml`. El cliente tiene perfil `prod`. ¿Qué valor recibe?
3. ¿Qué diferencia hay entre `spring.cloud.config.server.overrides` y propiedades normales en el repositorio de configuración?
4. `allow-override: false` en el cliente — ¿qué efecto tiene sobre los overrides del servidor?
5. ¿Puede un cliente tener múltiples perfiles activos simultáneamente? ¿Cómo afecta esto a la resolución?

---

← [1.3.2 Backends alternativos](sc-config-backends.md) | [Índice](README.md) | [1.5 Refresh de configuración](sc-config-refresh.md) →

