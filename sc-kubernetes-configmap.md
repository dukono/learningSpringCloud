# 9.2 Spring Cloud Kubernetes — ConfigMap como PropertySource

← [9.1 Motivación y Arquitectura](sc-kubernetes-arquitectura.md) | [Índice](README.md) | [9.3 Secrets como PropertySource](sc-kubernetes-secrets.md) →

---

## Introducción

Spring Cloud Kubernetes permite usar un ConfigMap de Kubernetes como fuente de propiedades de Spring Boot, de la misma forma que se usaría un fichero `application.yml` o un Config Server. Cuando la aplicación arranca, el framework lee el ConfigMap indicado a través de la API de Kubernetes y lo integra en el `Environment` de Spring, con mayor precedencia que las propiedades locales. Esto elimina la necesidad de un Spring Cloud Config Server para externalizar configuración en entornos K8s nativos.

## Diagrama de flujo de carga

El siguiente diagrama muestra cómo Spring Cloud Kubernetes carga el ConfigMap durante el arranque de la aplicación y lo inyecta en el `Environment` de Spring.

```
Arranque Spring Boot
        │
        ▼
BootstrapContext / ApplicationContext
        │
        ▼
KubernetesConfigDataLoader
        │ lee spring.cloud.kubernetes.config.*
        ▼
API K8s: GET /api/v1/namespaces/{ns}/configmaps/{name}
        │
        ▼
ConfigMapPropertySource
        │ se añade al Environment con mayor precedencia
        ▼
@Value / @ConfigurationProperties resueltos
```

> [CONCEPTO] El ConfigMap puede contener datos en formato `key=value` (estilo `application.properties`) o como un bloque YAML bajo la clave `application.yml`. Spring Cloud Kubernetes detecta automáticamente el formato y convierte el contenido en propiedades de Spring.

> [CONCEPTO] La propiedad `spring.cloud.kubernetes.config.sources` permite declarar una lista de ConfigMaps (con nombre y namespace opcionales) cuando se necesita cargar más de un ConfigMap. Es la alternativa a `config.name` para escenarios con múltiples fuentes.

> [PREREQUISITO] El ServiceAccount del pod necesita el verbo `get` (y `watch` para reload) sobre el recurso `configmaps` en el namespace correspondiente.

## Ejemplo central

El siguiente ejemplo cubre el escenario completo: un ConfigMap K8s con datos en formato YAML embebido, la configuración Spring que lo referencia, y una clase que lee las propiedades inyectadas.

```yaml
# kubernetes/configmap.yaml — ConfigMap con application.yml embebido
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-service
  namespace: default
  labels:
    app: my-service
data:
  application.yml: |
    app:
      message: "Hello from ConfigMap"
      max-retries: 3
    server:
      port: 8080
```

```yaml
# kubernetes/configmap-prod.yaml — ConfigMap de perfil activo
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-service-prod
  namespace: default
data:
  application.yml: |
    app:
      message: "Hello from ConfigMap PROD"
      max-retries: 5
```

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: my-service
  profiles:
    active: prod
  cloud:
    kubernetes:
      config:
        enabled: true
        name: my-service           # nombre del ConfigMap principal
        namespace: default         # namespace donde buscar
        # sources permite múltiples ConfigMaps:
        sources:
          - name: my-service
            namespace: default
          - name: shared-config
            namespace: infrastructure
