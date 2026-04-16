# 1.6 Actualización dinámica de configuración (Refresh)

← [1.5 Backends alternativos al Git backend](sc-config-backends.md) | [Índice (README.md)](README.md) | [1.7 Seguridad del Config Server →](sc-config-security.md)

---

## Introducción

Externalizar la configuración en el Config Server solo resuelve la mitad del problema: el microservicio puede cargar su configuración desde el servidor al arrancar, pero sin un mecanismo de refresco, un cambio en el repositorio Git solo tiene efecto después de un reinicio del microservicio. Esto elimina la ventaja principal de la externalización en producción.

El mecanismo de refresco dinámico de Spring Cloud Config permite actualizar las propiedades de los beans que las consumen **en caliente**, sin reiniciar el proceso JVM. El precio es la complejidad añadida: no todos los beans son refrescables, el refresco tiene un coste de rendimiento, y existen casos edge importantes en los que `@RefreshScope` no funciona como se espera.

> [CONCEPTO] `@RefreshScope` crea un bean en un scope de ciclo de vida especial: cuando se dispara un refresco, el bean se destruye y se recrea con las nuevas propiedades. No es un mecanismo de actualización en tiempo real (push), sino de reinicialización bajo demanda (pull o por evento).

> [PREREQUISITO] El microservicio debe tener `spring-boot-starter-actuator` en el classpath y el endpoint `/actuator/refresh` expuesto (`management.endpoints.web.exposure.include: refresh`).

## Diagrama: flujo de refresco dinámico

El siguiente diagrama muestra la secuencia de eventos que ocurre cuando se dispara un refresco, tanto en el modo manual (POST directo) como con Spring Cloud Bus.

```
MODO MANUAL (un solo microservicio):

Operador / CI/CD
      │
      ▼
POST /actuator/refresh  (al microservicio destino)
      │
      ▼
ContextRefresher.refresh()
      ├── Vuelve a importar propiedades del Config Server
      ├── Detecta propiedades cambiadas
      ├── Publica EnvironmentChangeEvent con la lista de cambios
      └── Destruye y recrea todos los beans marcados con @RefreshScope


MODO BUS (broadcast a todas las instancias):

Operador / Webhook /monitor
      │
      ▼
POST /actuator/busrefresh  (a cualquier instancia)
      │
      ▼  (via Spring Cloud Bus — Kafka o RabbitMQ)
      ├──► microservicio-1: ContextRefresher.refresh()
      ├──► microservicio-2: ContextRefresher.refresh()
      └──► microservicio-N: ContextRefresher.refresh()

Para propagar el refresco a todas las instancias, añade
spring-cloud-starter-bus-kafka (o -amqp) y publica POST /actuator/busrefresh.
Ver módulo Spring Cloud Bus.
```

## Ejemplo central

A continuación se muestra cómo anotar correctamente los beans que deben refrescarse, incluyendo el patrón recomendado con `@ConfigurationProperties`.

### pom.xml (dependencias necesarias para el refresco)

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### application.yml del microservicio

```yaml
spring:
  application:
    name: notification-service
  config:
    import: "configserver:http://config-server:8888"

management:
  endpoints:
    web:
      exposure:
        include: health,refresh     # expone /actuator/refresh
```

### Patrón recomendado: @ConfigurationProperties + @RefreshScope

```java
package com.example.notification.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

// @RefreshScope hace que este bean se destruya y recree cuando se dispara un refresh
// @ConfigurationProperties agrupa las propiedades y permite validación con @Validated
@RefreshScope
@Component
@ConfigurationProperties(prefix = "notification")
public class NotificationProperties {

    private String smtpHost;
    private int smtpPort;
    private String templatePath;
    private boolean debugMode;

    // getters y setters
    public String getSmtpHost() { return smtpHost; }
    public void setSmtpHost(String smtpHost) { this.smtpHost = smtpHost; }

    public int getSmtpPort() { return smtpPort; }
    public void setSmtpPort(int smtpPort) { this.smtpPort = smtpPort; }

    public String getTemplatePath() { return templatePath; }
    public void setTemplatePath(String templatePath) { this.templatePath = templatePath; }

    public boolean isDebugMode() { return debugMode; }
    public void setDebugMode(boolean debugMode) { this.debugMode = debugMode; }
}
```

### Uso en un servicio de negocio

```java
package com.example.notification.service;

import com.example.notification.config.NotificationProperties;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    private final NotificationProperties props;

    public EmailService(NotificationProperties props) {
        this.props = props;
    }

    public void sendEmail(String to, String subject, String body) {
        // props.getSmtpHost() siempre devuelve el valor actualizado
        // porque NotificationProperties está marcado con @RefreshScope
        String host = props.getSmtpHost();
        int port = props.getSmtpPort();
        // ... lógica de envío
    }
}
```

`EmailService` **no** necesita `@RefreshScope` porque inyecta `NotificationProperties` por referencia (proxy de Spring). Cuando `NotificationProperties` se recrea tras un refresh, las llamadas a través del proxy apuntan al nuevo bean.

