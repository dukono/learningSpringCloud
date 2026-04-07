# Parte 2 — Arquitectura de Spring Cloud

← [Parte 1 — Fundamentos](./01-fundamentos.md) | [Volver al índice](./README.md) | Siguiente: [Parte 3 — Config](./03-config.md) →

---

## 2.1 Visión general de la arquitectura

Un ecosistema Spring Cloud completo se compone de capas que trabajan juntas. Cada capa tiene una responsabilidad clara:

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTES                             │
│            (Browser, App móvil, Servicio externo)           │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTP/HTTPS
┌─────────────────────────▼───────────────────────────────────┐
│                    API GATEWAY                              │
│         (Spring Cloud Gateway — puerto único de entrada)    │
│    Routing · Auth · Rate Limiting · SSL Termination         │
└──────┬──────────────────┬──────────────────┬────────────────┘
       │                  │                  │
┌──────▼──────┐   ┌───────▼──────┐   ┌──────▼──────┐
│  Servicio A │   │  Servicio B  │   │  Servicio C │
│  (Spring    │   │  (Spring     │   │  (Spring    │
│   Boot)     │   │   Boot)      │   │   Boot)     │
└──────┬──────┘   └───────┬──────┘   └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │ todos se registran en
┌─────────────────────────▼───────────────────────────────────┐
│              SERVICE REGISTRY (Eureka Server)               │
│         "Directorio telefónico" de microservicios           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│               CONFIG SERVER                                 │
│    (centraliza application.yml de todos los servicios)      │
│    Backend: repositorio Git con archivos de configuración   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              OBSERVABILIDAD                                 │
│   Zipkin (trazas) · Prometheus (métricas) · ELK (logs)      │
└─────────────────────────────────────────────────────────────┘
```

**Flujo de arranque de un microservicio:**

1. El microservicio arranca y lee su configuración del **Config Server**
2. Se registra en **Eureka Server** (anuncia su IP y puerto)
3. Comienza a atender peticiones
4. Para llamar a otro servicio, consulta Eureka para descubrir sus instancias
5. Usa **LoadBalancer** para elegir una instancia y **Feign** para hacer la llamada HTTP
6. Si el servicio destino falla, el **Circuit Breaker** corta el flujo y devuelve un fallback

---

## 2.2 Patrones de diseño en microservicios que implementa Spring Cloud

Spring Cloud no es arbitrario: implementa **patrones de diseño catalogados** por la industria.

### Patrón: Service Registry and Discovery
- **Problema:** Las IPs de los servicios son dinámicas
- **Solución:** Un registro central donde cada servicio publica su localización
- **Implementación:** Eureka, Consul

### Patrón: Client-Side Load Balancing
- **Problema:** Hay múltiples instancias del mismo servicio, ¿a cuál llamar?
- **Solución:** El cliente (no un balanceador externo) elige la instancia
- **Implementación:** Spring Cloud LoadBalancer

### Patrón: API Gateway / Edge Server
- **Problema:** Los clientes externos no deben conocer la topología interna
- **Solución:** Un único punto de entrada que enruta, autentica y filtra
- **Implementación:** Spring Cloud Gateway

### Patrón: Circuit Breaker
- **Problema:** Un servicio lento bloquea recursos y produce fallos en cascada
- **Solución:** Cortar el circuito cuando el servicio falla y usar un fallback
- **Implementación:** Resilience4j

### Patrón: Externalized Configuration
- **Problema:** Configuración hardcodeada o dispersa en múltiples repos
- **Solución:** Repositorio central de configuración, accesible en tiempo de arranque
- **Implementación:** Spring Cloud Config

### Patrón: Distributed Tracing
- **Problema:** Seguir una petición a través de N servicios es imposible sin correlación
- **Solución:** Inyectar IDs de traza y span en cada petición HTTP y log
- **Implementación:** Micrometer Tracing + Zipkin

### Patrón: Sidecar
- **Problema:** Integrar servicios escritos en otros lenguajes al ecosistema Spring Cloud
- **Solución:** Un proceso Java secundario (sidecar) gestiona la comunicación
- **Implementación:** Spring Cloud Netflix Sidecar

### Patrón: Bulkhead
- **Problema:** Un servicio sobreusa todos los recursos compartidos
- **Solución:** Aislar recursos por servicio (pools de hilos separados)
- **Implementación:** Resilience4j Bulkhead

---

## 2.3 Mapa de componentes y sus relaciones

```
                    ┌─────────────────────────────────────┐
                    │         spring-cloud-config          │
                    │    Config Server ←── Git Repo        │
                    │         ↑                            │
                    │    (todos los servicios leen aquí)   │
                    └─────────────────┬───────────────────┘
                                      │ bootstrap / startup
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
┌─────────▼──────────┐   ┌───────────▼──────────┐  ┌────────────▼────────────┐
│   Gateway Service  │   │  Eureka Server        │  │  Microservicio A/B/C    │
│ spring-cloud-      │   │  spring-cloud-netflix │  │  spring-boot            │
│ gateway            │   │  -eureka-server       │  │  + eureka-client        │
│                    │   │                       │  │  + config-client        │
│ Routes ──────────────→ │  Registry             │ ←─ registers itself        │
│ Predicates         │   │  (IP:port catalog)  ←────→ heartbeat               │
│ Filters            │   └───────────────────────┘  │                         │
│                    │             ↑                 │  Feign Client ──────────┼──→ calls B
│ LoadBalancer ←─────┼─────────────┘                 │  Circuit Breaker        │
│ Circuit Breaker    │                               │  LoadBalancer           │
└────────────────────┘                               └─────────────────────────┘
          │
          │ propagates traceId
          ▼
