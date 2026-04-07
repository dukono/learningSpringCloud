# Parte 4 — Service Discovery (Descubrimiento de servicios)

← [Parte 3 — Config](./03-config.md) | [Volver al índice](./README.md) | Siguiente: [Parte 5 — Load Balancing](./05-load-balancing.md) →

---

## 4.1 Qué es el Service Discovery y por qué es necesario

En una arquitectura tradicional, si el Servicio A quiere llamar al Servicio B, hardcodea su dirección:

```yaml
# MALO — dirección hardcodeada
servicio-b:
  url: http://192.168.1.50:8082
```

En producción con microservicios esto falla por:
- Las IPs cambian con cada redespliegue en contenedores (Docker, Kubernetes)
- El auto-escalado crea y destruye instancias dinámicamente
- Hay múltiples instancias del mismo servicio y no hay una sola IP

**Service Discovery** es el mecanismo por el que los servicios se encuentran entre sí de forma dinámica, sin IPs hardcodeadas:

```
Servicio A pregunta: "¿Dónde está pedidos-service?"
Registry responde:   "En 10.0.0.5:8083 y en 10.0.0.6:8083"
Servicio A elige una instancia y llama
```

---

## 4.2 Modelos: Client-Side vs Server-Side Discovery

### Client-Side Discovery (Spring Cloud / Eureka)

El **cliente** consulta el registro directamente y decide a qué instancia llamar:

```
[Servicio A]
    │ 1. pregunta a Eureka: instancias de B
    ▼
[Eureka Registry]
    │ 2. devuelve lista de IPs
    ▼
[Servicio A]
    │ 3. elige instancia (Round Robin)
    │ 4. llama directamente
    ▼
[Servicio B — instancia elegida]
```

**Ventajas:** menos latencia (sin proxy), el cliente tiene control total  
**Desventajas:** el cliente necesita la librería de discovery integrada

### Server-Side Discovery (AWS ALB, Kubernetes Service, Nginx)

Un **balanceador externo** consulta el registro y enruta la petición:

```
[Servicio A] → [Load Balancer / Proxy] → [Registry] → [Servicio B]
```

**Ventajas:** el cliente no sabe nada del registry, cualquier lenguaje funciona  
**Desventajas:** un salto extra en la red, el load balancer es punto único de fallo

**Spring Cloud usa Client-Side Discovery** con Eureka por defecto.

---

## 4.3 Spring Cloud Netflix Eureka — Servidor

Eureka Server es el **registro central** donde todos los microservicios se registran al arrancar y consultan para encontrar a otros.

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### Clase principal

```java
@SpringBootApplication
@EnableEurekaServer    // ← convierte la app en Eureka Server
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### application.yml del Eureka Server

```yaml
server:
  port: 8761   # puerto convencional de Eureka

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    # En modo standalone, el servidor NO se registra a sí mismo
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

  server:
    # Tiempo de espera antes de evictar instancias sin heartbeat
    eviction-interval-timer-in-ms: 10000
    # Deshabilitar self-preservation en desarrollo (ver 4.6)
    enable-self-preservation: false
```

### Dashboard de Eureka

Eureka expone un dashboard web en `http://localhost:8761` donde se pueden ver todos los servicios registrados, su estado y metadatos.

---

## 4.4 Spring Cloud Netflix Eureka — Cliente

Todo microservicio que quiere registrarse y/o descubrir otros servicios es un **Eureka Client**.

### Dependencia Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### Clase principal

```java
// @EnableDiscoveryClient es opcional desde Spring Cloud 2020.x
// Con solo tener la dependencia en el classpath, el cliente se activa automáticamente
@SpringBootApplication
public class PedidosServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PedidosServiceApplication.class, args);
    }
}
```

### application.yml del cliente

```yaml
spring:
  application:
    name: pedidos-service   # ← CRÍTICO: este es el nombre con el que se registra en Eureka

server:
  port: 8083

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    # Intervalo de refresco del caché local de instancias (defecto: 30s)
    registry-fetch-interval-seconds: 10
  instance:
    # Intervalo de heartbeat al servidor (defecto: 30s)
    lease-renewal-interval-in-seconds: 10
    # Tiempo que Eureka espera sin heartbeat antes de evictar (defecto: 90s)
    lease-expiration-duration-in-seconds: 30
    # Preferir IP sobre hostname (necesario en Docker/contenedores)
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
```

### Usar DiscoveryClient programáticamente

```java
@Service
public class ServicioDescubrimiento {

    @Autowired
    private DiscoveryClient discoveryClient;

    public List<ServiceInstance> obtenerInstancias(String serviceId) {
        return discoveryClient.getInstances(serviceId);
    }

    public String obtenerUrlPrimera(String serviceId) {
        List<ServiceInstance> instancias = discoveryClient.getInstances(serviceId);
        if (instancias.isEmpty()) {
            throw new RuntimeException("Servicio no encontrado: " + serviceId);
        }
        ServiceInstance instancia = instancias.get(0);
        return instancia.getUri().toString();  // ej: http://10.0.0.5:8082
    }
}
```

---

## 4.5 Heartbeat, lease y evicción de instancias

### Ciclo de vida de una instancia en Eureka

