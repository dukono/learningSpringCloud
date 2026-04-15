# 6.3.4 Control de flujo: CircuitBreaker, Retry, RedirectTo

← [6.3.3 Headers de respuesta](./06-15-gateway-filtros-headers-response.md) | [Índice](./README.md) | [6.3.5 Protección y utilidades →](./06-17-gateway-filtros-proteccion.md)

---

Estos filtros controlan qué ocurre cuando el servicio destino falla, tarda demasiado o no existe. Son la diferencia entre un Gateway que propaga fallos directamente al cliente y uno que los gestiona con elegancia: reintentando llamadas transitorias, cortando el flujo cuando un servicio está caído y redirigiendo el tráfico cuando una ruta ha cambiado.

## Diagrama: flujo de una petición con CircuitBreaker y fallback

El diagrama muestra cómo se encadenan CircuitBreaker y Retry: el circuito evalúa primero si está abierto —en cuyo caso redirige al `fallbackUri` sin llegar al servicio— y solo cuando está cerrado cede el control al filtro Retry, que gestiona los reintentos y reporta el resultado final al circuito.

```
Cliente
  │
  ▼
Gateway ──► CircuitBreaker (CLOSED)
                │
                │  ¿circuito abierto?
                │
           NO   │                    SÍ
                ▼                    ▼
          Retry filter         fallbackUri
                │              (sin llamar al servicio)
                │
           ¿éxito?
                │
           SÍ   │         NO (reintentos agotados)
                ▼                    ▼
         Servicio destino     fallbackUri / error
                │
                ▼
         Respuesta al cliente
```

---

## CircuitBreaker

El filtro `CircuitBreaker` envuelve la llamada al servicio destino con el circuito de Resilience4j. Si el servicio falla repetidamente, el circuito se abre y las peticiones siguientes se redirigen directamente al `fallbackUri` sin llamar al servicio.

> [PREREQUISITO] CircuitBreaker requiere la dependencia `spring-cloud-starter-circuitbreaker-reactor-resilience4j` en el classpath.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - name: CircuitBreaker
              args:
                name: productosCB          # nombre del circuito en Resilience4j
                fallbackUri: forward:///fallback/productos
                statusCodes:
                  - 500
                  - 503                    # estos códigos cuentan como fallo

        - id: productos-fallback
          uri: lb://fallback-service
          predicates:
            - Path=/fallback/productos
```

> [CONCEPTO] Si el circuito está abierto y no hay `fallbackUri` configurado, el Gateway devuelve directamente un `503 Service Unavailable` al cliente sin intentar contactar con el servicio. El `fallbackUri` es opcional: su ausencia no impide que el circuito funcione, pero el cliente recibe un error genérico en lugar de una respuesta degradada útil.

Configuración del circuito en `application.yml`:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      productosCB:
        slidingWindowSize: 10
        failureRateThreshold: 50           # abre el circuito si >50% de las últimas 10 llamadas fallan
        waitDurationInOpenState: 10s       # tiempo en estado abierto antes de pasar a half-open
        permittedNumberOfCallsInHalfOpenState: 3
        slowCallDurationThreshold: 2s      # llamadas más lentas que 2s cuentan como lentas
        slowCallRateThreshold: 100
  timelimiter:
    instances:
      productosCB:
        timeoutDuration: 3s               # timeout de la llamada individual
```

Diagrama de estados del circuito:

```
         fallo > umbral               timeout
  CLOSED ─────────────────► OPEN ──────────────► HALF-OPEN
    ▲                                                  │
    │                   éxito en sonda                │
    └──────────────────────────────────────────────────┘
                              │ fallo en sonda
                              ▼
                            OPEN (reinicio del timer)
```

---

## Retry

El filtro `Retry` reintenta la petición automáticamente cuando el servicio devuelve ciertos códigos de estado o cuando la conexión falla. Se ejecuta antes de que el CircuitBreaker registre el resultado, por lo que los reintentos cuentan como llamadas individuales en el circuito.

```yaml
filters:
  - name: Retry
    args:
      retries: 3                   # número máximo de reintentos (sin contar el intento inicial)
      statuses: SERVICE_UNAVAILABLE  # códigos HTTP que disparan reintento (nombre del HttpStatus)
      methods: GET,HEAD            # solo reintentar métodos idempotentes
      backoff:
        firstBackoff: 100ms        # espera antes del primer reintento
        maxBackoff: 500ms          # espera máxima entre reintentos
        factor: 2                  # factor multiplicador (100ms → 200ms → 400ms → 500ms)
        basedOnPreviousValue: false
```

> [ADVERTENCIA] NO reintentar automáticamente métodos POST, PUT o DELETE a menos que los servicios sean idempotentes. Un POST reintentado puede crear recursos duplicados.

