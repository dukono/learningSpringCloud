# 1.3.1 Backend Git — autenticación, search-paths y etiquetas

← [1.2 Config Client](sc-config-client.md) | [Índice](README.md) | [1.3.2 Backends alternativos](sc-config-backends.md) →

---

## Introducción

El backend Git es la elección por defecto y la más evaluada en el examen VMware Spring Professional para Spring Cloud Config. Resuelve el problema de versionar la configuración junto al código: cada cambio en la configuración queda registrado con un commit, se puede hacer rollback a cualquier estado anterior, y distintas ramas pueden representar distintos entornos. El Config Server actúa como un servidor HTTP que traduce peticiones `{application}/{profile}/{label}` a operaciones Git sobre el repositorio clonado localmente.

> [CONCEPTO] El Config Server con backend Git clona el repositorio en un directorio temporal local y sirve los ficheros desde ahí. La propiedad `label` en la petición del cliente se traduce directamente a una rama, etiqueta o hash de commit Git.

> [PREREQUISITO] Antes de configurar autenticación SSH o search-paths avanzados, el backend Git básico (con URI pública) debe funcionar correctamente.

## Estructura del repositorio y resolución de ficheros

La forma en que el servidor busca ficheros en el repositorio es el concepto más crítico del backend Git. El servidor construye una lista ordenada de ficheros a buscar en función de la aplicación y el perfil, y la posición de búsqueda dentro del repositorio depende de `search-paths`.

```
REPOSITORIO GIT (estructura recomendada con search-paths: '{application}')
│
├── application.yml              ← Propiedades globales (todos los servicios)
├── application-prod.yml         ← Propiedades globales para perfil prod
│
├── order-service/
│   ├── order-service.yml        ← Propiedades base de order-service
│   └── order-service-prod.yml   ← Propiedades de order-service en prod
│
└── inventory-service/
    ├── inventory-service.yml
    └── inventory-service-prod.yml

ORDEN DE RESOLUCIÓN para GET /order-service/prod/main:
  1. order-service/order-service-prod.yml   (máxima prioridad)
  2. order-service/order-service.yml
  3. application-prod.yml
  4. application.yml                         (mínima prioridad, base global)
```

Con `search-paths: '{application}'`, el servidor busca primero en la carpeta del servicio y luego en la raíz. El placeholder `{application}` se sustituye dinámicamente por el nombre de la aplicación cliente.

> [EXAMEN] La diferencia entre `label` (rama/tag Git) y `profile` (entorno: dev/prod) es pregunta directa. `label` controla la versión del código de configuración; `profile` controla el entorno de despliegue.

## Ejemplo central

El siguiente ejemplo muestra la configuración completa del backend Git con los tres métodos de autenticación disponibles: público, usuario/contraseña, y SSH.

**application.yml — Variante 1: repositorio público**:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          clone-on-start: true
          search-paths:
            - '{application}'
            - '{application}-{profile}'
          timeout: 10
          refresh-rate: 30
```

**application.yml — Variante 2: repositorio privado con usuario/contraseña**:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          clone-on-start: true
          search-paths: '{application}'
          username: ${GIT_USERNAME}
          password: ${GIT_TOKEN}
          force-pull: true
```

**application.yml — Variante 3: repositorio privado con SSH**:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:myorg/config-repo.git
          default-label: main
          clone-on-start: true
          search-paths: '{application}'
          ignore-local-ssh-settings: true
          private-key: |
            -----BEGIN OPENSSH PRIVATE KEY-----
            b3BlbnNzaC1rZXktdjEAAAAA...
            -----END OPENSSH PRIVATE KEY-----
          host-key: AAAAB3NzaC1yc2EAAAADAQABAAABAQC...
          host-key-algorithm: ssh-rsa
```

**ConfigServerApplication.java**:

```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**Verificación manual de resolución** (curl):