┌─────────────────────┐
│  Zipkin / Tempo     │   ←── todos los servicios envían spans
│  (distributed       │
│   tracing UI)       │
└─────────────────────┘
```

---

## 2.4 Flujo de una petición en un ecosistema Spring Cloud

Traza completa de: **"Cliente externo pide el detalle de un pedido"**

```
1. CLIENTE
   GET https://api.miempresa.com/pedidos/42
   
2. API GATEWAY (Spring Cloud Gateway)
   - Recibe la petición en el puerto 443
   - Valida el JWT (filtro de autenticación)
   - Identifica la ruta: /pedidos/** → servicio "pedidos-service"
   - Consulta al LoadBalancer: ¿qué instancia de "pedidos-service" uso?
   
3. LOAD BALANCER (Spring Cloud LoadBalancer)
   - Consulta Eureka: "dame las instancias registradas de pedidos-service"
   - Eureka responde: [192.168.1.10:8083, 192.168.1.11:8083]
   - Round Robin → elige 192.168.1.10:8083
   
4. PEDIDOS-SERVICE (192.168.1.10:8083)
   - Recibe GET /pedidos/42
   - Necesita los datos del producto → llama a "productos-service"
   - Usa Feign Client: @FeignClient("productos-service")
   - Feign + LoadBalancer → Eureka → instancia de productos-service
   - Circuit Breaker: ¿está productos-service saludable?
     - SI → hace la llamada HTTP
     - NO → devuelve fallback (producto genérico o error controlado)
   
5. PRODUCTOS-SERVICE
   - Responde con los datos del producto

6. PEDIDOS-SERVICE
   - Combina datos del pedido + producto
   - Responde al Gateway

7. API GATEWAY
   - Añade headers de respuesta (CORS, cache-control)
   - Devuelve al cliente

EN PARALELO (todo el flujo):
   - Cada servicio inyecta traceId/spanId en sus logs
   - Cada servicio envía spans a Zipkin
   - Zipkin muestra el árbol completo de la petición
```

**Puntos clave de este flujo:**
- El cliente solo conoce la URL del Gateway
- Ningún servicio conoce las IPs hardcodeadas de los demás
- La configuración de todos los servicios viene del Config Server
- Si cualquier servicio falla, el Circuit Breaker evita el fallo en cascada
- Todo el flujo es trazable desde Zipkin con un único `traceId`

---

← [Parte 1 — Fundamentos](./01-fundamentos.md) | [Volver al índice](./README.md) | Siguiente: [Parte 3 — Config](./03-config.md) →
