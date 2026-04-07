# Parte 1 — Fundamentos Conceptuales

← [Volver al índice](./README.md) | Siguiente: [Parte 2 — Arquitectura](./02-arquitectura.md) →

---

## 1.1 ¿Qué es Spring Cloud?

Spring Cloud es un **conjunto de frameworks y librerías** que se construyen sobre Spring Boot y proporcionan soluciones listas para usar a los problemas más comunes que surgen al construir sistemas distribuidos y arquitecturas de microservicios.

No es un único producto, sino una **colección de proyectos** bajo el paraguas de Spring Cloud, cada uno resolviendo un problema concreto:

| Problema | Proyecto que lo resuelve |
|---|---|
| Configuración centralizada | Spring Cloud Config |
| Descubrimiento de servicios | Spring Cloud Netflix Eureka / Consul |
| Balanceo de carga | Spring Cloud LoadBalancer |
| Puerta de entrada única | Spring Cloud Gateway |
| Tolerancia a fallos | Spring Cloud Circuit Breaker (Resilience4j) |
| Cliente HTTP declarativo | Spring Cloud OpenFeign |
| Trazabilidad distribuida | Micrometer Tracing + Zipkin |
| Mensajería asíncrona | Spring Cloud Stream |

**Definición formal:** Spring Cloud provee herramientas para que los desarrolladores construyan rápidamente patrones comunes en sistemas distribuidos como: gestión de configuración, descubrimiento de servicios, circuit breakers, routing inteligente, micro-proxies, bus de control, tokens de acceso, sesión global y contratos de prueba.

**Punto clave:** Spring Cloud **no reinventa la rueda**. Integra y adapta soluciones ya probadas (Netflix OSS, HashiCorp Consul, Apache Kafka, etc.) bajo una API unificada y coherente con el ecosistema Spring.

---

## 1.2 Historia y evolución

```
2014 — Netflix publica su stack de microservicios como Open Source (Eureka, Ribbon, Hystrix, Zuul)
2015 — Spring Cloud 1.x integra el stack de Netflix (Spring Cloud Netflix)
2018 — Netflix pone en modo mantenimiento Hystrix, Ribbon y Zuul 1.x
2019 — Spring Cloud introduce alternativas propias: Gateway, LoadBalancer, Circuit Breaker SPI
2020 — Spring Cloud 2020.x (Ilford) consolida la migración fuera de Netflix OSS
2022 — Spring Boot 3 / Spring Cloud 2022.x migra a Jakarta EE y Java 17
2023 — Micrometer Tracing reemplaza a Spring Cloud Sleuth
2024 — Spring Cloud 2023.x (Leyton), soporte completo para Spring Boot 3.2+
```

**Por qué importa conocer la historia:**  
Muchos proyectos empresariales usan versiones antiguas con Hystrix o Ribbon. En entrevistas y pruebas técnicas aparecen ambas generaciones. Este documento cubre **la generación actual** pero señala las diferencias con la anterior donde es relevante.

---

## 1.3 Spring Cloud vs Spring Boot vs Spring Framework

Es una confusión muy frecuente en principiantes. La relación es jerárquica:

```
Spring Framework
    └── Spring Boot          (simplifica la configuración de Spring)
            └── Spring Cloud (añade capacidades para sistemas distribuidos)
```

| Aspecto | Spring Framework | Spring Boot | Spring Cloud |
|---|---|---|---|
| **Nivel** | Base | Sobre Spring Framework | Sobre Spring Boot |
| **Propósito** | IoC, DI, AOP, MVC | Auto-configuración, arranque rápido | Sistemas distribuidos |
| **Sin él** | Mucha configuración XML/Java | Posible pero tedioso | Habría que integrar todo manualmente |
| **Ejemplo** | `@Component`, `@Autowired` | `@SpringBootApplication`, starters | `@EnableEurekaServer`, `@FeignClient` |
| **Aplicación standalone** | Sí | Sí | No (siempre necesita Spring Boot) |

**Regla de oro:** Todo proyecto Spring Cloud **es** un proyecto Spring Boot. No todo proyecto Spring Boot necesita Spring Cloud (solo si tiene múltiples servicios que necesitan coordinarse).

---

## 1.4 El problema que resuelve: arquitectura monolítica vs microservicios

### Arquitectura Monolítica

En una aplicación monolítica, **todo el código vive en un único desplegable** (un WAR o JAR):

