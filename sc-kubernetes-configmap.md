# 9.2 ConfigMap PropertySource — lectura y fuentes

← [9.1 Setup y starters de Spring Cloud Kubernetes](sc-kubernetes-setup.md) | [Índice (README.md)](README.md) | [9.3 ConfigMap PropertySource — recarga dinámica](sc-kubernetes-configmap-reload.md) →

---

## Introducción

En Kubernetes, `ConfigMap` es el objeto estándar para externalizar configuración no sensible: URLs de servicios dependientes, parámetros de negocio, timeouts, flags de feature. Sin SCK, una aplicación Spring Boot solo puede acceder al ConfigMap si lo lee directamente vía la Kubernetes API o si el operador lo monta como fichero de volumen o variable de entorno. Spring Cloud Kubernetes transforma automáticamente el contenido de un ConfigMap en un `PropertySource` de Spring, haciéndolo disponible como cualquier propiedad de `application.yml` sin que el código conozca Kubernetes. Este nodo cubre exclusivamente la lectura estática del ConfigMap (qué fuentes se cargan y cómo se resuelven); la recarga dinámica cuando el ConfigMap cambia se trata en [9.3 ConfigMap PropertySource — recarga dinámica](sc-kubernetes-configmap-reload.md).

> **[PREREQUISITO]** Es necesario haber configurado el starter y el RBAC mínimo descritos en [9.1 Setup y starters de Spring Cloud Kubernetes](sc-kubernetes-setup.md). Sin permiso `get`/`list`/`watch` sobre `configmaps` en el namespace, SCK lanza un error 403 en el arranque.

> **[ADVERTENCIA]** Usar `spring.config.import` en `application.yml` en lugar de `bootstrap.yml` (Spring Boot 4.x). Un `bootstrap.yml` sin la dependencia `spring-cloud-starter-bootstrap` no se procesa en Oakwood.

## Representación visual

SCK resuelve el PropertySource de un ConfigMap siguiendo un orden de fuentes definido. El diagrama muestra el flujo desde el arranque de la aplicación hasta la inyección de propiedades.

```
Arranque Spring Boot 4.x
        │
        ▼
spring.config.import: "kubernetes:configmap/mi-app-config"
        │
        ▼
┌────────────────────────────────────────────────────────┐
│         ConfigMapPropertySourceLocator (SCK)           │
│                                                        │
│  1. Busca ConfigMap por nombre (config.name)           │
│     en namespace (config.namespace)                    │
│  2. Si config.sources → lista de múltiples ConfigMaps  │
│  3. Si config.paths   → lee desde volumen montado      │
│  4. Aplica perfil activo: busca clave                  │
│     "application-{profile}.properties" en el ConfigMap │
└────────────────────────────────────────────────────────┘
        │
        ▼
Spring Environment (PropertySource con prioridad alta)
        │
        ▼
@Value / @ConfigurationProperties inyectados
```

> **[CONCEPTO]** El `ConfigMapPropertySourceLocator` se ejecuta durante la fase de inicialización del `Environment`, antes de que los beans de la aplicación sean instanciados, garantizando que las propiedades del ConfigMap estén disponibles para `@ConfigurationProperties`.

## Ejemplo central

El siguiente ejemplo muestra una aplicación que lee configuración de dos fuentes: un ConfigMap principal y un ConfigMap adicional de base de datos, con soporte de perfil `prod`.

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
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: pedidos-service
  config:
    import: "kubernetes:configmap/pedidos-config,kubernetes:configmap/db-config"
  profiles:
    active: prod

spring:
  cloud:
    kubernetes:
      config:
        enabled: true
        name: pedidos-config
        namespace: produccion
        sources:
          - name: pedidos-config
            namespace: produccion
          - name: db-config
            namespace: produccion
```

**ConfigMap pedidos-config** (manifiesto Kubernetes):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pedidos-config
  namespace: produccion
data:
  application.properties: |
    pedidos.max-por-pagina=50
    pedidos.timeout-ms=3000
  application-prod.properties: |
    pedidos.max-por-pagina=100
    pedidos.timeout-ms=1500
```

**ConfigMap db-config** (manifiesto Kubernetes):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: produccion
data:
  application.properties: |
    spring.datasource.url=jdbc:postgresql://postgres-svc:5432/pedidos
    spring.datasource.username=pedidos_user
