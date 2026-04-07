# Parte 3.4 — Spring Cloud Config: Config Client y Perfiles

← [Config Server](./03-03-config-server.md) | [Volver al índice](./README.md) | Siguiente: [Refresco →](./03-05-config-refresh.md)

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
    name: pedidos-service   # CRÍTICO: debe coincidir con el nombre de fichero en Git
                            # el Config Server buscará pedidos-service.yml

  config:
    import: "configserver:http://localhost:8888"  # Spring Boot 3.x

  profiles:
    active: dev   # el perfil activo determina qué fichero carga
```

> **Nota histórica:** En Spring Boot 2.x se usaba `bootstrap.yml` en lugar de `spring.config.import`. En Spring Boot 3.x el `bootstrap.yml` ya no es necesario ni recomendado.

---

### Opciones del `spring.config.import`

```yaml
spring:
  config:
    # Falla si el Config Server no está disponible (comportamiento por defecto)
    import: "configserver:http://localhost:8888"

    # Config Server opcional — si no está disponible, arranca con configuración local
    import: "optional:configserver:http://localhost:8888"
```

> **[EXAMEN]** `optional:configserver:` es clave para el **desarrollo local**: permite arrancar un microservicio sin necesidad de tener el Config Server levantado. En producción se usa sin `optional` para detectar problemas de conectividad en el arranque.

---

### Deshabilitar el Config Client completamente

Para servicios o tests que no deben usar el Config Server:

```yaml
spring:
  cloud:
    config:
      enabled: false   # deshabilita completamente el Config Client
                       # el servicio usa solo su application.yml local
```

Útil en:
- Tests de integración donde no hay Config Server disponible
- Servicios de infraestructura que no necesitan configuración centralizada
- Entornos de desarrollo donde se prefiere configuración local

> A diferencia de `optional:configserver:`, que intenta conectar y sigue si falla, `enabled: false` no intenta la conexión en absoluto.

---

### Múltiples fuentes en `spring.config.import`

`spring.config.import` acepta una lista de fuentes que se cargan en orden. Las fuentes al principio de la lista tienen **mayor prioridad** (sobreescriben a las que vienen después). Esto permite combinar el Config Server con fuentes locales de forma precisa:

```yaml
spring:
  config:
    import:
      # 1. Config Server — fuente principal (mayor prioridad)
      - "configserver:http://config-server:8888"

      # 2. Fichero local opcional — para sobreescribir en desarrollo sin tocar el repo
      #    Solo se carga si el fichero existe; no falla si no está
      - "optional:file:./local-override.yml"

      # 3. Propiedades por defecto en el JAR — menor prioridad
      - "classpath:defaults.yml"
```

> **Orden de prioridad:** la primera fuente de la lista gana ante conflictos. En el ejemplo, si `config-server` y `local-override.yml` definen la misma propiedad, gana `config-server`. Para que el fichero local gane, colocarlo antes del `configserver:`.

```yaml
# Para que el override local tenga prioridad máxima (útil en desarrollo):
spring:
  config:
    import:
      - "optional:file:./local-override.yml"   # primero = prioridad más alta
      - "configserver:http://config-server:8888"
```

---

### Cuando el nombre del fichero difiere del nombre de la aplicación

Por defecto, el Config Server busca ficheros con el mismo nombre que `spring.application.name`. Si el nombre del fichero es diferente:

```yaml
spring:
  application:
    name: pedidos-service

  cloud:
    config:
      name: pedidos-config      # el Config Server buscará pedidos-config.yml
                                # en lugar de pedidos-service.yml
      label: release/1.2        # fuerza una rama/tag específico del repo Git
                                # sobreescribe el default-label del servidor
```

---

### Timeouts de conexión al Config Server

Sin configurar estos valores, el cliente espera indefinidamente si el Config Server no responde:

```yaml
spring:
  cloud:
    config:
      request-connect-timeout: 3000    # ms máximo para establecer la conexión TCP
      request-read-timeout: 10000      # ms máximo esperando la respuesta una vez conectado
