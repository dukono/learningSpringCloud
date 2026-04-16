# 2.1 Arquitectura y rol de Eureka en el ecosistema de microservicios

← [1.9 Testing / Verificación de Spring Cloud Config](sc-config-testing.md) | [Índice (README.md)](README.md) | [2.2 Configuración y arranque del Eureka Server →](sc-eureka-server.md)

---

En un sistema de microservicios desplegado dinámicamente, las instancias de servicio cambian de dirección IP y puerto en cada despliegue, escalado horizontal o reinicio. Un cliente que almacena la URL de destino en su configuración estática rompe en cuanto esa instancia se reemplaza. Eureka resuelve este problema introduciendo un registro centralizado donde cada servicio publica su localización en el arranque y la retira al detenerse, permitiendo que los consumidores consulten el registro por nombre lógico en vez de por IP fija. Sin Eureka —o algún mecanismo equivalente— cualquier sistema con más de dos microservicios que escalan independientemente requiere gestión manual de endpoints, lo que es inviable en producción.

> [CONCEPTO] Spring Cloud Netflix Eureka implementa el patrón **Service Registry & Discovery**: un servidor mantiene el catálogo de instancias activas (Eureka Server) y cada microservicio se registra en él al arrancar y lo consulta al consumir otro servicio (Eureka Client).

## Diagrama: posición de Eureka en el ecosistema

El siguiente diagrama muestra cómo Eureka Server actúa como punto central de coordinación entre servicios productores y consumidores, y cómo Spring Cloud LoadBalancer usa el registro para elegir la instancia destino en cada llamada.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EUREKA SERVER :8761                          │
│                    Registro centralizado                             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  servicio-pedidos  → 10.0.1.5:8081  UP                      │   │
│  │  servicio-pedidos  → 10.0.1.6:8081  UP                      │   │
│  │  servicio-catalogo → 10.0.2.3:8082  UP                      │   │
│  │  servicio-pago     → 10.0.3.1:8083  UP                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────┬────────────────────────────────────┬─────────────────────┘
           │ 1. Registro (PUT /eureka/apps)      │ 1. Registro (PUT)
           │    heartbeat (PUT cada 30s)         │    heartbeat
           ▼                                     ▼
┌─────────────────────┐              ┌─────────────────────┐
│  servicio-pedidos   │              │  servicio-catalogo  │
│  (Eureka Client)    │              │  (Eureka Client)    │
│  Puerto 8081        │              │  Puerto 8082        │
└──────────┬──────────┘              └─────────────────────┘
           │ 2. Fetch registry (GET /eureka/apps cada 30s)
           │    Caché local de instancias
           │
           │ 3. Llamada HTTP via LoadBalancer
           │    http://servicio-catalogo/api/productos
           │    Spring Cloud LoadBalancer → resuelve a 10.0.2.3:8082
           ▼
┌─────────────────────┐
│  Spring Cloud       │
│  LoadBalancer       │
│  (cliente-side lb)  │
└─────────────────────┘
```

## Implementación: patrones de discovery comparados

Eureka implementa **client-side discovery**, que es el modelo predominante en Spring Cloud. El desarrollador debe conocer la diferencia con **server-side discovery** para justificar la elección arquitectónica en una entrevista.

En el **client-side discovery**, el cliente obtiene la lista de instancias del registro y él mismo elige a cuál llamar. Spring Cloud LoadBalancer implementa este rol dentro del proceso cliente. En el **server-side discovery**, un componente externo (como un load balancer de infraestructura o AWS ALB) recibe la petición y decide el destino; el cliente no conoce las instancias individuales.

```
Client-side discovery (Eureka + Spring Cloud LoadBalancer)
──────────────────────────────────────────────────────────
Cliente → consulta Eureka → obtiene [inst1, inst2, inst3]
        → elige inst2 (round-robin)
        → llama directamente a inst2

Server-side discovery (AWS ALB, Nginx, Kubernetes Service)
──────────────────────────────────────────────────────────
Cliente → llama a load-balancer.dominio.com
        → el LB consulta el registro
        → el LB elige inst2 y hace proxy
        → responde al cliente
