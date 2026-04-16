# 1.5 Backends alternativos al Git backend

← [1.4 Git backend — configuración y autenticación](sc-config-git-backend.md) | [Índice (README.md)](README.md) | [1.6 Actualización dinámica de configuración (Refresh) →](sc-config-refresh.md)

---

## Introducción

El Git backend es el caso de uso mayoritario de Spring Cloud Config, pero no el único. Existen escenarios donde Git no es la solución adecuada: entornos con secretos que no deben vivir en un repositorio (aunque estén cifrados), equipos que ya operan un gestor de secretos como HashiCorp Vault, sistemas legacy con configuración en bases de datos relacionales, o arquitecturas en AWS donde un bucket S3 es la fuente de verdad más natural.

Spring Cloud Config Server implementa cada backend como una `EnvironmentRepository` intercambiable. La interfaz del servidor HTTP es idéntica en todos los casos: lo que cambia es dónde y cómo se leen las propiedades. Además, el backend Composite permite combinar varios backends en paralelo con una política de precedencia explícita.

> [CONCEPTO] El punto de extensión es la interfaz `EnvironmentRepository`: implementar un solo método `findOne(application, profile, label)` es suficiente para conectar cualquier backend personalizado al Config Server sin modificar la capa HTTP.

## Tabla comparativa de backends

La siguiente tabla resume las características principales de cada backend para facilitar la elección según el contexto.

| Backend | Dependencia adicional | Casos de uso | Gestión de secretos | Soporte de label |
|---|---|---|---|---|
| Git (default) | Ninguna | General, auditoría, entornos por rama | Cifrado `{cipher}` o delegación a Vault | Sí (rama/tag/commit) |
| Native (filesystem) | Ninguna | Desarrollo local, CI sin Git | No (ficheros planos) | No |
| Vault | `spring-cloud-starter-vault-config` | Secretos gestionados, rotación automática | Nativo (Vault policies) | No |
| JDBC | `spring-cloud-config-server` + driver JDBC | Configuración en BD existente | Solo si la BD está securizada | No |
| AWS S3 | `spring-cloud-config-server-aws` o `awspring` | Ecosistemas AWS, buckets como repositorio | IAM policies | Sí (prefijo de path) |
| Composite | (combina los anteriores) | Secretos en Vault + resto en Git | Combinado | Según backend |
| Custom | Implementación `EnvironmentRepository` | Cualquier fuente propietaria | Personalizado | Configurable |

## Ejemplo central

A continuación se muestra la configuración de cada backend, con el caso de Composite como ejemplo integrador.

### Backend Native (filesystem)

El backend native lee propiedades desde el sistema de ficheros local del servidor. Es el más sencillo y se usa principalmente en desarrollo y tests.

```yaml
# application.yml del Config Server
spring:
  profiles:
    active: native                  # activa el EnvironmentRepository de filesystem

  cloud:
    config:
      server:
        native:
          search-locations:
            - classpath:/config/       # busca en src/main/resources/config/
            - file:/etc/config/        # busca también en el sistema de ficheros
```

Con `search-locations: classpath:/config/`, los ficheros de configuración van en `src/main/resources/config/`:

```
src/main/resources/config/
├── application.yml
├── order-service.yml
└── order-service-prod.yml
```

> [ADVERTENCIA] El backend native no tiene historial de cambios ni mecanismo de `label`. Si el Config Server se reinicia con ficheros modificados, los clientes reciben la nueva configuración en el siguiente refresh pero sin trazabilidad de qué cambió. No usar en producción si la auditoría de cambios es un requisito.

### Backend Vault

El backend Vault delega la lectura de secretos a HashiCorp Vault. El Config Server actúa como proxy: recibe el token del cliente y lo usa para consultar Vault.

```yaml
spring:
  profiles:
    active: vault

  cloud:
    config:
      server:
        vault:
          host: vault.example.com
          port: 8200
          scheme: https
          backend: secret            # motor de secretos KV (default: secret)
          default-key: application   # clave raíz para propiedades globales
          token: ${VAULT_TOKEN}      # token del Config Server para leer de Vault
```

> El cliente pasa su propio token de Vault al Config Server mediante `spring.cloud.config.token`. La gestión de políticas, motores secretos y autenticación nativa de Vault es responsabilidad de HashiCorp Vault, no del Config Server.

El Config Server consulta las rutas `secret/application`, `secret/{application}` y `secret/{application}/{profile}` en Vault.

### Backend JDBC

El backend JDBC lee propiedades de una tabla relacional. Requiere que la tabla `PROPERTIES` exista con columnas `APPLICATION`, `PROFILE`, `LABEL`, `KEY` y `VALUE`.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/configdb
    username: ${DB_USER}
    password: ${DB_PASSWORD}

  profiles:
    active: jdbc

  cloud:
    config:
      server:
        jdbc:
          sql: "SELECT KEY, VALUE FROM PROPERTIES WHERE APPLICATION=? AND PROFILE=? AND LABEL=?"
          order: 1
