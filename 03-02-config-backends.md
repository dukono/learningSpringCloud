# Parte 3.2 — Spring Cloud Config: Backends

← [Concepto y Arquitectura](./03-01-config-concepto.md) | [Volver al índice](./README.md) | Siguiente: [Config Server →](./03-03-config-server.md)

---

## 3.3 Backends soportados

Spring Cloud Config Server soporta múltiples fuentes de configuración. La elección del backend depende del entorno y las necesidades del equipo.

| Backend | Uso recomendado | Ventajas |
|---|---|---|
| **Git** | Producción, estándar | Historial, auditoría, PRs para cambios |
| **Filesystem** | Desarrollo local | Simple, sin servidor externo |
| **HashiCorp Vault** | Secretos en producción | Cifrado nativo, rotación de secretos |
| **JDBC** | Equipos con BD como estándar | Sin Git, queries SQL |
| **AWS S3 / GCS** | Entornos cloud | Sin servidor Git propio |
| **Composite** | Combinación de backends | Git para config + Vault solo para secrets |

```
Árbol de decisión: ¿qué backend elegir?

¿El entorno ya tiene AWS/GCP como plataforma cloud?
├── SÍ → ¿Secretos con control de acceso granular por servicio?
│         ├── SÍ → Vault (motor Transit + AppRole/K8s auth)
│         └── NO → AWS SSM Parameter Store / S3 / GCS
└── NO → ¿El equipo gestiona todo via Git (Infrastructure as Code)?
         ├── SÍ → Git (HTTPS o SSH)
         │         └── ¿Secretos junto a la config en Git?
         │              ├── SÍ con protección → Git + cifrado {cipher}
         │              └── NO (secretos separados) → Composite: Git + Vault
         └── NO → ¿BD existente como fuente de verdad operacional?
                   ├── SÍ → JDBC (scripts SQL con Flyway/Liquibase)
                   └── NO → Filesystem (solo desarrollo local / tests)
```

---

### Backend Git (más común en producción)

Git es el backend estándar en producción por tres razones concretas: cada cambio de configuración queda registrado con fecha, autor y mensaje de commit; se pueden usar Pull Requests como mecanismo de revisión y aprobación antes de que un cambio llegue a producción; y es posible hacer rollback instantáneo a cualquier versión anterior. Tratar la configuración como código en Git — el patrón _Config as Code_ — es la práctica recomendada en entornos con múltiples servicios y múltiples equipos.

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          default-label: main       # "label" en Config Server = rama, tag o commit de Git
          search-paths: '{application}' # {application} se sustituye por el spring.application.name
                                        # del servicio cliente. Busca en subcarpeta con ese nombre.
          clone-on-start: true      # clona el repo al arrancar el servidor, no en la primera petición
                                    # recomendado: detecta errores de acceso al repo en el arranque
          # Autenticación HTTPS
          username: ${GIT_USER}
          password: ${GIT_TOKEN}
          # Parámetros de refresco
          refresh-rate: 30          # segundos entre comprobaciones de cambios en el repo
          force-pull: true          # descarta cambios locales y fuerza pull (útil si el repo local se corrompe)
          timeout: 5                # segundos máximo para operaciones Git
```

#### Opciones adicionales del backend Git

```yaml
spring:
  cloud:
    config:
      server:
        git:
          # Certificados autofirmados (entornos corporativos internos)
          skip-ssl-validation: true     # omite la validación del certificado SSL del servidor Git
                                        # NUNCA usar en repos Git públicos o accesibles desde internet

          # search-paths con múltiples variables de sustitución
          search-paths: 'config/{application}/{profile}'
          # {application} → spring.application.name del cliente
          # {profile}     → perfil activo del cliente
          # Resultado: buscará en config/pedidos-service/prod/
```

---

#### Autenticación SSH (estándar en empresas)

En entornos corporativos, SSH es preferible a HTTPS por dos motivos: muchos firewalls bloquean el tráfico HTTPS saliente a hosts externos como `github.com`, pero permiten SSH en el puerto 22. Además, los tokens HTTPS tienen fecha de expiración y deben rotarse manualmente; las claves SSH tienen vida útil indefinida y su revocación es inmediata — solo hay que eliminar la clave pública del servidor Git, sin necesidad de actualizar configuración en los servicios.

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:mi-org/config-repo.git   # URI SSH
          ignore-local-ssh-settings: true
          private-key: |
            -----BEGIN RSA PRIVATE KEY-----
            MIIEowIBAAKCAQEA...
            -----END RSA PRIVATE KEY-----
          # O mejor: apuntar al fichero de clave
          # private-key: ${SSH_PRIVATE_KEY}
          host-key: AAAA...         # clave pública del servidor Git (known_hosts)
          host-key-algorithm: ssh-rsa
```

