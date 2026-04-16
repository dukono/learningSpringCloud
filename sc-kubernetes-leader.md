# 9.11 Leader Election con Spring Cloud Kubernetes

← [9.10 Health Indicators de Spring Cloud Kubernetes](sc-kubernetes-health.md) | [Índice (README.md)](README.md) | [9.12 Testing de Spring Cloud Kubernetes](sc-kubernetes-testing.md) →

---

## Introducción

Cuando un microservicio tiene múltiples réplicas en Kubernetes, ciertas tareas deben ejecutarse en una única instancia: el envío de notificaciones diarias, la ejecución de jobs de reconciliación, la rotación de datos. Sin coordinación, todas las réplicas ejecutarían la tarea simultáneamente, causando duplicados, condiciones de carrera o conflictos sobre recursos compartidos. Spring Cloud Kubernetes Leader usa un `ConfigMap` de Kubernetes como mecanismo de lock distribuido: la instancia que logra escribir su identidad en el ConfigMap (con garantía de write-once via optimistic locking de Kubernetes) se convierte en líder y ejecuta las tareas exclusivas. Las demás instancias quedan en espera y están listas para tomar el liderazgo si el líder actual falla o renuncia. Este mecanismo es más simple que Zookeeper o etcd para este propósito, pero requiere que el ServiceAccount tenga permisos de **escritura** sobre ConfigMaps.

> **[PREREQUISITO]** Leader Election requiere añadir el verbo `update` (y opcionalmente `create`/`patch`) sobre `configmaps` al Role del ServiceAccount. El Role mínimo de [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) solo incluye `get`/`list`/`watch` y debe ser ampliado explícitamente para este módulo.

> **[ADVERTENCIA]** El permiso `update` sobre `configmaps` concedido para Leader Election permite también modificar el ConfigMap de configuración de la aplicación (el que SCK usa como PropertySource). En entornos de alta seguridad, crear un ConfigMap dedicado para el lock y restringir el `update` a ese ConfigMap específico usando un `ResourceName` en el Role.

## Representación visual

El diagrama muestra el ciclo de vida del leader election y las transiciones entre instancias candidatas.

```
Kubernetes Cluster — 3 réplicas de mi-app
                │
    ┌───────────┼───────────┐
    │           │           │
  Pod-1       Pod-2       Pod-3
 (Candidato) (Candidato) (Candidato)
    │           │           │
    └─────────┬─┘           │
              │  Todos intentan escribir en:
              ▼
  ConfigMap "leader-lock" (namespace: produccion)
  {
    "leader": "pod-1",          ← Primera escritura gana
    "acquireTime": "2025-04-16T10:00:00Z",
    "renewTime":   "2025-04-16T10:00:10Z"
  }
              │
    ┌─────────┴──────────┐
    │                    │
  Pod-1 gana          Pod-2 y Pod-3 pierden
  onGranted() llamado  esperan hasta lease-duration
    │                    │
    ▼                    ▼
  Ejecuta tareas       Intentan de nuevo si
  de líder             Pod-1 deja de renovar
                       (renew-deadline excedido)
```

> **[CONCEPTO]** El mecanismo de lock usa las anotaciones del ConfigMap y el `resourceVersion` de Kubernetes como sistema de optimistic locking. Solo una instancia puede hacer `update` con un `resourceVersion` concreto; las demás reciben un error `409 Conflict` y reintentarán tras `retry-period`. Kubernetes garantiza que solo una actualización con el mismo `resourceVersion` tiene éxito, lo que hace el lock atómico sin necesidad de un coordinador externo como ZooKeeper.

## Ejemplo central

El siguiente ejemplo muestra la implementación completa de Leader Election con Spring Cloud Kubernetes: configuración, implementación de `Candidate` y un servicio que ejecuta tareas periódicas solo cuando es líder.

**pom.xml**:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
  </dependency>
  <!-- Módulo específico de leader election -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-kubernetes-fabric8-leader</artifactId>
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
    name: reconciliador-service
  config:
    import: "kubernetes:configmap/reconciliador-config"

spring:
  cloud:
    kubernetes:
      leader:
        enabled: true
        config-map-name: reconciliador-leader-lock   # ConfigMap dedicado para el lock
        namespace: produccion
        role: reconciliador-lider                    # Rol del candidato (identificador lógico)
        lease-duration: 15          # Segundos que dura el liderazgo sin renovación
        renew-deadline: 10          # Segundos máximos para renovar antes de perder liderazgo
        retry-period: 2             # Segundos entre reintentos de candidatos no-líderes
```

**Implementación de Candidate**:

```java
package com.ejemplo.reconciliador.leader;

import org.springframework.cloud.kubernetes.commons.leader.Candidate;
import org.springframework.cloud.kubernetes.commons.leader.Context;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class ReconciliadorCandidate implements Candidate {

    private static final Logger log = LoggerFactory.getLogger(ReconciliadorCandidate.class);

    private volatile Context leaderContext;

    @Override
    public String getId() {
        // Identificador único de esta instancia — típicamente el nombre del pod
        return System.getenv("HOSTNAME");
    }

    @Override
    public String getRole() {
        return "reconciliador-lider";
    }

    @Override
    public void onGranted(Context context) {
        log.info("Liderazgo obtenido por instancia: {}", getId());
        this.leaderContext = context;
        // Aquí se puede publicar un evento Spring o activar un flag
    }

    @Override
    public void onRevoked(Context context) {
        log.info("Liderazgo perdido por instancia: {}", getId());
        this.leaderContext = null;
    }

    public boolean esLider() {
        return leaderContext != null && leaderContext.isLeader();
    }

    public void cederLiderazgo() {
        if (leaderContext != null) {
            leaderContext.yield();
        }
    }
}
```

**Servicio con tarea exclusiva del líder**:

```java
package com.ejemplo.reconciliador.service;

