# 1.1 Arquitectura y concepto del Config Server

← [Índice (README.md)](README.md) | [1.2 Config Client — configuración y arranque →](sc-config-client.md)

---

## Introducción

Cuando un sistema de microservicios crece, cada servicio lleva su propia configuración embebida en el JAR o en un `application.yml` local. El problema aparece en producción: cambiar un parámetro —un timeout, una URL de base de datos, un flag de feature— obliga a recompilar y redesplegar. Con veinte servicios y cuatro entornos (dev, test, staging, prod) la situación se vuelve inmanejable.

Spring Cloud Config resuelve este problema mediante la **externalización de configuración**: un servidor centralizado (Config Server) almacena todas las propiedades en un backend externo (Git, filesystem, Vault…) y las sirve bajo demanda vía HTTP a cualquier microservicio (Config Client). El cliente arranca, llama al servidor, recibe sus propiedades y las inyecta en el contexto de Spring antes de que ningún bean se inicialice.

Sin Config Server, un cambio de configuración implica redeploy. Con Config Server, implica un `POST /actuator/refresh` —o nada si se usa Spring Cloud Bus con refresco automático.

> [CONCEPTO] Config Server no es un servicio de configuración de la aplicación: es la **fuente de verdad** de configuración del ecosistema de microservicios. Su disponibilidad es un prerequisito para que cualquier otro servicio arranque correctamente.

## Diagrama: flujo de resolución cliente → servidor → backend

El siguiente diagrama muestra las tres capas del sistema y el orden en que se produce la resolución de configuración durante el arranque de un microservicio.

```
┌─────────────────────────────────────────────────────────┐
│                   MICROSERVICIO (Config Client)          │
│                                                         │
│  1. Lee spring.config.import=configserver:              │
│     http://config-server:8888                           │
│  2. GET /order-service/prod/main                        │
└──────────────────────┬──────────────────────────────────┘
                       │  HTTP (puede ser HTTPS + Basic Auth)
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   CONFIG SERVER                         │
│                                                         │
│  @EnableConfigServer                                    │
│  EnvironmentRepository (Git / native / Vault / ...)     │
│                                                         │
│  3. Resuelve: application + profile + label             │
│  4. Devuelve PropertySource[] (JSON)                    │
└──────────────────────┬──────────────────────────────────┘
                       │  protocolo del backend
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   BACKEND DE CONFIGURACIÓN              │
│                                                         │
│  Git remoto / filesystem local / HashiCorp Vault /      │
│  JDBC / AWS S3 / Composite                              │
└─────────────────────────────────────────────────────────┘
```

Los pasos ocurren **antes** de que Spring cree cualquier bean: el bootstrap de configuración es la primera fase del ciclo de vida del contexto.

## Ejemplo central

A continuación se muestra un Config Server mínimo y funcional con backend Git, incluyendo dependencias Maven y configuración completa.

### pom.xml (Config Server)

```xml
<dependencies>
    <!-- Spring Cloud Config Server -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <!-- Actuator para health y métricas -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
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

### ConfigServerApplication.java

```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer          // activa el EnvironmentRepository y los endpoints HTTP
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### application.yml (Config Server)

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
          uri: https://github.com/mi-org/mi-config-repo
          default-label: main
          clone-on-start: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,env,refresh
```

Con esta configuración, el servidor responde peticiones del tipo:

```
GET http://localhost:8888/order-service/prod
GET http://localhost:8888/order-service/prod/main
```

y devuelve un JSON con la lista de `PropertySource` que el cliente inyecta en su contexto.

> [PREREQUISITO] El repositorio Git configurado en `spring.cloud.config.server.git.uri` debe ser accesible desde la red del Config Server. Si es privado, configura autenticación SSH o HTTPS antes de arrancar (ver sección 1.4).

## Tabla de componentes

La siguiente tabla resume los cuatro componentes principales del ecosistema y su responsabilidad.

| Componente | Clase / Anotación clave | Responsabilidad |
|---|---|---|
| Config Server | `@EnableConfigServer` | Servir configuración vía HTTP a los clientes |
| EnvironmentRepository | `JGitEnvironmentRepository` (default) | Acceder al backend y construir el `Environment` |
| Config Client | `spring-cloud-starter-config` | Importar configuración del servidor al arrancar |
| Bootstrap / Import | `spring.config.import=configserver:` | Punto de entrada del cliente (modo moderno) |

La separación entre **servidor** y **EnvironmentRepository** es el punto de extensión clave: implementar `EnvironmentRepository` es suficiente para conectar cualquier backend personalizado sin tocar la capa HTTP.

## Buenas y malas prácticas

**Hacer:**
- Activar `clone-on-start: true` en el backend Git para que el servidor valide la conexión al arrancar y no falle en la primera petición de un cliente.
- Exponer únicamente los endpoints Actuator necesarios (`health`, `env`, `refresh`) y proteger el servidor con HTTP Basic o mTLS desde el día uno: el Config Server tiene acceso a **todos** los secretos del ecosistema.
- Desplegarlo como servicio dedicado con su propio pipeline, sin colocarlo como módulo dentro de un microservicio de negocio.

**Evitar:**
- Usar el Config Server como único punto de configuración sin `fail-fast: true` en los clientes: si el servidor cae, los clientes arrancan con un contexto vacío y producen `NullPointerException` en runtime, en lugar de fallar de forma rápida y evidente.
- Almacenar secretos en texto plano en el repositorio Git del Config Server. Usar cifrado (`{cipher}`) o delegar en HashiCorp Vault.
- Olvidar configurar `default-label` explícitamente: en entornos con ramas `main` vs `master` el comportamiento por defecto varía entre versiones y provoca resoluciones silenciosamente incorrectas.

## Comparación: @EnableConfigServer vs configuración sin Config Server

La siguiente tabla contrasta el enfoque centralizado frente al modelo embebido que sustituye.

| Aspecto | Configuración embebida (sin Config Server) | Spring Cloud Config Server |
|---|---|---|
| Cambio de propiedad | Requiere rebuild + redeploy | `POST /actuator/refresh` o bus |
| Secretos | En el artefacto o variables de entorno por servicio | Cifrados en un único repositorio auditado |
| Auditoría de cambios | Difícil, distribuida | Historial Git completo |
| Consistencia entre instancias | No garantizada | Un solo origen de verdad |
| Complejidad operativa | Baja inicialmente | Añade un servicio a operar |
| Disponibilidad como prerequisito | No aplica | Crítico: si cae, los clientes no arrancan |

> [EXAMEN] La pregunta más frecuente en entrevista sobre Config Server es: "¿Qué pasa si el Config Server no está disponible cuando arranca un microservicio?" La respuesta depende de `spring.cloud.config.fail-fast` y `retry.*` en el cliente — no de la configuración del servidor.

---

← [Índice (README.md)](README.md) | [1.2 Config Client — configuración y arranque →](sc-config-client.md)
