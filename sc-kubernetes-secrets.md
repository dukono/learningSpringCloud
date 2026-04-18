# 9.3 Spring Cloud Kubernetes — Secrets como PropertySource

← [9.2 ConfigMap como PropertySource](sc-kubernetes-configmap.md) | [Índice](README.md) | [9.4 Service Discovery](sc-kubernetes-discovery.md) →

---

## Introducción

Kubernetes Secrets almacena datos sensibles —contraseñas, tokens, claves API— en formato base64 en el cluster. Spring Cloud Kubernetes puede exponer esos Secrets como propiedades del `Environment` de Spring de dos maneras: leyéndolos mediante la API de Kubernetes (`secrets.enabled=true`) o consumiendo el contenido de un Secret montado como volumen o variable de entorno en el pod. Elegir la estrategia correcta tiene implicaciones de seguridad significativas que el examen evalúa directamente.

## Comparación: API vs Volumen

La siguiente tabla resume las diferencias entre ambos enfoques de acceso a Secrets, que es el punto de evaluación más frecuente en el examen VMware Spring Professional.

| Aspecto | API de Kubernetes (`secrets.enabled=true`) | Volumen / Env Var |
|---|---|---|
| Configuración Spring | `spring.cloud.kubernetes.secrets.enabled=true` | Solo Spring Boot estándar (`${SECRET_VAR}`) |
| RBAC necesario | `get` (o `list`) sobre `secrets` en el namespace | Ninguno adicional (K8s monta automáticamente) |
| Riesgo de seguridad | Alto: si el pod se compromete, puede listar otros Secrets con el token SA | Bajo: el pod solo ve el contenido del volumen montado |
| Compatibilidad con rotación de secretos | Requiere reload de configuración (sc-kubernetes-reload.md) | Actualización automática del volumen (en K8s 1.21+) |
| Recomendación | Evitar en producción si hay alternativa con volumen | Preferida por el principio de mínimo privilegio |

> [CONCEPTO] La lectura de Secrets mediante la API de Kubernetes requiere que el ServiceAccount tenga el verbo `get` sobre `secrets`. Esto es más restrictivo que `configmaps` porque los Secrets contienen credenciales. Muchos equipos de seguridad prohíben dar este permiso y exigen el enfoque de volumen.

> [CONCEPTO] Cuando un Secret se monta como volumen en el pod, Kubernetes escribe cada clave del Secret como un fichero independiente en el directorio de montaje. Spring Cloud Kubernetes puede leer esos ficheros usando `spring.cloud.kubernetes.secrets.paths`.

> [ADVERTENCIA] Activar `spring.cloud.kubernetes.secrets.enabled=true` con un ClusterRole que permite `list` sobre Secrets en todos los namespaces es un riesgo crítico de seguridad. Un pod comprometido podría exfiltrar todos los Secrets del clúster.

## Ejemplo central

El siguiente ejemplo muestra ambos enfoques: acceso vía API y montado como volumen, con la configuración RBAC mínima para cada caso.

```yaml
# kubernetes/secret.yaml — Secret con credenciales de base de datos
apiVersion: v1
kind: Secret
metadata:
  name: my-service-db-credentials
  namespace: default
  labels:
    app: my-service
type: Opaque
data:
  db.username: bXl1c2Vy          # base64("myuser")
  db.password: c2VjcmV0cGFzcw==  # base64("secretpass")
```

```yaml
# kubernetes/deployment-volume.yaml — Montado como volumen (RECOMENDADO)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    spec:
      serviceAccountName: my-service-sa
      containers:
        - name: my-service
          image: my-service:1.0.0
          volumeMounts:
            - name: db-credentials
              mountPath: /etc/secrets/db
              readOnly: true
      volumes:
        - name: db-credentials
          secret:
            secretName: my-service-db-credentials
```

```yaml
# src/main/resources/application.yml — Enfoque volumen
spring:
  application:
    name: my-service
  cloud:
    kubernetes:
      secrets:
        enabled: false          # no se usa la API de Secrets
        paths:
          - /etc/secrets/db     # Spring Cloud K8s lee los ficheros del volumen
```

```yaml
# src/main/resources/application-api.yml — Enfoque API (solo si es necesario)
spring:
  cloud:
    kubernetes:
      secrets:
        enabled: true
        name: my-service-db-credentials
        namespace: default
        # Filtrar por label en lugar de por nombre (alternativa)
        labels:
          app: my-service
```