import com.ejemplo.reconciliador.leader.ReconciliadorCandidate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class ReconciliadorService {

    private static final Logger log = LoggerFactory.getLogger(ReconciliadorService.class);

    private final ReconciliadorCandidate candidate;

    public ReconciliadorService(ReconciliadorCandidate candidate) {
        this.candidate = candidate;
    }

    @Scheduled(fixedDelay = 30000)   // Cada 30 segundos en todas las instancias
    public void ejecutarReconciliacion() {
        if (!candidate.esLider()) {
            log.debug("No es líder — saltando reconciliación");
            return;
        }
        log.info("Ejecutando reconciliación como líder");
        // Lógica exclusiva del líder
        reconciliarEstado();
    }

    private void reconciliarEstado() {
        // Implementación real de la reconciliación
        log.info("Estado reconciliado exitosamente");
    }
}
```

**Clase principal** con `@EnableScheduling`:

```java
package com.ejemplo.reconciliador;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableDiscoveryClient
@EnableScheduling
public class ReconciliadorApplication {

    public static void main(String[] args) {
        SpringApplication.run(ReconciliadorApplication.class, args);
    }
}
```

**RBAC ampliado para Leader Election** (agrega `update`/`create`/`patch` al Role):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: reconciliador-role
  namespace: produccion
rules:
  # Lectura general (ConfigMap config, discovery)
  - apiGroups: [""]
    resources: ["configmaps", "services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  # Escritura solo sobre el ConfigMap de lock de leader election
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["reconciliador-leader-lock"]   # Restricción por nombre
    verbs: ["get", "create", "update", "patch"]
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué ocurre si el líder falla repentinamente sin llamar a `yield()`?" El liderazgo permanece "bloqueado" hasta que transcurre el `lease-duration` sin que el líder renueve (falla el `renew-deadline`). Después de ese tiempo, otro candidato intenta adquirir el lock. En la configuración del ejemplo (lease=15s, renew=10s), el máximo tiempo de "leader vacante" es 15 segundos. Este es el tradeoff entre latencia de failover y estabilidad del liderazgo.

## Tabla de elementos clave

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.leader.enabled` | boolean | `true` | Activa el módulo de leader election |
| `spring.cloud.kubernetes.leader.config-map-name` | String | `leaders` | Nombre del ConfigMap usado como lock |
| `spring.cloud.kubernetes.leader.namespace` | String | namespace del pod | Namespace del ConfigMap de lock |
| `spring.cloud.kubernetes.leader.role` | String | — | Rol lógico del candidato (múltiples roles posibles por app) |
| `spring.cloud.kubernetes.leader.lease-duration` | int (s) | `15` | Duración del liderazgo sin renovación (s) |
| `spring.cloud.kubernetes.leader.renew-deadline` | int (s) | `10` | Tiempo máximo para renovar antes de perder el liderazgo (s) |
| `spring.cloud.kubernetes.leader.retry-period` | int (s) | `2` | Intervalo entre reintentos de candidatos no-líderes (s) |

## Buenas y malas prácticas

**Hacer:**

- Usar un ConfigMap dedicado para el lock (`config-map-name` distinto al ConfigMap de configuración) y restringir el permiso `update` a ese ConfigMap específico via `resourceNames` en el Role: evita que el código de leader election pueda modificar la configuración de la aplicación.
- Implementar la lógica de negocio del líder como tareas periódicas con `@Scheduled` que comprueban `candidate.esLider()` antes de ejecutar: más robusto que activar/desactivar el scheduler en `onGranted`/`onRevoked`, que puede perder eventos en condiciones de alta carga.
- Configurar `lease-duration` > `renew-deadline` > `retry-period`: es la relación invariante del algoritmo de leader election. Si `renew-deadline >= lease-duration`, el líder puede perder el liderazgo incluso si está funcionando correctamente.

**Evitar:**

- Poner lógica irreversible (envío de emails, transacciones financieras) directamente en `onGranted()`: este callback puede llamarse durante el arranque antes de que el contexto Spring esté completamente inicializado. Usar eventos de aplicación (`ApplicationEventPublisher`) para garantizar que la lógica se ejecuta solo cuando la aplicación está lista.
- Usar `lease-duration` muy corto (< 5s) en clusters con latencia alta en el API server: con alta latencia, el líder puede fallar en renovar el lock dentro del `renew-deadline` aunque esté completamente funcional, causando cambios de liderazgo innecesarios (split-brain).
- Ignorar el callback `onRevoked()`: es el momento para limpiar recursos, cerrar conexiones exclusivas o persistir el estado del trabajo en curso antes de cederlo al nuevo líder.

---

← [9.10 Health Indicators de Spring Cloud Kubernetes](sc-kubernetes-health.md) | [Índice (README.md)](README.md) | [9.12 Testing de Spring Cloud Kubernetes](sc-kubernetes-testing.md) →
