# Parte 3 — Spring Cloud Config (Configuración centralizada)

← [Parte 2 — Arquitectura](./02-arquitectura.md) | [Volver al índice](./README.md) | Siguiente: [Parte 4 — Service Discovery](./04-service-discovery.md) →

---

## 3.1 Qué es y para qué sirve

En una arquitectura de microservicios con 20, 50 o 100 servicios, cada uno tiene su propio `application.yml`. Gestionar esa configuración de forma individual es inmanejable:

- Cambiar la URL de una base de datos implica modificar y redesplegar N servicios
- Los secretos (contraseñas, API keys) están dispersos en múltiples repos
- No hay historial ni auditoría de cambios de configuración
- Distintos entornos (dev, staging, prod) requieren configuraciones distintas sin mecanismo unificado

**Spring Cloud Config** resuelve esto con un **servidor centralizado de configuración** que actúa como fuente única de verdad para todos los microservicios.

**Características clave:**
- Almacena la configuración en un repositorio Git (u otros backends)
- Los servicios la obtienen en tiempo de arranque mediante HTTP
- Soporta múltiples perfiles (`dev`, `staging`, `prod`)
- Permite refresco de configuración **sin reiniciar** los servicios
- Puede cifrar/descifrar valores sensibles

---

## 3.2 Arquitectura: Config Server y Config Client

```
┌─────────────────────────────────────┐
│         Repositorio Git             │
│  ├── application.yml                │  ← config global para todos
│  ├── pedidos-service.yml            │  ← config específica del servicio
│  ├── pedidos-service-prod.yml       │  ← config de producción
│  └── productos-service.yml          │
└─────────────────┬───────────────────┘
                  │ git pull / fetch
┌─────────────────▼───────────────────┐
│          CONFIG SERVER              │
│   (Spring Boot App con             │
│    @EnableConfigServer)             │
│                                     │
│  GET /pedidos-service/prod          │
│  → devuelve la config fusionada     │
└─────────────────┬───────────────────┘
                  │ HTTP al arrancar
        ┌─────────┼─────────┐
        ▼         ▼         ▼
  [pedidos]  [productos]  [gateway]
  (Config    (Config      (Config
   Client)    Client)      Client)
```

**Regla de fusión de configuración:** El Config Server combina propiedades en este orden de prioridad (mayor prioridad primero):

```
{service-name}-{profile}.yml   (ej: pedidos-service-prod.yml)
{service-name}.yml             (ej: pedidos-service.yml)
application-{profile}.yml      (ej: application-prod.yml)
application.yml                (global)
```

Las propiedades del nivel más específico **sobreescriben** las del nivel más genérico.

---

## 3.3 Backends soportados: Git, filesystem, Vault, JDBC

| Backend | Uso recomendado | Ventajas |
|---|---|---|
| **Git** | Producción, estándar | Historial, auditoría, PRs para cambios |
| **Filesystem** | Desarrollo local | Simple, sin servidor externo |
| **HashiCorp Vault** | Secretos en producción | Cifrado nativo, rotación de secretos |
| **JDBC** | Equipos con BD como estándar | Sin Git, queries SQL |
| **AWS S3 / GCS** | Entornos cloud | Sin servidor Git propio |

### Configurar backend Git (más común):
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mi-org/config-repo
          default-label: main           # rama por defecto
          search-paths: '{application}' # busca en subcarpeta con nombre del servicio
          clone-on-start: true          # clona el repo al arrancar (recomendado)
```

### Configurar backend Filesystem (desarrollo):
```yaml
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:///ruta/local/configs
```

---

## 3.4 Configuración del Config Server

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

### Paso 4: Estructura del repositorio Git de configuración

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

### Verificar que funciona

El Config Server expone una API REST. Se puede verificar manualmente:

```bash
# Obtener config de pedidos-service en perfil prod
curl http://localhost:8888/pedidos-service/prod

# Obtener solo un fichero específico
curl http://localhost:8888/pedidos-service/prod/main/pedidos-service.yml
```

Respuesta JSON con todas las propiedades fusionadas del servicio.

---

## 3.5 Configuración del Config Client

Todo microservicio que quiera obtener su configuración del Config Server es un **Config Client**.

### Paso 1: Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### Paso 2: application.yml del cliente

```yaml
spring:
  application:
    name: pedidos-service   # ← CRÍTICO: debe coincidir con el nombre de fichero en Git

  config:
    import: "configserver:http://localhost:8888"  # Spring Boot 3.x

  profiles:
    active: dev   # el perfil activo determina qué fichero carga
```

> **Nota histórica:** En Spring Boot 2.x se usaba `bootstrap.yml` en lugar de `spring.config.import`. En Spring Boot 3.x el `bootstrap.yml` ya no es necesario.

### Paso 3: Uso de las propiedades

Las propiedades del Config Server se inyectan exactamente igual que las locales:

```java
@RestController
public class PedidosController {

    @Value("${pedidos.max-items:100}")   // con valor por defecto
    private int maxItems;

    @Value("${database.url}")
    private String databaseUrl;

    // También con @ConfigurationProperties:
}
```

### Paso 4: Configuración de reintentos (recomendado en producción)

Si el Config Server no está disponible al arrancar, el servicio falla. Se puede configurar reintentos:

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    config:
      fail-fast: true       # falla si no puede conectar (recomendado en prod)
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.1
        max-interval: 2000
```

---

## 3.6 Perfiles y entornos (application-{profile}.yml)