```

Esquema mínimo de la tabla:

```sql
CREATE TABLE PROPERTIES (
    APPLICATION VARCHAR(255) NOT NULL,
    PROFILE     VARCHAR(255) NOT NULL,
    LABEL       VARCHAR(255) NOT NULL,
    KEY         VARCHAR(255) NOT NULL,
    VALUE       TEXT,
    PRIMARY KEY (APPLICATION, PROFILE, LABEL, KEY)
);
```

### Backend AWS S3

El backend S3 lee ficheros de propiedades desde un bucket de Amazon S3.

```yaml
spring:
  profiles:
    active: awss3

  cloud:
    config:
      server:
        awss3:
          region: eu-west-1
          bucket: mi-config-bucket   # nombre del bucket S3
```

El Config Server necesita credenciales AWS estándar (variables de entorno `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, perfil `~/.aws/credentials`, o rol IAM adjunto al pod/instancia). La gestión del bucket S3, políticas IAM y cifrado en reposo es externa al Config Server.

> Una línea: 'El Config Server solo necesita la URI `s3://bucket/path` y credenciales AWS estándar (variables de entorno o perfil). La gestión del bucket es externa.'

### Backend Composite (combinación de backends)

El backend Composite combina múltiples `EnvironmentRepository` con orden de precedencia explícito. El caso más habitual es Vault para secretos y Git para el resto.

```yaml
spring:
  profiles:
    active: composite

  cloud:
    config:
      server:
        composite:
          - type: vault
            order: 1                  # mayor precedencia: ganan los secretos de Vault
            host: vault.example.com
            port: 8200
            scheme: https
            token: ${VAULT_TOKEN}
          - type: git
            order: 2                  # menor precedencia: base de configuración en Git
            uri: https://github.com/mi-org/config-repo
            default-label: main
            clone-on-start: true
```

Con esta configuración, si la misma clave aparece en Vault y en Git, gana el valor de Vault (order=1).

### EnvironmentRepository personalizado

```java
package com.example.configserver;

import org.springframework.cloud.config.environment.Environment;
import org.springframework.cloud.config.environment.PropertySource;
import org.springframework.cloud.config.server.environment.EnvironmentRepository;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

// Ejemplo: backend que lee de un sistema de configuración propietario
@Component
public class CustomEnvironmentRepository implements EnvironmentRepository {

    @Override
    public Environment findOne(String application, String profile, String label) {
        // Lógica de consulta al sistema propietario
        Map<String, Object> props = fetchFromProprietarySystem(application, profile);

        PropertySource propertySource = new PropertySource(
                "custom:" + application + "-" + profile,
                props
        );

        Environment env = new Environment(application, profile);
        env.add(propertySource);
        env.setLabel(label);
        return env;
    }

    private Map<String, Object> fetchFromProprietarySystem(String app, String profile) {
        // implementación real aquí
        Map<String, Object> props = new HashMap<>();
        props.put("custom.property", "value-from-custom-source");
        return props;
    }
}
```

Si hay múltiples `EnvironmentRepository` beans en el contexto sin backend Composite, el servidor usa el último registrado. Para controlar el orden, implementar también `Ordered` o usar `@Primary`.

## Buenas y malas prácticas

**Hacer:**
- Usar el backend Composite (Vault + Git) cuando el sistema tiene tanto secretos como configuración ordinaria: Vault gestiona rotación automática de secretos; Git gestiona historial y revisión de configuración no sensible.
- Añadir la dependencia `spring-cloud-starter-vault-config` solo cuando se necesita el backend Vault. Es una dependencia pesada con configuración propia que no debe estar en el classpath si no se usa.
- Definir explícitamente el `order` de cada backend en Composite para que la precedencia sea predecible y no dependa del orden de declaración en YAML.

**Evitar:**
- Usar el backend native en producción si los ficheros de configuración viven en el filesystem del propio servidor: no hay historial, no hay rollback, y un reinicio del servidor puede causar inconsistencias si los ficheros se modifican concurrentemente.
- Combinar el backend JDBC con una base de datos de negocio (la misma BD que usan los microservicios): el Config Server puede saturar el pool de conexiones de la BD en picos de arranque simultáneo de múltiples servicios.
- Poner `VAULT_TOKEN` en el `application.yml` del Config Server como valor literal: usar variables de entorno, Kubernetes Secrets o el propio Vault Agent para inyectar el token en el entorno de ejecución.

---

← [1.4 Git backend — configuración y autenticación](sc-config-git-backend.md) | [Índice (README.md)](README.md) | [1.6 Actualización dinámica de configuración (Refresh) →](sc-config-refresh.md)