```

**Clase de configuración Java**:

```java
package com.ejemplo.pedidos.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "pedidos")
public class PedidosProperties {

    private int maxPorPagina = 20;
    private long timeoutMs = 5000;

    public int getMaxPorPagina() {
        return maxPorPagina;
    }

    public void setMaxPorPagina(int maxPorPagina) {
        this.maxPorPagina = maxPorPagina;
    }

    public long getTimeoutMs() {
        return timeoutMs;
    }

    public void setTimeoutMs(long timeoutMs) {
        this.timeoutMs = timeoutMs;
    }
}
```

**Lectura de propiedad desde volumen montado** — alternativa via `paths`:

```yaml
# application.yml con paths (el ConfigMap está montado como volumen en /etc/config)
spring:
  cloud:
    kubernetes:
      config:
        paths:
          - /etc/config/pedidos
```

```yaml
# Deployment con volumen montado
spec:
  volumes:
    - name: config-vol
      configMap:
        name: pedidos-config
  containers:
    - name: pedidos-service
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config/pedidos
```

> **[EXAMEN]** Pregunta frecuente: "¿Qué pasa si el ConfigMap tiene una clave `application-prod.properties` y el perfil activo es `prod`?" SCK carga ambas: primero `application.properties` (base) y luego `application-prod.properties` (override de perfil), con la misma semántica de precedencia que los ficheros de `application.yml` por perfil en Spring Boot.

## Tabla de elementos clave

Las propiedades siguientes controlan qué ConfigMaps se leen y desde qué fuentes. Son las configuradas en el arranque de cualquier aplicación SCK en Kubernetes.

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.config.enabled` | boolean | `true` | Activa el ConfigMap PropertySource |
| `spring.cloud.kubernetes.config.name` | String | `${spring.application.name}` | Nombre del ConfigMap principal |
| `spring.cloud.kubernetes.config.namespace` | String | namespace del pod | Namespace donde buscar el ConfigMap |
| `spring.cloud.kubernetes.config.paths` | List\<String\> | — | Rutas de ficheros de un volumen montado con el ConfigMap |
| `spring.cloud.kubernetes.config.sources` | List | — | Lista de pares `{name, namespace}` para múltiples ConfigMaps |
| `spring.cloud.kubernetes.config.use-name-as-prefix` | boolean | `false` | Si `true`, prefija las claves con el nombre del ConfigMap para evitar colisiones |
| `spring.config.import` | String | — | Declaración explícita de importación en Spring Boot 4.x |

## Buenas y malas prácticas

**Hacer:**

- Separar la configuración por responsabilidad en varios ConfigMaps (uno por servicio dependiente, uno por parámetros de negocio) y listarlos en `config.sources`: facilita los permisos RBAC granulares y el versionado independiente de cada ConfigMap.
- Usar `application-{profile}.properties` dentro del ConfigMap para sobrescribir valores por entorno: mantiene un único ConfigMap por aplicación en lugar de un ConfigMap por entorno.
- Usar `config.paths` cuando la aplicación necesita ficheros de configuración no-propiedades (por ejemplo, un fichero `logback.xml` o un JSON de configuración): el volumen montado soporta cualquier formato.
- Activar `use-name-as-prefix=true` cuando se leen varios ConfigMaps con claves del mismo nombre: evita que un ConfigMap sobrescriba silenciosamente las propiedades de otro.

**Evitar:**

- Poner credenciales (contraseñas, tokens API) en ConfigMaps: Kubernetes los almacena en texto plano en etcd. Usar Secrets PropertySource (ver [9.4 Secrets PropertySource](sc-kubernetes-secrets.md)).
- Depender de que el ConfigMap siempre esté presente en el arranque sin gestión de errores: si el ConfigMap no existe y `config.enabled=true`, la aplicación lanza `IllegalStateException` durante la inicialización del `Environment`. Usar `spring.cloud.kubernetes.config.fail-fast=false` solo en desarrollo.
- Usar claves con puntos en el nombre del ConfigMap data (ej: `mi.clave.profunda`) sin verificar que el tipo de destino en `@ConfigurationProperties` los resuelva correctamente: SCK los convierte con la misma lógica de relaxed binding de Spring Boot.

---

← [9.1 Setup y starters de Spring Cloud Kubernetes](sc-kubernetes-setup.md) | [Índice (README.md)](README.md) | [9.3 ConfigMap PropertySource — recarga dinámica](sc-kubernetes-configmap-reload.md) →
