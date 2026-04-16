# 7.4 Refresh de configuración distribuido con Spring Cloud Bus

← [7.3 Configuración del broker subyacente para Spring Cloud Bus](sc-bus-broker-config.md) | [Índice](README.md) | [7.5 Eventos personalizados de Spring Cloud Bus](sc-bus-eventos-personalizados.md) →

## Introducción

El problema que resuelve este módulo es concreto: en un clúster de cincuenta instancias de un mismo servicio, un cambio en el repositorio de configuración Git no llega a ningún nodo hasta que alguien llama explícitamente a `/actuator/refresh` en cada instancia. Esto es inviable en producción sin automatización, y la automatización ingenua —un script que hace curl a todas las IPs— crea acoplamiento con la topología dinámica del clúster. Spring Cloud Bus proporciona una solución basada en mensajería: un único `POST /actuator/busrefresh` publica un `RefreshRemoteApplicationEvent` en el bus, que lo entrega a todos los nodos suscritos simultáneamente, independientemente de cuántos sean o de qué IPs tengan. El endpoint `/actuator/busrefresh/{destination}` añade granularidad permitiendo el refresh selectivo de un subconjunto de nodos sin afectar al resto del clúster.

> [PREREQUISITO] Comprende `@RefreshScope` de Spring Cloud Context (documentado en sc-config-refresh.md) antes de este fichero. El refresh distribuido es el mecanismo de transporte; `@RefreshScope` es el mecanismo de recarga en el bean receptor.

## Representación visual

El diagrama de secuencia siguiente muestra el flujo completo desde el push en Git hasta la re-instanciación del bean en todos los nodos.

```
Git Repository          Config Server            Bus Broker
     │                       │                       │
     │── git push ──────────►│                       │
     │                       │                       │
     │                       │── POST /busrefresh ──►│
     │                       │   (webhook trigger)   │
     │                       │                       │── RefreshRemoteApplicationEvent ──►│
     │                       │                       │                                    │
     │                       │                       │        ┌───────────────────────────┘
     │                       │                       │        │ Todos los nodos suscritos
     │                       │                       │        ▼
     │                       │                  Node-A:inst-1
     │                       │                  Node-A:inst-2
     │                       │                  Node-B:inst-1
     │                       │                       │
     │                       │◄── GET /config ───────│ (cada nodo recarga desde Config Server)
     │                       │                       │
     │                       │─── propiedades ──────►│
     │                       │                       │
     │                       │                       │── @RefreshScope beans re-instanciados
```

El parámetro `destination` permite el refresh selectivo. La tabla siguiente explica los formatos válidos:

| Formato de destination | Nodos afectados |
|---|---|
| (sin destination) | Todos los nodos del bus |
| `service-a:**` | Todas las instancias de `service-a`, todos los perfiles |
| `service-a:production:**` | Todas las instancias de `service-a` con perfil `production` |
| `service-a:production:instance-1` | Exactamente el nodo con ese `bus.id` |
| `**` | Equivalente a sin destination: todos los nodos |

## Ejemplo central

El ejemplo siguiente implementa la integración completa: Config Server con webhook Git configurado que dispara el busrefresh automáticamente, y un servicio cliente que recibe el refresh y recarga sus beans.

```xml
<!-- pom.xml del Config Server -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```yaml
# Config Server — application.yml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/company/config-repo
          clone-on-start: true
    bus:
      enabled: true
      id: config-server:${spring.profiles.active:default}:${random.value}
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, busenv, health, info

# El webhook de GitHub apuntará a: http://config-server:8888/actuator/busrefresh
# GitHub Webhook settings:
#   Payload URL: http://config-server:8888/actuator/busrefresh
#   Content type: application/json
#   Secret: (configurado en spring.cloud.bus.security o Spring Security)
#   Events: Just the push event
```

```xml
<!-- pom.xml del servicio cliente -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

```yaml
# Servicio cliente — application.yml
spring:
  application:
    name: pricing-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
    bus:
      enabled: true
      id: ${spring.application.name}:${spring.profiles.active:default}:${random.value}
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh, refresh, health, info
```

```java
// src/main/java/com/example/pricing/PricingApplication.java
package com.example.pricing;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PricingApplication {
    public static void main(String[] args) {
        SpringApplication.run(PricingApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/pricing/config/PricingConfig.java
package com.example.pricing.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Component
@RefreshScope
public class PricingConfig {

    private final double taxRate;
    private final String currency;
    private final boolean discountsEnabled;

    // Al recibir RefreshRemoteApplicationEvent, Spring destruye este bean
    // y lo re-instancia con los nuevos valores del Config Server
    public PricingConfig(
            @Value("${pricing.tax-rate:0.21}") double taxRate,
            @Value("${pricing.currency:EUR}") String currency,
            @Value("${pricing.discounts-enabled:true}") boolean discountsEnabled) {
        this.taxRate = taxRate;
        this.currency = currency;
        this.discountsEnabled = discountsEnabled;
    }

    public double getTaxRate() { return taxRate; }
    public String getCurrency() { return currency; }
    public boolean isDiscountsEnabled() { return discountsEnabled; }
}
```

```java
// src/main/java/com/example/pricing/controller/PricingController.java
package com.example.pricing.controller;

import com.example.pricing.config.PricingConfig;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Map;

@RestController
public class PricingController {

    private final PricingConfig pricingConfig;

    public PricingController(PricingConfig pricingConfig) {
        this.pricingConfig = pricingConfig;
    }

    // Cada llamada a este endpoint obtiene los valores actuales de PricingConfig
    // Después de un busrefresh, devolverá los valores actualizados
    @GetMapping("/pricing/config")
    public Map<String, Object> getConfig() {
        return Map.of(
            "taxRate", pricingConfig.getTaxRate(),
            "currency", pricingConfig.getCurrency(),
            "discountsEnabled", pricingConfig.isDiscountsEnabled()
        );
    }
}
```