```

> En producción, combinar con `fail-fast: true` y `retry` para un comportamiento robusto ante arranques lentos del Config Server.

---

### Precedencia: propiedades remotas vs locales

> **[CONCEPTO]** Por defecto, las propiedades del **Config Server tienen mayor prioridad** que el `application.yml` local. El fichero local solo sirve para arrancar el contexto (URL del servidor, perfil activo). Cualquier propiedad de negocio en el `application.yml` local quedará silenciosamente ignorada si el servidor define la misma clave.

```
Orden de prioridad de fuentes de configuración (mayor prioridad arriba):

  ┌─────────────────────────────────────────────────────────┐
  │  server.overrides del Config Server                     │  ← ningún mecanismo puede sobreescribir
  ├─────────────────────────────────────────────────────────┤
  │  Propiedades del Config Server (repo Git):              │
  │    {service}-{profile}.yml  (más específico)            │
  │    {service}.yml                                        │
  │    application-{profile}.yml                            │
  │    application.yml          (menos específico)          │
  ├─────────────────────────────────────────────────────────┤
  │  Variables de entorno / argumentos del sistema          │  ← si override-system-properties=true
  ├─────────────────────────────────────────────────────────┤
  │  application.yml local del microservicio                │  ← solo URL del servidor + perfil
  └─────────────────────────────────────────────────────────┘

  Con override-none=true: las tres capas inferiores tienen prioridad sobre el Config Server
  (excepto server.overrides, que siempre gana)
```

El comportamiento por defecto sorprende a muchos desarrolladores nuevos a Spring Cloud Config: **las propiedades del Config Server ganan sobre las del `application.yml` local**. Esto es deliberado — el Config Server es la fuente de verdad centralizada y no tiene sentido que cada instancia pueda sobreescribirla localmente. Pero en algunos escenarios sí se necesita que las variables de entorno del contenedor o las propiedades locales tengan prioridad (por ejemplo, un `server.port` asignado por la plataforma de despliegue que no debe sobreescribirse desde Git). Este comportamiento es configurable:

```yaml
spring:
  cloud:
    config:
      allow-override: true              # permite que propiedades locales sobreescriban las remotas
                                        # defecto: true — pero solo funciona si override-none también es true
      override-none: false              # si true, la config remota tiene la MÍNIMA prioridad
                                        # las propiedades locales, de entorno y de sistema siempre ganan
      override-system-properties: true  # si false, variables de entorno y propiedades de sistema
                                        # NO pueden sobreescribir la config remota
```

**Ejemplo práctico:**

```yaml
# Para que en desarrollo las propiedades locales siempre ganen sobre las remotas:
spring:
  cloud:
    config:
      allow-override: true
      override-none: true
```

```yaml
# Para que las variables de entorno del contenedor no puedan sobreescribir config remota:
spring:
  cloud:
    config:
      override-system-properties: false
```

> **[ADVERTENCIA]** El comportamiento por defecto (remoto tiene prioridad) puede sorprender: si el repo Git tiene `server.port=9090` y el `application.yml` local tiene `server.port=8080`, el servicio arranca en el puerto 9090.

### `server.overrides` — propiedades que ningún cliente puede sobreescribir

Existe una categoría de propiedades que el Config Server puede declarar en su propia configuración bajo `spring.cloud.config.server.overrides`. Estas propiedades llegan a todos los clientes con la prioridad máxima posible: **no pueden sobreescribirse con `override-none=true`, ni con variables de entorno, ni con argumentos de sistema**. El cliente las recibe pero las trata como si tuvieran una prioridad mayor que cualquier fuente local.

```yaml
# Esto NO sobreescribirá una propiedad declarada en server.overrides del Config Server
spring:
  cloud:
    config:
      override-none: true          # solo afecta a propiedades normales del repo Git
      allow-override: true         # idem — no afecta a server.overrides
      override-system-properties: false  # idem
```

> Si una propiedad tiene un valor inesperado y no varía aunque se configure `override-none=true`, probablemente está declarada en `server.overrides` del Config Server. Verificar con `/actuator/env/{propiedad}` — la fuente aparecerá como `"configService:overrides"`.

---

### Autenticación al Config Server (si tiene seguridad)

```yaml
spring:
  cloud:
    config:
      uri: http://localhost:8888
      username: config-user
      password: ${CONFIG_SERVER_PASSWORD}