```bash
# Verificar resolución de configuración para order-service en prod, rama main
curl http://localhost:8888/order-service/prod/main

# Respuesta esperada (PropertySource fusionado):
# {
#   "name": "order-service",
#   "profiles": ["prod"],
#   "label": "main",
#   "propertySources": [
#     {"name": "...order-service/order-service-prod.yml", "source": {...}},
#     {"name": "...order-service/order-service.yml", "source": {...}},
#     {"name": ".../application-prod.yml", "source": {...}},
#     {"name": ".../application.yml", "source": {...}}
#   ]
# }
```

## Tabla de propiedades del backend Git

Todas las propiedades se configuran bajo el prefijo `spring.cloud.config.server.git.*`.

| Propiedad | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `uri` | String | — | URI del repositorio. Acepta https://, git@, file:// |
| `default-label` | String | `master` | Rama/tag por defecto cuando el cliente no especifica label |
| `search-paths` | String[] | `[""]` | Rutas a buscar en el repo. Acepta `{application}`, `{profile}`, `{label}` |
| `username` | String | — | Usuario para autenticación HTTPS |
| `password` | String | — | Contraseña/token para autenticación HTTPS |
| `clone-on-start` | boolean | `false` | Clona al arrancar el servidor (falla rápido si el repo no es accesible) |
| `force-pull` | boolean | `false` | Descarta cambios locales en el clon y fuerza git pull |
| `timeout` | int | `5` | Timeout en segundos para operaciones Git |
| `refresh-rate` | int | `0` | Segundos entre actualizaciones automáticas del clon (0 = en cada petición) |
| `ignore-local-ssh-settings` | boolean | `false` | Usa config SSH inline en lugar del `~/.ssh` del OS |
| `private-key` | String | — | Clave privada SSH (cuando ignore-local-ssh-settings=true) |

> [ADVERTENCIA] `force-pull: true` es crítico en producción. Sin él, si alguien modifica manualmente el directorio temporal de caché del Config Server, el servidor servirá configuración incorrecta silenciosamente.

## Buenas y malas prácticas

Hacer:
- Usar `clone-on-start: true` en producción para detectar problemas de acceso al repositorio en el momento del arranque, no en la primera petición.
- Usar `search-paths: '{application}'` para organizar la configuración por servicio en subdirectorios separados; escala bien con muchos servicios.
- Usar `force-pull: true` en entornos donde el clon local puede corromperse (contenedores efímeros, pruebas CI/CD).
- Almacenar credenciales en variables de entorno (`${GIT_TOKEN}`) nunca en el fichero de configuración del servidor.

Evitar:
- Hardcodear `username` y `password` en `application.yml` porque el fichero puede acabar en un repositorio de código y exponer credenciales.
- Dejar `default-label: master` en repositorios migrados a `main`; el servidor fallará silenciosamente (devuelve 404 en cada petición del cliente).
- Usar `search-paths` sin placeholder cuando hay múltiples servicios; todos los servicios reciben la misma configuración de la raíz del repositorio.
- Ignorar el `timeout` en redes lentas; el valor por defecto de 5s puede causar timeouts en repositorios remotos con latencia alta.

## Verificación y práctica

Para verificar que el backend Git funciona correctamente, el endpoint de diagnóstico del Config Server devuelve el estado del repositorio:

```bash
# Endpoint de estado del servidor (Actuator)
curl http://localhost:8888/actuator/health

# Resolución directa (sin cliente)
curl http://localhost:8888/order-service/default

# Ver propiedades en formato .properties
curl http://localhost:8888/order-service-default.properties
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Qué representa el tercer segmento de la URL de resolución del Config Server (`/app/profile/label`)? ¿Qué ocurre si se omite?
2. ¿Qué diferencia hay entre `spring.cloud.config.server.git.default-label` y `spring.profiles.active` en el cliente?
3. El placeholder `{application}` en `search-paths` — ¿en qué valor se sustituye? ¿Qué propiedad del cliente lo determina?
4. ¿Por qué `clone-on-start: false` (valor por defecto) puede ser problemático en producción?
5. Un Config Server con `force-pull: false` sirve configuración desactualizada aunque el repositorio Git haya cambiado. ¿Por qué? ¿Cómo se soluciona?

---

← [1.2 Config Client](sc-config-client.md) | [Índice](README.md) | [1.3.2 Backends alternativos](sc-config-backends.md) →
