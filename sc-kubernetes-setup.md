# 9.1 Setup y starters de Spring Cloud Kubernetes

← [8.15 Testing / Verificación de Spring Cloud Security](sc-security-testing.md) | [Índice (README.md)](README.md) | [9.2 ConfigMap PropertySource — lectura y fuentes](sc-kubernetes-configmap.md) →

---

## Introducción

Cuando una aplicación Spring Boot se despliega en Kubernetes, la plataforma ya ofrece mecanismos nativos de configuración (ConfigMaps, Secrets), descubrimiento de servicios (DNS y Endpoints API) y observabilidad. Spring Cloud Kubernetes (SCK) actúa como puente entre esos mecanismos nativos y las abstracciones de Spring Cloud (`PropertySource`, `DiscoveryClient`, `HealthIndicator`), de modo que el código de la aplicación no necesita conocer la API de Kubernetes directamente. Sin SCK, el desarrollador tendría que consultar la Kubernetes API manualmente para leer configuración o descubrir servicios, rompiendo la portabilidad del código. SCK existe como módulo separado porque los mecanismos de Kubernetes son distintos a los de Consul, Vault o Eureka: hay dos implementaciones del cliente Java de Kubernetes (fabric8 y el cliente oficial de Kubernetes), y la elección determina qué starter se usa.

> **[PREREQUISITO]** Se asume conocimiento de Kubernetes a nivel usuario: pods, services, configmaps, secrets y namespaces. Los internals de Kubernetes están fuera del alcance (ver meta.md).

> **[ADVERTENCIA]** Spring Boot 4.0.x (Oakwood) no usa `bootstrap.yml` ni `BootstrapContext` legado. Todos los ejemplos de configuración utilizan `spring.config.import` para importar ConfigMaps y Secrets. Un `bootstrap.yml` en Spring Boot 4.x no se procesa y la configuración no se carga.

## Representación visual

Los tres starters de Spring Cloud Kubernetes cubren combinaciones distintas de cliente Java y capacidades incluidas. Elegir el starter correcto evita conflictos de dependencias en runtime.

```
┌─────────────────────────────────────────────────────────────────────┐
│              Spring Cloud Kubernetes — árbol de starters            │
│                                                                     │
│  spring-cloud-starter-kubernetes-fabric8                            │
│  ├── Fabric8 KubernetesClient (fluent DSL)                          │
│  ├── ConfigMap PropertySource                                       │
│  ├── Secrets PropertySource                                         │
│  └── KubernetesDiscoveryClient (fabric8)                            │
│                                                                     │
│  spring-cloud-starter-kubernetes-client                             │
│  ├── Official Kubernetes Java Client (ApiClient)                    │
│  ├── ConfigMap PropertySource                                       │
│  ├── Secrets PropertySource                                         │
│  └── KubernetesInformerDiscoveryClient                              │
│                                                                     │
│  spring-cloud-starter-kubernetes-fabric8-all  ← todo en uno        │
│  ├── spring-cloud-starter-kubernetes-fabric8                        │
│  └── spring-cloud-starter-kubernetes-fabric8-loadbalancer           │
│      └── Spring Cloud LoadBalancer integrado                        │
└─────────────────────────────────────────────────────────────────────┘
```

> **[CONCEPTO]** `spring-cloud-starter-kubernetes-fabric8-all` incluye el starter de LoadBalancer y es la opción recomendada para proyectos nuevos que usan fabric8 y necesitan descubrimiento + balanceo sin añadir dependencias adicionales manualmente.

## Ejemplo central

El siguiente ejemplo muestra la configuración mínima para integrar una aplicación Spring Boot 4.x en un cluster Kubernetes usando el starter fabric8-all, con `spring.config.import` para importar un ConfigMap llamado `mi-app-config`.

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
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: mi-app
  config:
    import: "kubernetes:configmap/mi-app-config"   # Spring Boot 4.x — reemplaza bootstrap.yml

spring:
  cloud:
    kubernetes:
      enabled: true
      config:
        enabled: true
        name: mi-app-config
        namespace: default
      discovery:
        enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

**Clase principal**:

```java
package com.ejemplo.kubernetes;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class MiAppKubernetesApplication {

    public static void main(String[] args) {
        SpringApplication.run(MiAppKubernetesApplication.class, args);
    }
}
```

**ConfigMap en Kubernetes** (YAML de manifiesto, no un fichero de aplicación):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mi-app-config
  namespace: default
data:
  application.properties: |
    mensaje.bienvenida=Hola desde Kubernetes
    server.port=8080
```

**ServiceAccount con RBAC mínimo** (requerido para que SCK lea el ConfigMap):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mi-app-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mi-app-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["configmaps", "pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mi-app-rolebinding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: mi-app-sa
    namespace: default
roleRef:
  kind: Role
  apiRef: mi-app-role
  apiGroup: rbac.authorization.k8s.io
```

