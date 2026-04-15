# 6.1.1 Qué es un API Gateway: responsabilidades y casos de uso

← [5 — Balanceo de Carga](./05-load-balancing.md) | [Índice](./README.md) | [6.1.2 Arquitectura reactiva →](./06-02-gateway-arquitectura.md)

---

Sin Gateway, cada cliente externo necesita conocer la URL, el puerto y la interfaz de cada microservicio que quiere llamar. Cuando el sistema crece de 3 a 30 servicios, cualquier refactorización interna —renombrar un servicio, moverlo de puerto, escalar horizontalmente— se convierte en un cambio coordinado con todos los clientes. Además, responsabilidades transversales como autenticación, control de tasa o logging se duplican en cada servicio o simplemente se omiten. El **API Gateway** resuelve ambos problemas a la vez: ofrece un único punto de entrada estable hacia la arquitectura de microservicios, y centraliza las responsabilidades transversales que no pertenecen a la lógica de negocio de ningún servicio concreto.

> [PREREQUISITO] El módulo Gateway requiere `spring-cloud-starter-gateway`, que trae WebFlux (reactivo, no bloqueante). Si el proyecto incluye también `spring-boot-starter-web` (Servlet), la aplicación fallará al arrancar. Los dos stacks son incompatibles en el mismo módulo.

---

## Responsabilidades del Gateway

El Gateway actúa como fachada de toda la arquitectura. No contiene lógica de negocio: su trabajo es recibir peticiones externas, aplicar políticas transversales y enrutarlas al microservicio correcto.

| Responsabilidad | Qué hace | Sin Gateway |
|---|---|---|
| **Enrutamiento** | Reenvía al servicio correcto según path, método u headers | El cliente conoce cada URL interna |
| **Autenticación** | Valida el token JWT antes de dejar pasar la petición | Cada servicio valida su propio token |
| **Rate Limiting** | Limita peticiones por IP, usuario o plan de suscripción | Cada servicio implementa su propio límite |
| **SSL Termination** | Gestiona HTTPS; los servicios internos usan HTTP plano | Cada servicio gestiona sus propios certificados |
| **CORS** | Centraliza las políticas Cross-Origin | Cada servicio configura sus propios headers CORS |
| **Transformación** | Modifica headers, paths o body antes de reenviar | Transformaciones duplicadas en cada servicio |
| **Logging / Trazabilidad** | Registra todas las peticiones y propaga Trace ID | Logging inconsistente entre servicios |
| **Circuit Breaker** | Protege los servicios de sobrecarga desde el perímetro | Cada servicio gestiona su propio CB |
| **Load Balancing** | Distribuye carga entre instancias del mismo servicio | El cliente necesita conocer todas las instancias |

> [CONCEPTO] El Gateway no reemplaza la lógica de negocio: la delega en los microservicios. Todo lo que es política de acceso, transformación de protocolo o control de tráfico pertenece al Gateway. Todo lo que es regla de negocio pertenece al servicio.

Estas responsabilidades no son arbitrarias: responden a la necesidad de que la lógica transversal tenga un único lugar donde vivir, evolucionar y auditarse. Si la autenticación está en 12 servicios distintos, actualizar el algoritmo de validación de tokens requiere 12 despliegues coordinados. Si está en el Gateway, requiere uno.

### Qué NO hace el Gateway

Es igual de importante conocer los límites. El Gateway no debe:

- **Tomar decisiones de negocio**: si un pedido puede realizarse, si un usuario tiene saldo suficiente, si un producto está disponible. Eso pertenece al microservicio correspondiente.
- **Mantener estado de sesión de usuario**: las sesiones de negocio (carrito, wizard de compra) pertenecen al servicio o a una caché compartida, no al Gateway.
- **Orquestar flujos entre servicios**: llamar a A, luego a B, luego a C y combinar resultados. Eso es responsabilidad de un servicio orquestador o del patrón BFF.
- **Acceder directamente a bases de datos de dominio**: en raras excepciones un filtro puede consultar una caché Redis para validar una API key, pero nunca una base de datos de negocio.

---

## Topología: sin Gateway vs con Gateway

El diagrama muestra cómo cambia la topología visible para un cliente externo al introducir un Gateway.

```
SIN GATEWAY

  Cliente móvil ──────────────────► productos-service  :8081
  Cliente móvil ──────────────────► pedidos-service    :8082
  Cliente móvil ──────────────────► usuarios-service   :8083
  Cliente web   ──────────────────► productos-service  :8081
  Cliente web   ──────────────────► pedidos-service    :8082
  Cliente web   ──────────────────► usuarios-service   :8083

  Cada cliente conoce 3 URLs internas y 3 puertos.
  Un cambio de puerto o nombre de servicio rompe todos los clientes.
  Auth, rate limiting y CORS se implementan (o no) en cada servicio.
  No hay punto único de logging ni de propagación de trazas.


CON GATEWAY

  Cliente móvil ─┐
                 ├──► Gateway :443 ──► productos-service  :8081
  Cliente web   ─┘        │   └──────► pedidos-service    :8082
                           └─────────► usuarios-service   :8083

  Los clientes solo conocen https://api.miempresa.com.
  La topología interna es invisible y modificable sin impacto en clientes.
  Auth, rate limiting y CORS se aplican una sola vez, en el Gateway.
  Un único punto de logging y de propagación de Trace ID.
```