```
Microservicio arranca
        │
        ▼
1. REGISTRO: envía POST /eureka/apps/{appId} con metadata
   (nombre, IP, puerto, estado UP, metadatos personalizados)
        │
        ▼
2. HEARTBEAT: cada 30s envía PUT /eureka/apps/{appId}/{instanceId}
   → renueva el "lease" (arrendamiento)
        │
   Si no llega heartbeat en 90s (lease expiration)...
        │
        ▼
3. EVICCIÓN: Eureka marca la instancia como DOWN y la elimina del registry
        │
        ▼
4. CANCELACIÓN: al apagar correctamente (SIGTERM), el servicio
   envía DELETE /eureka/apps/{appId}/{instanceId}
```

### Parámetros clave

| Parámetro | Default | Descripción |
|---|---|---|
| `lease-renewal-interval-in-seconds` | 30 | Frecuencia del heartbeat (cliente) |
| `lease-expiration-duration-in-seconds` | 90 | Timeout sin heartbeat → evicción (cliente) |
| `eviction-interval-timer-in-ms` | 60000 | Frecuencia de limpieza en el servidor |
| `registry-fetch-interval-seconds` | 30 | Frecuencia de refresco del caché local |

**En desarrollo** se recomienda reducir estos valores (10s/30s) para que los cambios se reflejen más rápido. En producción los valores por defecto son adecuados para evitar falsos negativos por picos de carga.

---

## 4.6 Self-preservation mode

Eureka tiene un mecanismo de protección llamado **self-preservation mode** (modo de auto-preservación).

### El problema que resuelve:

Si la red sufre una partición temporal, Eureka podría no recibir heartbeats de instancias que en realidad están saludables. Sin protección, las evictaría y los clientes perderían instancias válidas.

### Cómo funciona:

Eureka calcula el **número esperado de heartbeats por minuto** basándose en las instancias registradas. Si el porcentaje de heartbeats recibidos cae por debajo del 85%, Eureka **activa el modo de auto-preservación** y deja de evictar instancias aunque no envíen heartbeat.

```
Dashboard de Eureka muestra:
"EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP 
WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE 
THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE."
```

### Comportamiento:
- **Self-preservation ACTIVO:** Eureka no evicta instancias (puede mostrar instancias caídas como UP)
- **Self-preservation INACTIVO:** Eureka evicta agresivamente instancias sin heartbeat

### Recomendación:
- **Desarrollo:** deshabilitar (`enable-self-preservation: false`) para que los cambios se reflejen inmediatamente
- **Producción:** mantener habilitado (comportamiento por defecto)

```yaml
eureka:
  server:
    enable-self-preservation: false  # solo en desarrollo
```

---

## 4.7 Eureka en alta disponibilidad (clúster)

En producción, un solo Eureka Server es un punto único de fallo. Se usa un **clúster de Eureka** donde cada nodo se registra en los demás.

### Configuración de clúster con 2 nodos (peer replication):

```yaml
# Nodo 1 — application-peer1.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: eureka1.miempresa.com
  client:
    register-with-eureka: true   # en clúster SÍ se registra
    fetch-registry: true
    service-url:
      defaultZone: http://eureka2.miempresa.com:8762/eureka/  # apunta al otro nodo
```

```yaml
# Nodo 2 — application-peer2.yml
server:
  port: 8762

eureka:
  instance:
    hostname: eureka2.miempresa.com
  client:
    service-url:
      defaultZone: http://eureka1.miempresa.com:8761/eureka/  # apunta al primer nodo
```

### Los clientes apuntan a todos los nodos:

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka1.miempresa.com:8761/eureka/,http://eureka2.miempresa.com:8762/eureka/
```

Si uno de los nodos cae, los clientes siguen usando el otro. Los nodos sincronizan sus registros entre sí mediante **peer replication**.

---

## 4.8 Alternativas a Eureka: Consul y Zookeeper

| Característica | Eureka | Consul | Zookeeper |
|---|---|---|---|
| **Tipo** | Solo Service Discovery | Discovery + Config + Health | Coordinación distribuida |
| **Consistencia** | AP (disponibilidad > consistencia) | CP (consistencia > disponibilidad) | CP |
| **Protocolo** | HTTP REST | HTTP + DNS | ZAB (Zookeeper Atomic Broadcast) |
| **Health checks** | Heartbeat pasivo | Activos (HTTP, TCP, scripts) | Sesiones con timeouts |
| **Complejidad** | Baja | Media | Alta |
| **Uso en Spring Cloud** | `spring-cloud-starter-netflix-eureka-*` | `spring-cloud-starter-consul-*` | `spring-cloud-starter-zookeeper-*` |

### Cuándo usar Consul en lugar de Eureka:
- Se necesitan **health checks activos** (Eureka solo usa heartbeat)
- Ya se usa Consul en la organización para otras cosas (secrets, service mesh)
- Se necesita la interfaz DNS para descubrimiento (Consul expone DNS)
- Se prefiere consistencia fuerte sobre disponibilidad

### Integración con Consul (configuración básica):

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: pedidos-service
        health-check-path: /actuator/health
        health-check-interval: 15s
```

---

← [Parte 3 — Config](./03-config.md) | [Volver al índice](./README.md) | Siguiente: [Parte 5 — Load Balancing](./05-load-balancing.md) →
