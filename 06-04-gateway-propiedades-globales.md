# 6.1.4 Propiedades globales y arranque

← [6.1.3 Spring Cloud Gateway vs Zuul](./06-03-gateway-vs-zuul.md) | [Índice](./README.md) | [6.1.5 SSL/TLS →](./06-05-gateway-ssl.md)

---

Spring Cloud Gateway expone un conjunto de propiedades de configuración bajo el prefijo `spring.cloud.gateway.*` que controlan el comportamiento del Gateway como conjunto, independientemente de las rutas individuales que se definan. Comprender la diferencia entre estas propiedades globales y las propiedades de ruta es fundamental antes de empezar a configurar rutas, porque errores en las propiedades globales afectan a todo el tráfico que pasa por el Gateway, no solo a una ruta concreta.

Las propiedades dentro de `routes[].filters` o `routes[].predicates` son exclusivas de esa ruta: solo el tráfico que coincide con esa ruta las aplica. Las propiedades fuera de `routes:`, como `httpclient.*`, `globalcors.*` o `metrics.*`, afectan al Gateway en su totalidad. El caso intermedio es `default-filters:`, que se define fuera de las rutas pero se aplica a todas ellas, actuando como un conjunto de filtros comunes que evitan repetición en cada definición de ruta.

## Jerarquía de propiedades `spring.cloud.gateway.*`

El espacio de nombres de configuración del Gateway tiene una estructura jerárquica bien definida. Conocer esta estructura de un vistazo facilita navegar la documentación y localizar rápidamente qué propiedad controla cada aspecto del comportamiento.

```
spring.cloud.gateway
├── enabled                        # habilita/deshabilita el Gateway
├── default-filters                # filtros aplicados a todas las rutas
├── routes                         # lista de rutas (ver 6.2.x)
├── globalcors                     # CORS global
│   └── cors-configurations
│       └── '[/**]'
│           ├── allowed-origins
│           ├── allowed-methods
│           └── allowed-headers
├── httpclient                     # cliente HTTP hacia los servicios
│   ├── connect-timeout
│   ├── response-timeout
│   ├── pool.*
│   └── ssl.*
├── metrics
│   └── enabled
├── discovery
│   └── locator
│       ├── enabled
│       └── lower-case-service-id
└── loadbalancer
    └── use404
```

## Ejemplo completo de `application.yml`

El siguiente ejemplo muestra las propiedades globales más relevantes en un fichero `application.yml` realista. Cada propiedad está acompañada de un comentario que explica su efecto. Este fichero es un punto de partida funcional para un Gateway de producción.

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      # Habilita o deshabilita completamente el Gateway.
      # Útil para desactivarlo en perfiles de prueba sin eliminar la dependencia.
      enabled: true

      # Filtros que se aplican a TODAS las rutas, en el orden aquí definido,
      # antes de los filtros específicos de cada ruta.
      default-filters:
        - AddResponseHeader=X-Gateway-Version, 1.0
        - DedupeResponseHeader=Access-Control-Allow-Origin

      # Configuración del cliente HTTP reactivo (Reactor Netty) que el Gateway
      # utiliza para conectarse a los servicios de destino (upstream).
      httpclient:
        # Tiempo máximo en ms para establecer la conexión TCP con el upstream.
        connect-timeout: 3000
        # Tiempo máximo de espera para recibir la respuesta completa del upstream.
        # Acepta formato de Duration de Spring (10s, 500ms, PT10S…).
        response-timeout: 10s
        pool:
          # FIXED: pool de tamaño fijo. ELASTIC: crece bajo demanda. DISABLED: sin pool.
          type: FIXED
          # Número máximo de conexiones en el pool (solo aplica si type=FIXED).
          max-connections: 500
          # Tiempo máximo en ms que una solicitud espera para obtener una conexión del pool.
          acquire-timeout: 5000

      # Configuración de CORS a nivel global (afecta a todas las rutas).
      globalcors:
        # Imprescindible para que las preflight OPTIONS requests sean procesadas
        # correctamente por el Gateway y no devuelvan 404.
        add-to-simple-url-handler-mapping: true
        cors-configurations:
          '[/**]':
            allowed-origins:
              - "https://app.miempresa.com"
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            # '*' permite cualquier cabecera; en producción considera listar explícitamente.
            allowed-headers: "*"
            allow-credentials: true
            # Tiempo en segundos que el navegador puede cachear la respuesta preflight.
            max-age: 3600

      # Expone métricas del Gateway a través de Micrometer (requiere actuator).
      metrics:
        enabled: true

      # Controla el comportamiento cuando el LoadBalancer (lb://) no encuentra
      # instancias disponibles del servicio en el registro (p. ej. Eureka).
      loadbalancer:
        # false (default): devuelve 503 Service Unavailable.
        # true: devuelve 404 Not Found — útil en desarrollo.
        use404: true

