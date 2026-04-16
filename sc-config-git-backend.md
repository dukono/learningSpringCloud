# 1.4 Git backend — configuración y autenticación

← [1.3 Resolución de configuración por perfiles y labels](sc-config-resolucion.md) | [Índice (README.md)](README.md) | [1.5 Backends alternativos al Git backend →](sc-config-backends.md)

---

## Introducción

El Git backend es la implementación por defecto de `EnvironmentRepository` en Spring Cloud Config Server y el caso de uso más extendido en producción. Almacena la configuración en un repositorio Git estándar, lo que aporta historial de cambios auditado, posibilidad de rollback mediante revert, revisión de cambios por pull request y separación de entornos por ramas o directorios.

La configuración del Git backend cubre dos dimensiones: la **localización** del repositorio (URI, ramas, directorios) y la **autenticación** (SSH con clave privada, HTTPS con usuario/contraseña o token). Ambas dimensiones tienen múltiples opciones con implicaciones de seguridad y rendimiento que es imprescindible dominar para un entorno productivo.

> [PREREQUISITO] Para las opciones de autenticación SSH es necesario disponer de un par de claves SSH generado y la clave pública añadida al proveedor Git (GitHub, GitLab, Bitbucket). Para autenticación HTTPS, un token de acceso personal con permiso de lectura sobre el repositorio.

## Diagrama: relación entre Config Server y el repositorio Git

El siguiente diagrama muestra los pasos que ejecuta el `JGitEnvironmentRepository` cuando recibe una petición de resolución de configuración.

```
Config Server recibe: GET /order-service/prod/main
         │
         ▼
JGitEnvironmentRepository
         │
         ├── ¿Existe clone local en basedir?
         │         NO ──► git clone {uri}  (autenticación SSH o HTTPS)
         │         SÍ ──► git fetch (si refreshRate ha expirado)
         │
         ▼
         git checkout {label}   (rama/tag/commit)
         │
         ▼
         Busca ficheros en search-paths:
         application.yml, {app}.yml, {app}-{profile}.yml
         │
         ▼
         Construye PropertySource[] y los devuelve al cliente
```

El clon local en `basedir` es un **caché**: el servidor no clona en cada petición, sino que hace `fetch` periódicamente según `refreshRate`.

## Ejemplo central

A continuación se presenta la configuración completa del Git backend para los escenarios más habituales.

### Configuración básica con repositorio HTTPS público

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          default-label: main
          clone-on-start: true      # valida la conexión al arrancar el servidor
          timeout: 10               # segundos máximos esperando respuesta del servidor Git
          refresh-rate: 30          # segundos entre fetch automáticos
```

### Autenticación HTTPS con usuario/contraseña o token

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          username: ${GIT_USERNAME}          # usuario o nombre del token en GitHub
          password: ${GIT_TOKEN}             # contraseña o valor del personal access token
          default-label: main
          clone-on-start: true
```

Para GitHub y GitLab modernos, `username` puede ser cualquier string y `password` es el Personal Access Token con scope `read_contents` (GitHub) o `read_repository` (GitLab).

### Autenticación SSH con clave privada

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:mi-org/config-repo.git
          ignore-local-ssh-settings: true     # ignora ~/.ssh del sistema operativo
          private-key: |                       # clave privada RSA/ECDSA en línea
            -----BEGIN OPENSSH PRIVATE KEY-----
            b3BlbnNzaC1rZXktdjEAAAAA...
            -----END OPENSSH PRIVATE KEY-----
          strict-host-key-checking: false      # solo en entornos sin known_hosts configurado
          default-label: main
          clone-on-start: true
```

> [ADVERTENCIA] `strict-host-key-checking: false` deshabilita la verificación de la identidad del servidor Git remoto, abriendo la puerta a ataques man-in-the-middle. En producción, configura el `known-hosts-file` o usa HTTPS. Si debes usar SSH en producción, establece `strict-host-key-checking: true` (valor por defecto) y asegúrate de que el fichero `known_hosts` del sistema o el especificado en `known-hosts-file` contiene la huella del servidor Git.

### Configuración con search-paths y múltiples directorios

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          search-paths:
            - "config/{application}"     # busca en subdirectorios por nombre de app
            - "config/global"            # busca también en directorio global
          default-label: main
          clone-on-start: true
```

Con esta configuración, `order-service` buscaría en `config/order-service/` y en `config/global/`. El placeholder `{application}` se sustituye por el nombre del cliente.

### Configuración con múltiples repositorios (multi-repo)

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-default   # repositorio por defecto
          default-label: main
          repos:
            team-payments:
              uri: https://github.com/mi-org/config-payments
              pattern:
                - "payment-*"           # aplica a todos los servicios que empiecen por payment-
              default-label: main
            team-orders:
              uri: https://github.com/mi-org/config-orders
              pattern:
                - "order-*"
                - "invoice-*"
              default-label: release
