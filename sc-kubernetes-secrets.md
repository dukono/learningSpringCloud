# 9.4 Secrets PropertySource

← [9.3 ConfigMap PropertySource — recarga dinámica](sc-kubernetes-configmap-reload.md) | [Índice (README.md)](README.md) | [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) →

---

## Introducción

Kubernetes `Secret` es el objeto destinado a almacenar datos sensibles: contraseñas, tokens, certificados y claves API. A diferencia del `ConfigMap`, Kubernetes aplica controles adicionales sobre los Secrets (almacenamiento cifrado en etcd con `EncryptionConfiguration`, acceso restringido por RBAC por defecto, omisión en logs del cluster). Spring Cloud Kubernetes puede transformar un `Secret` en un `PropertySource` de la misma forma que lo hace con un `ConfigMap`, pero esta funcionalidad está **desactivada por defecto** precisamente porque requiere que el ServiceAccount de la aplicación tenga permisos de lectura sobre el recurso `secrets`, lo que amplía la superficie de ataque. Este nodo explica cómo activar y configurar el Secrets PropertySource de forma segura, incluyendo la lectura desde volumen montado como alternativa más segura, y la rotación de credenciales en caliente.

> **[PREREQUISITO]** Comprender el RBAC de SCK descrito en [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) antes de activar este PropertySource. El verbo `get`/`list`/`watch` sobre `secrets` debe añadirse explícitamente al Role del ServiceAccount.

> **[ADVERTENCIA]** Activar `spring.cloud.kubernetes.secrets.enabled=true` hace que la aplicación lea Secrets de Kubernetes vía la API. Si el ServiceAccount tiene permisos excesivos, un atacante que comprometa el pod puede listar todos los Secrets del namespace. La alternativa más segura es montar el Secret como volumen y usar `secrets.paths`.

## Representación visual

Existen dos formas de exponer un Kubernetes Secret como `PropertySource` en Spring. La elección determina el riesgo de seguridad y la complejidad del RBAC necesario.

```
Opción A — API PropertySource (secrets.enabled=true)
┌─────────────────────────────────────────────────────┐
│  Pod (ServiceAccount con get/list/watch secrets)    │
│           │                                         │
│           ▼                                         │
│  SecretsPropertySourceLocator ──► Kubernetes API    │
│           │         Lee Secret por nombre/labels    │
│           ▼                                         │
│  Spring PropertySource (claves en memoria)          │
└─────────────────────────────────────────────────────┘

Opción B — Volumen montado (más segura, recomendada)
┌─────────────────────────────────────────────────────┐
│  Kubernetes monta el Secret como volumen            │
│  en /etc/secrets/mi-app                             │
│           │                                         │
│           ▼                                         │
│  spring.cloud.kubernetes.secrets.paths              │
│           │                                         │
│           ▼                                         │
│  Spring PropertySource (lee ficheros del volumen)   │
│  ← No requiere acceso API al recurso secrets        │
└─────────────────────────────────────────────────────┘
```

> **[CONCEPTO]** Con la opción de volumen montado (`secrets.paths`), el pod no necesita permiso `get`/`list`/`watch` sobre `secrets` en el RBAC. Kubernetes inyecta los valores del Secret en el sistema de ficheros del contenedor, y SCK los lee como ficheros de propiedades. Además, si el Secret se actualiza, Kubernetes actualiza el volumen montado automáticamente (con latencia de hasta 2 minutos).

## Ejemplo central

El siguiente ejemplo muestra las dos estrategias: lectura directa via API con filtrado por etiquetas, y lectura desde volumen montado, que es la opción recomendada en producción.

**pom.xml** (igual al de 9.1, con `spring-cloud-starter-kubernetes-fabric8-all`):

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
</dependency>
```

**application.yml — Opción A (via API)**:

```yaml
spring:
  application:
    name: pagos-service
  config:
    import: "kubernetes:secret/pagos-credentials"

spring:
  cloud:
    kubernetes:
      secrets:
        enabled: true
        name: pagos-credentials
        namespace: produccion
        labels:
          app: pagos-service
          tier: backend
```

**application.yml — Opción B (via volumen montado, recomendada)**:

```yaml
spring:
  application:
    name: pagos-service

spring:
  cloud:
    kubernetes:
      secrets:
        enabled: true
        paths:
          - /etc/secrets/pagos
```

**Secret de Kubernetes** (manifiesto con valores en base64):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pagos-credentials
  namespace: produccion
  labels:
    app: pagos-service
    tier: backend
type: Opaque
data:
  # echo -n "mi-password-seguro" | base64
  db.password: bWktcGFzc3dvcmQtc2VndXJv
  # echo -n "sk-live-xxxx" | base64
  stripe.api-key: c2stbGl2ZS14eHh4
```

