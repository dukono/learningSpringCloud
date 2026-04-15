# 6.4.1 AbstractGatewayFilterFactory: filtro por ruta con parámetros YAML

← [6.3.5 Protección y utilidades](./06-17-gateway-filtros-proteccion.md) | [Índice](./README.md) | [6.4.2 GlobalFilter y Ordered →](./06-19-gateway-global-filter.md)

---

Los filtros predefinidos de Spring Cloud Gateway cubren la mayoría de los casos habituales, pero en proyectos reales siempre aparecen requisitos específicos: añadir cabeceras calculadas dinámicamente, validar formatos de datos propietarios, enriquecer peticiones con metadatos internos o aplicar políticas de negocio que no tienen un filtro equivalente en el catálogo estándar. `AbstractGatewayFilterFactory` es el punto de extensión para este tipo de filtros: permite crear filtros configurables por ruta mediante YAML, de la misma forma que los filtros predefinidos del framework, con soporte para notación abreviada y expandida.

La diferencia clave con `GlobalFilter` es el ámbito de aplicación. Un filtro basado en `AbstractGatewayFilterFactory` solo actúa en las rutas en las que se declara explícitamente. Esto lo hace adecuado para lógica que aplica a un subconjunto de rutas, mientras que `GlobalFilter` aplica a todas las rutas del Gateway sin excepción.

## Diagrama: factory vs filtro en el pipeline

El ciclo de vida tiene dos fases diferenciadas: la factory se invoca una sola vez en el arranque para construir el `GatewayFilter` por ruta, y ese filtro se invoca después en cada petición que coincide con el predicado de la ruta. Entender esta separación es clave para saber dónde poner la lógica costosa (en la factory) y dónde la lógica por petición (en el filtro).

```
ARRANQUE DE LA APLICACIÓN:
┌──────────────────────────────────────────────────────┐
│  Spring container detecta @Component                 │
│  LogRequestGatewayFilterFactory                      │
│        │                                             │
│        ▼                                             │
│  Spring registra la factory por nombre:              │
│  "LogRequest" (nombre de clase sin sufijo Factory)   │
│        │                                             │
│        ▼                                             │
│  YAML parser: `- LogRequest=INFO-GW, true`           │
│  llama a factory.apply(Config{prefix="INFO-GW",      │
│                               includeHeaders=true})  │
│  → devuelve GatewayFilter (instancia por ruta)       │
└──────────────────────────────────────────────────────┘

CADA PETICIÓN:
Cliente → [pre-filter: loguea método+path] → Servicio → [post-filter: loguea status] → Cliente
```

La factory se ejecuta una vez en el arranque para crear el `GatewayFilter`. El `GatewayFilter` resultante se ejecuta en cada petición que coincide con la ruta.

## Implementación completa: LogRequestGatewayFilterFactory

`LogRequestGatewayFilterFactory` registra en el pre-filter el método y el path de la petición entrante, y en el post-filter el código de estado de la respuesta. Los dos parámetros configurables desde YAML son `prefix` (etiqueta que identifica la ruta en el log) e `includeHeaders` (si se vuelcan todos los headers de la petición).

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.util.Arrays;
import java.util.List;

@Component
public class LogRequestGatewayFilterFactory
        extends AbstractGatewayFilterFactory<LogRequestGatewayFilterFactory.Config> {

    public LogRequestGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // PRE-FILTER: ejecuta antes de reenviar la petición al servicio
            StringBuilder log = new StringBuilder()
                .append("[").append(config.getPrefix()).append("] ")
                .append(request.getMethod()).append(" ")
                .append(request.getURI().getPath());

            if (config.isIncludeHeaders()) {
                request.getHeaders().forEach((name, values) ->
                    log.append("\n  ").append(name).append(": ").append(values)
                );
            }

            System.out.println(log); // en producción usar Logger de SLF4J

            // POST-FILTER: ejecuta después de que el servicio responde
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                ServerHttpResponse response = exchange.getResponse();
                System.out.println("[" + config.getPrefix() + "] Response status: "
                    + response.getStatusCode());
            }));
        };
    }

    // shortcutFieldOrder define el orden de argumentos en la notación abreviada:
    // `- LogRequest=MI-PREFIJO, true` → Config{prefix="MI-PREFIJO", includeHeaders=true}
    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("prefix", "includeHeaders");
    }

    public static class Config {
        private String prefix = "GATEWAY";
        private boolean includeHeaders = false;

        public String getPrefix() { return prefix; }
        public void setPrefix(String prefix) { this.prefix = prefix; }

        public boolean isIncludeHeaders() { return includeHeaders; }
        public void setIncludeHeaders(boolean includeHeaders) {
            this.includeHeaders = includeHeaders;
        }
    }
}
```

> [CONCEPTO] Spring Cloud Gateway registra la factory por el nombre de la clase sin el sufijo `GatewayFilterFactory`. La clase `LogRequestGatewayFilterFactory` se referencia en YAML como `LogRequest`. Si la clase se llamara `LogFilter`, Spring no la reconocería automáticamente como una factory y habría que sobreescribir el método `name()`.

## Configuración YAML

La factory `LogRequestGatewayFilterFactory` implementada arriba se declara en YAML de dos formas: abreviada (shortcut) y expandida. La forma abreviada requiere que `shortcutFieldOrder()` esté sobreescrito y devuelva los campos en el mismo orden que los argumentos del YAML; la expandida funciona siempre y es recomendada para configuraciones con muchos argumentos o tipos complejos.

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Forma abreviada (shortcut): usa el orden de shortcutFieldOrder()
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - LogRequest=PRODUCTOS-API, true

        # Forma expandida (expanded): explícita, recomendada para args complejos
        - id: pedidos-route
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - name: LogRequest
              args:
                prefix: PEDIDOS-API
                includeHeaders: false
```

