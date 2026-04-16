# 3.10 WebSockets y Server-Sent Events

← [3.9 Rutas dinámicas y recarga en caliente](sc-gateway-rutas-dinamicas.md) | [Índice](README.md) | [3.11 Filtros y predicados personalizados (extension points)](sc-gateway-extension-points.md) →

---

## Introducción

Los protocolos de conexión persistente (WebSocket y SSE) presentan requisitos distintos al proxying HTTP convencional: la conexión no se cierra tras la respuesta, sino que permanece abierta para el intercambio bidireccional (WebSocket) o para el streaming unidireccional desde el servidor (SSE). Spring Cloud Gateway soporta ambos de forma nativa gracias a su modelo reactivo: `WebsocketRoutingFilter` detecta el header `Upgrade: websocket` y hace proxy del WebSocket sin necesidad de configuración adicional; SSE funciona a través del streaming reactivo `Flux` del propio WebFlux. El desarrollador necesita configurar Gateway para proxiear conexiones WebSocket y streams SSE sin interrumpir la conexión persistente.

## Diagrama: ciclo de vida de una conexión WebSocket en Gateway

El siguiente diagrama muestra cómo Gateway gestiona el upgrade HTTP→WebSocket y el proxying bidireccional.

```
Cliente (browser)                 Gateway                      Backend WS
     │                               │                              │
     │── GET /ws/chat ──────────────►│                              │
     │   Upgrade: websocket          │                              │
     │   Connection: Upgrade         │                              │
     │                               │                              │
     │                    WebsocketRoutingFilter detecta Upgrade     │
     │                               │── GET /ws/chat ─────────────►│
     │                               │   Upgrade: websocket         │
     │                               │◄─ 101 Switching Protocols ───│
     │◄─ 101 Switching Protocols ────│                              │
     │                               │                              │
     │═══════════ WebSocket bidireccional ══════════════════════════│
     │  frame →                      │  frame →                     │
     │  ← frame                      │  ← frame                     │
     │                               │                              │
     │── CLOSE frame ───────────────►│── CLOSE frame ──────────────►│
     │◄─ CLOSE frame ────────────────│◄─ CLOSE frame ───────────────│
     │                               │                              │

Nota: durante la conexión WebSocket activa, los GatewayFilters PRE
ya se ejecutaron (al momento del upgrade); NO se re-ejecutan para
cada frame. Los POST-filters se ejecutan al cerrar la conexión.
```

## Ejemplo central

El siguiente ejemplo muestra la configuración de una ruta de WebSocket y una de SSE. No se requiere configuración especial más allá del predicado `Path`: la detección del WebSocket upgrade es automática.

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # ── Ruta WebSocket ─────────────────────────────────────────────
        # El predicado Path funciona igual para HTTP y WebSocket.
        # Gateway detecta Upgrade: websocket automáticamente.
        - id: websocket-chat-route
          uri: lb://chat-service          # ws:// se usa internamente; lb:// es válido
          predicates:
            - Path=/ws/chat/**
          # Los filtros aplicables en WebSocket son limitados:
          # - Se pueden usar: AddRequestHeader, RemoveRequestHeader, SetRequestHeader
          # - NO se pueden usar: ModifyRequestBody, ModifyResponseBody (el body es el stream WS)
          filters:
            - AddRequestHeader=X-Gateway-Source, websocket-proxy
          metadata:
            # connect-timeout aplica al upgrade inicial
            connect-timeout: 2000
            # response-timeout NO aplica a WS (la conexión es persistente)

        # ── Ruta SSE (Server-Sent Events) ─────────────────────────────
        # SSE es HTTP estándar con Content-Type: text/event-stream.
        # Gateway hace proxy del Flux<ServerSentEvent> sin configuración especial.
        - id: sse-events-route
          uri: lb://events-service
          predicates:
            - Path=/api/events/stream
          filters:
            - AddRequestHeader=X-Client-Id, "#{T(java.util.UUID).randomUUID()}"
          metadata:
            # response-timeout DEBE deshabilitarse o ponerse muy alto para SSE:
            # si el backend tarda en emitir el primer evento, el timeout corta la conexión.
            response-timeout: 3600000    # 1 hora (en ms); equivale a timeout virtual

      # ── Configuración del cliente WebSocket de Netty ────────────────
      httpclient:
        websocket:
          # Tamaño máximo de un frame WebSocket (en bytes)
          # Default: 65536 (64KB). Aumentar si los mensajes son más grandes.
          max-frame-payload-length: 1048576    # 1 MB
          # proxy-ping: si Gateway debe responder a los PING frames del backend
          proxy-ping: true
```

### Backend de ejemplo: servicio WebSocket con Spring WebFlux

```java
package com.example.chat;

import org.springframework.stereotype.Component;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Mono;

/**
 * Handler WebSocket reactivo en el servicio backend.
 * Gateway hace proxy transparente de los frames sin modificación.
 */