**Deployment con volumen montado (Opción B)**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pagos-service
spec:
  template:
    spec:
      serviceAccountName: pagos-sa
      volumes:
        - name: pagos-secrets-vol
          secret:
            secretName: pagos-credentials
      containers:
        - name: pagos-service
          image: mi-empresa/pagos-service:latest
          volumeMounts:
            - name: pagos-secrets-vol
              mountPath: /etc/secrets/pagos
              readOnly: true
```

**Uso en código Java**:

```java
package com.ejemplo.pagos.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class PagosCredentials {

    // SCK expone las claves del Secret como propiedades Spring normales
    @Value("${db.password}")
    private String dbPassword;

    @Value("${stripe.api-key}")
    private String stripeApiKey;

    public String getDbPassword() {
        return dbPassword;
    }

    public String getStripeApiKey() {
        return stripeApiKey;
    }
}
```

**Recarga de Secrets en caliente** (rotación de credenciales):

```yaml
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        strategy: RESTART_CONTEXT   # Secrets normalmente requieren reinicio
        monitoring-secrets: true    # Activar explícitamente
        mode: polling
        period: 60000               # Cada 60 segundos para secrets
```

> **[EXAMEN]** Pregunta frecuente: "¿Por qué `spring.cloud.kubernetes.secrets.enabled` está en `false` por defecto cuando `config.enabled` está en `true`?" Porque los Secrets almacenan datos sensibles y conceder acceso API a todos los Secrets del namespace a cualquier ServiceAccount viola el principio de mínimo privilegio. Los ConfigMaps son no-sensibles por diseño; los Secrets requieren una decisión explícita del operador.

## Tabla de elementos clave

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.secrets.enabled` | boolean | `false` | Activa el Secrets PropertySource (desactivado por seguridad) |
| `spring.cloud.kubernetes.secrets.name` | String | `${spring.application.name}` | Nombre del Secret a leer |
| `spring.cloud.kubernetes.secrets.namespace` | String | namespace del pod | Namespace donde buscar el Secret |
| `spring.cloud.kubernetes.secrets.labels` | Map | — | Filtrado de Secrets por etiquetas (lee todos los que coincidan) |
| `spring.cloud.kubernetes.secrets.paths` | List\<String\> | — | Rutas de volumen montado con el Secret (alternativa más segura) |
| `spring.cloud.kubernetes.secrets.use-name-as-prefix` | boolean | `false` | Prefija las claves con el nombre del Secret para evitar colisiones |
| `spring.cloud.kubernetes.reload.monitoring-secrets` | boolean | `false` | Activa la detección de cambios en Secrets para recarga |

## Buenas y malas prácticas

**Hacer:**

- Preferir la opción de volumen montado (`secrets.paths`) sobre la lectura directa via API: el RBAC del ServiceAccount no necesita acceso a `secrets`, reduciendo la superficie de ataque.
- Usar `secrets.labels` para filtrar Secrets específicos de la aplicación cuando hay múltiples Secrets en el namespace: evita que la aplicación lea Secrets de otros servicios accidentalmente.
- Activar el cifrado en reposo de etcd (`EncryptionConfiguration` en el cluster) para que los Secrets no se almacenen en texto plano en el backend de Kubernetes.
- Para la rotación de credenciales, usar `RESTART_CONTEXT` en lugar de `REFRESH`: los beans de conexión a base de datos o clientes HTTP no son `@RefreshScope` por defecto, y un `REFRESH` no propagaría la nueva contraseña al connection pool.

**Evitar:**

- Loguear propiedades que provengan de Secrets: Spring Boot puede loguear propiedades en modo DEBUG durante el arranque; desactivar `logging.level.org.springframework.boot.context.config=DEBUG` en producción.
- Activar `secrets.enabled=true` sin RBAC explícito y auditado: el error 403 en runtime indica que el ServiceAccount no tiene permisos, pero si el cluster tiene RBAC mal configurado, el pod podría leer Secrets de otros servicios.
- Usar Secrets de Kubernetes para claves de cifrado de alto valor (HSM keys, claves de firma JWT) sin rotación automatizada: considerar Vault u otros gestores de secretos especializados para ese nivel de sensibilidad.

---

← [9.3 ConfigMap PropertySource — recarga dinámica](sc-kubernetes-configmap-reload.md) | [Índice (README.md)](README.md) | [9.5 Namespace awareness y RBAC](sc-kubernetes-namespace-rbac.md) →
