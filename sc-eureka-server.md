# 2.2 Configuración y arranque del Eureka Server

← [2.1 Arquitectura y rol de Eureka en el ecosistema de microservicios](sc-eureka-arquitectura.md) | [Índice (README.md)](README.md) | [2.3 Configuración y arranque del Eureka Client →](sc-eureka-cliente.md)

---

El Eureka Server es el componente que centraliza el registro de todas las instancias de microservicios del sistema. Sin él, ningún cliente puede localizar a otro servicio por nombre lógico; cada consumidor tendría que conocer la dirección IP del proveedor en tiempo de despliegue. Levantar el servidor correctamente —con las propiedades adecuadas para evitar que se registre a sí mismo o que intente conectarse a un peer inexistente— es el primer paso operativo de cualquier arquitectura basada en Eureka. Un servidor mal configurado en modo standalone genera bucles de error en los logs cada 30 segundos y puede confundir al equipo de operaciones sobre el estado real del registro.

> [PREREQUISITO] El desarrollador debe tener Spring Boot 4.0.x y el BOM de Spring Cloud 2025.1.1 (Oakwood) configurado en el proyecto antes de añadir la dependencia del servidor.

## Diagrama: arranque del Eureka Server en modo standalone

El siguiente diagrama muestra el flujo de inicialización del servidor y las dos propiedades críticas que impiden los intentos de autorregistro y sincronización con peers inexistentes.

```
Arranque de la aplicación Spring Boot
       │
       ▼
@EnableEurekaServer detectado
       │
       ▼
EurekaServerAutoConfiguration activa
       │
       ├─► eureka.client.register-with-eureka: false
       │       └── El servidor NO intenta registrarse en sí mismo
       │
       ├─► eureka.client.fetch-registry: false
       │       └── El servidor NO intenta descargar un registro externo
       │
       ▼
Eureka Server listo en :8761/eureka
       │
       ├─► Dashboard disponible en http://localhost:8761
       └─► API REST disponible en http://localhost:8761/eureka/apps
```

## Implementación: Eureka Server standalone completo

El siguiente ejemplo es un proyecto Spring Boot 4.0.x completamente funcional como Eureka Server en modo standalone. Incluye dependencias, configuración y clase principal.

**pom.xml (dependencias relevantes)**

```xml
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

<dependencies>
  <!-- Eureka Server -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>

  <!-- Spring Boot Web (necesario para el dashboard y API REST) -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- Actuator para health checks -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

**EurekaServerApplication.java**

```java
package com.ejemplo.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer                    // Activa el registro Eureka
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**application.yml (modo standalone)**

```yaml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    # El servidor NO se registra a sí mismo en el registro
    register-with-eureka: false
    # El servidor NO intenta descargar el registro de otro peer
    fetch-registry: false
    service-url:
      # La URL apunta a sí mismo; no tiene efecto real en standalone
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    # Intervalo de comprobación de instancias expiradas (ms)
    eviction-interval-timer-in-ms: 10000
    # Desactivar self-preservation en desarrollo para que las instancias
    # caídas se eliminen rápidamente del registro
    enable-self-preservation: false

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

> [ADVERTENCIA] En un servidor standalone, si no se configuran `register-with-eureka: false` y `fetch-registry: false`, el servidor intenta registrarse en sí mismo y descargar su propio registro. Esto genera excepciones `com.netflix.discovery.shared.transport.TransportException` repetidas en los logs cada 30 segundos pero no impide el funcionamiento; sin embargo, contamina el log y genera confusión en operaciones.

## Tabla de propiedades clave del Eureka Server

La siguiente tabla recoge las propiedades que un senior debe dominar para configurar y operar el servidor en producción.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `eureka.client.register-with-eureka` | boolean | `true` | Si `false`, el servidor no se registra en el registro (obligatorio en standalone) |
| `eureka.client.fetch-registry` | boolean | `true` | Si `false`, el servidor no descarga el registro (obligatorio en standalone) |
| `eureka.instance.hostname` | String | hostname del SO | Hostname con el que se anuncia el servidor; crucial en entornos multi-red |
| `eureka.client.service-url.defaultZone` | String | `http://localhost:8761/eureka/` | URL del propio servidor (standalone) o de los peers (cluster) |
| `eureka.server.enable-self-preservation` | boolean | `true` | Activa el modo de autopreservación ante pérdida masiva de heartbeats |
| `eureka.server.renewal-percent-threshold` | double | `0.85` | Umbral mínimo de renovaciones para no activar self-preservation |
| `eureka.server.eviction-interval-timer-in-ms` | long | `60000` | Intervalo en ms para el hilo que expulsa instancias sin lease vigente |
| `eureka.server.response-cache-update-interval-ms` | long | `30000` | Frecuencia de actualización de la caché de respuestas del servidor |
| `eureka.server.peer-eureka-nodes-update-interval-ms` | long | `600000` | Intervalo de refresco de la lista de peers en modo cluster |
| `eureka.server.max-threads-for-peer-replication` | int | `20` | Máximo de hilos dedicados a la replicación entre peers |