Para disparar un refresh selectivo solo en instancias de `pricing-service`:

```bash
# Refresh de todas las instancias de pricing-service
curl -X POST http://config-server:8888/actuator/busrefresh/pricing-service:**

# Refresh de una instancia específica
curl -X POST "http://config-server:8888/actuator/busrefresh/pricing-service:production:abc123"

# Propagación de un cambio de environment variable en caliente (sin restart)
curl -X POST http://config-server:8888/actuator/busenv \
  -H "Content-Type: application/json" \
  -d '{"name":"pricing.tax-rate","value":"0.23"}'
```

## Tabla de elementos clave

Esta tabla recoge los parámetros y endpoints del refresh distribuido que un senior debe dominar en profundidad.

| Elemento | Tipo/Valor | Default | Descripción |
|---|---|---|---|
| `POST /actuator/busrefresh` | Endpoint | — | Dispara `RefreshRemoteApplicationEvent` a todos los nodos del bus. |
| `POST /actuator/busrefresh/{destination}` | Endpoint | — | Refresh selectivo. `destination` en formato `app:profile:index`. |
| `POST /actuator/busenv` | Endpoint | — | Propaga `EnvironmentChangeRemoteApplicationEvent` con cambios de variables de entorno. |
| `spring.cloud.bus.refresh.enabled` | Boolean | `true` | Habilita el listener de `RefreshRemoteApplicationEvent` en este nodo. |
| `spring.cloud.bus.env.enabled` | Boolean | `true` | Habilita el listener de `EnvironmentChangeRemoteApplicationEvent`. |
| `destination` param | String | `**` (todos) | Patrón de destino. Soporta wildcards: `app:**`, `**`, o `app:profile:id` exacto. |
| `@RefreshScope` | Anotación | — | Marca el bean para re-instanciación al recibir `RefreshRemoteApplicationEvent`. |
| `RefreshRemoteApplicationEvent` | Clase | — | Evento Bus que desencadena el refresh. Extiende `RemoteApplicationEvent`. |
| `EnvironmentChangeRemoteApplicationEvent` | Clase | — | Evento Bus que propaga cambios de variables de entorno sin restart del contexto. |

## Buenas y malas prácticas

**Hacer:**

- Integrar el webhook de GitHub/GitLab apuntando al Config Server, no a los servicios cliente. El Config Server actúa como punto de entrada único al bus: recibe el evento Git, publica el `RefreshRemoteApplicationEvent` y cada nodo recarga desde el Config Server. Si el webhook apunta directamente a un servicio cliente, ese servicio se convierte en coordinador del bus, lo que rompe la topología hub-and-spoke.
- Usar el endpoint `/actuator/busrefresh/{destination}` en despliegues canary. Al actualizar primero `pricing-service:production:canary-instance`, se puede verificar el comportamiento con los nuevos valores antes de propagarlos al resto del clúster.
- Verificar que los beans que necesitan actualización tienen `@RefreshScope`. Un bean sin esta anotación cacheará los valores del momento del arranque indefinidamente, incluso si el Config Server ya devuelve valores nuevos. Este es el error de configuración más común con Bus.

**Evitar:**

- No disparar `/actuator/busrefresh` desde pipelines CI/CD sin rate limiting. Un pipeline que hace deploy de diez microservicios simultáneamente puede generar diez refresh eventos concurrentes, y si el Config Server tarda en responder, los nodos pueden recibir respuestas inconsistentes durante la ventana de actualización.
- No usar `/actuator/busenv` para cambios permanentes de configuración en producción. Los cambios propagados por `busenv` se almacenan en memoria y se pierden al reiniciar los nodos. Esta funcionalidad es para ajustes temporales de diagnóstico, no para cambios de configuración que deban persistir.
- No confundir `/actuator/refresh` (refresh local del nodo actual) con `/actuator/busrefresh` (broadcast a todos). En un sistema con Bus activo, la operación correcta para actualizar la configuración del clúster es siempre `busrefresh`; usar `refresh` directo en cada nodo no es un error, pero es operacionalmente ineficiente y no escala.

## Comparación: refresh local vs refresh distribuido

El siguiente contraste clarifica cuándo usar cada mecanismo, una pregunta habitual en evaluaciones técnicas.

| Criterio | `/actuator/refresh` (local) | `/actuator/busrefresh` (Bus) |
|---|---|---|
| Alcance | Solo el nodo receptor de la llamada HTTP | Todos los nodos suscritos al bus |
| Requiere Bus | No | Sí |
| Requiere broker | No | Sí |
| Latencia de propagación | Inmediata en el nodo llamado | Milisegundos (latencia del broker) |
| Útil en | Dev/debug local, refresh quirúrgico | Producción con múltiples instancias |
| Riesgo | Inconsistencia si no se llama a todos los nodos | Ninguno si el broker está disponible |
| Integración webhook | Requiere llamar a cada instancia | Un solo POST al Config Server |

---

← [7.3 Configuración del broker subyacente para Spring Cloud Bus](sc-bus-broker-config.md) | [Índice](README.md) | [7.5 Eventos personalizados de Spring Cloud Bus](sc-bus-eventos-personalizados.md) →