```

---

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
    // @ConfigurationProperties(prefix = "pedidos")
}
```

---

### Refresco automático por polling (`spring.cloud.config.watch`)

Alternativa a `@RefreshScope` + llamada manual o Spring Cloud Bus: el cliente consulta periódicamente al Config Server si hay cambios y, si los hay, refresca automáticamente los beans `@RefreshScope`:

```yaml
spring:
  cloud:
    config:
      watch:
        enabled: true           # defecto: false — activa el polling automático
        initial-delay: 180000   # ms antes del primer sondeo tras arrancar (defecto: 3 min)
                                # dar tiempo al sistema a estabilizarse antes del primer poll
        delay: 30000            # ms entre sondeos sucesivos (defecto: 30 seg)
```

> **Diferencia con Spring Cloud Bus:** el watch funciona por polling (el cliente pregunta al servidor periódicamente). Bus funciona por eventos push (el servidor notifica a los clientes). Watch es más simple de configurar (no necesita Kafka/RabbitMQ) pero genera carga constante de peticiones al Config Server.

---

### Debug logging — diagnosticar problemas de configuración

Cuando una propiedad tiene un valor inesperado o el cliente no conecta al Config Server, activar el logging de debug es el primer paso:

```yaml
logging:
  level:
    org.springframework.cloud.config: DEBUG
    org.springframework.cloud.bootstrap: DEBUG
    # Muestra en los logs:
    # - URL exacta del Config Server que se consulta
    # - Nombre de la aplicación y perfil usados en la búsqueda
    # - Ficheros recibidos del servidor y las propiedades de cada uno
    # - Qué fuente "gana" cuando hay conflicto de propiedades
```

Ejemplo de salida con DEBUG activo:
```
Fetching config from server at: http://localhost:8888
Located environment: name=pedidos-service, profiles=[dev], label=main
Located property source: CompositePropertySource [
  name='configService',
  propertySources=[
    MapPropertySource {name='configserver:file:/tmp/config-repo/pedidos-service-dev.yml'},
    MapPropertySource {name='configserver:file:/tmp/config-repo/pedidos-service.yml'},
    MapPropertySource {name='configserver:file:/tmp/config-repo/application-dev.yml'},
    MapPropertySource {name='configserver:file:/tmp/config-repo/application.yml'}
  ]
]
```

> Combinado con `/actuator/env/{propiedad}` (ver `03-03-config-server.md`), el debug logging permite identificar exactamente qué fichero está ganando para cada propiedad.

---

### Paso 4: Configuración de reintentos (recomendado en producción)

Si el Config Server no está disponible al arrancar, el servicio falla. Se puede configurar reintentos para que espere a que el Config Server esté listo:

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
      fail-fast: true       # REQUERIDO para que los reintentos funcionen
                            # sin fail-fast=true, la configuración de retry se ignora
      retry:
        max-attempts: 6           # número total de intentos (incluye el primero)
        initial-interval: 1000    # ms de espera antes del primer reintento
        multiplier: 1.5           # factor de incremento entre reintentos (backoff exponencial)
                                  # ej: 1000ms → 1500ms → 2250ms → ...
        max-interval: 10000       # ms máximo entre reintentos (techo del backoff)
```

> **[ADVERTENCIA]** `retry` solo funciona si `fail-fast: true` está activo. Sin él, el cliente falla inmediatamente sin reintentar, independientemente de la configuración de `retry`.

---

### Bootstrap context — compatibilidad con Spring Boot 2.x (`bootstrap.yml`)

> **[LEGACY]** El **Bootstrap Context** y `bootstrap.yml` son el mecanismo de Spring Boot 2.x para cargar configuración remota antes del contexto principal. En Spring Boot 3.x están reemplazados por `spring.config.import`. Solo añadir `spring-cloud-starter-bootstrap` si se mantiene código legado de Spring Boot 2.x.

En Spring Boot 2.x, el Config Client usaba un **Bootstrap Context**: una fase de arranque previa al contexto principal donde se cargaba `bootstrap.yml` y se consultaba al Config Server antes de que Spring Boot inicializara ningún otro bean. Este mecanismo fue reemplazado en Spring Boot 3.x por `spring.config.import`.

Si se trabaja con código legacy que usa `bootstrap.yml`, o se migra gradualmente un proyecto de Spring Boot 2.x a 3.x, se puede reactivar el Bootstrap Context añadiendo la dependencia:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

Con esta dependencia, `bootstrap.yml` vuelve a funcionar:

```yaml
# bootstrap.yml (solo con spring-cloud-starter-bootstrap activo)
spring:
  application:
    name: pedidos-service
  cloud:
    config:
      uri: http://localhost:8888    # se usa uri: en lugar de spring.config.import
      fail-fast: true
