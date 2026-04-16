# 1.7 Seguridad del Config Server

← [1.6 Actualización dinámica de configuración (Refresh)](sc-config-refresh.md) | [Índice (README.md)](README.md) | [1.8 Operación del Config Server en producción →](sc-config-operacion.md)

---

## Introducción

El Config Server tiene acceso a **toda la configuración de todos los microservicios del ecosistema**, incluyendo credenciales de bases de datos, claves de API y tokens de servicio. Un Config Server sin protección adecuada es una brecha de seguridad que expone en un solo endpoint toda la superficie de ataque del sistema.

La seguridad del Config Server tiene dos dimensiones complementarias: la **protección del acceso HTTP** al servidor (autenticación, TLS) y el **cifrado de los valores sensibles** almacenados en el backend (de modo que incluso con acceso al repositorio Git, los secretos no sean legibles en texto plano).

> [ADVERTENCIA] El Config Server expone endpoints sin autenticación por defecto si no se añade Spring Security al classpath. En producción, nunca desplegar el Config Server sin protección HTTP.

> [PREREQUISITO] Para configurar HTTPS es necesario disponer de un certificado TLS válido (o autofirmado para entornos internos) en formato JKS o PKCS12. Para cifrado asimétrico, un keystore JKS o PKCS12 con un par de claves RSA.

## Diagrama: capas de seguridad del Config Server

El siguiente diagrama muestra las dos capas de protección independientes que se pueden aplicar.

```
CAPA 1 — Protección del acceso HTTP
┌────────────────────────────────────────────────────────┐
│ Config Client                                          │
│  spring.cloud.config.username: config                  │
│  spring.cloud.config.password: secret                  │
└───────────────────┬────────────────────────────────────┘
                    │  HTTPS + HTTP Basic Auth
                    ▼
┌────────────────────────────────────────────────────────┐
│ Config Server (Spring Security)                        │
│  spring.security.user.name: config                     │
│  spring.security.user.password: secret                 │
└───────────────────┬────────────────────────────────────┘
                    │  lee ficheros del backend
                    ▼
CAPA 2 — Cifrado de valores en el backend
┌────────────────────────────────────────────────────────┐
│ Git repo / filesystem                                  │
│  db.password: '{cipher}AQA7K3m...'  ← cifrado         │
│  api.key:     '{cipher}BXB8L4n...'  ← cifrado         │
└────────────────────────────────────────────────────────┘
    │ servidor llama a /decrypt antes de devolver el valor
    ▼ (o el cliente descifra, según configuración)
Valor en claro solo en memoria del cliente que lo consume
```

Las dos capas son independientes y complementarias: se pueden usar juntas o por separado.

## Ejemplo central

### Capa 1a: Protección HTTP con Spring Security (HTTP Basic)

```xml
<!-- pom.xml del Config Server — añadir Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```yaml
# application.yml del Config Server
spring:
  security:
    user:
      name: ${CONFIG_SERVER_USER:config}
      password: ${CONFIG_SERVER_PASSWORD:changeme}
      roles: ACTUATOR, USER

  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          default-label: main
```

El cliente configura las credenciales correspondientes:

```yaml
# application.yml del Config Client
spring:
  cloud:
    config:
      username: ${CONFIG_SERVER_USER:config}
      password: ${CONFIG_SERVER_PASSWORD:changeme}
```

> Las credenciales del cliente hacia el Config Server son las únicas propiedades que **no pueden** venir del propio Config Server. Deben estar en `application.yml` local o como variables de entorno.

### Capa 1b: HTTPS/TLS en el Config Server

```yaml
# application.yml del Config Server
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.jks      # o file:/etc/ssl/keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: JKS                    # o PKCS12
    key-alias: config-server
```

El cliente apunta al puerto HTTPS:

```yaml
spring:
  config:
    import: "configserver:https://config-server:8443"
```

Si el certificado es autofirmado (entornos internos), el cliente necesita confiar en él añadiendo el certificado raíz al truststore de la JVM o configurando `spring.cloud.config.tls.*`.

### Capa 2a: Cifrado simétrico con `encrypt.key`

El cifrado simétrico usa una clave AES compartida para cifrar y descifrar los valores.

```yaml
# application.yml del Config Server
encrypt:
  key: ${ENCRYPT_KEY}    # clave AES de al menos 128 bits (32 caracteres hex recomendado)
```

Para cifrar un valor, usar el endpoint `/encrypt` del Config Server:

```bash
# Cifrar un valor (requiere autenticación si el servidor la tiene activada)
curl -s -X POST http://config-server:8888/encrypt \
     -u config:secret \
     -d 'mi-password-de-bd'
# Resultado: AQA7K3m8f9X...

# Verificar descifrando
curl -s -X POST http://config-server:8888/decrypt \
     -u config:secret \
     -d 'AQA7K3m8f9X...'
# Resultado: mi-password-de-bd
```

El valor cifrado se almacena en el fichero YAML del backend con el prefijo `{cipher}`:

```yaml
# order-service-prod.yml en el repositorio Git
order:
  db:
    password: '{cipher}AQA7K3m8f9X...'   # las comillas simples son necesarias en YAML
  api-key: '{cipher}BXB8L4n7p2...'
```

### Capa 2b: Cifrado asimétrico con keystore JKS/PKCS12

El cifrado asimétrico usa un par de claves RSA: la clave pública para cifrar, la privada para descifrar. Solo el Config Server (con acceso a la clave privada) puede descifrar.

```yaml
# application.yml del Config Server
encrypt:
  key-store:
    location: classpath:config-server-keystore.jks
    password: ${KEYSTORE_PASSWORD}
    alias: config-server-key
    secret: ${KEY_PASSWORD}       # contraseña de la clave privada dentro del keystore
