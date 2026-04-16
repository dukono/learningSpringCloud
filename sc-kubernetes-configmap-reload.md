# 9.3 ConfigMap PropertySource — recarga dinámica

← [9.2 ConfigMap PropertySource — lectura y fuentes](sc-kubernetes-configmap.md) | [Índice (README.md)](README.md) | [9.4 Secrets PropertySource](sc-kubernetes-secrets.md) →

---

## Introducción

Un `ConfigMap` en Kubernetes puede actualizarse en cualquier momento mediante `kubectl apply` o la API. Sin recarga dinámica, la aplicación Spring Boot solo lee el ConfigMap en el arranque y requiere un reinicio para recoger los cambios. Spring Cloud Kubernetes ofrece un mecanismo de recarga automática que detecta cambios en el ConfigMap (o en un Secret) y propaga esos cambios a la aplicación sin necesidad de desplegar una nueva imagen. Esto es especialmente útil para cambios de parámetros operacionales (timeouts, flags de feature, niveles de log) que no justifican un redespliegue completo. El mecanismo de recarga tiene tres estrategias con comportamientos muy distintos, y elegir la incorrecta puede provocar pérdida de estado en producción.

> **[PREREQUISITO]** La recarga requiere que la lectura básica del ConfigMap esté configurada correctamente según [9.2 ConfigMap PropertySource — lectura y fuentes](sc-kubernetes-configmap.md). Sin `config.enabled=true` y RBAC de lectura, la recarga no funciona.

> **[ADVERTENCIA]** `RESTART_CONTEXT` y `SHUTDOWN` causan que el pod se reinicie o termine. En un Deployment con una sola réplica, esto produce downtime. Usar solo `REFRESH` para cambios de configuración en caliente sin interrupción de servicio.

## Representación visual

El diagrama muestra los dos modos de detección de cambios y las tres estrategias de aplicación. La elección correcta depende del tipo de beans que consumen la configuración.

```
ConfigMap actualizado en K8s
          │
          ├── Modo WATCH (evento K8s inmediato)
          │       │
          │       └── ConfigMapWatcher observa eventos
          │           del API server en tiempo real
          │
          └── Modo POLLING (intervalo configurable)
                  │
                  └── ConfigMapPropertySourceLocator
                      compara hash cada `reload.period`
                      │
                      ▼
          ┌─────────────────────────────────────┐
          │    Cambio detectado → estrategia     │
          ├─────────────────────────────────────┤
          │ REFRESH      → @RefreshScope beans  │  ← sin downtime
          │               republished via Bus   │
          │                                     │
          │ RESTART_CONTEXT → ApplicationContext│  ← downtime ~seg
          │               reinicia el contexto  │
          │                                     │
          │ SHUTDOWN      → JVM termina         │  ← K8s reinicia pod
          └─────────────────────────────────────┘
```

> **[CONCEPTO]** El modo `WATCH` requiere permiso `watch` sobre `configmaps` en el RBAC del ServiceAccount. El modo `POLLING` solo necesita `get`/`list`. Si el cluster restringe el uso de watchers (por cuota de conexiones al API server), usar polling con un período adecuado (mínimo recomendado: 15 segundos).

## Ejemplo central

El siguiente ejemplo configura la recarga dinámica con estrategia `REFRESH` y modo `WATCH` para que los beans anotados con `@RefreshScope` recarguen sus propiedades cuando el ConfigMap cambia, sin reiniciar la aplicación.

**pom.xml** (dependencias relevantes):

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
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <!-- Necesario para @RefreshScope -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-context</artifactId>
  </dependency>
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: catalogo-service
  config:
    import: "kubernetes:configmap/catalogo-config"

spring:
  cloud:
    kubernetes:
      config:
        enabled: true
        name: catalogo-config
        namespace: default
      reload:
        enabled: true
        strategy: REFRESH          # REFRESH | RESTART_CONTEXT | SHUTDOWN
        mode: watch                # watch | polling
        period: 15000              # solo relevante si mode=polling (ms)
        monitoring-config-maps: true
        monitoring-secrets: false

management:
  endpoints:
    web:
      exposure:
        include: health,refresh