```

| | Spring Boot 2.x | Spring Boot 3.x |
|---|---|---|
| **Fichero de config** | `bootstrap.yml` | `application.yml` |
| **Propiedad** | `spring.cloud.config.uri` | `spring.config.import` |
| **Dependencia** | incluida por defecto | requiere `spring-cloud-starter-bootstrap` si se quiere `bootstrap.yml` |

> En proyectos nuevos con Spring Boot 3.x, usar siempre `spring.config.import`. El Bootstrap Context es legacy — soportado para compatibilidad pero no recomendado en proyectos nuevos.

---

## 3.6 Perfiles y entornos

Spring Cloud Config y los perfiles de Spring trabajan juntos para manejar múltiples entornos.

### Estructura de ficheros en el repo Git

Los siguientes son **ficheros distintos** en el repositorio:

**Fichero: `application.yml`** (base global, todos los servicios)
```yaml
logging:
  level:
    root: INFO
```

**Fichero: `application-dev.yml`** (sobreescribe para dev)
```yaml
logging:
  level:
    root: DEBUG
    com.miempresa: DEBUG
```

**Fichero: `pedidos-service.yml`** (específico del servicio)
```yaml
pedidos:
  max-items: 100
  timeout-ms: 5000
```

**Fichero: `pedidos-service-prod.yml`** (producción del servicio)
```yaml
pedidos:
  max-items: 1000
  timeout-ms: 2000

spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/pedidos
    username: ${DB_USER}      # variable de entorno, no hardcodeada
    password: ${DB_PASSWORD}
```

---

### Activar perfil en el cliente

```yaml
# application.yml del microservicio
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}  # dev por defecto, override con variable de entorno
```

### En Docker / Kubernetes

```bash
# Variable de entorno en contenedor
SPRING_PROFILES_ACTIVE=prod
```

### Múltiples perfiles simultáneos

Spring permite activar varios perfiles a la vez. El Config Server los gestiona haciendo una petición que fusiona los ficheros de todos los perfiles activos:

```yaml
spring:
  profiles:
    active: dev,cloud   # dos perfiles activos simultáneamente
```

El Config Server buscará y fusionará **en este orden de prioridad**:
```
pedidos-service-dev.yml
pedidos-service-cloud.yml
pedidos-service.yml
application-dev.yml
application-cloud.yml
application.yml
```

Útil para separar configuración por dimensiones ortogonales: `dev` para el entorno de desarrollo y `cloud` para la infraestructura cloud, aplicando ambas a la vez.

---

### `@ConfigurationProperties` y refresco

`@ConfigurationProperties` **no se refresca automáticamente** tras llamar a `/actuator/refresh` a menos que se anote explícitamente con `@RefreshScope`:

```java
// SIN @RefreshScope — los valores no se actualizan tras el refresco
@ConfigurationProperties(prefix = "pedidos")
@Component
public class PedidosProperties {
    private int maxItems;      // este valor NO cambia aunque se llame a /actuator/refresh
    private int timeoutMs;
}
```

```java
// CON @RefreshScope — los valores sí se actualizan
@ConfigurationProperties(prefix = "pedidos")
@RefreshScope
@Component
public class PedidosProperties {
    private int maxItems;      // se actualiza tras /actuator/refresh
    private int timeoutMs;
}
```

```java
// Alternativa: definir el bean con @RefreshScope en una clase @Configuration
@Configuration
public class PedidosConfig {

