# 1.2 Config Client — configuración y arranque

← [1.1 Arquitectura y concepto del Config Server](sc-config-arquitectura.md) | [Índice (README.md)](README.md) | [1.3 Resolución de configuración por perfiles y labels →](sc-config-resolucion.md)

---

## Introducción

Un microservicio se convierte en Config Client en el momento en que necesita obtener su configuración del Config Server en lugar de leerla de un fichero local. La pregunta no es si hacerlo —en un ecosistema con Config Server es el comportamiento estándar— sino cómo configurar el cliente correctamente para que la importación ocurra en el momento preciso del ciclo de vida de Spring Boot.

El reto técnico está en el **orden de inicialización**: las propiedades deben estar disponibles antes de que cualquier bean se cree, lo que obliga a que la importación del Config Server ocurra durante la fase de carga del `Environment`, antes del refresh del contexto. En Spring Boot 4.0.x, este mecanismo se implementa mediante `spring.config.import`. El modo anterior (bootstrap context) existía para el mismo fin pero con una arquitectura más compleja que está desaconsejada en las versiones actuales.

> [ADVERTENCIA] En Spring Cloud 2025.1.1 con Spring Boot 4.0.x, el bootstrap context está **desaconsejado**. El modo canónico es `spring.config.import=configserver:`. El bootstrap context se documenta aquí únicamente como referencia de migración de proyectos legacy.

> [PREREQUISITO] El Config Server debe estar levantado y accesible en la URL configurada antes de arrancar el cliente. Si no está disponible y `fail-fast=true`, el cliente fallará al iniciar. Si `fail-fast=false` (default), el cliente arrancará con un contexto de propiedades vacío, lo que puede causar errores en runtime difíciles de diagnosticar.

## Diagrama: ciclo de carga de configuración en Spring Boot 4.0.x

El siguiente diagrama muestra en qué fase del arranque de Spring Boot interviene el Config Client y por qué debe ocurrir antes que la creación de beans.

```
ARRANQUE Spring Boot 4.0.x
        │
        ▼
1. Carga application.properties / application.yml local
        │
        ▼
2. Procesa spring.config.import=configserver:http://config-server:8888
        │
        │  ──► GET /order-service/prod/main  (al Config Server)
        │  ◄── PropertySource[] (JSON)
        │
        ▼
3. Environment completo (local + remoto, por precedencia)
        │
        ▼
4. Creación del ApplicationContext
   - Creación de beans (@Component, @Service, @RestController…)
   - Inyección de @Value, @ConfigurationProperties   ← ya disponibles
        │
        ▼
5. Aplicación lista para recibir peticiones
```

Si el import falla en el paso 2, el contexto no llega al paso 4: el fallo es explícito y temprano.

## Ejemplo central

A continuación se muestra la configuración completa de un Config Client en Spring Boot 4.0.x, incluyendo dependencias, propiedades y gestión de fallos.

### pom.xml (fragmento de dependencias)

```xml
<dependencies>
    <!-- Config Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <!-- Actuator (necesario para /actuator/refresh) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Retry (necesario para spring.cloud.config.retry.*) -->
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>
    <!-- AOP necesario para que spring-retry funcione -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2025.1.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### application.yml (Config Client — modo moderno)

```yaml
spring:
  application:
    name: order-service          # se usa como {application} en la resolución del servidor

  config:
    import: "configserver:http://config-server:8888"   # modo canónico Spring Boot 4.x

  cloud:
    config:
      profile: ${spring.profiles.active:default}       # se puede omitir; usa el perfil activo
      label: main                                       # rama/tag/commit del Git backend
      fail-fast: true                                   # falla al arrancar si el servidor no responde
      retry:
        max-attempts: 6          # reintentos antes de fallar definitivamente
        initial-interval: 1000   # ms entre el primer y segundo intento
        multiplier: 1.5          # factor de backoff exponencial
        max-interval: 2000       # techo de espera entre intentos (ms)
      # Autenticación básica al Config Server (si el servidor la requiere):
      username: ${CONFIG_SERVER_USER:config}
      password: ${CONFIG_SERVER_PASSWORD:secret}

management:
  endpoints:
    web:
      exposure:
        include: health,refresh
```

### Consumo de propiedades con @ConfigurationProperties

```java
package com.example.orderservice.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;

// @RefreshScope aquí permite recargar las propiedades sin reiniciar
// Ver sección 1.6 para el mecanismo de refresco completo
@RefreshScope
@ConfigurationProperties(prefix = "order")
public record OrderProperties(
        String paymentServiceUrl,
        int maxRetries,
        boolean asyncProcessing
) {}
```

```java
package com.example.orderservice;

import com.example.orderservice.config.OrderProperties;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

@SpringBootApplication
@EnableConfigurationProperties(OrderProperties.class)
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

El fichero `order-service.yml` en el repositorio Git del Config Server contendría:

```yaml
order:
  payment-service-url: http://payment-service:8080
  max-retries: 3
  async-processing: false
```

