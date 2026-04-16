# 2.4 Registro de instancias: metadata y estado

← [2.3 Configuración y arranque del Eureka Client](sc-eureka-cliente.md) | [Índice (README.md)](README.md) | [2.5.1 Heartbeat y lease del registro →](sc-eureka-lease.md)

---

Cuando un microservicio se registra en Eureka, no transmite solo su nombre y dirección: publica un conjunto de metadatos estructurados que otros clientes y el propio servidor usan para tomar decisiones de enrutamiento, balanceo y health check. Si esa metadata está mal configurada —por ejemplo, publicando el hostname interno de un contenedor en lugar de su IP, o usando el estado de Eureka desacoplado del estado real de la aplicación— los consumidores reciben información incorrecta y dirigen tráfico a instancias inaccesibles. Esta sección cubre qué metadatos se publican, cómo controlarlos y cómo conectar el estado de Eureka al ciclo de salud real de la aplicación.

> [PREREQUISITO] Spring Boot Actuator debe estar en el classpath y el endpoint `/actuator/health` debe estar expuesto para que la integración con el `HealthCheckHandler` de Eureka funcione correctamente.

## Diagrama: estructura de la metadata de instancia

El siguiente diagrama muestra los campos que Eureka almacena por cada instancia registrada y su origen de configuración.

```
InstanceInfo (registro Eureka)
│
├── appName          ← spring.application.name
├── instanceId       ← eureka.instance.instance-id
├── hostName         ← eureka.instance.hostname
│                      (o IP si prefer-ip-address: true)
├── ipAddr           ← IP de red detectada automáticamente
├── port             ← server.port
├── securePort       ← server.ssl.port (si HTTPS habilitado)
├── vipAddress       ← eureka.instance.virtual-host-name
│                      (default: spring.application.name)
├── secureVipAddress ← eureka.instance.secure-virtual-host-name
├── status           ← UP | DOWN | STARTING | OUT_OF_SERVICE | UNKNOWN
│                      controlado por HealthCheckHandler
└── metadata         ← eureka.instance.metadata-map.*
        ├── zona: "eu-west-1a"
        ├── version: "2.1.0"
        └── environment: "production"
```

## Implementación: configuración completa de metadata e integración con Actuator

El siguiente ejemplo muestra un servicio con metadata personalizada, preferencia de IP sobre hostname, y la integración de Spring Actuator para que el estado de Eureka refleje el estado real de la aplicación.

**application.yml**

```yaml
server:
  port: 8082

spring:
  application:
    name: servicio-catalogo

eureka:
  instance:
    # Registrar la IP en lugar del hostname del SO.
    # Obligatorio en entornos Docker/Kubernetes donde el hostname
    # del contenedor no resuelve desde la red del host.
    prefer-ip-address: true

    # Hostname explícito: solo se usa si prefer-ip-address: false.
    # En producción, configurar con el FQDN del host o servicio DNS.
    # hostname: catalogo.internal.empresa.com

    # Metadatos personalizados: pares clave-valor accesibles vía DiscoveryClient.
    # Los consumidores pueden leerlos con instance.getMetadata().get("zona").
    metadata-map:
      zona: "eu-west-1a"
      version: "2.1.0"
      environment: "production"
      tipo: "catalogo"

    # Parámetros de lease (heartbeat y expiración — ver sección 2.5.1)
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90

  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    # Activar la integración de Actuator health con el estado de Eureka.
    # Con esta propiedad, si /actuator/health devuelve DOWN, Eureka
    # marca la instancia como DOWN automáticamente.
    healthcheck:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

**Lectura de metadata personalizada desde un consumidor**

```java
package com.ejemplo.pedidos.service;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class CatalogoLocalizador {

    private final DiscoveryClient discoveryClient;

    public CatalogoLocalizador(DiscoveryClient discoveryClient) {
        this.discoveryClient = discoveryClient;
    }

    /**
     * Obtiene la instancia de servicio-catalogo en la zona "eu-west-1a".
     * Usa la metadata personalizada para filtrar entre instancias disponibles.
     */
    public ServiceInstance obtenerInstanciaEnZona(String zonaDeseada) {
        List<ServiceInstance> instancias = discoveryClient.getInstances("servicio-catalogo");

        return instancias.stream()
            .filter(inst -> {
                Map<String, String> metadata = inst.getMetadata();
                return zonaDeseada.equals(metadata.get("zona"));
            })
            .findFirst()
            .orElseThrow(() -> new RuntimeException(
                "No hay instancias de servicio-catalogo en zona: " + zonaDeseada
            ));
    }
}
```

**HealthCheckHandler personalizado (estado avanzado)**

```java
package com.ejemplo.catalogo.health;

import com.netflix.appinfo.HealthCheckHandler;
import com.netflix.appinfo.InstanceInfo.InstanceStatus;
import org.springframework.boot.actuate.health.HealthComponent;
import org.springframework.boot.actuate.health.HealthEndpoint;
import org.springframework.boot.actuate.health.Status;
import org.springframework.stereotype.Component;

