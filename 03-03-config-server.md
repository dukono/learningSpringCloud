# 3.3 Config Server

← [3.2 Backends](./03-02-config-backends.md) | [Índice](./README.md) | [3.4 Config Client →](./03-04-config-client.md)

---

## 3.3 Configuración del Config Server

### Paso 1: Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

### Paso 2: Habilitar con anotación

```java
@SpringBootApplication
@EnableConfigServer          // ← esta anotación lo convierte en Config Server
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### Paso 3: application.yml del Config Server

```yaml
server:
  port: 8888   # puerto convencional del Config Server

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          default-label: main
          clone-on-start: true
          # Para repos privados:
          username: ${GIT_USER}
          password: ${GIT_TOKEN}
```

### Flujo de una petición de configuración

```
Config Client                Config Server              Repositorio Git
     │                            │                           │
     │  GET /pedidos-service/prod │                           │
     │───────────────────────────►│                           │
     │                            │  ¿Caché local vigente?    │
     │                            │  (< refresh-rate segundos)│
     │                            │  Si no → git fetch/pull   │
     │                            │──────────────────────────►│
     │                            │◄──────────────────────────│
     │                            │  pedidos-service-prod.yml │
     │                            │  pedidos-service.yml      │
     │                            │  application-prod.yml     │
     │                            │  application.yml          │
     │                            │                           │
     │                            │  fusiona en orden         │
     │                            │  de prioridad             │
     │◄───────────────────────────│                           │
     │  JSON con propiedades      │                           │
     │  fusionadas (todos los     │                           │
     │  ficheros combinados)      │                           │
     │                            │                           │
     │  aplica al contexto        │                           │
     │  de Spring Boot            │                           │
```

> La caché local del Config Server evita llamar a Git en cada petición. El parámetro `refresh-rate` (en segundos) controla cada cuánto se comprueba si hay cambios en Git. Ver `03-02-config-backends.md`.

### Paso 4: Verificar que funciona

El Config Server expone una API REST. Se puede verificar manualmente:

```bash
# Obtener config de pedidos-service en perfil prod
curl http://localhost:8888/pedidos-service/prod

# Obtener solo un fichero específico
curl http://localhost:8888/pedidos-service/prod/main/pedidos-service.yml
```

El patrón de la URL es: `/{application}/{profile}[/{label}]`

> **[CONCEPTO]** La **API REST del Config Server** sigue el patrón `/{application}/{profile}[/{label}]`. Cada segmento mapea directamente a un atributo del cliente: `application` es el `spring.application.name`, `profile` es el perfil activo, y `label` es la rama o tag del repositorio.

- `application` → `spring.application.name` del cliente
- `profile` → perfil activo del cliente
- `label` → rama/tag/commit de Git (opcional, usa `default-label` si se omite)

Respuesta JSON con todas las propiedades fusionadas del servicio.

---

## 3.3.1 Directorio de caché local del repositorio Git

El Config Server clona el repositorio Git en un directorio temporal del sistema operativo. En contenedores (Docker/Kubernetes) ese directorio desaparece cuando el pod se reinicia, forzando un clon completo en cada arranque.

En producción con K8s se fija el `basedir` a un volumen persistente:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          basedir: /data/config-repo-cache   # ruta fija para el clon local
          # En Kubernetes: montar este path como PersistentVolume
```

```yaml
# Ejemplo de volumen en el Deployment de K8s
spec:
  containers:
    - name: config-server
      volumeMounts:
        - name: git-cache
          mountPath: /data/config-repo-cache
  volumes:
    - name: git-cache
      persistentVolumeClaim:
        claimName: config-server-git-cache
```

> Si no se configura `basedir`, el clon se hace en el directorio temporal del SO (`/tmp/...`). Válido para desarrollo, problemático en producción con múltiples reinicios.

---

## 3.3.2 Proteger el Config Server con autenticación básica

El Config Server contiene configuración sensible y debe estar protegido. La forma más sencilla es autenticación básica:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```yaml
spring:
  security:
    user:
      name: config-user
      password: ${CONFIG_SERVER_PASSWORD}   # nunca hardcodear
```

Los clientes deben incluir las credenciales (ver sección [Config Client](./03-04-config-client.md)).

---

## 3.3.3 Actuator del Config Server

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, env, refresh, busrefresh
```

Endpoints útiles:

```bash
# Estado del servidor y conectividad con el backend (Git, Vault...)
GET /actuator/health

# Forzar refresco del caché interno del Config Server
# Descarta el clon local y fuerza un git fetch en la próxima petición
# DISTINTO del /actuator/refresh del cliente: este opera sobre el servidor, no sobre un microservicio
POST /actuator/refresh
```

> **`POST /actuator/refresh` en el Config Server vs en el cliente:**
>
> | Endpoint | En quién | Qué hace |
> |---|---|---|
> | `POST /actuator/refresh` en el **Config Server** | El propio Config Server | Fuerza re-sincronización de la caché local con Git |
> | `POST /actuator/refresh` en el **cliente** | Un microservicio Config Client | Recarga los beans `@RefreshScope` con los valores actuales del Config Server |
>
> Son dos endpoints distintos con el mismo nombre pero en servicios diferentes. Normalmente se llama primero al del servidor (para que tenga la config actualizada) y luego al del cliente (para que aplique esa config actualizada).

### `/actuator/env` — inspeccionar configuración en runtime

Endpoint esencial para depurar qué valor tiene una propiedad en un servicio en ejecución, y desde qué fuente viene (Config Server, fichero local, variable de entorno, etc.):

```bash
# En cualquier microservicio cliente (no en el Config Server)
# Ver todas las propiedades activas y su origen
GET http://localhost:8083/actuator/env