> [EXAMEN] Para que la notación abreviada (`- LogRequest=valor1, valor2`) funcione, el método `shortcutFieldOrder()` debe estar sobreescrito y devolver los nombres de los campos en el mismo orden en que se pasan los argumentos. Si no se sobreescribe, Spring solo acepta la forma expandida y lanza un error de configuración al usar la forma abreviada.

## Parámetros de la clase Config

La clase `Config` encapsula la configuración que el YAML inyecta en cada instancia del filtro. Los campos que define `LogRequestGatewayFilterFactory.Config` son los siguientes:

| Parámetro | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `prefix` | `String` | `"GATEWAY"` | Prefijo que aparece al inicio de cada línea de log para identificar la ruta origen del filtro |
| `includeHeaders` | `boolean` | `false` | Si es `true`, añade al log todos los headers HTTP de la petición; útil en desarrollo para depuración de headers |

> [CONCEPTO] Los valores por defecto del `Config` deben inicializarse en la declaración del campo (como en `private String prefix = "GATEWAY"`), no en el constructor. Spring asigna los valores del YAML mediante setters después de construir el objeto `Config`, de modo que los valores por defecto se aplican solo a los campos que el YAML no sobreescribe explícitamente.

## El método `shortcutFieldOrder()` en detalle

El método `shortcutFieldOrder()` devuelve una lista de nombres de campos de la clase `Config` que se mapean posicionalmente a los argumentos de la notación abreviada. Spring Cloud Gateway usa esta lista para asignar cada argumento del YAML al campo correcto del objeto `Config` mediante reflection.

```java
// Si shortcutFieldOrder devuelve ["prefix", "includeHeaders"]:
// - LogRequest=GATEWAY-LOG, true
//   → Config.prefix = "GATEWAY-LOG"
//   → Config.includeHeaders = true

// Si shortcutFieldOrder NO está sobreescrito:
// - LogRequest=GATEWAY-LOG, true  → ERROR o comportamiento indefinido
// Solo funciona la forma expandida con name/args
```

> [ADVERTENCIA] Si los campos del `Config` tienen tipos no estándar (como `Duration` o `List<String>`), Spring puede necesitar conversores adicionales para parsear los valores del YAML. Para tipos complejos, la forma expandida es más robusta que la abreviada.

## Acceso a metadatos de la ruta desde el filtro

Dentro del `apply()` se puede acceder al `id` de la ruta activa para logging o decisiones condicionales:

```java
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;

// Dentro del GatewayFilter:
Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
String routeId = route != null ? route.getId() : "unknown";
```

## Transformación del body de respuesta

Cuando se necesita modificar el body de la respuesta, la clase base correcta es `AbstractGatewayFilterFactory` combinada con `ModifyResponseBodyGatewayFilterFactory` como dependencia, o extender directamente `ModifyResponseBodyGatewayFilterFactory`:

```java
@Component
public class MaskSensitiveDataGatewayFilterFactory
        extends AbstractGatewayFilterFactory<MaskSensitiveDataGatewayFilterFactory.Config> {

    public MaskSensitiveDataGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpResponse originalResponse = exchange.getResponse();
            DataBufferFactory bufferFactory = originalResponse.bufferFactory();

            ServerHttpResponseDecorator decoratedResponse = new ServerHttpResponseDecorator(originalResponse) {
                @Override
                public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                    if (body instanceof Flux) {
                        Flux<? extends DataBuffer> fluxBody = (Flux<? extends DataBuffer>) body;
                        return super.writeWith(fluxBody.map(dataBuffer -> {
                            byte[] content = new byte[dataBuffer.readableByteCount()];
                            dataBuffer.read(content);
                            DataBufferUtils.release(dataBuffer);
                            // Enmascara el campo "cardNumber" en la respuesta JSON
                            String bodyStr = new String(content, StandardCharsets.UTF_8)
                                .replaceAll("\"cardNumber\"\\s*:\\s*\"[^\"]+\"",
                                            "\"cardNumber\":\"****-****-****-****\"");
                            byte[] bytes = bodyStr.getBytes(StandardCharsets.UTF_8);
                            return bufferFactory.wrap(bytes);
                        }));
                    }
                    return super.writeWith(body);
                }
            };

            return chain.filter(exchange.mutate().response(decoratedResponse).build());
        };
    }

    public static class Config {
        // Sin parámetros de configuración en este ejemplo;
        // se puede extender con campos como List<String> fieldsToMask
    }
}
```

