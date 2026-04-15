# 6.1.6 WebSocket y Server-Sent Events

← [6.1.5 SSL/TLS](./06-05-gateway-ssl.md) | [Índice](./README.md) | [6.1.7 Discovery Locator y Actuator →](./06-07-gateway-discovery-actuator.md)

---

HTTP clásico sigue un ciclo de vida corto: el cliente abre la conexión, envía la petición, el servidor responde y la conexión se cierra. WebSocket y Server-Sent Events (SSE) rompen ese ciclo. WebSocket convierte la conexión HTTP inicial en un canal bidireccional persistente; SSE mantiene la conexión abierta para que el servidor pueda empujar eventos al cliente cuando quiera. Para un Gateway, estas conexiones de larga duración suponen una consideración especial: los timeouts pensados para peticiones HTTP normales (segundos) romperán conexiones que deben durar minutos u horas. La buena noticia es que Spring Cloud Gateway, al estar construido sobre Netty y WebFlux, gestiona ambos protocolos de forma nativa sin cambios en la configuración de las rutas.

---

## Flujo de upgrade: HTTP → WebSocket a través del Gateway

El establecimiento de una conexión WebSocket comienza con una petición HTTP GET ordinaria que incluye las cabeceras `Upgrade: websocket` y `Connection: Upgrade`. El Gateway propaga esas cabeceras al backend, que responde con un 101 Switching Protocols. A partir de ese momento la conexión deja de ser HTTP y el Gateway actúa como proxy TCP transparente, reenviando frames binarios en ambas direcciones sin interpretarlos.

```
Cliente              Gateway              Backend WS
   │                    │                    │
   │─GET /ws/chat──────►│                    │
   │ Upgrade: websocket │                    │
   │ Connection: Upgrade│                    │
   │                    │─GET /ws/chat──────►│
   │                    │ Upgrade: websocket │
   │                    │◄─101 Switching─────│
   │◄─101 Switching─────│  Protocols         │
   │  Protocols         │                    │
   │                    │                    │
   │◄══ WS frame ═══════╪════════════════════│
   │═══ WS frame ═══════╪═══════════════════►│
   │         (canal bidireccional abierto)   │
   │                    │                    │
   │─Close frame───────►│─Close frame───────►│
   │◄─Close frame───────│◄─Close frame───────│
```

El Gateway intercepta la cabecera `Upgrade: websocket` del request inicial y la propaga al backend. A partir del 101, actúa como un proxy TCP transparente: reenvía los frames en ambas direcciones sin interpretarlos.

---

## Configuración de rutas