# Exponer los endpoints de actuator necesarios para gestionar y observar el Gateway.
management:
  endpoints:
    web:
      exposure:
        include: gateway,health,info
```

## Tabla de propiedades globales

La siguiente tabla recoge todas las propiedades globales relevantes de `spring.cloud.gateway.*`, sus tipos, valores por defecto y una descripción de su efecto.

| Propiedad | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `spring.cloud.gateway.enabled` | `boolean` | `true` | Habilita o deshabilita el Gateway por completo. Al desactivarlo, el contexto de Spring se inicia pero el Gateway no procesa ninguna petición. |
| `spring.cloud.gateway.default-filters` | `List<String>` | `[]` | Lista de filtros aplicados a todas las rutas antes que los filtros específicos de cada ruta. Sigue la misma sintaxis que `routes[].filters`. |
| `spring.cloud.gateway.httpclient.connect-timeout` | `int` (ms) | `45000` | Tiempo máximo en milisegundos para establecer la conexión TCP con el servicio destino. |
| `spring.cloud.gateway.httpclient.response-timeout` | `Duration` | sin límite | Tiempo máximo de espera para recibir la respuesta completa del servicio destino. Acepta formatos como `10s`, `500ms` o ISO-8601 `PT10S`. |
| `spring.cloud.gateway.httpclient.pool.type` | `enum` (`ELASTIC`, `FIXED`, `DISABLED`) | `ELASTIC` | Estrategia del pool de conexiones. `FIXED` limita el número máximo; `ELASTIC` crece bajo demanda; `DISABLED` no usa pool. |
| `spring.cloud.gateway.httpclient.pool.max-connections` | `int` | `2 × nº núcleos` | Número máximo de conexiones del pool. Solo tiene efecto si `pool.type=FIXED`. |
| `spring.cloud.gateway.httpclient.pool.max-idle-time` | `Duration` | `null` | Tiempo máximo que una conexión puede permanecer inactiva en el pool antes de cerrarse. `null` significa sin límite. |
| `spring.cloud.gateway.httpclient.pool.acquire-timeout` | `Duration` | `45s` | Tiempo máximo que una petición espera para obtener una conexión del pool. Superado este tiempo, la petición falla. |
| `spring.cloud.gateway.globalcors.cors-configurations` | `Map<String, CorsConfiguration>` | `{}` | Mapa de configuraciones CORS indexadas por patrón de ruta. La clave `[/**]` aplica a todas las rutas. |
| `spring.cloud.gateway.globalcors.add-to-simple-url-handler-mapping` | `boolean` | `false` | Registra la configuración CORS también en el `SimpleUrlHandlerMapping`, necesario para que las preflight `OPTIONS` sean gestionadas correctamente. |
| `spring.cloud.gateway.metrics.enabled` | `boolean` | `false` | Activa las métricas de Micrometer para el Gateway. Requiere `spring-boot-starter-actuator` en el classpath. |
| `spring.cloud.gateway.discovery.locator.enabled` | `boolean` | `false` | Activa la creación automática de rutas a partir del registro de servicios (Eureka, Consul…). |
| `spring.cloud.gateway.discovery.locator.lower-case-service-id` | `boolean` | `false` | Convierte el `serviceId` a minúsculas en las rutas generadas automáticamente. Necesario cuando los IDs de servicio en Eureka están en mayúsculas. |
| `spring.cloud.gateway.loadbalancer.use404` | `boolean` | `false` | Cuando el LoadBalancer (`lb://`) no encuentra instancias del servicio, devuelve `404` en lugar del `503` por defecto. |

## `default-filters` vs filtros en la ruta

`default-filters` y los filtros definidos dentro de cada ruta no son mutuamente excluyentes: ambos coexisten y se aplican de forma complementaria. Entender el orden de aplicación evita comportamientos inesperados.

Los `default-filters` se aplican a todas las rutas, en el orden en que están definidos en el fichero de configuración, y se ejecutan **antes** que los filtros específicos de la ruta. Esto significa que si se define un `default-filter` que añade una cabecera `X-Request-Id` y también hay un filtro en la ruta que añade otra cabecera, ambas cabeceras estarán presentes en la petición al servicio destino.

Un aspecto importante es que si un `default-filter` y un filtro de ruta tienen el mismo nombre y tipo, **ambos se aplican**; el filtro de ruta no sobreescribe al `default-filter`. Por ejemplo, si `default-filters` contiene `AddResponseHeader=X-Gateway-Version, 1.0` y una ruta también define `AddResponseHeader=X-Gateway-Version, 2.0`, la respuesta tendrá dos cabeceras `X-Gateway-Version` con ambos valores.

```yaml
spring:
  cloud:
    gateway:
      # Este filtro se aplica a TODAS las rutas
      default-filters:
        - AddResponseHeader=X-Gateway-Version, 1.0

      routes:
        - id: servicio-usuarios
          uri: lb://usuarios-service
          predicates:
            - Path=/api/usuarios/**
          filters:
            # Este filtro solo aplica a esta ruta, DESPUÉS de los default-filters
            - AddResponseHeader=X-Service-Name, usuarios
```

> [CONCEPTO] `default-filters` es la forma idiomática de aplicar comportamiento transversal a todas las rutas sin duplicar configuración. Úsalo para aspectos como añadir cabeceras de versión, deduplicar cabeceras CORS o registrar métricas comunes.

## `globalcors.add-to-simple-url-handler-mapping`

La configuración CORS del Gateway funciona a través de un handler específico del contexto reactivo de Spring WebFlux. Sin embargo, las preflight requests (`OPTIONS`) son un tipo especial de petición que los navegadores envían antes de la petición real para verificar los permisos CORS. En ciertos escenarios, estas peticiones pueden no llegar al handler del Gateway y ser rechazadas por el `DispatcherHandler` antes de que el Gateway pueda procesarlas.

> [ADVERTENCIA] Sin `add-to-simple-url-handler-mapping: true`, las preflight `OPTIONS` pueden devolver `404` en lugar de la respuesta CORS esperada, lo que hará que el navegador bloquee la petición real. Activa esta propiedad siempre que configures `globalcors` en producción.

La propiedad `add-to-simple-url-handler-mapping: true` registra la configuración CORS en el `SimpleUrlHandlerMapping`, asegurando que las preflight `OPTIONS` sean interceptadas y respondidas correctamente por el Gateway antes de que el `DispatcherHandler` pueda devolver un 404.

## `loadbalancer.use404`

Cuando el Gateway utiliza el esquema `lb://` para resolver el destino de una ruta, delega la resolución en el `ReactorLoadBalancer`. Si el servicio referenciado no tiene instancias registradas en el registro de servicios (por ejemplo, Eureka), el LoadBalancer no puede resolver la URI y el Gateway devuelve un error.

El comportamiento por defecto en este caso es devolver un `503 Service Unavailable`, que semánticamente es correcto: el servicio existe pero no está disponible en este momento. Sin embargo, en entornos de desarrollo donde los microservicios se levantan y apagan con frecuencia, recibir un 503 puede ser confuso.

Con `use404: true`, el Gateway devuelve `404 Not Found` cuando no hay instancias disponibles, lo que en desarrollo puede resultar más informativo para identificar que el servicio destino no está registrado.

> [EXAMEN] En producción, `503 Service Unavailable` es el comportamiento correcto y debe mantenerse (`use404: false`): indica que el servicio existe pero no está disponible, lo que permite que los clientes y proxies intermedios actúen en consecuencia (p. ej. reintentos). El uso de `use404: true` está pensado exclusivamente para facilitar el desarrollo local.

## Buenas y malas prácticas

Las siguientes recomendaciones se derivan de errores comunes al configurar el Gateway en proyectos reales.

**Buenas prácticas:**

- Definir `httpclient.connect-timeout` y `httpclient.response-timeout` siempre de forma explícita. Los valores por defecto (45 segundos) son demasiado altos para APIs web y pueden provocar que hilos y conexiones del pool queden bloqueados durante mucho tiempo.
- Usar `pool.type: FIXED` con un `max-connections` calculado en función de la carga esperada. `ELASTIC` puede provocar agotamiento de recursos en picos de tráfico.
- Activar `metrics.enabled: true` en todos los entornos (desarrollo, preproducción y producción) para detectar cuellos de botella en el Gateway.
- Mantener `globalcors` centralizado en el Gateway y deshabilitar CORS en los servicios individuales para evitar configuraciones CORS duplicadas y potencialmente contradictorias.

**Malas prácticas:**

- Dejar `httpclient.response-timeout` sin valor (sin límite) en producción. Una respuesta lenta de un upstream puede hacer que el Gateway consuma todas las conexiones del pool.
- Usar `use404: true` en producción. Los sistemas de monitorización y los clientes interpretan 503 y 404 de forma muy diferente; devolver 404 cuando el servicio no está disponible puede ocultar problemas reales.
- Configurar `discovery.locator.enabled: true` en producción sin revisar las rutas que genera automáticamente. Las rutas auto-generadas no tienen filtros de seguridad ni limitación de tasa, y pueden exponer endpoints no deseados.
- Poner credenciales o secrets directamente en `application.yml`. Las propiedades como `allowed-origins` que contienen nombres de dominio son seguras, pero cualquier token o contraseña debe externalizarse mediante Spring Cloud Config o variables de entorno.

---

← [6.1.3 Spring Cloud Gateway vs Zuul](./06-03-gateway-vs-zuul.md) | [Índice](./README.md) | [6.1.5 SSL/TLS →](./06-05-gateway-ssl.md)
