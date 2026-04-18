# 1.1 Concepto, arquitectura y configuración base del Config Server

← [Índice](README.md) | [Índice](README.md) | [1.2 Config Client](sc-config-client.md) →

---

## Introducción

Spring Cloud Config Server resuelve uno de los problemas fundamentales de las arquitecturas de microservicios: la dispersión de configuración. Cuando decenas de servicios tienen sus propios ficheros `application.properties` empaquetados dentro del JAR, cambiar una URL de base de datos o una clave de API requiere recompilar y redesplegar cada servicio afectado. Config Server centraliza toda la configuración en un repositorio externo (Git, filesystem, Vault, JDBC) y la sirve a través de una API REST, permitiendo cambios de configuración sin tocar el código ni reiniciar —opcionalmente— los servicios.

> [CONCEPTO] Spring Cloud Config Server es un servidor HTTP que actúa como proxy entre los microservicios y un repositorio de configuración centralizado. Cada microservicio (Config Client) pregunta al servidor por su configuración al arrancar, usando la triple tupla `{application}/{profile}/{label}`.

## Arquitectura del sistema

El flujo de resolución de configuración sigue un camino bien definido: el Config Client arranca, construye la URL de petición usando su nombre de aplicación, perfil activo y etiqueta (rama), y llama al Config Server. El servidor resuelve la configuración desde el backend (Git por defecto), fusiona los ficheros relevantes y devuelve un documento de propiedades al cliente.

```
┌──────────────────────────────────────────────────────────────────┐
│                      ARQUITECTURA CONFIG                          │
│                                                                   │
│  ┌─────────────┐    GET /{app}/{profile}/{label}  ┌───────────┐  │
│  │ Config      │ ─────────────────────────────→  │  Config   │  │
│  │ Client      │ ←─────────────────────────────  │  Server   │  │
│  │ (microsvs)  │     PropertySource[]             │           │  │
│  └─────────────┘                                  └─────┬─────┘  │
│                                                         │         │
│                                           ┌─────────────▼──────┐ │
│                                           │  Backend (Git/FS/  │ │
│                                           │  Vault/JDBC)       │ │
│                                           └────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

La triple tupla `{application}/{profile}/{label}` determina exactamente qué ficheros de configuración entrega el servidor:

- **application**: el valor de `spring.application.name` del cliente (ej. `order-service`)
- **profile**: el perfil activo del cliente (ej. `dev`, `prod`, `default`)
- **label**: la rama, etiqueta o commit del repositorio Git (ej. `main`, `v2.1.0`)

El servidor construye una lista de PropertySources por prioridad decreciente: primero `{application}-{profile}.yml`, luego `{application}.yml`, luego `application-{profile}.yml`, finalmente `application.yml`. Las claves del primer fichero tienen mayor precedencia.

> [EXAMEN] La triple tupla `{application}/{profile}/{label}` es la pregunta más frecuente sobre Config Server. `application` = nombre del servicio cliente, `profile` = entorno, `label` = rama Git.

## Ejemplo central

El siguiente ejemplo muestra un Config Server completo y operativo con backend Git. Incluye la clase principal, las dependencias Maven y la configuración mínima.

**pom.xml (dependencias)**:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2025.0.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
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

**application.yml (Config Server)**:

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          clone-on-start: true
          search-paths: '{application}'
```

Con esta configuración, si un cliente llamado `order-service` con perfil `prod` arranca, el servidor responderá a `GET /order-service/prod/main` buscando ficheros en la carpeta `order-service/` del repositorio.

> [ADVERTENCIA] `@EnableConfigServer` es OBLIGATORIO aunque el starter esté en el classpath. Sin esta anotación, el servidor no expone sus endpoints de configuración y arranca como una aplicación web vacía.

## Tabla de propiedades clave del servidor

Las siguientes propiedades controlan el comportamiento del servidor y son evaluadas directamente en el examen VMware Spring Professional.

