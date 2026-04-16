# 8.9 Propagación de identidad en contextos asíncronos y mensajería

← [8.8 Propagación de identidad entre microservicios síncronos](sc-security-propagation.md) | [Índice (README.md)](README.md) | [8.10 Spring Authorization Server — configuración base](sc-security-authorization-server.md) →

---

El `SecurityContextHolder` de Spring Security usa por defecto `ThreadLocal`, lo que significa que el contexto de seguridad solo es visible en el thread del request HTTP original. Cuando la lógica pasa a un thread diferente —mediante `@Async`, `CompletableFuture`, virtual threads con ejecutores, o mensajes de Kafka/RabbitMQ— el `SecurityContext` del usuario se pierde, y cualquier operación que requiera autorización fallará con `AccessDeniedException` o con un principal vacío. Existen tres patrones para manejar este problema: propagar el contexto explícitamente con `DelegatingSecurityContextExecutor`, usar `MODE_INHERITABLETHREADLOCAL` para threads hijo, y modelar la identidad en el mensaje para contextos de mensajería donde no hay usuario real.

> [PREREQUISITO] Requiere `spring-boot-starter-security`. Para mensajería: `spring-cloud-starter-stream-kafka` o `spring-cloud-starter-stream-rabbit`.

## Modos de propagación del SecurityContext

```
┌────────────────────────────────────────────────────────────────┐
│  Thread del Request (SecurityContext visible por ThreadLocal)   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Authentication: JwtAuthenticationToken (user-123)       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│         ┌─────────────────┴──────────────────┐                 │
│         ▼                                     ▼                 │
│  Thread Async Pool                    Mensaje Kafka              │
│  (ThreadLocal vacío                   (sin SecurityContext       │
│   → Authentication: null)              → sistema/servicio)      │
└────────────────────────────────────────────────────────────────┘
```

## Ejemplo central: propagación en @Async, ejecutores y mensajería

### Patrón 1: DelegatingSecurityContextExecutor para @Async

```java
package com.example.async;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.security.concurrent.DelegatingSecurityContextExecutor;
import org.springframework.security.task.DelegatingSecurityContextAsyncTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    // Ejecutor que propaga el SecurityContext del thread padre a threads del pool
    @Bean("securityAwareExecutor")
    public Executor securityAwareExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setThreadNamePrefix("async-security-");
        executor.initialize();
        // Envolver con DelegatingSecurityContextAsyncTaskExecutor
        return new DelegatingSecurityContextAsyncTaskExecutor(executor);
    }
}
```

```java
package com.example.async;

import org.springframework.scheduling.annotation.Async;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;

@Service
public class NotificationService {

    // El SecurityContext del thread del request se propaga al thread async
    @Async("securityAwareExecutor")
    public void sendNotificationAsync(String message) {
        // SecurityContextHolder.getContext().getAuthentication() devuelve
        // el mismo Authentication que en el thread del request original
        var auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth != null ? auth.getName() : "anonymous";
        // Lógica de notificación con acceso a la identidad del usuario
        System.out.println("Notificando a " + username + ": " + message);
    }
}
```

### Patrón 2: SecurityContext en CompletableFuture / virtual threads

```java
package com.example.async;

import org.springframework.security.concurrent.DelegatingSecurityContextRunnable;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SecurityContextPropagation {

    private final ExecutorService virtualThreadExecutor =
        Executors.newVirtualThreadPerTaskExecutor();

    public CompletableFuture<String> processAsync(String input) {
        // Capturar el SecurityContext del thread actual
        SecurityContext securityContext = SecurityContextHolder.getContext();

        return CompletableFuture.supplyAsync(() -> {
            // Propagar el contexto capturado al thread virtual
            SecurityContextHolder.setContext(securityContext);
            try {
                var auth = SecurityContextHolder.getContext().getAuthentication();
                return "Procesado para: " + (auth != null ? auth.getName() : "anon");
            } finally {
                // Limpiar al terminar para evitar leaks
                SecurityContextHolder.clearContext();
            }
        }, virtualThreadExecutor);
    }

    // Alternativa con DelegatingSecurityContextRunnable
    public void runWithContext(Runnable task) {
        Runnable delegating = new DelegatingSecurityContextRunnable(task);
        virtualThreadExecutor.submit(delegating);
    }
}
```

### Patrón 3: Identidad en mensajes de Kafka/RabbitMQ

Cuando la identidad debe cruzar una frontera de mensajería, se incluye como header o campo del mensaje — no hay SecurityContext que propagar.