```java
// src/main/java/com/example/DataSourceConfig.java
package com.example;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.sql.DataSource;
import org.springframework.boot.jdbc.DataSourceBuilder;

@Configuration
public class DataSourceConfig {

    // Spring lee db.username del fichero /etc/secrets/db/db.username
    // o de la propiedad inyectada por el PropertySource de Secrets
    @Value("${db.username}")
    private String username;

    @Value("${db.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url("jdbc:postgresql://postgres:5432/mydb")
                .username(username)
                .password(password)
                .build();
    }
}
```

## Tabla de propiedades de Secrets

La siguiente tabla cubre las propiedades más relevantes para configurar el PropertySource de Secrets en Spring Cloud Kubernetes.

| Propiedad | Valor por defecto | Descripción |
|---|---|---|
| `spring.cloud.kubernetes.secrets.enabled` | `false` | Activa la lectura de Secrets vía API de Kubernetes |
| `spring.cloud.kubernetes.secrets.name` | `${spring.application.name}` | Nombre del Secret a cargar |
| `spring.cloud.kubernetes.secrets.namespace` | namespace del pod | Namespace donde buscar el Secret |
| `spring.cloud.kubernetes.secrets.labels` | — | Mapa de etiquetas para filtrar Secrets por label selector |
| `spring.cloud.kubernetes.secrets.paths` | — | Lista de rutas de volúmenes montados con Secrets |
| `spring.cloud.kubernetes.secrets.use-name-as-prefix` | `false` | Prefija propiedades con el nombre del Secret |
| `spring.cloud.kubernetes.secrets.fail-fast` | `false` | Falla si el Secret no existe al arrancar |

## Buenas y malas prácticas

**Buenas prácticas:**
- Preferir siempre el montado como volumen o variables de entorno sobre `secrets.enabled=true` para reducir la superficie de ataque RBAC.
- Si se usa el enfoque de volumen con `secrets.paths`, activar el modo de lectura de ficheros y no conceder ningún permiso adicional sobre `secrets` al ServiceAccount.
- Usar `secrets.labels` para agrupar múltiples Secrets relacionados bajo un selector de etiquetas, en lugar de listar nombres individuales.
- Cifrar los Secrets en reposo con el mecanismo de encryption at rest de Kubernetes (etcd encryption) o usar Vault como backend.

**Malas prácticas:**
- Activar `secrets.enabled=true` y asignar un ClusterRole con `list` sobre `secrets` en todos los namespaces: es un riesgo crítico.
- Almacenar en un ConfigMap (no Secret) valores que son credenciales por considerarlos "menos importantes".
- Usar `secrets.use-name-as-prefix=false` cuando se cargan múltiples Secrets con propiedades de igual nombre: las propiedades se sobreescribirán silenciosamente.

> [EXAMEN] La distinción entre el enfoque API (`secrets.enabled=true`) y el montado como volumen es una pregunta recurrente en el examen VMware Spring Professional. El punto clave es el riesgo de seguridad: el enfoque API requiere permisos RBAC adicionales que amplían la superficie de ataque.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia de seguridad principal entre acceder a un Secret de Kubernetes mediante `spring.cloud.kubernetes.secrets.enabled=true` y montarlo como volumen en el pod?

> [EXAMEN] 2. ¿Qué verbo RBAC adicional necesita el ServiceAccount cuando se activa `spring.cloud.kubernetes.secrets.enabled=true` que no se necesita para los ConfigMaps?

> [EXAMEN] 3. ¿Para qué sirve la propiedad `spring.cloud.kubernetes.secrets.paths` y en qué escenario se usa?

> [EXAMEN] 4. Un equipo de seguridad prohíbe dar permisos `get` sobre Secrets al ServiceAccount del pod. ¿Cómo puede la aplicación Spring Boot seguir accediendo a las credenciales almacenadas en un Secret de Kubernetes?

> [EXAMEN] 5. ¿Qué ocurre cuando dos Secrets distintos contienen una propiedad con el mismo nombre y `use-name-as-prefix=false`?

---

← [9.2 ConfigMap como PropertySource](sc-kubernetes-configmap.md) | [Índice](README.md) | [9.4 Service Discovery](sc-kubernetes-discovery.md) →
