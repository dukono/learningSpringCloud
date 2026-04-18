# 1.3.2 Backends alternativos — native, Vault, JDBC y composite

← [1.3.1 Backend Git](sc-config-git-backend.md) | [Índice](README.md) | [1.4 Resolución de configuración](sc-config-resolucion.md) →

---

## Introducción

Aunque Git es el backend por defecto y el más utilizado en producción, Spring Cloud Config Server soporta cuatro backends alternativos que resuelven necesidades específicas: el backend **native** (filesystem/classpath) es imprescindible para desarrollo local y tests de integración sin infraestructura Git; **Vault** (HashiCorp) gestiona secretos sensibles con rotación automática y auditoría; **JDBC** permite usar una base de datos relacional existente como fuente de configuración; y el backend **composite** combina múltiples fuentes con orden de precedencia explícito.

> [CONCEPTO] Cada backend se activa mediante un perfil de Spring en el Config Server. `native` con `spring.profiles.active=native`, `vault` con `spring.profiles.active=vault`, y `jdbc` con `spring.profiles.active=jdbc`. El composite no requiere perfil especial: se configura listando backends bajo `spring.cloud.config.server.composite`.

## Backend native — filesystem y classpath

El backend native sirve configuración desde el sistema de archivos local o desde el classpath de la aplicación. Es el más simple de configurar y es la opción estándar para tests de integración: no requiere Git, no requiere red, y los ficheros de configuración pueden vivir en `src/test/resources`.

```
ESTRUCTURA DE FICHEROS CON BACKEND NATIVE
src/
├── main/
│   └── resources/
│       └── config/
│           ├── application.yml           ← Config global
│           ├── order-service.yml         ← Config de order-service
│           └── order-service-prod.yml    ← Config de order-service en prod
└── test/
    └── resources/
        └── config/
            ├── application.yml           ← Config de test (sobrescribe main)
            └── order-service.yml
```

La activación requiere añadir el perfil `native` al Config Server y definir las rutas de búsqueda:

```yaml
# application.yml del Config Server (modo native)
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations:
            - classpath:/config
            - file:///opt/config
            - file:///${user.home}/config
```

> [EXAMEN] El perfil `native` en `spring.profiles.active` activa el backend filesystem. Sin este perfil, el servidor intentará usar Git y fallará si no hay `spring.cloud.config.server.git.uri` configurado.

## Backend Vault — gestión de secretos

> [ADVERTENCIA] Esta sección es contenido avanzado. Requiere haber completado el backend Git (1.3.1) antes de continuar.

HashiCorp Vault es el estándar de facto para gestión de secretos en microservicios. El backend Vault de Config Server permite que los secretos (contraseñas de BD, API keys, certificados) vivan en Vault con rotación automática, auditoría de acceso y control de permisos granular, mientras que la configuración no sensible sigue en Git.

El backend Vault se activa con el perfil `vault` y requiere configurar la conexión al servidor Vault:

```yaml
spring:
  profiles:
    active: vault
  cloud:
    config:
      server:
        vault:
          host: vault.internal.company.com
          port: 8200
          scheme: https
          backend: secret
          default-key: application
          profile-separator: /
          authentication: TOKEN
          token: ${VAULT_TOKEN}
```

El Config Server usa el token de Vault para leer secretos del path `secret/{application}/{profile}`. El cliente pasa su propio Vault token en la cabecera `X-Config-Token` de cada petición al Config Server.

> [EXAMEN] En el composite backend, Vault y Git pueden coexistir. Los secretos de Vault tienen mayor prioridad que las propiedades de Git cuando Vault aparece primero en la lista composite.

## Backend JDBC — base de datos relacional

> [ADVERTENCIA] Esta sección es contenido avanzado. Requiere conocer el flujo de resolución base (1.1) antes de continuar.

El backend JDBC permite almacenar propiedades en una tabla de base de datos relacional. Es útil cuando la organización ya tiene una BD centralizada y no quiere añadir infraestructura Git o Vault. La tabla de propiedades tiene un esquema fijo que el Config Server espera encontrar.

**Esquema de tabla requerido**:

```sql
CREATE TABLE PROPERTIES (
    KEY     VARCHAR(256) NOT NULL,
    VALUE   VARCHAR(4096),
    APPLICATION VARCHAR(256) NOT NULL DEFAULT 'application',
    PROFILE VARCHAR(256) NOT NULL DEFAULT 'default',
    LABEL   VARCHAR(256) NOT NULL DEFAULT 'master'
);

-- Ejemplo de datos
INSERT INTO PROPERTIES (KEY, VALUE, APPLICATION, PROFILE, LABEL)
VALUES ('server.port', '8081', 'order-service', 'prod', 'master');

INSERT INTO PROPERTIES (KEY, VALUE, APPLICATION, PROFILE, LABEL)
VALUES ('spring.datasource.url', 'jdbc:postgresql://prod-db:5432/orders',
        'order-service', 'prod', 'master');
```

