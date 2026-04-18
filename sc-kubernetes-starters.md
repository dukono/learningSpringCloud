# 9.5 Spring Cloud Kubernetes — Starters: Fabric8 vs Cliente Oficial

← [9.4 Service Discovery](sc-kubernetes-discovery.md) | [Índice](README.md) | [9.6 Health Indicators y Actuator](sc-kubernetes-health.md) →

---

## Introducción

Spring Cloud Kubernetes ofrece dos líneas de starters basadas en clientes HTTP distintos para comunicarse con la API de Kubernetes: el cliente Java oficial (`io.kubernetes:client-java`, promovido por la CNCF) y el cliente Fabric8 (`io.fabric8:kubernetes-client`, con mayor historia en el ecosistema Java). Seleccionar el starter correcto determina las dependencias transitivas del proyecto y afecta a módulos adicionales como leader election. Mezclar ambos en el mismo `pom.xml` genera conflictos de classpath que son difíciles de diagnosticar.

## Comparativa de starters

La siguiente tabla compara los dos ecosistemas de starters para facilitar la decisión de selección.

| Aspecto | Cliente oficial (client-java) | Cliente Fabric8 |
|---|---|---|
| Artefacto cliente base | `io.kubernetes:client-java` | `io.fabric8:kubernetes-client` |
| Starter config+discovery | `spring-cloud-starter-kubernetes-client` | `spring-cloud-starter-kubernetes-fabric8` |
| Starter all-in-one | `spring-cloud-starter-kubernetes-client-all` | `spring-cloud-starter-kubernetes-fabric8-all` |
| Starter leader election | `spring-cloud-kubernetes-client-leader` | `spring-cloud-kubernetes-fabric8-leader` |
| Respaldo CNCF | Sí (cliente oficial del proyecto Kubernetes) | No (proyecto independiente de Red Hat/Fabric8) |
| Recomendación desde SC 2022.x | Recomendado | Alternativo / legado |
| Soporte de Watch/Informers | Sí (ApiClient + Watch) | Sí (Watcher + Informers propios) |

> [CONCEPTO] Desde Spring Cloud 2022.x (correspondiente a Spring Boot 3.x), el starter `spring-cloud-starter-kubernetes-client` y su variante `*-all` son los starters recomendados por defecto. El starter Fabric8 sigue siendo compatible pero no es el recomendado para proyectos nuevos.

> [ADVERTENCIA] No incluir simultáneamente `spring-cloud-starter-kubernetes-client` y `spring-cloud-starter-kubernetes-fabric8` en el mismo proyecto. Ambos definen beans de auto-configuración con el mismo nombre que entran en conflicto y pueden causar `NoSuchBeanDefinitionException` o comportamientos inesperados en runtime.

> [PREREQUISITO] El `spring-cloud-starter-kubernetes-client-all` incluye: config (ConfigMap/Secret PropertySource), discovery (KubernetesDiscoveryClient) y Spring Cloud LoadBalancer. Para proyectos que solo necesitan una funcionalidad concreta, usar los starters individuales es preferible.

## Ejemplo central

El siguiente ejemplo muestra cómo declarar cada starter en Maven y cómo se ve el pom.xml según la elección del cliente.

```xml
<!-- pom.xml — OPCIÓN 1: Cliente oficial (recomendado para proyectos nuevos) -->
<dependencies>
    <!-- Todo en uno: config + discovery + loadbalancer -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
    </dependency>

    <!-- O starters individuales si solo necesitas config: -->
    <!--
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-client-discovery</artifactId>
    </dependency>
    -->

    <!-- Leader election con cliente oficial -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-kubernetes-client-leader</artifactId>
    </dependency>
</dependencies>
```

```xml
<!-- pom.xml — OPCIÓN 2: Cliente Fabric8 (proyectos que ya lo usan) -->
<dependencies>
    <!-- Todo en uno Fabric8 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
    </dependency>

    <!-- Leader election con Fabric8 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-kubernetes-fabric8-leader</artifactId>
    </dependency>
</dependencies>
```

```xml
<!-- pom.xml — BOM de Spring Cloud para gestión de versiones -->
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
```