Spring Cloud Config y los perfiles de Spring trabajan juntos para manejar múltiples entornos.

### Estructura de ficheros recomendada en el repo Git:

```yaml
# application.yml (base global)
logging:
  level:
    root: INFO

# application-dev.yml (sobreescribe para dev)
logging:
  level:
    root: DEBUG
    com.miempresa: DEBUG

# pedidos-service.yml (específico del servicio)
pedidos:
  max-items: 100
  timeout-ms: 5000

# pedidos-service-prod.yml (producción del servicio)
pedidos:
  max-items: 1000
  timeout-ms: 2000

spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/pedidos
    username: ${DB_USER}      # variable de entorno, no hardcodeada
    password: ${DB_PASSWORD}
```

### Activar perfil en el cliente:

```yaml
# application.yml del microservicio
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}  # dev por defecto, override con variable de entorno
```

### En Docker / Kubernetes:
```bash
# Variable de entorno en contenedor
SPRING_PROFILES_ACTIVE=prod
```

---

## 3.7 Refresco de configuración: @RefreshScope y Spring Cloud Bus

Por defecto, los microservicios leen la configuración **una sola vez al arrancar**. Si se cambia una propiedad en el repo Git, el servicio no lo verá hasta que se reinicie.

### Solución 1: Refresco manual con @RefreshScope

`@RefreshScope` marca un bean para que sus propiedades se recarguen cuando se llame al endpoint `/actuator/refresh`.

```java
@RestController
@RefreshScope                    // ← el bean se recrea al hacer refresh
public class FeatureController {

    @Value("${feature.nueva-ui:false}")
    private boolean nuevaUiEnabled;

    @GetMapping("/feature-flag")
    public boolean isEnabled() {
        return nuevaUiEnabled;
    }
}
```

```xml
<!-- Necesario para exponer /actuator/refresh -->
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
        include: refresh, health, info
```

```bash
# Trigger del refresco (POST sin body)
curl -X POST http://localhost:8083/actuator/refresh
```

**Problema:** Si hay 20 instancias del servicio, hay que llamar a los 20 endpoints individualmente.

### Solución 2: Spring Cloud Bus (refresco masivo)

Spring Cloud Bus usa un broker de mensajes (Kafka o RabbitMQ) para **propagar el evento de refresco a todas las instancias** de todos los servicios de una sola vez.

```xml
<!-- En todos los microservicios -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    <!-- o spring-cloud-starter-bus-amqp para RabbitMQ -->
</dependency>
```

```bash
# Un solo POST al Config Server o a cualquier instancia
# refresca TODOS los servicios conectados al Bus
curl -X POST http://localhost:8888/actuator/busrefresh
```

```
Config Server recibe POST /actuator/busrefresh
         ↓
  publica evento en Kafka topic "springCloudBus"
         ↓
  todos los microservicios suscritos al topic
  reciben el evento y refrescan sus @RefreshScope beans
```

---

## 3.8 Cifrado y descifrado de propiedades sensibles

El Config Server puede almacenar propiedades cifradas en el repo Git y descifrarlas antes de entregarlas a los clientes.

### Configurar clave simétrica (AES):

```yaml
# bootstrap.yml del Config Server
encrypt:
  key: mi-clave-secreta-muy-larga-de-256-bits
```

### Cifrar un valor:

```bash
curl -X POST http://localhost:8888/encrypt -d "mi-password-secreto"
# responde: AQA8bX3+kY...texto-cifrado...
```

### Usar en el fichero de configuración del repo Git:

```yaml
# pedidos-service-prod.yml (en el repo Git)
spring:
  datasource:
    password: '{cipher}AQA8bX3+kY...texto-cifrado...'
```

El `{cipher}` prefijo indica al Config Server que debe descifrar ese valor antes de entregarlo al cliente.

### Usar clave asimétrica (RSA — recomendado en producción):

```yaml
encrypt:
  key-store:
    location: classpath:/keystore.jks
    password: ${KEYSTORE_PASSWORD}
    alias: config-server-key
```

---

## 3.9 Alta disponibilidad del Config Server

El Config Server es un **punto único de fallo** si solo hay una instancia. En producción se deben tener múltiples instancias:

### Estrategia 1: Múltiples instancias detrás de un load balancer

```
[Load Balancer]
      ├── Config Server :8888 (instancia 1)
      ├── Config Server :8888 (instancia 2)
      └── Config Server :8888 (instancia 3)
           (todas apuntan al mismo repo Git)
```

Los Config Clients usan la URL del Load Balancer:
```yaml
spring:
  config:
    import: "configserver:http://config-lb.interno:8888"
```

### Estrategia 2: Config Server registrado en Eureka

```yaml
# Config Server registrado como eureka client
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/

spring:
  application:
    name: config-server
```

```yaml
# Config Client usa discovery en lugar de URL directa
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server   # nombre en Eureka
```

**Consideración importante:** Si el Config Server también usa Eureka, hay un problema de arranque (bootstrap problem): el servicio necesita Config Server para arrancar, pero necesita Eureka para encontrar el Config Server. La solución es configurar la URL de Eureka directamente en `bootstrap.yml` o dar prioridad al Config Server sobre el Discovery.

---

← [Parte 2 — Arquitectura](./02-arquitectura.md) | [Volver al índice](./README.md) | Siguiente: [Parte 4 — Service Discovery](./04-service-discovery.md) →