```

Generar el keystore con `keytool`:

```bash
keytool -genkeypair \
        -alias config-server-key \
        -keyalg RSA \
        -keysize 4096 \
        -dname "CN=Config Server,OU=Platform,O=MiOrg" \
        -keystore config-server-keystore.jks \
        -storepass ${KEYSTORE_PASSWORD} \
        -keypass ${KEY_PASSWORD} \
        -validity 3650
```

El proceso de cifrado/descifrado de valores es idéntico al simétrico: usar los endpoints `/encrypt` y `/decrypt` del Config Server.

### Configuración de cifrado en cliente vs en servidor

Por defecto, el Config Server descifra los valores en el servidor antes de devolverlos al cliente (`spring.cloud.config.server.encrypt.enabled=true`). Los clientes reciben los valores en claro sobre el canal HTTPS.

Para descifrar en el cliente (cada microservicio descifra con su propio keystore):

```yaml
# application.yml del Config Server
spring:
  cloud:
    config:
      server:
        encrypt:
          enabled: false    # el servidor devuelve los valores cifrados tal cual
```

```yaml
# application.yml del Config Client (con su propio keystore)
encrypt:
  key-store:
    location: classpath:client-keystore.jks
    password: ${CLIENT_KEYSTORE_PASSWORD}
    alias: client-key
```

> [ADVERTENCIA] Cifrado en el cliente requiere que cada microservicio tenga acceso al keystore de descifrado. Esto multiplica la superficie de ataque (N microservicios con N copias del keystore) frente al cifrado en el servidor (un único keystore en el Config Server). El cifrado en servidor sobre HTTPS es el enfoque recomendado para la mayoría de los sistemas.

## Tabla de parámetros de seguridad

La siguiente tabla lista los parámetros de seguridad clave con sus valores por defecto.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.security.user.name` | `String` | `user` | Usuario HTTP Basic del Config Server. |
| `spring.security.user.password` | `String` | (generado) | Contraseña HTTP Basic. Por defecto se genera aleatoriamente en cada arranque. |
| `server.ssl.enabled` | `boolean` | `false` | Activa TLS en el servidor. |
| `server.ssl.key-store` | `String` | — | Ruta al keystore TLS. |
| `encrypt.key` | `String` | — | Clave AES para cifrado simétrico. |
| `encrypt.key-store.location` | `String` | — | Ruta al keystore RSA para cifrado asimétrico. |
| `encrypt.key-store.password` | `String` | — | Contraseña del keystore RSA. |
| `encrypt.key-store.alias` | `String` | — | Alias de la clave RSA en el keystore. |
| `spring.cloud.config.server.encrypt.enabled` | `boolean` | `true` | `true` = descifra en servidor; `false` = devuelve el valor cifrado al cliente. |

## Buenas y malas prácticas

**Hacer:**
- Usar cifrado asimétrico (RSA con keystore JKS/PKCS12) en lugar de simétrico en producción: la clave pública puede distribuirse sin riesgo para que cualquier operador cifre nuevos valores, mientras que la clave privada solo vive en el Config Server.
- Rotar la contraseña de HTTP Basic del Config Server periódicamente y mediante variables de entorno (no en el código fuente).
- Desplegar el Config Server detrás de un proxy inverso (Nginx, API Gateway) que termine TLS y restrinja el acceso por red solo a los microservicios autorizados.
- Usar `spring.security.user.roles` para restringir quién puede acceder a los endpoints de `/encrypt` y `/decrypt`: estos endpoints no deben estar accesibles para los microservicios clientes, solo para operadores.

**Evitar:**
- Almacenar `encrypt.key` o las contraseñas del keystore en el `application.yml` del Config Server en texto plano: usar variables de entorno o Kubernetes Secrets.
- Usar el mismo keystore para TLS y para cifrado de propiedades: si el certificado TLS expira o se rota, el cambio no debe afectar al descifrado de propiedades existentes.
- Desactivar la verificación TLS en los clientes (`spring.cloud.config.tls.enabled: false`) en producción: anula la protección de TLS contra ataques de interceptación.

## Comparación: cifrado simétrico vs asimétrico

La siguiente tabla compara los dos mecanismos de cifrado para facilitar la elección.

| Aspecto | Cifrado simétrico (`encrypt.key`) | Cifrado asimétrico (keystore RSA) |
|---|---|---|
| Clave para cifrar nuevos valores | La misma clave AES (secreta) | Clave pública (distribuible) |
| Clave para descifrar | La misma clave AES | Clave privada (solo en el servidor) |
| Operador puede cifrar sin acceso a la clave privada | No | Sí |
| Complejidad de configuración | Baja | Media (keytool, keystore) |
| Tamaño del valor cifrado | Menor | Mayor |
| Recomendado para | Desarrollo, entornos internos simples | Producción, equipos con múltiples operadores |

> El módulo Spring Cloud Security documenta OAuth2, JWT y autorización avanzada. Este fichero cubre solo los mecanismos de protección HTTP del Config Server (HTTP Basic) y el cifrado de propiedades. No documenta OAuth2 ni JWT.

---

← [1.6 Actualización dinámica de configuración (Refresh)](sc-config-refresh.md) | [Índice (README.md)](README.md) | [1.8 Operación del Config Server en producción →](sc-config-operacion.md)