```
[ Monolito ]
  ├── Módulo Usuarios
  ├── Módulo Productos  
  ├── Módulo Pedidos
  ├── Módulo Pagos
  └── Módulo Notificaciones
        ↕
   [ Base de datos única ]
```

**Ventajas del monolito:**
- Desarrollo inicial simple
- Una sola unidad a desplegar
- Llamadas entre módulos son llamadas de método (rápidas, sin red)
- Transacciones ACID triviales

**Problemas del monolito a escala:**
- Un bug en el módulo de notificaciones puede tumbar todo el sistema
- Para escalar el módulo de productos hay que escalar **todo** el monolito
- El equipo crece → conflictos en el repositorio, deploys coordinados obligatorios
- Tecnología única: si el módulo de IA necesita Python, no puede usarlo
- Tiempo de compilación y arranque crece con el sistema

### Arquitectura de Microservicios

Cada módulo se convierte en un **servicio independiente**, con su propia base de datos, desplegable por separado:

```
[API Gateway]
     |
     ├──→ [Servicio Usuarios   :8081]  ←→ [DB Usuarios]
     ├──→ [Servicio Productos  :8082]  ←→ [DB Productos]
     ├──→ [Servicio Pedidos    :8083]  ←→ [DB Pedidos]
     │         ↓ (llama a)
     │    [Servicio Productos  :8082]
     ├──→ [Servicio Pagos      :8084]  ←→ [DB Pagos]
     └──→ [Servicio Notif.     :8085]  ←→ [Cola Mensajes]
```

**Ventajas:**
- Escalar solo el servicio que lo necesita
- Equipos autónomos por servicio
- Distintas tecnologías por servicio
- Fallo aislado: si cae Notificaciones, los pedidos siguen funcionando
- Despliegues independientes

**Nuevos problemas que aparecen** (y que Spring Cloud resuelve — ver sección 1.5):

---

## 1.5 Los 8 problemas clásicos de los microservicios

Al pasar a microservicios, aparecen problemas que no existían en el monolito:

### Problema 1: ¿Dónde está cada servicio?
En un monolito, un módulo llama a otro con una llamada de método. En microservicios, el Servicio A necesita conocer la IP y puerto del Servicio B. Con auto-escalado, esas IPs **cambian dinámicamente**.  
→ **Solución: Service Discovery (Eureka)**

### Problema 2: ¿Cómo gestiono la configuración de 50 servicios?
Cada microservicio tiene su `application.yml`. En producción hay que actualizar URLs, credenciales, flags de feature… en 50 repositorios distintos.  
→ **Solución: Config Server**

### Problema 3: ¿Cómo enrutan las peticiones del cliente?
El cliente (app móvil, SPA) no puede conocer las URLs internas de 50 servicios. Tampoco puede manejar autenticación, rate limiting y CORS en cada uno.  
→ **Solución: API Gateway**

### Problema 4: ¿Qué pasa si un servicio se cae o está lento?
Si el Servicio A llama al Servicio B y B tarda 30 segundos, A acumula hilos esperando → A se queda sin recursos → C que llama a A también falla → **fallo en cascada** que tumba todo el sistema.  
→ **Solución: Circuit Breaker (Resilience4j)**

### Problema 5: ¿Cómo distribuyo la carga entre instancias?
Si hay 3 instancias del Servicio Productos, ¿quién decide a cuál de las tres enviar cada petición?  
→ **Solución: Load Balancer**

### Problema 6: ¿Cómo sigo una petición entre 10 servicios?
Un error en producción: la petición pasó por Gateway → Auth → Pedidos → Productos → Inventario. ¿En cuál falló? Los logs están en 5 máquinas distintas.  
→ **Solución: Distributed Tracing (Zipkin + Micrometer)**

### Problema 7: ¿Cómo comunico servicios sin acoplamiento temporal?
Algunos eventos no necesitan respuesta inmediata. El Servicio de Pagos confirma un pago y el de Notificaciones debe enviar un email, pero no necesita bloquear al de Pagos.  
→ **Solución: Spring Cloud Stream (Kafka/RabbitMQ)**

### Problema 8: ¿Cómo autentico/autorizo en cada servicio?
¿Valido el JWT en cada microservicio? ¿Quién emite los tokens? ¿Cómo propago la identidad del usuario entre llamadas internas?  
→ **Solución: OAuth2 + JWT centralizado en Gateway**

---

## 1.6 Cómo Spring Cloud resuelve esos problemas