**Configuración del servidor con backend JDBC**:

```yaml
spring:
  profiles:
    active: jdbc
  datasource:
    url: jdbc:postgresql://localhost:5432/config_db
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
  cloud:
    config:
      server:
        jdbc:
          sql: SELECT KEY, VALUE FROM PROPERTIES
               WHERE APPLICATION=? AND PROFILE=? AND LABEL=?
          order: 1
```

**Dependencia adicional requerida para JDBC**:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
</dependency>
```

> [EXAMEN] El esquema de tabla JDBC espera columnas `KEY`, `VALUE`, `APPLICATION`, `PROFILE` y `LABEL`. El nombre exacto de las columnas y la consulta SQL son configurables, pero el esquema por defecto usa exactamente esos nombres en mayúsculas.

## Composite backend — múltiples fuentes con prioridad

> [ADVERTENCIA] Esta sección es la más avanzada. Requiere conocer los backends Git, Vault y JDBC antes de continuar. Es contenido de Grupo 2 del itinerario.

El composite backend permite combinar múltiples fuentes de configuración. El orden en la lista determina la prioridad: el primer backend tiene mayor precedencia. Un patrón común en producción es Vault (secretos) + Git (configuración general), donde Vault siempre gana cuando hay conflicto de claves.

**Ejemplo completo: Vault + Git composite**:

```yaml
spring:
  cloud:
    config:
      server:
        composite:
          - type: vault
            host: vault.internal.company.com
            port: 8200
            scheme: https
            authentication: TOKEN
            token: ${VAULT_TOKEN}
            backend: secret
            default-key: application
            order: 1
          - type: git
            uri: https://github.com/myorg/config-repo
            default-label: main
            search-paths: '{application}'
            username: ${GIT_USERNAME}
            password: ${GIT_TOKEN}
            order: 2
```

Con esta configuración, una clave que existe en Vault y en Git tendrá el valor de Vault. Las claves que solo existen en Git se entregan desde Git. El cliente no sabe de qué backend viene cada propiedad.

## Tabla comparativa de backends

La siguiente tabla resume las características clave de cada backend para seleccionar el apropiado según el contexto.

| Backend | Perfil Spring | Caso de uso | Versionado | Secretos | Tests |
|---------|--------------|-------------|-----------|----------|-------|
| Git | (default) | Producción general | Sí (Git) | No directo | Complejo |
| Native | `native` | Desarrollo local, CI/CD | No | No | Ideal |
| Vault | `vault` | Secretos y credenciales | Sí (Vault) | Sí (cifrado) | Requiere Vault |
| JDBC | `jdbc` | BD existente, sin Git | No | No | Requiere BD |
| Composite | (ninguno) | Mezcla de fuentes | Depende | Depende | Complejo |

## Buenas y malas prácticas

Hacer:
- Usar el backend `native` con `classpath:/config` en tests de integración; es la opción más rápida y reproducible sin infraestructura externa.
- Combinar Git + Vault en composite para separar configuración general (Git) de secretos (Vault); es el patrón recomendado en producción.
- Declarar el backend `native` como perfil de test en `@SpringBootTest(properties = "spring.profiles.active=native")` para evitar dependencias Git en tests.

Evitar:
- Usar el backend `native` con rutas absolutas del sistema de archivos en producción; las rutas cambian entre entornos y rompen el servidor.
- Mezclar secretos (contraseñas, API keys) en el repositorio Git aunque estén "obfuscados"; usar Vault o el cifrado `{cipher}` del Config Server.
- Poner el perfil `native` en el `application.yml` de producción; sobreescribe el backend Git y el servidor sirve ficheros locales vacíos.
- Usar el backend JDBC sin índices en la tabla `PROPERTIES`; con muchos registros las consultas se vuelven lentas y bloquean el arranque de clientes.

## Verificación y práctica

Para verificar el backend native en tests:

```bash
# Estructura para test con backend native
src/test/resources/config/
├── application.yml
└── order-service.yml

# Test de integración
@SpringBootTest(properties = {
    "spring.profiles.active=native",
    "spring.cloud.config.server.native.search-locations=classpath:/config"
})
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Qué perfil de Spring activa el backend native en el Config Server?
2. ¿Cuál es la diferencia entre `classpath:/config` y `file:///opt/config` como `search-locations`?
3. En un composite backend con Vault (order=1) y Git (order=2), una clave existe en ambos backends con valores distintos. ¿Cuál gana?
4. ¿Qué esquema de tabla espera el backend JDBC? ¿Cuáles son los nombres de columna por defecto?
5. ¿Por qué el backend native es ideal para tests de integración pero no para producción?

---

← [1.3.1 Backend Git](sc-config-git-backend.md) | [Índice](README.md) | [1.4 Resolución de configuración](sc-config-resolucion.md) →

