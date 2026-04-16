# 13.7 Sidecar Pattern con Spring Cloud Netflix Sidecar

← [13.6 API Composition con Spring Cloud Gateway y OpenFeign](sc-patrones-api-composition.md) | [Índice (README.md)](README.md) | [13.8 Resiliencia transversal en patrones distribuidos](sc-patrones-resiliencia-transversal.md) →

---

El Sidecar Pattern permite que aplicaciones no-JVM (Node.js, Python, Go, aplicaciones legacy) participen en la infraestructura de Service Discovery de Spring Cloud Eureka. Spring Cloud Netflix Sidecar actúa como proxy: registra la aplicación non-JVM en Eureka en su nombre, y le da acceso a las APIs de Eureka y Ribbon/LoadBalancer para que pueda descubrir y llamar a otros servicios. El sidecar se despliega como un proceso JVM junto a la aplicación non-JVM (típicamente en el mismo pod de Kubernetes o en el mismo host). En el ecosistema 2025.1.x, el uso de Sidecar es menos frecuente que en versiones anteriores debido a que Service Meshes (Istio, Linkerd) cubren el mismo caso de uso con mayor generalidad.

> [PREREQUISITO] Requiere `spring-cloud-netflix-sidecar`. La aplicación non-JVM debe tener un health endpoint HTTP accesible.

## Ejemplo central: Sidecar para una aplicación Node.js

### Dependencias Maven del Sidecar JVM

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
    <!-- Sidecar: registra la app non-JVM en Eureka -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-netflix-sidecar</artifactId>
    </dependency>
    <!-- Eureka Client: para el registro -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### Aplicación Sidecar Spring Boot

```java
package com.example.sidecar;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.sidecar.EnableSidecar;

@SpringBootApplication
@EnableSidecar  // habilita el proxy Sidecar
public class NodeJsSidecarApplication {

    public static void main(String[] args) {
        SpringApplication.run(NodeJsSidecarApplication.class, args);
    }
}
```

### application.yml del Sidecar

```yaml
server:
  port: 5678  # Puerto del Sidecar JVM

sidecar:
  # Puerto de la aplicación Node.js en el mismo host/pod
  port: 3000
  # Health endpoint de la app Node.js (el Sidecar lo consulta para el health de Eureka)
  health-uri: http://localhost:3000/health

spring:
  application:
    name: nodejs-analytics-service  # Nombre con el que aparece en Eureka

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    # El Sidecar registra la app Node.js con la IP y puerto del sidecar-port
    hostname: nodejs-analytics-host
    prefer-ip-address: true
```

### Aplicación Node.js: usar la API del Sidecar para Service Discovery

```javascript
// app.js — aplicación Node.js que usa el Sidecar para llamar a otros servicios
const express = require('express');
const axios = require('axios');

const app = express();
const SIDECAR_URL = 'http://localhost:5678';

// Health endpoint requerido por el Sidecar
app.get('/health', (req, res) => {
  res.json({ status: 'UP' });
});

// La aplicación Node.js llama a otros servicios a través del Sidecar
// El Sidecar actúa como proxy de Service Discovery
app.get('/analytics/orders', async (req, res) => {
  try {
    // El Sidecar expone /hosts/{serviceId} para obtener las instancias disponibles
    const hostsResponse = await axios.get(
      `${SIDECAR_URL}/hosts/order-service`
    );
    const instances = hostsResponse.data;
    
    if (instances.length === 0) {
      return res.status(503).json({ error: 'order-service not available' });
    }
    
    // Seleccionar una instancia (load balancing manual)
    const instance = instances[Math.floor(Math.random() * instances.length)];
    const serviceUrl = `http://${instance.host}:${instance.port}`;
    
    // Llamar al servicio usando la URL obtenida de Eureka via Sidecar
    const ordersResponse = await axios.get(`${serviceUrl}/orders`);
    res.json(ordersResponse.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Alternativamente: usar el proxy del Sidecar directamente
// El Sidecar expone /{serviceId}/{path} como proxy con LB automático
app.get('/analytics/products', async (req, res) => {
  try {
    // El Sidecar hace el load balancing automáticamente
    const response = await axios.get(
      `${SIDECAR_URL}/product-service/products`
    );
    res.json(response.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(3000, () => console.log('Node.js analytics service running on port 3000'));
```

> [CONCEPTO] El Sidecar expone tres APIs a la aplicación non-JVM: `/hosts/{serviceId}` (devuelve las instancias disponibles del servicio), `/{serviceId}/{path}` (proxy con load balancing automático), y `/ping` (health check del sidecar). La aplicación non-JVM no necesita ninguna librería Eureka — toda la comunicación con el Service Discovery la hace el Sidecar JVM en su nombre.

> [ADVERTENCIA] Spring Cloud Netflix Sidecar es un componente en modo mantenimiento en Spring Cloud 2025.x. Para nuevos proyectos con aplicaciones polyglot, considerar Service Mesh (Istio, Linkerd) como alternativa más moderna y mantenida. El Sidecar sigue siendo funcional pero no recibirá nuevas características.

## Tabla de elementos clave

| Componente / Endpoint | Descripción |
|---|---|
| `@EnableSidecar` | Activa el Sidecar en la aplicación JVM; habilita el proxy y el registro en Eureka |
| `sidecar.port` | Puerto de la aplicación non-JVM; el Sidecar usa este puerto para registrar en Eureka |
| `sidecar.health-uri` | Health endpoint de la app non-JVM; el Sidecar lo consulta para determinar el estado en Eureka |
| `GET /hosts/{serviceId}` | API del Sidecar que devuelve las instancias de un servicio en formato JSON |
| `GET /{serviceId}/{path}` | Proxy con load balancing automático del Sidecar hacia otros servicios |
| `GET /ping` | Health check del Sidecar mismo; útil para Kubernetes liveness probes |

## Buenas y malas prácticas

**Hacer:**
- Asegurarse de que el health endpoint de la app non-JVM (`sidecar.health-uri`) devuelve el formato esperado por Spring Boot Actuator: `{"status":"UP"}` o `{"status":"DOWN"}`; el Sidecar usa este endpoint para sincronizar el estado en Eureka.
- Desplegar el Sidecar y la aplicación non-JVM en el mismo pod de Kubernetes (contenedores múltiples); usar red de loopback (`localhost`) para la comunicación entre ellos.
- Evaluar Service Mesh (Istio, Linkerd) como alternativa para nuevos proyectos con múltiples aplicaciones polyglot; ofrecen más funcionalidades (mTLS automático, traffic management, observabilidad) que el Sidecar de Spring Cloud.

**Evitar:**
- Usar el Sidecar para aplicaciones JVM que pueden registrarse directamente en Eureka; el Sidecar es específicamente para aplicaciones non-JVM que no pueden usar el cliente Eureka nativo.
- Exponer el puerto del Sidecar (5678 por defecto) fuera del pod; el Sidecar debe ser accesible solo desde la aplicación non-JVM que lo acompaña, no desde el resto de la red.
- Asumir que el proxy `/{serviceId}/{path}` soporta todas las funcionalidades de Spring Cloud Gateway (filtros, transformaciones, seguridad); es un proxy simple de load balancing, no un Gateway completo.

---

← [13.6 API Composition con Spring Cloud Gateway y OpenFeign](sc-patrones-api-composition.md) | [Índice (README.md)](README.md) | [13.8 Resiliencia transversal en patrones distribuidos](sc-patrones-resiliencia-transversal.md) →