```java
// src/main/java/com/example/StarterVerificationBean.java
package com.example;

import io.kubernetes.client.openapi.ApiClient;
import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

/**
 * Verifica que el cliente oficial está en classpath comprobando
 * que ApiClient (io.kubernetes:client-java) está disponible.
 * Si se usara Fabric8 tendríamos io.fabric8.kubernetes.client.KubernetesClient.
 */
@Component
public class StarterVerificationBean implements InfoContributor {

    private final ApiClient apiClient;

    public StarterVerificationBean(ApiClient apiClient) {
        this.apiClient = apiClient;
    }

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("k8s-client", "official")
               .withDetail("k8s-basePath", apiClient.getBasePath());
    }
}
```

## Tabla de starters disponibles

La siguiente tabla lista todos los starters de Spring Cloud Kubernetes organizados por cliente base y funcionalidad.

| Starter | Cliente | Funcionalidad incluida |
|---|---|---|
| `spring-cloud-starter-kubernetes-client` | Oficial | Config + Discovery |
| `spring-cloud-starter-kubernetes-client-all` | Oficial | Config + Discovery + LoadBalancer |
| `spring-cloud-starter-kubernetes-client-config` | Oficial | Solo ConfigMap/Secret PropertySource |
| `spring-cloud-starter-kubernetes-client-discovery` | Oficial | Solo KubernetesDiscoveryClient |
| `spring-cloud-kubernetes-client-leader` | Oficial | Leader election |
| `spring-cloud-starter-kubernetes-fabric8` | Fabric8 | Config + Discovery |
| `spring-cloud-starter-kubernetes-fabric8-all` | Fabric8 | Config + Discovery + LoadBalancer |
| `spring-cloud-kubernetes-fabric8-leader` | Fabric8 | Leader election |

## Buenas y malas prácticas

**Buenas prácticas:**
- Usar `spring-cloud-starter-kubernetes-client-all` en proyectos nuevos para obtener todas las funcionalidades con una única dependencia alineada con la CNCF.
- Verificar que no hay dependencias transitivas que traigan el cliente Fabric8 cuando se usa el cliente oficial: ejecutar `mvn dependency:tree | grep fabric8`.
- Mantener consistencia: si el proyecto ya usa el cliente Fabric8 para otras integraciones (Tekton, OpenShift), usar el starter Fabric8 evita tener dos clientes K8s en classpath.
- Incluir solo los starters individuales necesarios cuando el binario final tiene restricciones de tamaño (entornos serverless o imágenes GraalVM).

**Malas prácticas:**
- Mezclar starters de ambas familias en el mismo proyecto.
- No declarar el BOM de Spring Cloud y gestionar versiones individuales de cada starter: genera desajustes de versión entre módulos.
- Usar el starter `*-all` en proyectos que solo necesitan config: incluye componentes de discovery y loadbalancer innecesarios que añaden beans al contexto.

## Verificación y práctica

> [EXAMEN] 1. ¿En qué se diferencia `spring-cloud-starter-kubernetes-client` de `spring-cloud-starter-kubernetes-fabric8` en términos del cliente HTTP subyacente de Kubernetes?

> [EXAMEN] 2. ¿Cuál es el riesgo de incluir tanto `spring-cloud-starter-kubernetes-client` como `spring-cloud-starter-kubernetes-fabric8` en el mismo `pom.xml`?

> [EXAMEN] 3. ¿Qué dependencia se necesita para usar leader election con el cliente oficial de Kubernetes?

> [EXAMEN] 4. ¿Qué incluye el starter `spring-cloud-starter-kubernetes-client-all` que no incluye `spring-cloud-starter-kubernetes-client`?

> [EXAMEN] 5. Un equipo ya usa el cliente Fabric8 en otros módulos del proyecto. ¿Qué starter de Spring Cloud Kubernetes debería usar para evitar conflictos de classpath?

---

← [9.4 Service Discovery](sc-kubernetes-discovery.md) | [Índice](README.md) | [9.6 Health Indicators y Actuator](sc-kubernetes-health.md) →