> [EXAMEN] El campo `statuses` usa el nombre del enum `HttpStatus` de Spring, no el código numérico: `SERVICE_UNAVAILABLE` en lugar de `503`. Usar el código numérico provoca que el filtro ignore la configuración silenciosamente.

---

## RedirectTo

El filtro `RedirectTo` devuelve una respuesta de redirección HTTP al cliente con el código y URL especificados. No llama al servicio destino.

```yaml
filters:
  # Redirección permanente a nueva versión de la API
  - RedirectTo=301, https://api.miempresa.com/api/v2/productos

  # Redirección temporal para mantenimiento
  - RedirectTo=302, https://status.miempresa.com/mantenimiento
```

Casos de uso habituales:

- Migrar URLs de v1 a v2 sin romper clientes existentes.
- Redirigir HTTP a HTTPS a nivel de ruta.
- Redirigir a página de mantenimiento durante despliegues.

---

## Tabla de parámetros

### CircuitBreaker

Los parámetros del filtro CircuitBreaker se declaran en el bloque `args` y controlan el nombre del circuito en Resilience4j, la URI de fallback y los códigos de estado HTTP que se registran como fallos en el circuito.

| Parámetro | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `name` | String | Sí | Nombre del circuito; debe coincidir con la configuración de Resilience4j |
| `fallbackUri` | String | No | URI de fallback; acepta `forward:///path` o URL externa |
| `statusCodes` | List\<String\> | No | Códigos HTTP que se registran como fallo en el circuito; por defecto solo las excepciones de red |

### Retry

El filtro Retry acepta los siguientes parámetros para controlar el número de reintentos, los métodos HTTP y códigos de estado que los disparan, y la estrategia de backoff exponencial que evita sobrecargar el servicio durante la recuperación.

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `retries` | int | 3 | Número de reintentos adicionales tras el intento inicial |
| `statuses` | List\<HttpStatus name\> | — | Códigos que disparan reintento (nombre del enum, no número) |
| `methods` | List\<HttpMethod\> | GET | Métodos HTTP que se pueden reintentar |
| `exceptions` | List\<Class\> | — | Excepciones que disparan reintento |
| `backoff.firstBackoff` | Duration | — | Espera antes del primer reintento |
| `backoff.maxBackoff` | Duration | — | Espera máxima entre reintentos |
| `backoff.factor` | int | — | Multiplicador exponencial entre reintentos |

### RedirectTo

`RedirectTo` acepta dos parámetros posicionales: el código HTTP de redirección y la URL de destino absoluta. No acepta variables de path ni expresiones SpEL; la URL de destino es un literal estático.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `status` | int | Código de redirección: 301, 302, 307 o 308 |
| `url` | String | URL de destino de la redirección |

---

## Buenas y malas prácticas

Hacer:
- Nombrar el circuito con un nombre que identifique el servicio: `productosCB`, `pedidosCB`. Facilita la monitorización de cada circuito en Actuator y en métricas de Resilience4j.
- Implementar el fallback como una respuesta degradada útil (datos en caché, respuesta vacía con status 200) en lugar de devolver un error genérico. Un fallback que devuelve 503 tiene el mismo impacto en el cliente que no tener fallback.
- Combinar CircuitBreaker con Retry en el orden correcto: declarar `CircuitBreaker` antes que `Retry` en la lista de filtros. En este orden, el CircuitBreaker envuelve al Retry: si el Retry agota todos sus intentos y la petición sigue fallando, el CircuitBreaker registra un único fallo. Si se invierte el orden (Retry después de CircuitBreaker), cada intento de reintento es evaluado de forma independiente por el CircuitBreaker, lo que puede abrir el circuito prematuramente antes de que los reintentos tengan oportunidad de recuperarse.

```yaml
filters:
  - name: CircuitBreaker     # envuelve todo, incluido el retry
    args:
      name: productosCB
      fallbackUri: forward:///fallback/productos
  - name: Retry              # se ejecuta dentro del circuito
    args:
      retries: 3
      statuses: SERVICE_UNAVAILABLE
      methods: GET
```

Evitar:
- Reintentar métodos no idempotentes (POST, PATCH, DELETE) sin asegurarse de que el servicio los trata correctamente ante duplicados.
- Configurar `fallbackUri` como un endpoint del mismo servicio que está fallando — si el servicio está caído, el fallback también lo estará.
- Omitir `statusCodes` en CircuitBreaker — por defecto solo las excepciones de conexión abren el circuito; los códigos 500 y 503 devueltos por el servicio no se registran como fallos sin configuración explícita.

---

← [6.3.3 Headers de respuesta](./06-15-gateway-filtros-headers-response.md) | [Índice](./README.md) | [6.3.5 Protección y utilidades →](./06-17-gateway-filtros-proteccion.md)