```

| Criterio | Client-side (Eureka) | Server-side (LB externo) |
|---|---|---|
| Quién elige la instancia | El cliente (Spring Cloud LB) | El load balancer de infraestructura |
| Dependencia en el cliente | `spring-cloud-starter-netflix-eureka-client` | Ninguna (URL fija al LB) |
| Visibilidad del cliente | Ve todas las instancias | Solo ve el LB |
| Latencia de descubrimiento | Caché local (~30 s de staleness) | Sin caché en cliente |
| Flexibilidad de políticas LB | Configurable en código | Configurable en infraestructura |

## Tabla de componentes clave

La siguiente tabla resume los componentes del ecosistema Eureka y su responsabilidad en el patrón.

| Componente | Módulo / Artefacto | Responsabilidad |
|---|---|---|
| Eureka Server | `spring-cloud-starter-netflix-eureka-server` | Mantiene el registro de instancias, replica entre peers |
| Eureka Client | `spring-cloud-starter-netflix-eureka-client` | Registra la instancia, mantiene heartbeat, cachea el registro |
| `DiscoveryClient` | `spring-cloud-commons` (interfaz) | API unificada de consulta del registro; independiente de Eureka |
| `EurekaDiscoveryClient` | `spring-cloud-netflix-eureka-client` | Implementación de `DiscoveryClient` sobre Eureka |
| Spring Cloud LoadBalancer | `spring-cloud-starter-loadbalancer` | Balanceo client-side usando la lista de instancias de `DiscoveryClient` |

> [ADVERTENCIA] Ribbon fue eliminado a partir de Spring Cloud 2022.x y no está disponible en 2025.1.1 (Oakwood). No añadas `spring-cloud-starter-netflix-ribbon` al proyecto; Spring Cloud LoadBalancer es el sustituto oficial incluido en `spring-cloud-commons`. Consulta el módulo Spring Cloud LoadBalancer para la configuración completa.

## Buenas y malas prácticas

Hacer:
- Usar `spring.application.name` como identificador lógico del servicio: ese nombre es el que los consumidores usan para hacer `getInstances("nombre-servicio")` y para construir URLs como `http://nombre-servicio/api/recurso`.
- Desacoplar el consumidor del proveedor exclusivamente a través del nombre lógico; dejar que Spring Cloud LoadBalancer resuelva la IP en tiempo de ejecución para tolerar cambios de infraestructura sin reconfiguración.
- En entornos de producción con múltiples zonas de disponibilidad, configurar `eureka.client.prefer-same-zone-eureka: true` para reducir la latencia de red entre el cliente y el servidor Eureka más próximo.

Evitar:
- Codificar la IP o el puerto del servicio destino en la configuración del cliente: rompe con cualquier escalado horizontal o reemplazo de instancia, exactamente el problema que Eureka resuelve.
- Usar Ribbon en proyectos nuevos o en migraciones a Spring Boot 4.0.x: la dependencia no existe en el BOM de Spring Cloud 2025.1.1 y su inclusión manual produce conflictos de dependencias imposibles de resolver en tiempo de compilación.
- Confundir `DiscoveryClient` (interfaz de Spring Cloud Commons, portátil) con `EurekaDiscoveryClient` (implementación concreta): el código de aplicación siempre debe inyectar la interfaz para facilitar el cambio a Consul, Zookeeper u otro backend.

## Comparación: Eureka vs alternativas de service registry

Eureka no es el único registro de servicios disponible en el ecosistema Spring Cloud. La elección depende del stack de infraestructura existente y los requisitos de consistencia.

| Registro | Modelo de consistencia | Integración Spring Cloud | Casos de uso |
|---|---|---|---|
| Eureka (Netflix) | AP (disponibilidad + partición tolerante) | `spring-cloud-starter-netflix-eureka-client` | Microservicios on-premise o nube sin Kubernetes |
| Consul (HashiCorp) | CP (consistencia + partición tolerante) | `spring-cloud-starter-consul-discovery` | Entornos con necesidad de consistencia fuerte o service mesh |
| Zookeeper (Apache) | CP | `spring-cloud-starter-zookeeper-discovery` | Ecosistemas Kafka/Hadoop donde ya existe Zookeeper |
| Kubernetes Service + DNS | DNS nativo (no registry explícito) | `spring-cloud-kubernetes` | Despliegues en Kubernetes; Eureka deja de tener rol |

> [EXAMEN] En entornos Kubernetes, el mecanismo de discovery cambia radicalmente: el DNS nativo del clúster y los Kubernetes Services sustituyen a Eureka. Spring Cloud Kubernetes provee la implementación de `DiscoveryClient` que consulta la API de Kubernetes en lugar del registro de Eureka. Si el equipo migra a Kubernetes, Eureka se vuelve redundante.

---

← [1.9 Testing / Verificación de Spring Cloud Config](sc-config-testing.md) | [Índice (README.md)](README.md) | [2.2 Configuración y arranque del Eureka Server →](sc-eureka-server.md)