# Ver el valor de una propiedad concreta
GET http://localhost:8083/actuator/env/pedidos.max-items
```

Respuesta de ejemplo:
```json
{
  "property": {
    "source": "configserver:https://github.com/mi-org/config-repo/pedidos-service-prod.yml",
    "value": "1000"
  }
}
```

El campo `source` indica exactamente de dónde viene el valor: Config Server, fichero local, variable de entorno o argumento de línea de comandos. Imprescindible cuando una propiedad tiene un valor inesperado y no se sabe cuál fuente está ganando.

```yaml
# Exponer en el cliente (además de refresh)
management:
  endpoints:
    web:
      exposure:
        include: health, refresh, env
```

---

## 3.3.4 `server.overrides` — propiedades forzadas en todos los clientes

`server.overrides` es el mecanismo del Config Server para imponer propiedades que ningún cliente puede sobreescribir. A diferencia de las propiedades normales del repositorio Git (que el cliente puede sobreescribir con `override-none=true`, variables de entorno o argumentos de sistema), las declaradas en `server.overrides` llegan a todos los clientes con la prioridad más alta posible: superan a cualquier fuente local, variable de entorno y argumento de línea de comandos.

Casos de uso típicos: deshabilitar features de emergencia en todos los servicios, forzar un nivel de logging mínimo obligatorio por política, o fijar configuración de seguridad que no debe variar entre instancias.

```yaml
# application.yml del Config Server
spring:
  cloud:
    config:
      server:
        overrides:
          # Estas propiedades se distribuyen a TODOS los clientes
          # y no pueden sobreescribirse por ningún mecanismo en el cliente
          feature.mantenimiento: false          # feature flag de emergencia
          logging.level.com.miempresa: INFO     # política de logging obligatoria
          management.endpoints.web.exposure.include: health,info  # seguridad
```

> **[ADVERTENCIA]** `server.overrides` no es lo mismo que poner propiedades en el `application.yml` del repo Git. Las propiedades Git pueden sobreescribirse localmente. Las de `server.overrides` no pueden. Usarlas solo para políticas que realmente deben ser uniformes en todos los servicios y en todos los entornos.

---

## 3.3.5 Endpoint de recursos: servir ficheros completos

Además de la API de propiedades (`/{app}/{profile}/{label}`), el Config Server expone un endpoint para servir ficheros enteros sin parsear: XMLs de Logback, certificados, scripts SQL, o cualquier otro recurso que los microservicios necesiten centralizar junto con la configuración.

```bash
# Patrón: /{application}/{profile}/{label}/{path}
# {path} es la ruta del fichero dentro del repo Git

# Obtener logback.xml de producción del servicio de pedidos
curl http://localhost:8888/pedidos-service/prod/main/logback.xml

# Obtener un script SQL de inicialización
curl http://localhost:8888/application/default/main/db/init.sql

# Obtener el fichero YAML completo sin fusionar (útil para debug)
curl http://localhost:8888/pedidos-service/prod/main/pedidos-service-prod.yml
```

> **[EXAMEN]** El endpoint de propiedades (`/{app}/{profile}`) devuelve JSON con propiedades fusionadas de múltiples ficheros. El endpoint de recursos (`/{app}/{profile}/{label}/{path}`) devuelve el fichero tal cual, sin fusión ni parseo. Son propósitos distintos.

---

## 3.3.6 Health check del Config Server

Por defecto el Config Server verifica la conectividad con el backend (Git, Vault, etc.) en cada llamada a `/actuator/health`. Se puede configurar qué repositorios se verifican:

```yaml
spring:
  cloud:
    config:
      server:
        health:
          repositories:
            pedidos-check:
              label: main
              name: pedidos-service
              profiles: prod
```

```yaml
management:
  health:
    config:
      enabled: true   # defecto: true — deshabilitar solo si el backend es muy lento
```

> Si `enabled: false`, el health check siempre devuelve `UP` independientemente del estado del backend. Útil en CI sin Git disponible, peligroso en producción porque oculta problemas reales de conectividad.

---

## 3.3.7 Endpoint `/monitor` — push notifications de Git

`/monitor` es la alternativa al webhook directo a `/actuator/busrefresh`. Diferencia clave: mientras `busrefresh` refresca todos los servicios, `/monitor` recibe el payload nativo de GitHub/GitLab/Bitbucket, detecta qué ficheros de configuración cambiaron, y publica el evento solo para los servicios afectados. Si solo cambió `pedidos-service-prod.yml`, solo `pedidos-service` recibe el refresco.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-monitor</artifactId>
</dependency>
<!-- Requiere Spring Cloud Bus (Kafka o RabbitMQ) -->
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, busrefresh, monitor
```

Webhook en GitHub apuntando a `POST /monitor` del Config Server. El endpoint interpreta las cabeceras `X-Github-Event`, `X-Gitlab-Event`, etc., y extrae los ficheros modificados del payload.

> Ver ciclo de refresco completo y comparativa en [03-05-config-refresh.md](./03-05-config-refresh.md).

---

→ **Extensión programática:** [3.3.8 — EnvironmentEncryptor, EnvironmentController, EnvironmentPostProcessor](./03-03-extension-programatica.md)

← [3.2 Backends](./03-02-config-backends.md) | [Índice](./README.md) | [3.4 Config Client →](./03-04-config-client.md)