| Propiedad | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `spring.cloud.config.server.git.uri` | String | — | URI del repositorio Git (obligatoria con backend git) |
| `spring.cloud.config.server.git.default-label` | String | `master` | Rama por defecto si el cliente no especifica label |
| `spring.cloud.config.server.git.clone-on-start` | boolean | `false` | Clona el repo al arrancar en lugar de esperar la primera petición |
| `spring.cloud.config.server.git.search-paths` | String[] | `[]` | Subdirectorios a buscar; acepta placeholder `{application}` |
| `spring.cloud.config.server.git.force-pull` | boolean | `false` | Descarta cambios locales y fuerza pull del remoto |

> [EXAMEN] El valor por defecto de `default-label` es `master`, no `main`. En repositorios nuevos de GitHub/GitLab la rama principal es `main`, por lo que se debe cambiar explícitamente esta propiedad.

## Buenas y malas prácticas

Hacer:
- Añadir `@EnableConfigServer` explícitamente aunque el starter esté en el classpath, ya que documenta la intención y es obligatorio para activar el servidor.
- Configurar `clone-on-start: true` en producción para detectar errores de acceso al repositorio en el arranque y no en la primera petición de un cliente.
- Usar un puerto dedicado (8888 es el convencional) para el Config Server, separado del puerto de los microservicios de negocio.
- Versionar el repositorio de configuración igual que el código: ramas por entorno o por release del servicio.

Evitar:
- Empaquetar el Config Server junto a otro microservicio de negocio; si ese servicio falla, toda la configuración centralizada desaparece.
- Usar el perfil `native` (filesystem local) en producción; es exclusivo para desarrollo y testing, no tiene persistencia ni versionado.
- Confundir `spring.application.name` del Config Server con el del cliente; el nombre del servidor solo afecta cómo se registra en Eureka, no a la resolución de configuración.
- Exponer el Config Server sin seguridad en entornos no aislados; cualquier cliente puede leer todas las propiedades de todos los servicios.

## Comparación: Config Server vs configuración local

La siguiente tabla compara el enfoque centralizado (Config Server) con la configuración local empaquetada en el JAR, para entender cuándo cada enfoque es apropiado.

| Criterio | Config Server | Local (application.yml en JAR) |
|----------|---------------|-------------------------------|
| Cambio sin redeploy | Sí (con @RefreshScope) | No (requiere rebuild y redeploy) |
| Versionado | Git nativo | Solo si el repo de código lo incluye |
| Secretos cifrados | Sí ({cipher}prefix) | No (texto plano o Vault separado) |
| Disponibilidad | Depende del servidor externo | Siempre disponible (embebido) |
| Complejidad operacional | Alta (servidor adicional) | Baja |
| Recomendado para | Producción con múltiples servicios | Proyectos pequeños o monolitos |

## Verificación y práctica

Para verificar que el Config Server está operativo, la forma más directa es hacer una petición a su endpoint de resolución con valores de prueba:

```bash
# Verificar estado del servidor
curl http://localhost:8888/actuator/health

# Solicitar configuración de "myapp" en perfil "default", rama "main"
curl http://localhost:8888/myapp/default/main

# El servidor debe responder con JSON conteniendo "propertySources"
# Si responde con {"propertySources":[]} el repositorio está vacío o la ruta es incorrecta
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Cuál es la anotación obligatoria para activar el Config Server? ¿Es suficiente tener el starter en el classpath?
2. La URL de resolución del Config Server es `/{application}/{profile}/{label}`. ¿Qué representa cada segmento? ¿Cuál es el valor de `label` por defecto si el cliente no lo especifica?
3. Un microservicio `payment-service` con perfil `prod` solicita configuración al Config Server. ¿En qué orden el servidor fusiona los PropertySources devueltos?
4. ¿Qué propiedad del cliente determina el valor de `{application}` en la URL de resolución?
5. ¿Por qué el Config Server necesita `spring-cloud-config-server` y no solo `spring-cloud-starter-config`?

---

← [Índice](README.md) | [Índice](README.md) | [1.2 Config Client](sc-config-client.md) →