```
PROBLEMA                     COMPONENTE SPRING CLOUD          TECNOLOGÍA SUBYACENTE
─────────────────────────    ──────────────────────────────   ─────────────────────
Descubrimiento de servicios  Spring Cloud Netflix Eureka      Netflix Eureka
Configuración centralizada   Spring Cloud Config              Git, Vault, JDBC
Puerta de entrada            Spring Cloud Gateway             Project Reactor (WebFlux)
Fallos en cascada            Spring Cloud Circuit Breaker     Resilience4j
Balanceo de carga            Spring Cloud LoadBalancer        Integrado en Spring
Cliente HTTP declarativo     Spring Cloud OpenFeign           OpenFeign + OkHttp
Trazabilidad distribuida     Micrometer Tracing               Zipkin, Jaeger
Mensajería asíncrona         Spring Cloud Stream              Kafka, RabbitMQ
Propagación de eventos       Spring Cloud Bus                 Kafka, RabbitMQ
```

---

## 1.7 Versiones, BOM y compatibilidad con Spring Boot

Spring Cloud usa un esquema de versiones por **nombre de ciudad** (hasta 2020) y luego por **año** (desde 2020):

| Spring Cloud | Nombre código | Spring Boot compatible |
|---|---|---|
| 2023.0.x | Leyton | 3.2.x, 3.3.x |
| 2022.0.x | Kilburn | 3.0.x, 3.1.x |
| 2021.0.x | Jubilee | 2.6.x, 2.7.x |
| 2020.0.x | Ilford | 2.4.x, 2.5.x |
| Hoxton | — | 2.2.x, 2.3.x |
| Greenwich | — | 2.1.x |

**CRÍTICO:** Mezclar versiones incompatibles de Spring Boot y Spring Cloud es la causa número uno de problemas en proyectos nuevos.

### Uso del BOM (Bill of Materials)

En lugar de gestionar versiones individuales de cada dependencia Spring Cloud, se usa el BOM:

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Con el BOM importado, las dependencias individuales **no necesitan versión**:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <!-- sin <version> — la gestiona el BOM -->
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

### Con Gradle:

```groovy
// build.gradle
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.3"
    }
}
```