---

## Setup mínimo funcional

Un Gateway operativo necesita tres elementos: la dependencia correcta, al menos una ruta configurada y acceso a un service registry si se usa balanceo de carga. El siguiente ejemplo enruta tres servicios y añade un header de respuesta global.

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <!-- NO incluir spring-boot-starter-web: incompatible con WebFlux -->
</dependencies>

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

```yaml
# application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service     # lb:// → resuelve instancia vía Eureka + LoadBalancer
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1               # elimina /api antes de reenviar al servicio

        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - StripPrefix=1

        - id: usuarios-route
          uri: lb://usuarios-service
          predicates:
            - Path=/api/usuarios/**
          filters:
            - StripPrefix=1

      default-filters:
        - AddResponseHeader=X-Gateway, spring-cloud-gateway   # aplica a todas las rutas

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

```java
// GatewayApplication.java — no necesita ninguna anotación adicional
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

Con esta configuración, una petición `GET /api/productos/123` llega al Gateway, que elimina el prefijo `/api` mediante `StripPrefix=1` y reenvía `GET /productos/123` a una instancia de `productos-service` seleccionada por el load balancer desde Eureka. La respuesta vuelve al cliente con el header `X-Gateway: spring-cloud-gateway` añadido por el filtro global.

> [EXAMEN] `lb://` indica al Gateway que resuelva la URI a través del load balancer integrado (Spring Cloud LoadBalancer + Eureka). Sin `lb://`, el Gateway usa la URL literal y no distribuye carga entre instancias. Es uno de los errores más frecuentes al configurar rutas por primera vez.

---

## Cuándo se necesita un API Gateway

No toda arquitectura necesita un Gateway desde el primer día. Añadirlo antes de tiempo introduce una capa de infraestructura que hay que mantener sin que aporte valor real. Estos son los criterios que justifican introducirlo:

| Situación | ¿Necesita Gateway? |
|---|---|
| 1 único servicio expuesto directamente | No — añade complejidad sin beneficio real |
| 2–3 servicios con clientes externos distintos | Sí — centraliza CORS, auth y logging |
| Auth o rate limiting comunes a varios servicios | Sí — evita duplicación en cada servicio |
| SSL Termination centralizada | Sí — simplifica gestión de certificados |
| Rutas que cambian con frecuencia | Sí — los clientes quedan aislados de cambios internos |
| Solo comunicación interna entre servicios | No — para tráfico este-oeste se usa service mesh |
| Necesidad de canary deployment o A/B testing | Sí — el Gateway distribuye tráfico por porcentaje con el predicate `Weight` |
| Múltiples equipos con APIs heterogéneas | Sí — el Gateway unifica la interfaz externa sin imponer cambios internos |

El criterio de ruptura más claro es este: en el momento en que dos servicios distintos comparten alguna política transversal (misma lógica de auth, mismo esquema de rate limiting, mismo CORS), el Gateway deja de ser una mejora opcional y pasa a ser la alternativa correcta frente a la duplicación.

---

## Buenas y malas prácticas

Hacer:
- **Centralizar en el Gateway todo lo que es política transversal**: autenticación, rate limiting, CORS, logging. Cada servicio se simplifica al no tener que implementarlo y el equipo que mantiene el Gateway tiene visibilidad completa sobre todas las políticas de acceso del sistema.
- **Usar `lb://nombre-servicio`** para que el load balancer seleccione la instancia automáticamente. El Gateway no necesita saber cuántas instancias hay ni sus IPs; Eureka las gestiona. Esto hace el Gateway resiliente a reinicios, escalados y movimientos de servicios.
- **Mantener el Gateway sin estado**. No guardar sesiones ni datos de negocio en él. Un Gateway con estado no puede escalarse horizontalmente sin sincronización entre réplicas, lo que convierte en un cuello de botella el elemento que debería ser el más escalable de la arquitectura.
- **Desplegarlo con al menos 2 réplicas** detrás de un balanceador externo (NGINX, AWS ALB, GCP Load Balancer). Es el único punto de entrada de toda la arquitectura externa; un Gateway con una sola instancia es un SPOF (Single Point of Failure) que anula la alta disponibilidad del resto del sistema.
- **Versionar las rutas públicas** desde el inicio (`/api/v1/...`). Cambiar la URL pública de una ruta en el futuro obliga a coordinar con todos los clientes; un prefijo de versión permite introducir `/api/v2/...` en paralelo sin romper los clientes existentes.