    @Bean
    @RefreshScope
    @ConfigurationProperties(prefix = "pedidos")
    public PedidosProperties pedidosProperties() {
        return new PedidosProperties();
    }
}
```

> **[ADVERTENCIA]** Este es uno de los errores más frecuentes: se llama a `/actuator/refresh`, el endpoint responde con la lista de propiedades cambiadas, pero los beans `@ConfigurationProperties` siguen usando los valores antiguos porque les falta `@RefreshScope`.

---

---

## 3.6.1 Testing del Config Client

En tests, el Config Client intenta conectar al Config Server al arrancar el contexto. Sin estrategia explícita, los tests fallan con `Connection refused` si no hay servidor disponible.

### Estrategia 1: Deshabilitar completamente el Config Client

La opción más simple y la más recomendada para tests unitarios y de slice (`@WebMvcTest`, `@DataJpaTest`). El contexto arranca solo con la configuración local.

```yaml
# src/test/resources/application.yml
spring:
  cloud:
    config:
      enabled: false
```

```java
@WebMvcTest(PedidosController.class)
// No necesita @ActiveProfiles — application.yml de test se carga automáticamente
class PedidosControllerTest {
    // el contexto arranca sin intentar conectar al Config Server
}
```

---

### Estrategia 2: Config Server como fuente opcional en tests de integración

Cuando el test levanta un contexto de Spring Boot completo y puede que haya un Config Server disponible o no:

```yaml
# src/test/resources/application.yml
spring:
  config:
    import: "optional:configserver:http://localhost:8888"
    # Si no hay servidor → usa la configuración local del test
    # Si hay servidor → la mezcla (util en tests de integración en CI con infraestructura real)
```

---

### Estrategia 3: Sobreescribir propiedades específicas en el test

Cuando el Config Client está activo pero se necesita controlar el valor de propiedades concretas sin modificar el repositorio Git:

```java
@SpringBootTest
@TestPropertySource(properties = {
    "pedidos.max-items=5",        // sobreescribe la propiedad remota para este test
    "feature.nueva-ui=true"
})
class PedidosIntegrationTest {
    // Las propiedades de @TestPropertySource tienen la prioridad más alta,
    // por encima del Config Server y del application.yml local
}
```

```java
// Alternativa con @SpringBootTest(properties = ...)
@SpringBootTest(properties = "pedidos.max-items=5")
class PedidosLimitTest { }
```

> **[ADVERTENCIA]** `@TestPropertySource` y `@SpringBootTest(properties)` tienen prioridad sobre el Config Server, pero NO sobre `server.overrides` del Config Server. Si una propiedad tiene un valor inesperado en el test a pesar del override, verificar si está declarada en `server.overrides`.

---

### Estrategia 4: Verificar que `@ConfigurationProperties` recibe los valores del servidor

```java
@SpringBootTest
@ActiveProfiles("test")
class ConfigPropertiesTest {

    @Autowired
    private PedidosProperties pedidosProperties;   // @ConfigurationProperties(prefix = "pedidos")

    @Test
    void propiedadesSeCarganDesdeConfigServer() {
        // Verificar que el Config Client cargó los valores correctos
        assertThat(pedidosProperties.getMaxItems()).isEqualTo(100);
        assertThat(pedidosProperties.getTimeoutMs()).isGreaterThan(0);
    }
}
```

```yaml
# src/test/resources/application-test.yml
spring:
  config:
    import: "optional:configserver:http://localhost:8888"
  cloud:
    config:
      enabled: false   # deshabilita si el test no necesita el servidor real
```

---

### Resumen: qué estrategia usar

| Caso | Estrategia |
|---|---|
| Tests unitarios / `@WebMvcTest` / `@DataJpaTest` | `enabled: false` |
| Tests de integración sin infraestructura | `enabled: false` |
| Tests de integración en CI con Config Server real | `optional:configserver:` |
| Controlar valor de una propiedad concreta | `@TestPropertySource` / `@SpringBootTest(properties)` |
| Verificar carga correcta de `@ConfigurationProperties` | `@SpringBootTest` + `@Autowired` de la clase de props |

---

→ **Extensión programática:** [03-04-extension-programatica.md](./03-04-extension-programatica.md) — `PropertySourceLocator`, `ConfigServicePropertySourceLocator`, `BootstrapConfiguration`

← [Config Server](./03-03-config-server.md) | [Volver al índice](./README.md) | Siguiente: [Refresco →](./03-05-config-refresh.md)