## Buenas y malas prácticas

Hacer:
- Asignar un `spring.application.name` descriptivo al servidor (por ejemplo `eureka-server`): aparece en el dashboard y en los logs de los clientes al registrarse, facilitando la identificación en sistemas con múltiples registros.
- Desactivar `enable-self-preservation` en entornos de desarrollo (`false`): en local, los servicios se detienen con frecuencia y sin self-preservation el registro limpia las instancias muertas en segundos. En producción, mantenerlo a `true`.
- Exponer el actuator (`/actuator/health`) del servidor para que la plataforma de orquestación (Kubernetes, ECS) pueda detectar su estado y reiniciarlo si falla.
- Reducir `eviction-interval-timer-in-ms` a 5000–10000 ms en entornos de desarrollo para que las instancias caídas desaparezcan rápidamente del registro sin esperar el minuto por defecto.

Evitar:
- Desplegar un único Eureka Server en producción sin alta disponibilidad: si el servidor falla, los clientes siguen funcionando con su caché local, pero ningún nuevo servicio puede registrarse hasta que el servidor se recupere. Ver sección 2.8 para cluster.
- Modificar `renewal-percent-threshold` por debajo de `0.5` en producción: umbrales muy bajos hacen que self-preservation no se active nunca, con el riesgo de que instancias realmente caídas permanezcan en el registro durante un fallo de red parcial.
- Usar el mismo `server.port` para varios servidores Eureka en la misma máquina física sin configurar `eureka.instance.hostname` diferente para cada uno: los peers no se distinguirán y la replicación fallará silenciosamente.

## Comparación: modo standalone vs modo cluster (peer-aware)

La diferencia entre ambos modos no es un parámetro binario sino un conjunto de propiedades que cambian el comportamiento de replicación del servidor.

| Aspecto | Standalone | Cluster (peer-aware) |
|---|---|---|
| `register-with-eureka` en el servidor | `false` | `true` (el servidor se registra con sus peers) |
| `fetch-registry` en el servidor | `false` | `true` (descarga el registro de sus peers) |
| `service-url.defaultZone` | URL del propio servidor | URLs de todos los peers separadas por coma |
| Replicación de registros | No aplica | Automática vía `peerEurekaNodes` |
| Tolerancia a fallo | Ninguna (SPOF) | Si cae un nodo, los otros continúan sirviendo |
| Configuración adicional | Ninguna | Perfiles Spring por instancia o variables de entorno |

El modo cluster se detalla en la sección 2.8. Lo que sí debe quedar claro aquí es que la transición de standalone a cluster no requiere recompilar la aplicación: es un cambio de configuración en `application.yml` o en las variables de entorno del despliegue.

> [EXAMEN] Una pregunta frecuente en entrevistas es: "¿Qué pasa si el Eureka Server cae en producción?". La respuesta correcta tiene dos partes: (1) los clientes existentes continúan funcionando con su caché local del registro durante el tiempo que dure esa caché (típicamente ~90 segundos antes de considerarla stale), y (2) los nuevos servicios que arranquen durante la caída no podrán registrarse y no serán descubiertos por otros hasta que el servidor se recupere.

---

← [2.1 Arquitectura y rol de Eureka en el ecosistema de microservicios](sc-eureka-arquitectura.md) | [Índice (README.md)](README.md) | [2.3 Configuración y arranque del Eureka Client →](sc-eureka-cliente.md)