@Component
public class ChatWebSocketHandler implements WebSocketHandler {

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        // Echo handler: reenvía cada mensaje recibido al mismo cliente
        return session.send(
            session.receive()
                .map(message -> session.textMessage(
                    "Echo: " + message.getPayloadAsText()))
        );
    }
}
```

### Backend de ejemplo: endpoint SSE con Spring WebFlux

```java
package com.example.events;

import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.time.Duration;

@RestController
public class EventsController {

    @GetMapping(value = "/api/events/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamEvents() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(seq -> ServerSentEvent.<String>builder()
                .id(String.valueOf(seq))
                .event("update")
                .data("Event #" + seq)
                .build());
    }
}
```

## Tabla de elementos clave

La siguiente tabla recoge los parámetros específicos de WebSocket y las restricciones de filtros en conexiones persistentes.

| Parámetro / Concepto | Valor / Descripción |
|---------------------|---------------------|
| `httpclient.websocket.max-frame-payload-length` | Tamaño máximo de frame WS en bytes (default: 65536) |
| `httpclient.websocket.proxy-ping` | Si `true`, Gateway responde a los PING del backend con PONG |
| Detección de WebSocket | Automática vía header `Upgrade: websocket`; no requiere configuración |
| Scheme URI para WS backend | `lb://`, `ws://`, `wss://` — todos son válidos |
| `response-timeout` en WS | No aplica a conexiones WebSocket activas; solo aplica al upgrade inicial |
| `response-timeout` en SSE | Debe deshabilitarse o configurarse muy alto; SSE es una conexión larga |
| Filtros aplicables en WS | `AddRequestHeader`, `RemoveRequestHeader`, `SetRequestHeader` |
| Filtros NO aplicables en WS | `ModifyRequestBody`, `ModifyResponseBody`, `CircuitBreaker` con timeout |

> [ADVERTENCIA] `CircuitBreakerGatewayFilterFactory` con `TimeLimiter` no es compatible con WebSocket: el timeout del TimeLimiter cierra la conexión después del tiempo configurado, interrumpiendo la sesión WebSocket. Si se necesita resiliencia en rutas WebSocket, usar lógica de reconexión en el cliente en lugar de CircuitBreaker en el gateway.

> [EXAMEN] La principal diferencia de comportamiento entre HTTP y WebSocket en Gateway es cuándo se ejecutan los filtros: en HTTP los filtros PRE se ejecutan antes de la llamada y POST después de la respuesta; en WebSocket los filtros PRE se ejecutan una sola vez al momento del upgrade, y los POST cuando se cierra la conexión. Durante el intercambio de frames no hay re-ejecución de filtros.

## Buenas y malas prácticas

**Hacer:**
- Configurar `response-timeout` muy alto (o metadata-level) para rutas SSE: un timeout de 5s cortará el stream SSE antes de que el backend emita el primer evento si hay latencia inicial.
- Aumentar `max-frame-payload-length` para aplicaciones que intercambian mensajes JSON grandes (ej: actualizaciones de estado completas): el default de 64KB es insuficiente para payloads de datos estructurados.
- Usar `lb://` con el nombre del servicio en rutas WebSocket: el balanceo entre instancias se aplica en el momento del upgrade (cada nueva conexión WS puede ir a una instancia diferente), no en cada frame.

**Evitar:**
- Aplicar `ModifyRequestBody` o `ModifyResponseBody` en rutas WebSocket: estos filtros no son compatibles con el protocolo WebSocket; si se configuran en la misma ruta, los frames no serán procesados correctamente.
- Esperar que `CircuitBreaker` con `TimeLimiter.timeout` funcione como timeout de conexión WebSocket: el TimeLimiter cierra el Mono de la conexión, que en WebSocket resulta en cierre abrupto del socket sin código de cierre correcto.

---

← [3.9 Rutas dinámicas y recarga en caliente](sc-gateway-rutas-dinamicas.md) | [Índice](README.md) | [3.11 Filtros y predicados personalizados (extension points)](sc-gateway-extension-points.md) →