**Regla práctica:** Siempre consultar la [matriz de compatibilidad oficial](https://spring.io/projects/spring-cloud) antes de crear un proyecto nuevo.

---

## 1.8 Las 8 falacias de la computación distribuida

Peter Deutsch (Sun Microsystems, 1994) formuló 8 suposiciones falsas que los desarrolladores suelen hacer al pasar de sistemas locales a distribuidos. Spring Cloud existe precisamente para mitigar los efectos de estas falacias.

| # | Falacia | Realidad | Cómo Spring Cloud ayuda |
|---|---------|----------|------------------------|
| 1 | La red es confiable | Los paquetes se pierden, las conexiones se cortan | Circuit Breaker, Retry (Resilience4j) |
| 2 | La latencia es cero | Las llamadas de red miden milisegundos, no nanosegundos | Timeouts en Feign/Gateway, Load Balancer |
| 3 | El ancho de banda es infinito | Los payloads grandes congestionan la red | Feign con compresión, mensajería asíncrona |
| 4 | La red es segura | Hay interceptación, man-in-the-middle, inyección | OAuth2/JWT en Gateway, SSL, Spring Cloud Security |
| 5 | La topología no cambia | Los servicios escalan, se mueven, se reinician | Service Discovery (Eureka) con registro dinámico |
| 6 | Hay un único administrador | Equipos distintos gestionan partes distintas | Config Server centralizado, convenciones uniformes |
| 7 | El coste de transporte es cero | Serialización/deserialización tiene coste CPU | Protobuf, mensajería asíncrona (Stream) |
| 8 | La red es homogénea | Hay distintos OS, versiones JVM, protocolos | Abstracciones de Spring Cloud sobre las diferencias |

**`[EXAMEN]`** En pruebas técnicas suelen preguntar cuántas falacias hay (8) y pedir ejemplos. La más frecuente en entrevistas: *"¿Por qué no puedes asumir que las llamadas entre microservicios son tan confiables como las llamadas locales?"* — respuesta: falacias 1 y 2.

---

## 1.9 CAP Theorem

El teorema CAP (Brewer, 2000) establece que un sistema distribuido **no puede garantizar simultáneamente** las tres propiedades siguientes. En un escenario de partición de red, solo puede ofrecer dos:

```
        Consistencia (C)
             /\
            /  \
           / CP \
          /------\
         /   AP   \
        /__________\
Disponibilidad (A)  Tolerancia a Particiones (P)
```

| Propiedad | Significado |
|-----------|-------------|
| **C** — Consistency | Todos los nodos ven los mismos datos al mismo tiempo |
| **A** — Availability | Cada petición recibe una respuesta (no necesariamente con el dato más reciente) |
| **P** — Partition tolerance | El sistema sigue funcionando aunque fallen enlaces de red entre nodos |

**En la práctica**, la partición de red (P) es inevitable en sistemas distribuidos → la elección real es entre **CP** y **AP**:

| Sistema | Tipo | Razonamiento |
|---------|------|--------------|
| Eureka | **AP** | Prefiere disponibilidad: sigue sirviendo entradas aunque algunos nodos no estén sincronizados |
| Consul | **CP** por defecto | Requiere quorum para responder; puede rechazar peticiones durante particiones |
| Zookeeper | **CP** | Consensus protocol (ZAB); rechaza writes sin quorum |
| Redis (Sentinel) | **CP** | Elección de master bloqueante durante failover |
| Cassandra | **AP** configurable | Nivel de consistencia ajustable por operación |

**`[EXAMEN]`** Pregunta clásica: *"Eureka vs Consul: ¿cuál es CP y cuál es AP?"* — Eureka es AP, Consul es CP. También: *"¿Por qué Eureka tiene self-preservation mode?"* — para mantener disponibilidad (AP) durante particiones de red.

**`[ADVERTENCIA]`** CAP Theorem asume partición total. En la práctica existe el espectro **PACELC**: incluso sin partición, hay trade-off entre latencia y consistencia.

---

## 1.10 Spring Cloud vs soluciones nativas de Kubernetes

A partir de Kubernetes 1.x, muchas de las funcionalidades que justificaron Spring Cloud están disponibles nativamente en la plataforma. Es fundamental entender cuándo usar cada enfoque.

### Tabla comparativa

| Capacidad | Spring Cloud | Kubernetes nativo |
|-----------|-------------|-------------------|
| **Service Discovery** | Eureka Server (proceso separado) | kube-dns + Services + Endpoints |
| **Load Balancing** | Spring Cloud LoadBalancer (en el cliente) | kube-proxy + Service (en la red) |
| **Config Management** | Config Server + Git | ConfigMaps + Secrets + etcd |
| **Circuit Breaker** | Resilience4j (en el código) | Istio/Linkerd (en el sidecar) |
| **API Gateway** | Spring Cloud Gateway (app JVM) | Ingress Controller + API Gateway |
| **Observabilidad** | Micrometer Tracing + Zipkin | Jaeger/Zipkin vía service mesh |

### ¿Cuándo usar Spring Cloud en Kubernetes?

**Usar Spring Cloud incluso en Kubernetes cuando:**
- El equipo tiene más expertise en Java/Spring que en Kubernetes
- Se necesita lógica de negocio en el Gateway (filtros personalizados complejos)
- Se usa Spring Cloud Stream para mensajería (no tiene equivalente nativo)
- Se necesita `@RefreshScope` para reconfiguración sin reinicio de pods
- El proyecto tiene requisitos de multi-cloud o no quiere acoplarse a K8s

**Delegar a Kubernetes cuando:**
- Se usa service mesh (Istio/Linkerd) → circuit breaking, retry, mTLS en el sidecar
- Se quiere eliminar Eureka → usar `spring-cloud-starter-kubernetes-discoveryclient`
- Se quiere eliminar Config Server → usar `spring-cloud-kubernetes-config` (lee ConfigMaps)

**`[EXAMEN]`** Pregunta frecuente: *"¿Necesitas Eureka si despliegas en Kubernetes?"* — No necesariamente. Kubernetes Service Discovery es suficiente para muchos casos. Spring Cloud Kubernetes permite usar ConfigMaps y Secrets como fuente de configuración y el DiscoveryClient nativo de K8s, eliminando la necesidad de Eureka y Config Server.

**`[ADVERTENCIA]`** No mezcles Eureka y Kubernetes Service Discovery sin una estrategia clara. Tener dos sistemas de descubrimiento paralelos genera inconsistencias difíciles de depurar.

---

← [Volver al índice](./README.md) | Siguiente: [Parte 2 — Arquitectura](./02-arquitectura.md) →