> **[ADVERTENCIA]** La clave privada nunca debe estar hardcodeada en el YAML del repo. Usar variables de entorno o un gestor de secretos como Vault.

---

### Backend Filesystem (desarrollo local)

El backend `native` (sistema de ficheros) existe para dos casos concretos: **desarrollo local**, donde levantar un servidor Git completo sería un sobrecoste innecesario, y **tests de integración** donde el Config Server se levanta embebido (ver `03-06`). Los ficheros de configuración están en el disco local del Config Server y se leen directamente, sin clon ni red. El precio es la pérdida de todas las ventajas de Git: sin historial, sin rollback, sin acceso concurrente desde múltiples instancias. No usar en producción.

```yaml
spring:
  profiles:
    active: native    # activa el backend de sistema de ficheros
  cloud:
    config:
      server:
        native:
          search-locations:
            - file:///ruta/local/configs         # ruta absoluta
            - classpath:/config                  # dentro del JAR
            - file:${HOME}/config-repo           # relativa al home
```

No requiere Git. Los ficheros se leen directamente del disco. Útil para desarrollo local sin necesidad de repo.

---

### Backend HashiCorp Vault (secretos en producción)

La pregunta que lleva a elegir Vault es: ¿es suficiente con que el repositorio Git sea privado para proteger los secretos? En entornos con cumplimiento regulatorio (PCI-DSS, HIPAA, GDPR) la respuesta es no: los ficheros en un repo privado siguen siendo texto plano en el disco del servidor, accesibles a cualquier administrador del sistema. Vault cifra los secretos en reposo, registra en un log de auditoría quién accedió a qué secreto y cuándo, permite revocar el acceso de forma instantánea sin cambiar el secreto, y puede generar credenciales dinámicas de base de datos que expiran automáticamente. Para secretos en producción, Vault no es un lujo: es la herramienta diseñada específicamente para ese problema.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.vault</groupId>
    <artifactId>spring-vault-core</artifactId>
</dependency>
```

```yaml
spring:
  profiles:
    active: vault
  cloud:
    config:
      server:
        vault:
          host: vault.miempresa.com
          port: 8200
          scheme: https
          authentication: TOKEN
          token: ${VAULT_TOKEN}
          kv-version: 2            # versión del KV secrets engine (1 o 2)
          backend: secret          # nombre del secrets engine en Vault
          default-context: application   # ruta base en Vault
          profile-separator: /     # separador para perfiles en la ruta de Vault
```

Los secretos en Vault se organizan por ruta: `secret/application`, `secret/pedidos-service`, `secret/pedidos-service/prod`.

#### Métodos de autenticación en Vault

`TOKEN` es el más sencillo pero el menos seguro en producción: el token es estático y no rota. Los métodos recomendados son:

```yaml
# AppRole (recomendado en producción para aplicaciones)
# El role-id es público; el secret-id es efímero y rotable
spring:
  cloud:
    config:
      server:
        vault:
          authentication: APPROLE
          app-role:
            role-id: ${VAULT_ROLE_ID}       # identificador del rol (no secreto)
            secret-id: ${VAULT_SECRET_ID}   # secreto efímero (se puede rotar)
            role: config-server             # nombre del rol en Vault
            app-role-path: approle          # mount path del AppRole engine
```

```yaml
# Kubernetes (recomendado cuando el Config Server corre en K8s)
# Usa el Service Account token del pod — sin secretos estáticos
spring:
  cloud:
    config:
      server:
        vault:
          authentication: KUBERNETES
          kubernetes:
            role: config-server             # rol de Vault asociado al Service Account
            kubernetes-path: kubernetes     # mount path del Kubernetes auth engine
            # El token del Service Account se lee automáticamente del pod
```

| Método | Cuándo usarlo | Secretos estáticos |
|---|---|---|
| `TOKEN` | Desarrollo local, pruebas | Sí (peligroso en prod) |
| `APPROLE` | Producción en servidores/VMs | No (secret-id rotable) |
| `KUBERNETES` | Producción en Kubernetes | No (usa SA token del pod) |

---

### Backend JDBC

El backend JDBC tiene sentido en organizaciones donde las operaciones giran alrededor de bases de datos: el equipo no tiene experiencia con Git, la infraestructura de BD ya existe y está monitorizada, y los cambios de configuración se gestionan mediante scripts SQL con herramientas ya conocidas como Flyway o Liquibase. Es la opción con menor curva de aprendizaje para equipos con perfil DBA. El coste es la pérdida del historial de cambios que Git proporciona de forma gratuita — aunque puede compensarse añadiendo un campo de `last_modified` y `modified_by` a la tabla.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

```yaml
spring:
  profiles:
    active: jdbc
  datasource:
    url: jdbc:postgresql://localhost:5432/config_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  cloud:
    config:
      server:
        jdbc:
          sql: SELECT prop_key, value FROM properties
               WHERE application=? AND profile=? AND label=?
          order: 1   # prioridad si se usa con composite backend