> **[EXAMEN]** En una entrevista puede preguntarse: "¿Qué diferencia hay entre `spring-cloud-starter-kubernetes-fabric8` y `spring-cloud-starter-kubernetes-client`?" La respuesta clave: el primero usa Fabric8 (fluent DSL, amplia comunidad Spring Cloud), el segundo usa el cliente oficial de Kubernetes (informers nativos, menor abstracción). La elección del starter determina qué `DiscoveryClient` se autoconfigurar. Ver detalles en [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md).

## Tabla de elementos clave

Las propiedades siguientes controlan la activación global de SCK y el contexto de importación de configuración. Son las primeras que se configuran en cualquier proyecto nuevo.

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.kubernetes.enabled` | boolean | `true` | Activa/desactiva toda la autoconfiguración de SCK |
| `spring.cloud.kubernetes.config.enabled` | boolean | `true` | Habilita ConfigMap PropertySource |
| `spring.cloud.kubernetes.config.name` | String | `${spring.application.name}` | Nombre del ConfigMap principal a leer |
| `spring.cloud.kubernetes.config.namespace` | String | namespace del pod | Namespace donde buscar el ConfigMap |
| `spring.cloud.kubernetes.discovery.enabled` | boolean | `true` | Habilita KubernetesDiscoveryClient |
| `spring.cloud.kubernetes.secrets.enabled` | boolean | `false` | Habilita Secrets PropertySource (desactivado por seguridad) |
| `spring.config.import` | String | — | Importa ConfigMap/Secret via `kubernetes:configmap/<nombre>` |
| `spring.cloud.kubernetes.client.namespace` | String | autodetectado | Override del namespace de operación del cliente K8s |

## Buenas y malas prácticas

**Hacer:**

- Usar `spring-cloud-starter-kubernetes-fabric8-all` en proyectos nuevos con fabric8: incluye LoadBalancer de serie y evita añadir dependencias sueltas que pueden quedar desincronizadas con el BOM.
- Declarar `spring.config.import: "kubernetes:configmap/<nombre>"` en `application.yml` en lugar de `bootstrap.yml`: con Spring Boot 4.x el Bootstrap Context está desactivado por defecto y no se procesa.
- Definir un `ServiceAccount` dedicado por aplicación con un `Role` de permisos mínimos: el pod no necesita acceso de escritura ni acceso a todos los namespaces para leer su propio ConfigMap.
- Incluir `spring.cloud.kubernetes.enabled=false` en perfiles de tests locales (fuera del cluster): evita errores de conexión al API server durante el ciclo de CI en entornos sin Kubernetes.

**Evitar:**

- Usar `spring-cloud-starter-kubernetes-fabric8` y `spring-cloud-starter-kubernetes-client` juntos en el mismo proyecto: los dos starters registran beans con el mismo nombre y causan conflictos de autoconfiguración en runtime.
- Dejar `spring.cloud.kubernetes.secrets.enabled=true` sin RBAC explícito para Secrets: el pod obtendría acceso a todos los Secrets del namespace, lo que viola el principio de mínimo privilegio.
- Copiar ejemplos de la documentación pre-2023 que usan `bootstrap.yml`: fallará silenciosamente en Spring Boot 4.x porque el BootstrapContext no se activa a menos que se añada `spring-cloud-starter-bootstrap` como dependencia adicional (lo que no es la práctica recomendada en Oakwood).

## Comparación de starters

La elección del starter tiene implicaciones en qué `DiscoveryClient` se registra y qué cliente Java se usa para llamar a la API de Kubernetes. La tabla siguiente resume las diferencias operativas más relevantes para la selección inicial.

| Starter | Cliente K8s | DiscoveryClient | LoadBalancer incluido | Caso de uso |
|---|---|---|---|---|
| `kubernetes-fabric8` | Fabric8 (fluent DSL) | `KubernetesDiscoveryClient` | No | Proyectos que ya usan Fabric8 |
| `kubernetes-fabric8-all` | Fabric8 (fluent DSL) | `KubernetesDiscoveryClient` | Sí (SCK LB) | Proyectos nuevos fabric8 (recomendado) |
| `kubernetes-client` | Oficial (ApiClient) | `KubernetesInformerDiscoveryClient` | No | Proyectos que prefieren el cliente oficial |

> **[CONCEPTO]** `KubernetesInformerDiscoveryClient` (cliente oficial) usa informers nativos de Kubernetes para mantener un cache local sincronizado, lo que reduce las llamadas al API server en clusters grandes. Ver [9.9 Kubernetes Java Client: fabric8 vs cliente oficial](sc-kubernetes-client.md) para comparación detallada.

---

← [8.15 Testing / Verificación de Spring Cloud Security](sc-security-testing.md) | [Índice (README.md)](README.md) | [9.2 ConfigMap PropertySource — lectura y fuentes](sc-kubernetes-configmap.md) →