### ContextRefresher: API programática

```java
package com.example.notification.admin;

import org.springframework.cloud.context.refresh.ContextRefresher;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Set;

@RestController
@RequestMapping("/admin")
public class RefreshController {

    private final ContextRefresher contextRefresher;

    public RefreshController(ContextRefresher contextRefresher) {
        this.contextRefresher = contextRefresher;
    }

    @PostMapping("/refresh")
    public Set<String> refresh() {
        // Devuelve el conjunto de claves de propiedades que han cambiado
        return contextRefresher.refresh();
    }
}
```

`ContextRefresher.refresh()` es equivalente a `POST /actuator/refresh` pero invocable desde código. Útil para tests de integración o lógica de negocio que necesite forzar un refresh en respuesta a un evento interno.

### Disparar el refresco manualmente

```bash
# Dispara el refresco en un microservicio concreto
curl -s -X POST http://notification-service:8080/actuator/refresh

# Respuesta: lista de propiedades que han cambiado
["notification.smtp-host", "notification.debug-mode"]
```

## Tabla de parámetros y conceptos del refresco

La siguiente tabla resume los elementos clave del mecanismo de refresco.

| Elemento | Tipo | Descripción |
|---|---|---|
| `@RefreshScope` | Anotación | Marca un bean para que se destruya y recree tras un refresh. |
| `POST /actuator/refresh` | Endpoint HTTP | Dispara el refresco en la instancia que lo recibe. |
| `ContextRefresher` | Bean de Spring | API programática para disparar el refresco y obtener las claves modificadas. |
| `EnvironmentChangeEvent` | Evento Spring | Publicado tras el refresh con el mapa de propiedades cambiadas. |
| `@ConfigurationProperties` | Anotación | Grupo de propiedades tipadas; combinado con `@RefreshScope` es el patrón recomendado. |
| `spring-cloud-starter-bus-amqp/kafka` | Dependencia | Activa `POST /actuator/busrefresh` para broadcast a todas las instancias. |

## Buenas y malas prácticas

**Hacer:**
- Usar `@RefreshScope` en el bean de propiedades (`@ConfigurationProperties`), no en todos los beans que las consumen. Los beans consumidores inyectan el proxy y se benefician del refresco automáticamente.
- Inyectar siempre las propiedades refrescables mediante el bean `@ConfigurationProperties` (inyección por referencia al proxy). Si se inyecta con `@Value` en un bean sin `@RefreshScope`, el valor no se actualiza hasta el reinicio.
- Monitorear `EnvironmentChangeEvent` en producción para registrar qué propiedades cambian y cuándo. Esto facilita la auditoría y el diagnóstico de comportamientos inesperados tras un refresh.

**Evitar:**
- Anotar beans de infraestructura crítica con `@RefreshScope` sin considerar el impacto: al refrescar, el bean se destruye y las peticiones en vuelo que lo usan pueden fallar. `DataSource`, pools de conexiones y clientes HTTP con estado interno son candidatos problemáticos.
- Poner `@RefreshScope` en beans `@Configuration`: puede provocar que beans hijos del contexto no se recreen correctamente, dejando el contexto en un estado inconsistente.
- Asumir que `@RefreshScope` funciona con beans cuyo ciclo de vida no controla Spring (por ejemplo, instancias creadas con `new`). Solo los beans gestionados por el `ApplicationContext` participan en el refresco.

## Comparación: refresco parcial vs refresco total

La siguiente tabla contrasta los dos tipos de refresco que ofrece `ContextRefresher`.

| Aspecto | Refresco parcial (`refresh()`) | Refresco total (reinicio del contexto) |
|---|---|---|
| Qué recrea | Solo los beans con `@RefreshScope` | Todo el `ApplicationContext` |
| Impacto en peticiones en curso | Mínimo (solo los beans refrescados) | Alto (se pierden las conexiones activas) |
| Tiempo de ejecución | Milisegundos | Segundos (equivale a un reinicio) |
| Casos de uso | Cambio de propiedades de configuración | Cambio de beans de infraestructura |
| Cómo dispararlo | `POST /actuator/refresh` | Reinicio del proceso JVM |
| Disponible con Spring Cloud Config | Sí | Sí (pero equivale a un redeploy) |

> [EXAMEN] Una pregunta habitual: "¿Por qué `@Value("${mi.propiedad}")` en un bean sin `@RefreshScope` no se actualiza tras un `POST /actuator/refresh`?" La respuesta es que `@Value` se resuelve una sola vez durante la creación del bean. Sin `@RefreshScope`, el bean no se recrea, y el campo `@Value` mantiene el valor original. La solución es combinar `@ConfigurationProperties` con `@RefreshScope` en el bean de propiedades.

---

← [1.5 Backends alternativos al Git backend](sc-config-backends.md) | [Índice (README.md)](README.md) | [1.7 Seguridad del Config Server →](sc-config-security.md)