```

Estructura de la tabla esperada:
```sql
CREATE TABLE properties (
    application VARCHAR(50) NOT NULL,
    profile     VARCHAR(50) NOT NULL,
    label       VARCHAR(50) NOT NULL,
    prop_key    VARCHAR(255) NOT NULL,
    value       TEXT,
    PRIMARY KEY (application, profile, label, prop_key)
);
```

---

### Backend AWS S3 / GCS (entornos cloud)

Los backends de almacenamiento cloud son la opción natural para equipos que ya están completamente en AWS o GCP y quieren evitar gestionar un servidor Git propio. La ventaja operacional es que S3 y GCS son servicios gestionados: replicación, durabilidad y disponibilidad son responsabilidad del proveedor. La autenticación se integra con los sistemas IAM existentes (roles de EC2/EKS, Workload Identity en GKE), sin necesidad de gestionar credenciales adicionales. La convención de nombres de ficheros es idéntica a Git, por lo que migrar de Git a S3 es trivial.

```xml
<!-- Para AWS S3 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-s3</artifactId>
</dependency>
```

```yaml
spring:
  profiles:
    active: awss3
  cloud:
    config:
      server:
        awss3:
          region: eu-west-1
          bucket: mi-empresa-config-repo   # nombre del bucket S3
```

Los ficheros en el bucket siguen la misma convención de nombres que en Git: `application.yml`, `pedidos-service-prod.yml`, etc.

Las credenciales AWS se resuelven automáticamente mediante la cadena estándar de AWS (variables de entorno `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, IAM role del EC2/EKS, etc.).

---

### Backend AWS SSM Parameter Store

AWS Systems Manager Parameter Store es la opción natural cuando el equipo ya usa SSM para gestionar secretos en AWS y quiere evitar añadir un servidor Git o un servidor de configuración adicional. A diferencia de S3 (que almacena ficheros YAML enteros), SSM almacena parámetros individuales organizados en jerarquías de ruta — cada clave de configuración es un parámetro SSM con su propia ACL y política de acceso. Esto permite un control de acceso más granular: el equipo de base de datos solo puede ver los parámetros de conexión, el equipo de pagos solo los suyos.

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-parameter-store</artifactId>
</dependency>
```

```yaml
spring:
  profiles:
    active: awsparamstore
  cloud:
    config:
      server:
        awsparamstore:
          region: eu-west-1
          # Prefijo base de las rutas en SSM (defecto: /config)
          # Patrón de rutas: /{prefix}/{application}_{profile}
          # Ejemplo: /config/pedidos-service_prod/database.url
```

Convención de nombres en SSM Parameter Store:

```
/config/application                  ← propiedades globales (todos los servicios)
/config/application_prod             ← globales para producción
/config/pedidos-service              ← específicas del servicio
/config/pedidos-service_prod         ← producción del servicio
```

> Las credenciales SSM se resuelven igual que S3: cadena estándar AWS o IAM role del pod/instancia. Para producción en EKS, usar IRSA (IAM Roles for Service Accounts) en lugar de credenciales estáticas.

---

### Backend Azure App Configuration

Para equipos en Azure, Azure App Configuration es el equivalente gestionado del Config Server: almacena configuración centralizada con soporte de perfiles (via etiquetas), integración con Azure Key Vault para secretos, y autenticación mediante Managed Identity (sin credenciales estáticas). A diferencia de los backends anteriores, no se configura como backend del Config Server sino directamente en cada servicio cliente como fuente de propiedades.

```xml
<dependency>
    <groupId>com.azure.spring</groupId>
    <artifactId>spring-cloud-azure-appconfiguration-config</artifactId>
</dependency>
```

```yaml
# application.yml del microservicio (configuración directa, sin Config Server)
spring:
  config:
    import: "optional:az-appconfig:"   # prefijo específico de Azure App Configuration

  cloud:
    azure:
      appconfiguration:
        stores:
          - endpoint: https://mi-appconfig.azconfig.io
            # Autenticación con Managed Identity (sin secrets):
            # managed-identity: true
            # O con connection-string:
            # connection-string: ${AZURE_APPCONFIG_CONNECTION_STRING}
        profile-separator: "_"   # equivalente al guión en perfiles de Spring: app_prod