/**
 * Implementación personalizada de HealthCheckHandler que mapea el estado
 * de Spring Actuator al estado de instancia de Eureka.
 * Eureka llama a este handler antes de cada heartbeat para actualizar el estado.
 */
@Component
public class ActuatorHealthCheckHandler implements HealthCheckHandler {

    private final HealthEndpoint healthEndpoint;

    public ActuatorHealthCheckHandler(HealthEndpoint healthEndpoint) {
        this.healthEndpoint = healthEndpoint;
    }

    @Override
    public InstanceStatus getStatus(InstanceStatus currentStatus) {
        HealthComponent health = healthEndpoint.health();
        Status status = health.getStatus();

        return switch (status.getCode()) {
            case "UP"      -> InstanceStatus.UP;
            case "DOWN"    -> InstanceStatus.DOWN;
            case "OUT_OF_SERVICE" -> InstanceStatus.OUT_OF_SERVICE;
            default        -> InstanceStatus.UNKNOWN;
        };
    }
}
```

> [ADVERTENCIA] Sin `eureka.client.healthcheck.enabled: true`, Eureka solo actualiza el estado de la instancia a través del heartbeat y la expiración del lease. Una instancia con `prefer-ip-address: false` que publica un hostname no resolvible aparecerá como UP en el registro aunque ningún consumidor pueda conectar a ella.

## Tabla de propiedades de metadata e instancia

La siguiente tabla recoge las propiedades que controlan la identidad y el estado de la instancia en el registro.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `eureka.instance.prefer-ip-address` | boolean | `false` | Registra la IP en lugar del hostname; necesario en Docker/K8s |
| `eureka.instance.hostname` | String | hostname del SO | FQDN o IP explícita; ignorado si `prefer-ip-address: true` |
| `eureka.instance.virtual-host-name` | String | `spring.application.name` | vipAddress: nombre lógico para balanceo |
| `eureka.instance.metadata-map.*` | Map<String,String> | `{}` | Metadatos personalizados; accesibles vía `ServiceInstance.getMetadata()` |
| `eureka.client.healthcheck.enabled` | boolean | `false` | Conecta Actuator health con el estado de instancia en Eureka |
| `management.endpoints.web.exposure.include` | String | `health` | Endpoints de Actuator expuestos; debe incluir `health` para integración Eureka |

**Estados de instancia de Eureka**

| Estado | Significado | Quién lo establece |
|---|---|---|
| `UP` | La instancia está disponible y acepta tráfico | Heartbeat exitoso + health check OK |
| `DOWN` | La instancia ha fallado | `HealthCheckHandler` o expiración de lease |
| `STARTING` | La instancia está arrancando, no acepta tráfico todavía | Fase inicial del arranque del cliente |
| `OUT_OF_SERVICE` | Retirada temporalmente del tráfico (mantenimiento) | PUT manual via API REST de Eureka |
| `UNKNOWN` | Estado indeterminado | `HealthCheckHandler` con estado no mapeado |

## Buenas y malas prácticas

Hacer:
- Activar `eureka.client.healthcheck.enabled: true` en producción: sin esta propiedad, una instancia cuyo servicio de base de datos está caído aparecerá como UP en Eureka hasta que expire su lease (90 segundos por defecto), enviando tráfico fallido durante ese tiempo.
- Incluir en `metadata-map` información de versión y zona: permite que consumidores avanzados implementen routing basado en versión (canary deployment) o routing de zona sin depender de infraestructura adicional.
- Configurar `prefer-ip-address: true` en entornos contenerizados: los hostnames de contenedores Docker son IDs hexadecimales que no resuelven desde la red host, haciendo que todas las llamadas fallen con `UnknownHostException`.

Evitar:
- Incluir secretos en `metadata-map`: los metadatos son visibles para cualquier cliente Eureka que consulte el registro sin autenticación; credenciales o tokens expuestos aquí son accesibles a todo el ecosistema.
- Cambiar el estado a `OUT_OF_SERVICE` manualmente sin automatizar su restauración: instancias marcadas manualmente como `OUT_OF_SERVICE` no vuelven a `UP` solas; si el proceso de mantenimiento no incluye el PUT de restauración, el servicio permanece oculto indefinidamente.
- Confiar únicamente en el estado UP de Eureka para inferir que una instancia acepta tráfico de aplicación: el heartbeat solo mide conectividad de red, no la disponibilidad de la lógica de negocio. Sin `healthcheck.enabled: true`, una instancia con la base de datos caída aparece como UP.

---

← [2.3 Configuración y arranque del Eureka Client](sc-eureka-cliente.md) | [Índice (README.md)](README.md) | [2.5.1 Heartbeat y lease del registro →](sc-eureka-lease.md)