> [ADVERTENCIA] Los filtros que modifican el body de respuesta (`ModifyResponseBody`) no son compatibles con conexiones WebSocket ni con streams SSE (Server-Sent Events). Si una ruta sirve WebSocket y tiene este tipo de filtro, el upgrade de protocolo falla silenciosamente o el stream se corrompe.

## Tabla de métodos sobreescribibles

`AbstractGatewayFilterFactory` define varios métodos que la subclase puede sobreescribir para personalizar el comportamiento de la factory. Solo `apply()` es obligatorio; el resto controlan el registro automático en el contexto de Spring, la notación abreviada en YAML y los valores por defecto del `Config`.

| Método | Obligatorio | Descripción |
|---|---|---|
| `apply(Config config)` | Sí | Crea y devuelve la instancia `GatewayFilter` que se ejecutará en cada petición |
| `shortcutFieldOrder()` | No | Define el orden de campos para la notación abreviada en YAML |
| `name()` | No | Sobreescribe el nombre con el que Spring registra la factory. Por defecto: nombre de clase sin sufijo `GatewayFilterFactory` |
| `shortcutType()` | No | Define cómo se mapean los argumentos abreviados (`DEFAULT`, `GATHER_LIST`, `GATHER_LIST_TAIL_FLAG`) |
| `newConfig()` | No | Permite crear instancias del `Config` con valores por defecto personalizados |

## Buenas y malas prácticas

Hacer:
- Nombrar la clase con el sufijo `GatewayFilterFactory` para que Spring la registre automáticamente y el nombre en YAML coincida con el nombre semántico del filtro. Una clase `HmacValidationGatewayFilterFactory` se usa en YAML como `HmacValidation`.
- Usar `Mono.fromRunnable()` para los efectos secundarios del post-filter (logs, métricas) en lugar de lambdas dentro de `.then()` que puedan bloquear. `Mono.fromRunnable()` garantiza que el código se ejecuta en el scheduler de Reactor sin bloquear el hilo reactivo.
- Documentar los campos del `Config` con comentarios o Javadoc que indiquen el propósito, el tipo esperado y el valor por defecto. El equipo que mantiene el YAML no siempre tiene acceso al código fuente.
- Testear la factory con `@WebFluxTest` + `WebTestClient` usando un servidor mock del backend para verificar que el filtro actúa correctamente en el pre-filter y el post-filter.

Evitar:
- No hacer llamadas bloqueantes (`.block()`, JDBC, `Thread.sleep()`) dentro del `apply()` ni dentro del `GatewayFilter` que devuelve. El scheduler reactivo de Netty no está diseñado para bloquear y el impacto en latencia afecta a todo el Gateway.
- No compartir estado mutable entre peticiones en la instancia del `Config` o en la factory. El `Config` se crea una vez por ruta en el arranque, pero el `GatewayFilter` se invoca concurrentemente para múltiples peticiones simultáneas.
- No omitir `shortcutFieldOrder()` si se planea usar la notación abreviada. El error que produce es confuso: Spring no lanza una excepción clara en el arranque sino que asigna los argumentos de forma incorrecta o ignora la configuración.

## Comparación: AbstractGatewayFilterFactory vs GlobalFilter

Ambos mecanismos producen filtros que participan en el mismo pipeline reactivo, pero difieren en el ámbito de aplicación y en la forma en que se configuran. La elección entre uno y otro depende de si la lógica es transversal a todas las rutas o específica de un subconjunto con parámetros distintos por ruta.

| Aspecto | AbstractGatewayFilterFactory | GlobalFilter |
|---|---|---|
| Ámbito | Solo rutas donde se declara explícitamente | Todas las rutas del Gateway |
| Configuración por ruta | Sí, mediante clase `Config` e inyección de YAML | No (misma lógica para todas las rutas) |
| Notación YAML | Shortcut y expandida, como filtros built-in | No aplica (es un bean, no se declara en YAML) |
| Orden en el pipeline | Después de los filtros globales del framework, en el orden de declaración de la ruta | Controlado por `getOrder()`, puede ser antes o después de cualquier filtro |
| Caso de uso | Lógica específica de un grupo de rutas con parámetros variables | Lógica transversal a todas las rutas: correlación, autenticación, métricas |

---

← [6.3.5 Protección y utilidades](./06-17-gateway-filtros-proteccion.md) | [Índice](./README.md) | [6.4.2 GlobalFilter y Ordered →](./06-19-gateway-global-filter.md)
