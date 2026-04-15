# 6.1.2 Arquitectura reactiva: WebFlux y el modelo no bloqueante

← [6.1.1 Qué es un API Gateway](./06-01-gateway-concepto.md) | [Índice](./README.md) | [6.1.3 Spring Cloud Gateway vs Zuul →](./06-03-gateway-vs-zuul.md)

---

Spring Cloud Gateway no puede ejecutarse sobre el stack Servlet clásico (Spring MVC). No es una limitación arbitraria: es una consecuencia directa de su modelo de rendimiento. Un Gateway que actúa como único punto de entrada de la arquitectura gestiona centenares de peticiones simultáneas, cada una de ellas esperando la respuesta de un servicio downstream. Con el modelo Servlet, cada petición en espera de I/O ocupa un hilo del pool. Si el pool tiene 200 hilos y 200 peticiones están simultáneamente esperando respuesta de backend, la petición 201 queda bloqueada hasta que algún hilo se libere. Con Netty y Project Reactor, los mismos 200 peticiones en espera de I/O se gestionan con 8–16 hilos: mientras un hilo espera la respuesta de un servicio, el event loop lo asigna inmediatamente a procesar el siguiente evento disponible.

> [PREREQUISITO] Este fichero asume familiaridad básica con Project Reactor: qué es `Mono<T>` (flujo de 0 o 1 elemento) y `Flux<T>` (flujo de 0 a N elementos). No se requiere dominar operadores avanzados, pero sí entender que son definiciones perezosas de computaciones asíncronas — no valores inmediatos.

---

## Modelo de hilos: Servlet vs Netty

En el modelo Servlet cada petición en espera de I/O retiene un hilo del pool hasta que el backend responde, lo que convierte el tamaño del pool en el techo de concurrencia efectivo. Netty con Project Reactor invierte esta relación: el mismo hilo del event loop atiende múltiples peticiones intercalando callbacks de I/O, de forma que el número de hilos crece con los núcleos de CPU, no con las peticiones en vuelo.

```
MODELO SERVLET (Spring MVC / Tomcat)

  Thread pool (200 hilos por defecto)

  Petición 1   ──► Hilo 1  [CPU ▒▒ I/O ████████████████ CPU ▒] ──► resp
  Petición 2   ──► Hilo 2  [CPU ▒ I/O ██████████████████ CPU ▒] ──► resp
  ...
  Petición 200 ──► Hilo 200 [CPU ▒ I/O █████████████████ CPU ▒] ──► resp
  Petición 201 ──► BLOQUEADA esperando hilo libre


MODELO NETTY + PROJECT REACTOR (Spring Cloud Gateway)

  Event loop (8 hilos = núcleos de CPU)

  Hilo 1:  ──P1send──P3recv──P7send──P2recv──P9send──P5recv──►
  Hilo 2:  ──P5send──P1recv──P8send──P4recv──P6send──P2recv──►
  ...

  Las esperas de I/O no ocupan hilo: se registra un callback que
  dispara el siguiente paso cuando llega el dato.
  1000 peticiones simultáneas con 8 hilos es normal.
```

El modelo Servlet es más sencillo de razonar (flujo imperativo secuencial por hilo), pero no escala para el caso de uso del Gateway donde prácticamente toda la latencia es I/O. El modelo reactivo escala linealmente con los núcleos de CPU en lugar de con el tamaño del thread pool.

---

## Pipeline interno del Gateway

Cuando una petición llega al Gateway atraviesa esta cadena de componentes antes de ser reenviada al servicio destino.

```
Petición HTTP entrante
       │
       ▼
  HttpWebHandlerAdapter         ← adapta la petición Netty a ServerWebExchange
       │
       ▼
  DispatcherHandler             ← busca el handler que puede procesar el exchange
       │
       ▼
  RoutePredicateHandlerMapping  ← evalúa los predicates de cada ruta en orden de prioridad
       │                           la primera ruta cuyos predicates se cumplen gana
       │                           si ninguna coincide → 404 inmediato
       ▼
  FilteringWebHandler           ← construye la cadena de filtros ordenada por getOrder():
       │                           [filtros globales] + [filtros de la ruta]
       ▼
  GatewayFilter[0]
       │  chain.filter(exchange)   ← cada filtro llama al siguiente devolviendo Mono<Void>
       ▼
  GatewayFilter[1]
       │
       ▼
  ...
       │
       ▼
  NettyRoutingFilter             ← realiza la llamada real al servicio destino
       │                           usa el cliente HTTP reactivo de Netty
       ▼
  Servicio destino (respuesta)
       │
       ▼  (tramo de vuelta: los filtros se deshacen en orden inverso)
  NettyWriteResponseFilter       ← escribe la respuesta al cliente
```

El `NettyRoutingFilter` es el último filtro de la cadena en el tramo de ida. Su `getOrder()` es `Integer.MAX_VALUE - 1`. Un filtro global de seguridad con orden `-100` se ejecuta antes; un filtro que modifica la respuesta con orden `Integer.MAX_VALUE - 2` se ejecuta después de que la respuesta del servicio llegue.

---

## Cómo actúa un filtro en el modelo reactivo

Todo filtro implementa `GatewayFilter` o `GlobalFilter`, cuyo único método es `filter(ServerWebExchange, GatewayFilterChain)` y devuelve `Mono<Void>`. El filtro no ejecuta la lógica directamente: construye una definición reactiva que el framework ejecuta al suscribirse.