Las rutas para WebSocket y SSE se configuran igual que cualquier ruta HTTP. No existe una sintaxis especial ni filtros adicionales obligatorios. El Gateway detecta automáticamente el tipo de conexión por las cabeceras del request.

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        response-timeout: 10s          # para peticiones HTTP normales

      routes:
        # Ruta WebSocket — idéntica a una ruta HTTP normal
        - id: websocket-chat
          uri: lb://chat-service        # lb:// funciona con WebSocket
          predicates:
            - Path=/ws/chat/**
          metadata:
            response-timeout: 3600000  # 1 hora en ms — sobreescribe el timeout global para esta ruta

        # Ruta SSE — sin ninguna configuración especial
        - id: sse-notifications
          uri: lb://notification-service
          predicates:
            - Path=/api/events/**
          metadata:
            response-timeout: 0        # 0 = sin timeout para SSE

        # Ruta HTTP normal con timeout global heredado
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1
```

Para conectarse directamente a un backend sin load balancer, usar `ws://` o `wss://`:

```yaml
- id: websocket-directo
  uri: ws://backend-ws-service:8080
  predicates:
    - Path=/ws/**
```

> [ADVERTENCIA] Si se configura `spring.cloud.gateway.httpclient.response-timeout` globalmente con un valor bajo (por ejemplo `10s`), todas las conexiones WebSocket y SSE se romperán cuando se supere ese tiempo, aunque el protocolo no haya terminado. Sobreescribir el timeout por ruta con `metadata.response-timeout` es la solución estándar.

---

## Parámetros relevantes para conexiones persistentes

El timeout global `httpclient.response-timeout` está diseñado para peticiones HTTP cortas y romperá cualquier conexión WebSocket o SSE que lo supere. La propiedad `metadata.response-timeout` por ruta permite sobreescribir ese valor para conexiones de larga duración sin afectar al resto de rutas.

| Propiedad | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `spring.cloud.gateway.httpclient.response-timeout` | Duration | sin límite | Timeout global para recibir la respuesta completa; rompe WS/SSE si se configura bajo |
| `routes[].metadata.response-timeout` | long (ms) | hereda el global | Timeout específico por ruta; `0` desactiva el timeout para esa ruta |
| `routes[].metadata.connect-timeout` | int (ms) | hereda el global | Timeout de conexión TCP específico por ruta |

---

## Buenas y malas prácticas

Hacer:
- Configurar `metadata.response-timeout` en las rutas WebSocket y SSE explícitamente. No asumir que el valor global es adecuado: un timeout de 10 segundos correcto para APIs REST romperá cualquier conexión WebSocket pasado ese tiempo.
- Usar `lb://` para rutas WebSocket si los backends están en Eureka. El load balancer selecciona la instancia en el momento del upgrade; una vez establecida la conexión, todos los frames van a esa misma instancia por diseño del protocolo.
- Dimensionar la infraestructura considerando que cada conexión WebSocket activa ocupa recursos (memoria de buffer, descriptor de fichero) durante toda su vida. Un Gateway que gestiona 10.000 conexiones WS simultáneas tiene un perfil de recursos muy diferente a uno que gestiona 10.000 peticiones/segundo HTTP normales.

Evitar:
- Aplicar filtros `ModifyResponseBody` o `ModifyRequestBody` en rutas WebSocket o SSE. Estos filtros acumulan el body completo en memoria antes de procesarlo, lo que es incompatible con un stream continuo y causará un error o un bloqueo indefinido.
- Asumir que los filtros que funcionan en rutas HTTP funcionarán igual en rutas WebSocket. La mayoría de filtros predefinidos de Spring Cloud Gateway operan sobre cabeceras HTTP del handshake inicial, no sobre los frames del canal WebSocket. Intentar modificar cabeceras después del upgrade no tiene efecto.

---

## WebSocket vs SSE desde la perspectiva del Gateway

Ambos protocolos se configuran igual en las rutas, pero tienen características distintas que afectan al diseño del sistema.

| Aspecto | WebSocket | SSE |
|---|---|---|
| Dirección | Bidireccional (cliente y servidor envían) | Unidireccional (solo servidor envía) |
| Protocolo subyacente | Upgrade a WS (RFC 6455) | HTTP chunked (`text/event-stream`) |
| Configuración en Gateway | `uri: lb://` o `ws://`, `metadata.response-timeout` | Igual que ruta HTTP; `metadata.response-timeout: 0` |
| Filtros de body compatibles | No | No |
| Reconexión automática en navegadores | No (el cliente debe implementarla) | Sí (nativa en `EventSource`) |
| Caso de uso típico | Chat, colaboración en tiempo real, juegos | Notificaciones push, feeds de datos, progreso de tareas |
| CORS | Header `Origin` en el handshake; no usa preflight OPTIONS | Igual que HTTP; preflight OPTIONS si hay credenciales |

> [EXAMEN] SSE no requiere ninguna configuración especial en Spring Cloud Gateway más allá de desactivar el `response-timeout`. El Gateway lo trata como una respuesta HTTP chunked ordinaria. La confusión frecuente es asumir que SSE necesita el mismo tratamiento que WebSocket porque ambos son "tiempo real" — no es así.

---

← [6.1.5 SSL/TLS](./06-05-gateway-ssl.md) | [Índice](./README.md) | [6.1.7 Discovery Locator y Actuator →](./06-07-gateway-discovery-actuator.md)