```java
package com.example.messaging;

import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Service;

@Service
public class OrderEventPublisher {

    private final org.springframework.cloud.stream.function.StreamBridge streamBridge;

    public OrderEventPublisher(org.springframework.cloud.stream.function.StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }

    public void publishOrderCreated(String orderId) {
        // Extraer identidad del SecurityContext antes de enviar el mensaje
        var auth = SecurityContextHolder.getContext().getAuthentication();
        String userId = auth != null ? auth.getName() : "system";
        String jwtToken = null;
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            jwtToken = jwtAuth.getToken().getTokenValue();
        }

        // Incluir la identidad como header del mensaje
        Message<OrderEvent> message = MessageBuilder
            .withPayload(new OrderEvent(orderId, "CREATED"))
            .setHeader("x-user-id", userId)
            .setHeader("x-jwt-token", jwtToken)  // solo si el consumer necesita verificar
            .build();

        streamBridge.send("order-events-out-0", message);
    }
}
```

```java
package com.example.messaging;

import org.springframework.context.annotation.Bean;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;

import java.util.function.Consumer;

@Component
public class OrderEventConsumer {

    @Bean
    public Consumer<Message<OrderEvent>> orderEventsIn() {
        return message -> {
            // En el consumer de mensajes, no hay SecurityContext de usuario
            // La identidad viene en los headers del mensaje
            String userId = (String) message.getHeaders().get("x-user-id");
            OrderEvent event = message.getPayload();

            // Procesar con la identidad extraída del header
            processOrder(event, userId);
        };
    }

    private void processOrder(OrderEvent event, String userId) {
        System.out.println("Procesando " + event.orderId() + " para usuario: " + userId);
    }
}
```

> [CONCEPTO] En mensajería asíncrona, no existe concepto de "usuario actual" en el consumer — el mensaje fue enviado en un momento distinto, posiblemente por un usuario que ya no tiene sesión activa. La identidad del usuario que originó la acción se modeliza como dato del mensaje (header o campo del payload), no como contexto de seguridad del thread. Si el consumer necesita actuar en nombre del usuario, debe validar el JWT del header usando un `JwtDecoder` explícito.

## Tabla de elementos clave

| Componente | Descripción |
|---|---|
| `DelegatingSecurityContextAsyncTaskExecutor` | Envuelve un `TaskExecutor` de Spring para propagar el `SecurityContext` en `@Async` |
| `DelegatingSecurityContextRunnable` | Envuelve un `Runnable` capturando el `SecurityContext` del thread actual |
| `DelegatingSecurityContextExecutor` | Envuelve un `Executor` de Java para propagar el `SecurityContext` |
| `SecurityContextHolder.setContext(ctx)` | Establece manualmente el `SecurityContext` en el thread actual; siempre limpiar con `clearContext()` en `finally` |
| `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` | Modo que propaga el contexto a threads hijo del mismo proceso; no funciona con pool de threads |
| `x-user-id`, `x-jwt-token` | Convención de headers para propagar identidad en mensajes; el nombre es libre, pero debe estar documentado |

## Buenas y malas prácticas

**Hacer:**
- Usar `DelegatingSecurityContextAsyncTaskExecutor` para todos los ejecutores de `@Async` que contengan lógica que use el SecurityContext; es la solución más limpia y no requiere cambios en el código del servicio.
- Incluir `userId` como campo del payload del evento en mensajería, no solo como header; los headers pueden ser ignorados o perdidos en algunos brokers/transformaciones, mientras que el payload es inmutable.
- Limpiar el `SecurityContext` con `SecurityContextHolder.clearContext()` en el bloque `finally` cuando se establece manualmente en un thread del pool; evita que el contexto de un request contamine el siguiente request que reutilice ese thread.

**Evitar:**
- Usar `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` en aplicaciones con pool de threads (Tomcat, Undertow); este modo propaga el contexto a threads hijo directos (como en `new Thread(...)`), pero no funciona correctamente con threads reutilizados de un pool — el contexto del request anterior queda en el thread.
- Propagar el JWT completo como header de mensaje de Kafka para "reutilizarlo" en el consumer; el JWT puede haber expirado para cuando el consumer lo procesa. El consumer debe extraer solo el `sub` y `userId` del token como datos del evento.
- Asumir que el SecurityContext está disponible en `@Scheduled` o `ApplicationListener`; estos contextos se ejecutan sin request HTTP y sin usuario autenticado — cualquier operación que requiera seguridad debe obtener un token de servicio explícitamente.

---

← [8.8 Propagación de identidad entre microservicios síncronos](sc-security-propagation.md) | [Índice (README.md)](README.md) | [8.10 Spring Authorization Server — configuración base](sc-security-authorization-server.md) →