```

El Config Server evalúa los patrones en orden de declaración. El primer patrón que coincide con `{application}/{profile}` determina qué repositorio se usa. Si ningún patrón coincide, se usa el repositorio raíz.

> [EXAMEN] El campo `pattern` puede incluir comodines y también combinar `{application}/{profile}`: `"payment-service/prod"` aplica solo al servicio `payment-service` en el perfil `prod`. Los comodines `*` y `?` siguen la semántica de Spring `AntPathMatcher`.

### Propiedades de caché y resiliencia

```yaml
spring:
  cloud:
    config:
      server:
        git:
          basedir: /tmp/config-cache       # directorio local del clon
          force-pull: true                 # descarta cambios locales y fuerza pull
          refresh-rate: 60                 # segundos entre refreshes del caché local
          clone-on-start: true
          timeout: 15
```

`force-pull: true` es necesario cuando el repositorio local puede quedar en estado inconsistente (por ejemplo, después de un reinicio abrupto del servidor). Sin `force-pull`, el servidor puede servir configuración stale del clon local.

## Tabla de parámetros del Git backend

La siguiente tabla recoge los parámetros más relevantes bajo `spring.cloud.config.server.git.*`.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `uri` | `String` | — | URL del repositorio (HTTPS o SSH). Obligatorio. |
| `default-label` | `String` | `master` | Rama/tag/commit usado cuando el cliente no especifica label. |
| `search-paths` | `List<String>` | `[""]` | Subdirectorios donde buscar los ficheros de configuración. Soporta `{application}`, `{profile}`, `{label}`. |
| `username` | `String` | — | Usuario para autenticación HTTPS. |
| `password` | `String` | — | Contraseña/token para autenticación HTTPS. |
| `private-key` | `String` | — | Clave privada SSH en formato PEM (inline). |
| `ignore-local-ssh-settings` | `boolean` | `false` | Si `true`, ignora `~/.ssh` del sistema y usa solo lo configurado. |
| `strict-host-key-checking` | `boolean` | `true` | Verifica la identidad del servidor Git remoto. |
| `known-hosts-file` | `String` | — | Ruta al fichero `known_hosts` personalizado para SSH. |
| `clone-on-start` | `boolean` | `false` | Si `true`, clona al arrancar el servidor en lugar de en la primera petición. |
| `force-pull` | `boolean` | `false` | Si `true`, descarta cambios locales en cada fetch. |
| `basedir` | `String` | directorio temporal | Directorio local para el clon del repositorio. |
| `timeout` | `int` | `5` | Segundos de timeout para operaciones Git remotas. |
| `refresh-rate` | `int` | `0` | Segundos entre refreshes automáticos del clon local. `0` = refresh en cada petición. |

## Buenas y malas prácticas

**Hacer:**
- Activar `clone-on-start: true` en todos los entornos no locales para que el servidor valide la conexión y la autenticación al arrancar, no en la primera petición de un microservicio.
- Usar `private-key` con `ignore-local-ssh-settings: true` en contenedores: el sistema de ficheros del contenedor no tiene `~/.ssh` configurado y confiar en él es frágil.
- Establecer `refresh-rate` en un valor razonable (30–120 segundos) en producción para evitar que el servidor haga un `git fetch` en cada petición de configuración bajo carga alta.
- Almacenar la clave privada SSH y los tokens HTTPS en variables de entorno o en un gestor de secretos (Vault, Kubernetes Secrets), nunca en el `application.yml` del Config Server.

**Evitar:**
- Usar `strict-host-key-checking: false` en producción. Si el servidor Git cambia su clave de host (por una migración o un incidente), el Config Server aceptará silenciosamente la nueva identidad, abriendo una ventana de ataque.
- Olvidar configurar `basedir` en entornos con disco efímero (contenedores sin volumen persistente): el clon local se pierde en cada reinicio y el servidor clona desde cero en cada arranque, lo que añade latencia de arranque y puede saturar el servidor Git.
- Combinar `force-pull: false` con un `basedir` compartido entre múltiples instancias del Config Server: las instancias pueden interferir en el estado del clon local.

## Comparación: autenticación SSH vs HTTPS

La siguiente tabla compara ambos mecanismos de autenticación para ayudar a elegir el más adecuado según el contexto.

| Aspecto | SSH con clave privada | HTTPS con token |
|---|---|---|
| Configuración | Más compleja (generación de par de claves, known_hosts) | Más sencilla (usuario + token en YAML) |
| Revocación | Por par de claves (independiente por servicio) | Por token (un token para todos) |
| Rotación | Requiere generar nuevo par y actualizar el servidor Git | Solo actualizar el token |
| Uso en contenedores | Requiere `ignore-local-ssh-settings: true` | Sin configuración adicional |
| Compatibilidad con proxies HTTP | No | Sí |
| Recomendado para | Entornos on-premise con Git propio | Servicios cloud (GitHub, GitLab, Bitbucket) |

---

← [1.3 Resolución de configuración por perfiles y labels](sc-config-resolucion.md) | [Índice (README.md)](README.md) | [1.5 Backends alternativos al Git backend →](sc-config-backends.md)