```

> Azure App Configuration es una alternativa directa al patrón Config Server + Git, no un backend del Config Server. Integra de forma nativa con Azure Key Vault para referencias a secretos (`{keyvault:https://...}`), lo que evita almacenar secretos directamente en la store de configuración.

---

### Backend Composite (combinar backends)

El backend Composite surge de una tensión real entre visibilidad y seguridad. La configuración general — nombres de dominio, timeouts, feature flags — se beneficia de estar en Git: cualquier miembro del equipo puede ver la historia, revisar cambios en PRs y entender la evolución del sistema. Pero los secretos no deberían estar en Git nunca, ni siquiera cifrados, porque cualquier administrador del repositorio puede acceder a los commits históricos. La solución es usar ambos backends a la vez, cada uno para lo que hace mejor: Git para visibilidad, Vault para seguridad.

```yaml
spring:
  profiles:
    active: composite
  cloud:
    config:
      server:
        composite:
          # Orden de prioridad: el primero tiene mayor precedencia
          - type: vault
            host: vault.miempresa.com
            port: 8200
            authentication: TOKEN
            token: ${VAULT_TOKEN}

          - type: git
            uri: https://github.com/mi-org/config-repo
            default-label: main
            clone-on-start: true
```

Con este backend, el Config Server busca primero en Vault y luego en Git. Una propiedad que exista en ambos, gana la de Vault.

---

### Múltiples repositorios Git por patrón de servicio

En organizaciones con múltiples equipos, un repositorio único de configuración para todos los servicios crea fricción: el equipo de infraestructura modifica la configuración del gateway mientras el equipo de pedidos modifica la suya, con riesgos de conflictos de merge y necesidades de acceso distintas (el equipo de pedidos no debería tener permisos para modificar la configuración del gateway y viceversa). La solución es asignar un repositorio de configuración por dominio o equipo, y configurar el Config Server para enrutar cada servicio al repositorio correcto según su nombre:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-general   # repo por defecto (fallback)
          repos:
            pedidos:
              pattern: pedidos-*                           # cualquier servicio que empiece por "pedidos-"
              uri: https://github.com/mi-org/config-pedidos
              default-label: main

            infraestructura:
              pattern: '{gateway,eureka,config}-service'   # lista exacta de nombres
              uri: https://github.com/mi-org/config-infra

            frontend:
              pattern:
                - web-app
                - mobile-*
              uri: https://github.com/mi-org/config-frontend
              username: ${GIT_USER}
              password: ${GIT_TOKEN}
```

El Config Server evalúa los patrones en orden. Si ninguno coincide, usa el repo definido en `uri` (el global).

---

### Estructura del repositorio Git de configuración

Independientemente del backend principal, esta es la estructura de ficheros de referencia:

```
config-repo/
├── application.yml              ← propiedades globales (todos los servicios)
├── application-prod.yml         ← global solo para producción
├── pedidos-service.yml          ← propiedades del servicio pedidos
├── pedidos-service-dev.yml      ← pedidos en entorno dev
├── pedidos-service-prod.yml     ← pedidos en entorno prod
├── productos-service.yml
└── gateway-service.yml
```

---

### Semántica del parámetro `label`

El parámetro `label` en Spring Cloud Config tiene significado distinto según el backend. Entenderlo evita confusiones al desplegar en distintos entornos o al querer fijar la configuración a una versión concreta:

| Backend | Qué significa `label` | Valor por defecto |
|---|---|---|
| **Git** | Nombre de rama, tag o hash corto de commit | `main` (configurable con `default-label`) |
| **Filesystem** | Se ignora completamente | — |
| **Vault** | Se ignora (Vault no tiene el concepto de rama) | — |
| **JDBC** | Columna `label` en la tabla `properties` | `master` |
| **S3 / SSM** | Se ignora | — |

**En Git**, los tres usos habituales de `label`:

```bash
# Obtener config de la rama main (por defecto)
curl http://localhost:8888/pedidos-service/prod

# Obtener config de una rama de feature
curl http://localhost:8888/pedidos-service/prod/feature/nueva-bd

# Fijar la configuración a un tag de release — reproducibilidad total
curl http://localhost:8888/pedidos-service/prod/v2.3.1

# Fijar a un commit concreto — auditoría exacta
curl http://localhost:8888/pedidos-service/prod/a3f8c12
```

En el cliente, `label` se puede fijar para que un servicio siempre use la configuración de una rama o tag específico, independientemente de lo que haya en `main`:

```yaml
# application.yml del cliente — usar siempre la rama release/2.3
spring:
  cloud:
    config:
      label: release/2.3
```

> **Caso de uso habitual con tags:** en CI/CD, antes de desplegar una versión, se crea un tag en el repo de configuración con el mismo número de versión (`v2.3.1`). Los servicios de producción apuntan a ese tag, lo que garantiza que un rollback del código y un rollback de la configuración se sincronicen con el mismo tag.

---

→ **Extensión programática:** [03-02-extension-programatica.md](./03-02-extension-programatica.md) — `EnvironmentRepository`, repos dinámicos desde BD, backend custom

← [Concepto y Arquitectura](./03-01-config-concepto.md) | [Volver al índice](./README.md) | Siguiente: [Config Server →](./03-03-config-server.md)