## Tabla de parámetros del Config Client

La siguiente tabla lista los parámetros que un profesional senior debe conocer de memoria, con sus valores por defecto y su efecto real.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.config.import` | `String` | — | URL del Config Server con prefijo `configserver:`. Obligatorio en modo moderno. |
| `spring.application.name` | `String` | `application` | Nombre con el que el cliente se identifica ante el servidor (`{application}`). |
| `spring.cloud.config.uri` | `String` | `http://localhost:8888` | URL del servidor. Alternativa a incluirla en `spring.config.import`. |
| `spring.cloud.config.profile` | `String` | perfil activo | Perfil a resolver (`{profile}`). Por defecto toma `spring.profiles.active`. |
| `spring.cloud.config.label` | `String` | (del servidor) | Rama/tag/commit del backend Git. |
| `spring.cloud.config.fail-fast` | `boolean` | `false` | Si `true`, falla al arrancar si el servidor no responde. **Recomendado `true` en producción.** |
| `spring.cloud.config.retry.max-attempts` | `int` | `6` | Número de reintentos antes de fallar. Requiere `spring-retry` en el classpath. |
| `spring.cloud.config.retry.initial-interval` | `long` | `1000` | Milisegundos de espera antes del primer reintento. |
| `spring.cloud.config.retry.multiplier` | `double` | `1.1` | Factor de backoff exponencial entre reintentos. |
| `spring.cloud.config.retry.max-interval` | `long` | `2000` | Techo de espera entre reintentos (ms). |
| `spring.cloud.config.username` | `String` | — | Usuario para HTTP Basic auth contra el Config Server. |
| `spring.cloud.config.password` | `String` | — | Contraseña para HTTP Basic auth. |
| `spring.cloud.config.token` | `String` | — | Token de Vault que el cliente pasa al servidor para resolver secretos de Vault. |

## Buenas y malas prácticas

**Hacer:**
- Establecer `fail-fast: true` en todos los entornos no locales. Un cliente que arranca sin configuración del servidor produce errores en runtime que son mucho más difíciles de diagnosticar que un fallo explícito en el arranque.
- Combinar `fail-fast: true` con `retry.*`: el cliente reintenta la conexión N veces con backoff antes de fallar, lo que tolera reinicios temporales del Config Server sin que el cliente falle de inmediato.
- Externalizar las credenciales del Config Server (`username`, `password`) en variables de entorno, no en `application.yml`: son las únicas propiedades que no pueden venir del propio Config Server.
- Usar `@ConfigurationProperties` (record o clase) en lugar de `@Value` para grupos de propiedades relacionadas: es más seguro frente a typos, permite validación con `@Validated` y funciona mejor con `@RefreshScope`.

**Evitar:**
- Poner `spring.config.import` en un fichero que a su vez viene del Config Server: se produce un ciclo de bootstrap imposible de resolver. La propiedad `spring.config.import` debe estar en el fichero local `application.yml` o como variable de entorno.
- Usar `spring.cloud.config.uri` y `spring.config.import` simultáneamente apuntando a URLs distintas: el comportamiento es confuso y puede resultar en resoluciones duplicadas o contradictorias.
- Omitir `spring.application.name`: sin él, el cliente solicita al servidor la aplicación llamada `application`, que solo devuelve la configuración global compartida, no la específica del servicio.

## Comparación: spring.config.import vs bootstrap context

La siguiente tabla compara el modo moderno (canónico en Spring Boot 4.0.x) con el modo legacy basado en bootstrap context.

| Aspecto | `spring.config.import` (moderno) | Bootstrap context [LEGACY] |
|---|---|---|
| Disponibilidad | Spring Boot 2.4+ / Spring Cloud 2020.0+ | Todas las versiones anteriores |
| Dependencia extra | No (incluida en `spring-cloud-starter-config`) | Requiere `spring-cloud-starter-bootstrap` |
| Orden de carga | Integrado en el mecanismo estándar de `ConfigDataLoader` | Contexto hijo separado, cargado antes del contexto principal |
| Propiedades de config | En `application.yml` bajo `spring.config.import` | En `bootstrap.yml` / `bootstrap.properties` |
| Depuración | Más predecible con `--debug` de Spring Boot | Contexto extra dificulta el trazado de resolución |
| Estado en 2025.1.1 | **Recomendado** | Desaconsejado; mantener solo en proyectos sin migrar |

> [ADVERTENCIA] Si coexisten `bootstrap.yml` y `spring.config.import` en el mismo proyecto (situación habitual durante una migración), las propiedades del bootstrap context tienen precedencia sobre las del import. Esto puede causar que propiedades del servidor sean silenciosamente sobreescritas por valores del bootstrap. Completar la migración antes de pasar a producción.

---

← [1.1 Arquitectura y concepto del Config Server](sc-config-arquitectura.md) | [Índice (README.md)](README.md) | [1.3 Resolución de configuración por perfiles y labels →](sc-config-resolucion.md)