```

```java
// src/main/java/com/example/AppProperties.java
package com.example;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {

    private String message;
    private int maxRetries;

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public int getMaxRetries() {
        return maxRetries;
    }

    public void setMaxRetries(int maxRetries) {
        this.maxRetries = maxRetries;
    }
}
```

```java
// src/main/java/com/example/ConfigController.java
package com.example;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ConfigController {

    private final AppProperties props;

    public ConfigController(AppProperties props) {
        this.props = props;
    }

    @GetMapping("/config")
    public String config() {
        return props.getMessage() + " (retries=" + props.getMaxRetries() + ")";
    }
}
```

## Tabla de elementos clave

La siguiente tabla detalla todas las propiedades relevantes para configurar ConfigMap como PropertySource.

| Propiedad | Valor por defecto | Descripción |
|---|---|---|
| `spring.cloud.kubernetes.config.enabled` | `true` | Activa el PropertySource de ConfigMap |
| `spring.cloud.kubernetes.config.name` | `${spring.application.name}` | Nombre del ConfigMap a cargar |
| `spring.cloud.kubernetes.config.namespace` | namespace del pod | Namespace donde buscar el ConfigMap |
| `spring.cloud.kubernetes.config.sources[]` | — | Lista de fuentes (nombre + namespace) para múltiples ConfigMaps |
| `spring.cloud.kubernetes.config.fail-fast` | `false` | Lanza excepción si el ConfigMap no existe al arrancar |
| `spring.cloud.kubernetes.config.use-name-as-prefix` | `false` | Prefija todas las propiedades con el nombre del ConfigMap |

## Soporte de perfiles Spring

Spring Cloud Kubernetes aplica soporte de perfiles al cargar ConfigMaps: además del ConfigMap con el nombre base (`my-service`), también carga automáticamente el ConfigMap `my-service-{profile}` si existe. Las propiedades del ConfigMap de perfil sobrescriben las del ConfigMap base, respetando la misma lógica de precedencia que los perfiles de Spring Boot.

Por ejemplo, con `spring.profiles.active=prod` la aplicación cargará primero `my-service` y luego sobreescribirá con `my-service-prod`. Esto permite tener configuración común en el ConfigMap base y configuración específica de entorno en los ConfigMaps de perfil, sin duplicar propiedades.

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar el formato `application.yml` embebido en el ConfigMap para mantener la misma estructura de propiedades que en el fichero local, facilitando la portabilidad entre entornos locales y K8s.
- Usar `spring.cloud.kubernetes.config.sources` cuando se necesitan múltiples ConfigMaps (p.ej., uno compartido de infraestructura y uno específico del servicio).
- Activar `fail-fast: true` en producción para detectar errores de RBAC o nombre incorrecto durante el arranque, en lugar de ejecutarse con valores por defecto silenciosamente.
- Aprovechar el soporte de perfiles (`my-service-prod`) para separar configuración de entorno sin duplicar propiedades.

**Malas prácticas:**
- Almacenar credenciales o secretos en un ConfigMap: los ConfigMaps no están cifrados; las credenciales deben ir en Secrets de Kubernetes.
- Usar `config.name` y `config.sources` a la vez: la propiedad `sources` tiene precedencia y puede generar confusión sobre qué se carga.
- Ignorar los permisos RBAC sobre `configmaps`: sin `get` el arranque falla con `403 Forbidden`.

> [ADVERTENCIA] La precedencia de las propiedades del ConfigMap es más alta que `application.properties` local pero más baja que las variables de entorno del sistema. Si una variable de entorno sobreescribe una propiedad del ConfigMap, el comportamiento puede ser inesperado.

## Verificación y práctica

> [EXAMEN] 1. ¿Cómo se configura Spring Cloud Kubernetes para leer propiedades desde un ConfigMap llamado `shared-config` en el namespace `infrastructure` además del ConfigMap propio del servicio?

> [EXAMEN] 2. ¿Qué nombre de ConfigMap carga automáticamente Spring Cloud Kubernetes cuando `spring.profiles.active=staging` y `spring.application.name=order-service`?

> [EXAMEN] 3. ¿Cuál es la diferencia entre declarar propiedades en un ConfigMap como `application.properties` embebido versus como pares clave-valor directos en `data:`?

> [EXAMEN] 4. ¿Qué ocurre si `spring.cloud.kubernetes.config.fail-fast=false` y el ConfigMap indicado no existe en el namespace?

> [EXAMEN] 5. ¿Qué precedencia tienen las propiedades del ConfigMap respecto a las de `application.yml` local y las variables de entorno?

---

← [9.1 Motivación y Arquitectura](sc-kubernetes-arquitectura.md) | [Índice](README.md) | [9.3 Secrets como PropertySource](sc-kubernetes-secrets.md) →