```

**Bean con @RefreshScope**:

```java
package com.ejemplo.catalogo.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Component
@RefreshScope
public class CatalogoConfig {

    @Value("${catalogo.max-resultados:20}")
    private int maxResultados;

    @Value("${catalogo.cache-ttl-segundos:300}")
    private int cacheTtlSegundos;

    public int getMaxResultados() {
        return maxResultados;
    }

    public int getCacheTtlSegundos() {
        return cacheTtlSegundos;
    }
}
```

**ConfigMap con binaryData** — para ficheros de configuración binarios o propiedades con caracteres especiales:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogo-config
  namespace: default
data:
  application.properties: |
    catalogo.max-resultados=50
    catalogo.cache-ttl-segundos=600
binaryData:
  logback.xml: <base64-encoded-content>
```

**RBAC para modo WATCH** (necesita el verbo `watch`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: catalogo-configmap-watcher
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué diferencia hay entre REFRESH y RESTART_CONTEXT en la recarga de SCK?" Con `REFRESH`, solo los beans anotados con `@RefreshScope` reciben los nuevos valores; los beans singleton no se re-crean. Con `RESTART_CONTEXT`, todo el `ApplicationContext` se reinicia, lo que garantiza que todos los beans reciben la nueva configuración pero causa un breve downtime y puede provocar pérdida de conexiones activas.

## Tabla de elementos clave

Las propiedades del bloque `spring.cloud.kubernetes.reload` controlan el comportamiento completo de la recarga dinámica.

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.reload.enabled` | boolean | `false` | Activa la recarga dinámica de ConfigMaps/Secrets |
| `spring.cloud.kubernetes.reload.strategy` | enum | `REFRESH` | Estrategia: `REFRESH`, `RESTART_CONTEXT`, `SHUTDOWN` |
| `spring.cloud.kubernetes.reload.mode` | enum | `polling` | Modo de detección: `polling` o `watch` |
| `spring.cloud.kubernetes.reload.period` | long (ms) | `15000` | Intervalo de polling en milisegundos |
| `spring.cloud.kubernetes.reload.monitoring-config-maps` | boolean | `true` | Monitoriza cambios en ConfigMaps |
| `spring.cloud.kubernetes.reload.monitoring-secrets` | boolean | `false` | Monitoriza cambios en Secrets |
| `spring.cloud.kubernetes.reload.max-wait-for-restart` | long (ms) | `2000` | Tiempo máximo de espera antes de reiniciar el contexto |

## Buenas y malas prácticas

**Hacer:**

- Usar estrategia `REFRESH` con `@RefreshScope` en los beans que consumen propiedades que cambian frecuentemente (flags de feature, parámetros de negocio): es la única estrategia que no causa downtime.
- Usar modo `watch` en producción cuando el cluster lo permite: la propagación del cambio es prácticamente inmediata (< 1 segundo) frente a los 15 segundos de polling.
- Combinar la recarga con Spring Cloud Bus si se tienen múltiples réplicas: el evento de refresh se propaga a todas las instancias vía el bus en lugar de que cada pod monitorice el ConfigMap por separado.
- Anotar con `@RefreshScope` solo los beans que realmente cambian con la configuración: anotar beans con estado (repositorios, connection pools) con `@RefreshScope` causa su destrucción y recreación, lo que puede provocar pérdida de conexiones activas.

**Evitar:**

- Usar `SHUTDOWN` en un Deployment sin un `PodDisruptionBudget` configurado: si la estrategia de recarga hace que todos los pods se terminen simultáneamente, Kubernetes los reinicia en orden pero hay un periodo sin instancias disponibles.
- Activar `monitoring-secrets: true` sin revisar las implicaciones de seguridad: el watcher de Secrets requiere permisos adicionales sobre el recurso `secrets` y aumenta la superficie de ataque del ServiceAccount.
- Usar `reload.period` menor de 5000 ms: sondeos muy frecuentes generan carga en el API server de Kubernetes y pueden activar rate limiting, especialmente en clusters compartidos.

---

← [9.2 ConfigMap PropertySource — lectura y fuentes](sc-kubernetes-configmap.md) | [Índice (README.md)](README.md) | [9.4 Secrets PropertySource](sc-kubernetes-secrets.md) →
