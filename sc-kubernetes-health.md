# 9.10 Health Indicators de Spring Cloud Kubernetes

← [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md) | [Índice (README.md)](README.md) | [9.11 Leader Election con Spring Cloud Kubernetes](sc-kubernetes-leader.md) →

---

## Introducción

Kubernetes utiliza las probes de `liveness` y `readiness` para decidir si reiniciar un pod o si incluirlo en el pool de tráfico del Service. Spring Boot Actuator expone los endpoints `/actuator/health/liveness` y `/actuator/health/readiness` para esas probes. Spring Cloud Kubernetes añade un `KubernetesHealthIndicator` que reporta el estado de conectividad de la aplicación con el API server de Kubernetes, de modo que si el API server no es accesible (por ejemplo, durante un mantenimiento del control plane), el health check puede reflejar ese estado degradado. Sin este indicador, una aplicación podría reportarse como `UP` aunque no pueda leer ConfigMaps o descubrir servicios, enmascarando un problema de infraestructura hasta que la configuración se desactualiza o el descubrimiento falla.

> **[PREREQUISITO]** Este nodo asume que Spring Boot Actuator está en el classpath. Para conocer cómo configurar las probes de liveness y readiness en Kubernetes con Spring Boot Actuator, ver la documentación de Spring Boot Actuator (`management.endpoint.health.probes.enabled`).

## Representación visual

El diagrama muestra cómo se integran el `KubernetesHealthIndicator` de SCK con las probes de Kubernetes.

```
Kubernetes kubelet
   │
   ├── liveness probe  → GET /actuator/health/liveness
   │       │               │
   │       │               └── LivenessStateHealthIndicator (Spring Boot)
   │       │                   Estado: LIVE | BROKEN
   │       │
   └── readiness probe → GET /actuator/health/readiness
           │               │
           │               ├── ReadinessStateHealthIndicator (Spring Boot)
           │               │   Estado: ACCEPTING_TRAFFIC | REFUSING_TRAFFIC
           │               │
           │               └── KubernetesHealthIndicator (SCK) ← [nuevo]
           │                   Verifica: conectividad con API server
           │                   Estado: UP | DOWN

GET /actuator/health (completo)
{
  "status": "UP",
  "components": {
    "kubernetes": {
      "status": "UP",
      "details": {
        "nodeName": "worker-node-1",
        "podName": "mi-app-abc123",
        "namespace": "produccion",
        "inside": true
      }
    },
    "diskSpace": { "status": "UP" },
    "ping": { "status": "UP" }
  }
}
```

> **[CONCEPTO]** Spring Boot Actuator expone las probes de liveness y readiness en endpoints separados para que Kubernetes pueda diferenciar entre "la JVM está viva pero no lista" (solo readiness falla, no reinicia el pod) y "la JVM está en un estado irrecuperable" (liveness falla, Kubernetes reinicia el pod). `KubernetesHealthIndicator` contribuye al grupo de readiness por defecto.

## Ejemplo central

El siguiente ejemplo muestra la configuración completa para exponer las probes de Kubernetes con el `KubernetesHealthIndicator` de SCK y el manifiesto de Deployment con las probes configuradas.

**pom.xml**:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
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

**application.yml**:

```yaml
spring:
  application:
    name: mi-app
  config:
    import: "kubernetes:configmap/mi-app-config"

spring:
  cloud:
    kubernetes:
      health:
        enabled: true    # KubernetesHealthIndicator activado (default: true)

management:
  endpoints:
    web:
      exposure:
        include: health, info
  endpoint:
    health:
      probes:
        enabled: true              # Activa /actuator/health/liveness y /readiness
      show-details: always         # Muestra detalles en el health check (solo dev/staging)
      group:
        readiness:
          include: readinessState, kubernetes   # Incluye KubernetesHealthIndicator en readiness
  health:
    kubernetes:
      enabled: true
```

**Deployment con probes configuradas**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  namespace: produccion
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      serviceAccountName: mi-app-sa
      containers:
        - name: mi-app
          image: mi-empresa/mi-app:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30    # Tiempo de arranque de la JVM
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
```

**HealthContributor personalizado** para verificar que el ConfigMap watch está activo:

```java
package com.ejemplo.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.cloud.kubernetes.fabric8.config.reload.ConfigMapPropertySourceLocator;
import org.springframework.stereotype.Component;

@Component("configmapWatcher")
public class ConfigMapWatchHealthIndicator implements HealthIndicator {

    private volatile boolean watchActivo = true;

    // El watcher puede llamar a este método si detecta un error
    public void reportarFalloWatch() {
        this.watchActivo = false;
    }

    @Override
    public Health health() {
        if (watchActivo) {
            return Health.up()
                .withDetail("configmap-watch", "activo")
                .build();
        }
        return Health.down()
            .withDetail("configmap-watch", "inactivo — polling como fallback")
            .build();
    }
}
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué ocurre si el API server de Kubernetes se desconecta mientras la aplicación está ejecutándose?" El `KubernetesHealthIndicator` reporta `DOWN` en el next health check. Si está incluido en el grupo `readiness`, Kubernetes marcará el pod como no-ready y lo sacará del pool de tráfico. La aplicación puede seguir funcionando con la configuración cargada en memoria, pero no recibirá actualizaciones de ConfigMap hasta que la conexión con el API server se restaure.

## Tabla de elementos clave

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.health.enabled` | boolean | `true` | Activa KubernetesHealthIndicator |
| `management.endpoint.health.probes.enabled` | boolean | `false` | Activa endpoints /liveness y /readiness de Spring Boot Actuator |
| `management.health.kubernetes.enabled` | boolean | `true` | Incluye el KubernetesHealthIndicator en el health check general |
| `management.endpoint.health.group.readiness.include` | String | — | Lista de indicadores incluidos en la probe de readiness |
| `management.endpoint.health.show-details` | enum | `never` | `always` muestra detalles; usar `when-authorized` en producción |

## Buenas y malas prácticas

**Hacer:**

- Incluir `KubernetesHealthIndicator` en el grupo `readiness` (no en `liveness`): la desconexión del API server no debería reiniciar el pod (liveness), sino sacarlo del pool de tráfico (readiness) mientras se restaura la conexión.
- Configurar `initialDelaySeconds` de la liveness probe a al menos 30 segundos en aplicaciones con Spring Cloud Kubernetes: el arranque incluye la carga de ConfigMaps y la inicialización de watchers, que pueden tardar más que en una aplicación Spring Boot estándar.
- Usar `show-details: when-authorized` en producción: exponer los detalles del health check (nombre del nodo, namespace, nombre del pod) sin autenticación facilita el reconocimiento del entorno a posibles atacantes.

**Evitar:**

- Incluir `KubernetesHealthIndicator` en el grupo `liveness`: si el API server tiene un mantenimiento breve, Kubernetes reiniciaría todos los pods de la aplicación simultáneamente, causando downtime innecesario.
- Deshabilitar las probes en producción para "evitar complejidad": sin probes, Kubernetes no puede detectar pods zombie (la JVM responde HTTP pero no procesa peticiones), que es uno de los problemas más frecuentes en entornos de microservicios bajo carga.

---

← [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md) | [Índice (README.md)](README.md) | [9.11 Leader Election con Spring Cloud Kubernetes](sc-kubernetes-leader.md) →