Evitar:
- **Incluir lógica de negocio en los filtros del Gateway**. La validación de reglas de dominio, el cálculo de precios o las consultas a bases de datos de negocio no pertenecen al Gateway. Cuando ocurre, el Gateway se convierte en un monolito de infraestructura donde los cambios de dominio requieren tocar la capa de enrutamiento.
- **Añadir `spring-boot-starter-web`** junto a `spring-cloud-starter-gateway`. Spring Boot detecta ambos stacks y falla al arrancar con un error de contexto ambiguo. Si se migra desde un proyecto Servlet, hay que eliminar explícitamente la dependencia web antes de añadir Gateway.
- **Usar URLs internas directas** (`http://localhost:8081`) en lugar de `lb://`. Pierde el load balancing, acopla la configuración a IPs y puertos concretos, y rompe inmediatamente en entornos con múltiples instancias o en Kubernetes donde las IPs son efímeras.
- **Omitir `StripPrefix`** cuando el servicio de destino no incluye el prefijo de la ruta pública. Si el Gateway expone `/api/productos/**` pero el servicio espera `/productos/**`, sin `StripPrefix=1` el servicio recibe `/api/productos/...` y devuelve 404. Es el error de configuración más frecuente en primeras integraciones.

---

## El Gateway en el ecosistema Spring Cloud

El Gateway no opera en aislamiento: colabora con varios componentes del ecosistema Spring Cloud para cumplir sus responsabilidades.

```
                         ┌─────────────────────────────────┐
                         │        Spring Cloud Config       │
                         │  (configuración de rutas y       │
                         │   propiedades del Gateway)       │
                         └────────────────┬────────────────┘
                                          │ spring.config.import
                                          ▼
Cliente ──► HTTPS ──► ┌────────────────────────────────┐
                       │        API Gateway              │
                       │                                 │
                       │  ① Eureka Client               │◄── Service Registry
                       │     descubre instancias         │    (Eureka Server)
                       │                                 │
                       │  ② Spring Cloud LoadBalancer   │
                       │     selecciona instancia        │
                       │                                 │
                       │  ③ Resilience4j CircuitBreaker │
                       │     protege llamadas salientes  │
                       │                                 │
                       │  ④ Micrometer Tracing          │
                       │     propaga Trace ID            │
                       └────────────────┬───────────────┘
                                        │ lb://servicio
                                        ▼
                              Microservicio destino
```

Cada componente resuelve una responsabilidad distinta:
- **Config Server**: externaliza la configuración de rutas. Permite modificar rutas sin redesplegar el Gateway.
- **Eureka**: el Gateway se registra como cliente y consulta las instancias disponibles de cada servicio antes de enrutar.
- **LoadBalancer**: integrado en el Gateway, selecciona la instancia cuando se usa `lb://`. No requiere configuración adicional.
- **Resilience4j**: se integra como filtro predefinido `CircuitBreaker` para abrir el circuito si un servicio downstream falla repetidamente.
- **Micrometer Tracing**: el Gateway añade el `Trace ID` a las peticiones salientes, permitiendo rastrear el flujo completo en Zipkin o Grafana Tempo.

---

## API Gateway vs Backend for Frontend (BFF)

El BFF es un patrón relacionado pero con un propósito distinto. Ambos pueden coexistir: el Gateway como perímetro de seguridad y enrutamiento, los BFF como microservicios internos a los que el Gateway enruta según el tipo de cliente.

```
                        ┌──────────────────────────────┐
  Cliente móvil ────────► Gateway  (:443)               │
  Cliente web   ────────►          │                    │
                        └──────────┼───────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              BFF Móvil       BFF Web       BFF TV
              (agrega         (agrega       (agrega
               datos para      datos para    datos para
               app móvil)      web app)      smart TV)
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             productos-svc   pedidos-svc   usuarios-svc
```

El Gateway aplica auth y rate limiting una sola vez. Cada BFF agrega y adapta los datos al formato que necesita su cliente específico, sin que los microservicios de dominio necesiten conocer a quién sirven.

| Aspecto | API Gateway | BFF |
|---|---|---|
| **Propósito** | Enrutar y aplicar políticas transversales | Adaptar la API a las necesidades de un cliente específico |
| **Instancias** | Una (o clúster) para todos los clientes | Una por tipo de cliente (móvil, web, TV…) |
| **Lógica** | Sin lógica de negocio | Puede agregar llamadas y transformar datos entre servicios |
| **Mantenimiento** | Equipo de plataforma / infraestructura | Equipo del cliente que consume el BFF |
| **Cuándo usarlo** | Siempre que haya más de un servicio expuesto | Cuando los clientes tienen requisitos de datos muy distintos |

> [ADVERTENCIA] No confundir API Gateway con Service Mesh (Istio, Linkerd). El Gateway gestiona tráfico norte-sur: cliente externo hacia el interior del sistema. El Service Mesh gestiona tráfico este-oeste: comunicación entre servicios internos. Son complementarios, no alternativos. Usar un Service Mesh no elimina la necesidad de un API Gateway si hay clientes externos.

---

← [5 — Balanceo de Carga](./05-load-balancing.md) | [Índice](./README.md) | [6.1.2 Arquitectura reactiva →](./06-02-gateway-arquitectura.md)