`ServerWebExchange` es el contenedor inmutable que encapsula el estado completo de una petición en curso: `exchange.getRequest()` devuelve la petición entrante, `exchange.getResponse()` devuelve la respuesta en construcción, y `exchange.getAttributes()` proporciona un mapa de atributos donde los filtros pueden comunicarse entre sí (por ejemplo, el predicate `Path` almacena ahí las variables capturadas `{version}`). Al ser inmutable, cualquier modificación requiere `exchange.mutate()` para obtener una copia con el cambio.

```java
// Filtro global que añade un correlation ID a cada petición saliente
@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // mutate() crea una copia inmutable con el cambio — no modifica el original
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-Correlation-Id", UUID.randomUUID().toString())
            .build();

        // chain.filter devuelve Mono<Void> — no bloquea, no espera aquí
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }

    @Override
    public int getOrder() {
        return -100; // negativo = antes de los filtros internos del Gateway
    }
}
```

Para modificar la respuesta, se encadena lógica después de `chain.filter`:

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    long inicio = System.currentTimeMillis();

    return chain.filter(exchange)
        .then(Mono.fromRunnable(() -> {
            // Se ejecuta cuando la respuesta del servicio ha llegado
            long duracion = System.currentTimeMillis() - inicio;
            exchange.getResponse().getHeaders()
                .add("X-Response-Time-Ms", String.valueOf(duracion));
        }));
}
```

> [ADVERTENCIA] `Mono.fromRunnable` solo es seguro para operaciones síncronas y rápidas (añadir un header, incrementar un contador en memoria). Para cualquier I/O (consultar Redis, llamar a un servicio externo, leer un fichero), usar `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`. Bloquear el hilo del event loop dentro de un filtro degrada el rendimiento de todas las peticiones que ese hilo gestiona simultáneamente, no solo la petición actual.

---

## Parámetros del cliente HTTP reactivo

La mayoría de proyectos no necesitan ajustar estos valores. Son relevantes cuando el Gateway se convierte en cuello de botella o cuando los servicios downstream tienen latencias altas.

| Propiedad | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `spring.cloud.gateway.httpclient.connect-timeout` | int (ms) | `45000` | Timeout para establecer conexión TCP con el servicio destino |
| `spring.cloud.gateway.httpclient.response-timeout` | Duration | sin límite | Timeout total para recibir la respuesta completa |
| `spring.cloud.gateway.httpclient.pool.type` | enum | `ELASTIC` | `FIXED` (tamaño fijo), `ELASTIC` (crece según demanda), `DISABLED` |
| `spring.cloud.gateway.httpclient.pool.max-connections` | int | `2 × núcleos` | Máximo de conexiones en pool FIXED |
| `spring.cloud.gateway.httpclient.pool.max-idle-time` | Duration | `null` | Tiempo máximo que una conexión puede estar idle en el pool |
| `spring.cloud.gateway.httpclient.pool.acquire-timeout` | Duration | `45s` | Tiempo máximo esperando una conexión libre del pool |

```yaml
# Configuración recomendada para servicios downstream con latencia variable
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 3000          # 3 segundos para establecer conexión
        response-timeout: 10s          # 10 segundos para recibir respuesta completa
        pool:
          type: FIXED
          max-connections: 500         # ajustar según carga esperada
          acquire-timeout: 5000        # 5 segundos esperando conexión del pool
```

---

## Buenas y malas prácticas

Hacer:
- Devolver siempre `chain.filter(exchange)` (o una cadena reactiva basada en él) desde el método `filter`. Un filtro que no llama a `chain.filter` interrumpe la cadena y el Gateway devuelve una respuesta vacía al cliente sin llamar al servicio destino.
- Usar `exchange.getRequest().mutate()` y `exchange.mutate()` para modificar la petición. Los objetos `ServerHttpRequest` y `ServerWebExchange` son inmutables en WebFlux; `mutate()` crea una copia con el cambio sin alterar el original.
- Controlar el orden de los filtros con `getOrder()`. Un filtro de autenticación debe ejecutarse antes (orden más bajo) que un filtro de logging para que las peticiones rechazadas también queden registradas.

Evitar:
- Llamar a métodos bloqueantes dentro de filtros: `Thread.sleep`, JDBC síncrono, `RestTemplate`, `Future.get()`. Bloquear el event loop de Netty es el error de rendimiento más grave en un Gateway reactivo: impacta a todas las peticiones en vuelo, no solo la petición actual.
- Modificar directamente `exchange.getRequest().getHeaders()`. Las colecciones de headers son de solo lectura en WebFlux. Intentarlo lanza `UnsupportedOperationException` en tiempo de ejecución. Siempre usar `mutate()`.
- Asumir que el código dentro de `.then(Mono.fromRunnable(...))` se ejecuta de forma síncrona después de `chain.filter(...)`. El código dentro de un `Mono` se ejecuta cuando el framework lo suscribe, no cuando se define. Las pruebas que no usan `StepVerifier` u operadores de bloqueo para los tests pueden no ejecutar ese código nunca.

---

← [6.1.1 Qué es un API Gateway](./06-01-gateway-concepto.md) | [Índice](./README.md) | [6.1.3 Spring Cloud Gateway vs Zuul →](./06-03-gateway-vs-zuul.md)
