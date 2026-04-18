# 1.5 Refresh de configuración en caliente — manual, Bus y webhooks

← [1.4 Resolución de configuración](sc-config-resolucion.md) | [Índice](README.md) | [1.6 Seguridad Config Server](sc-config-security.md) →

---

## Introducción

El refresh de configuración en caliente resuelve el problema más práctico de la configuración centralizada: cómo aplicar cambios sin reiniciar los servicios. Cuando alguien actualiza un fichero en el repositorio Git del Config Server, los microservicios en ejecución siguen usando los valores anteriores hasta que se reinicien, a menos que se active el mecanismo de refresh. Spring Cloud proporciona tres niveles de refresh: manual individual (endpoint Actuator), distribuido via Spring Cloud Bus (propaga a todos los servicios simultáneamente), y automático via webhooks (disparado por push al repositorio Git).

> [CONCEPTO] `@RefreshScope` es la anotación que marca un bean como "reconstruible en caliente". Cuando se invoca el endpoint `/actuator/refresh`, Spring destruye y recrea todos los beans anotados con `@RefreshScope`, inyectando los nuevos valores de configuración. Sin esta anotación, el bean conserva los valores con los que fue creado al arrancar.

> [PREREQUISITO] El endpoint Actuator `refresh` debe estar expuesto: `management.endpoints.web.exposure.include=refresh`. Sin esto, `/actuator/refresh` responde 404.

## Flujo completo de refresh manual

El refresh manual es el mecanismo base. Entenderlo es obligatorio antes de ver el refresh distribuido via Bus.

```
FLUJO DE REFRESH MANUAL
════════════════════════════════════════════════════════════════
  1. Developer hace commit en el repositorio Git de configuración
     └─ Cambia order-service.yml: app.max-orders=500 → 1000

  2. Developer llama manualmente al endpoint del cliente:
     POST http://order-service:8080/actuator/refresh

  3. El cliente contacta al Config Server:
     GET http://config-server:8888/order-service/prod/main

  4. Config Server lee el repositorio Git actualizado y devuelve
     las nuevas propiedades

  5. Spring recrea todos los beans @RefreshScope del cliente
     con los nuevos valores

  6. GET http://order-service:8080/actuator/env/app.max-orders
     → {"value": "1000"}  ← Confirmación del cambio
════════════════════════════════════════════════════════════════
IMPORTANTE: Solo afecta al microservicio al que se llama
            Las demás instancias siguen con los valores antiguos
```

> [EXAMEN] `@RefreshScope` es la pregunta más frecuente del módulo Config en el examen VMware. Sin esta anotación en el bean, el endpoint `/actuator/refresh` responde 200 pero el bean sigue con los valores anteriores. La anotación está en el paquete `org.springframework.cloud.context.config.annotation`.

## Ejemplo central — Refresh manual completo

El siguiente ejemplo muestra todos los componentes necesarios para el refresh manual: configuración del cliente, bean con `@RefreshScope`, y verificación.

**application.yml del cliente (con refresh habilitado)**:

```yaml
spring:
  application:
    name: order-service
  config:
    import: "configserver:http://localhost:8888"

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh,env
  endpoint:
    refresh:
      enabled: true
```

**OrderConfiguration.java (bean refreshable)**:

```java
package com.example.orderservice.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Component
@RefreshScope
public class OrderConfiguration {

    @Value("${app.max-orders:100}")
    private int maxOrders;

    @Value("${app.payment-url:http://localhost:9090/payment}")
    private String paymentUrl;

    @Value("${app.feature.new-checkout:false}")
    private boolean newCheckoutEnabled;

    public int getMaxOrders() { return maxOrders; }
    public String getPaymentUrl() { return paymentUrl; }
    public boolean isNewCheckoutEnabled() { return newCheckoutEnabled; }
}
```

**OrderController.java (expone el estado actual de configuración)**:

```java
package com.example.orderservice.controller;

import com.example.orderservice.config.OrderConfiguration;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api/config")
public class OrderController {

    private final OrderConfiguration config;

    public OrderController(OrderConfiguration config) {
        this.config = config;
    }

    @GetMapping
    public ResponseEntity<Map<String, Object>> getCurrentConfig() {
        return ResponseEntity.ok(Map.of(
            "maxOrders", config.getMaxOrders(),
            "paymentUrl", config.getPaymentUrl(),
            "newCheckoutEnabled", config.isNewCheckoutEnabled()
        ));
    }
}
```

**Secuencia de comandos para hacer y verificar el refresh**:

```bash
# 1. Ver configuración actual
curl http://localhost:8080/api/config
# Respuesta: {"maxOrders":100,"paymentUrl":"http://localhost:9090/payment","newCheckoutEnabled":false}

# 2. (Developer actualiza order-service.yml en el repositorio Git)
# app.max-orders: 250
# app.feature.new-checkout: true

# 3. Disparar el refresh en el cliente
curl -X POST http://localhost:8080/actuator/refresh
# Respuesta: ["app.max-orders","app.feature.new-checkout"]  ← propiedades cambiadas

# 4. Verificar que el bean tiene los nuevos valores
curl http://localhost:8080/api/config
# Respuesta: {"maxOrders":250,"paymentUrl":"http://localhost:9090/payment","newCheckoutEnabled":true}
```

## Refresh distribuido con Spring Cloud Bus

> [ADVERTENCIA] Esta sección es contenido avanzado (Grupo 2). Requiere haber completado el refresh manual antes de continuar. Spring Cloud Bus es un módulo independiente (ver dominio `sc-bus`).

Cuando hay múltiples instancias del mismo microservicio, el refresh manual requiere llamar al endpoint de cada instancia individualmente. Spring Cloud Bus resuelve este problema publicando un evento de refresh en un broker de mensajes (RabbitMQ o Kafka), y todas las instancias suscritas lo reciben y ejecutan el refresh simultáneamente.

```
FLUJO DE REFRESH DISTRIBUIDO CON SPRING CLOUD BUS
══════════════════════════════════════════════════
  POST /actuator/busrefresh (en el Config Server o en cualquier cliente)
         │
         ▼
  Spring Cloud Bus publica RefreshRemoteApplicationEvent
  en el broker (RabbitMQ / Kafka)
         │
         ├──▶ order-service:8080  (instancia 1)  → refresh local
         ├──▶ order-service:8081  (instancia 2)  → refresh local
         ├──▶ order-service:8082  (instancia 3)  → refresh local
         └──▶ payment-service:9090               → refresh local
══════════════════════════════════════════════════
```

**Dependencias adicionales para Bus con RabbitMQ**:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

**application.yml del cliente con Bus**:

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh,busrefresh,busenv
```

> [EXAMEN] La diferencia entre `/actuator/refresh` y `/actuator/busrefresh` es pregunta directa. `refresh` actualiza solo el servicio al que se llama. `busrefresh` propaga el evento a TODOS los servicios conectados al broker, usando Spring Cloud Bus.

## Webhooks y Config Monitor

> [ADVERTENCIA] Esta sección es la más avanzada del ciclo de refresh (Grupo 2). Requiere conocer tanto el refresh manual como Spring Cloud Bus.

El `spring-cloud-config-monitor` permite automatizar completamente el ciclo de refresh: cuando alguien hace push al repositorio Git, el proveedor (GitHub, GitLab, Bitbucket) envía un webhook HTTP al Config Server, que determina qué aplicaciones han cambiado y publica eventos de refresh selectivos via Spring Cloud Bus.

**Dependencia del Config Monitor**:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-monitor</artifactId>
</dependency>
```

**Configuración del webhook en GitHub**: apuntar a `http://config-server:8888/monitor` con tipo `application/json` y el secreto configurado.

**application.yml del Config Server con Monitor**:

```yaml
spring:
  cloud:
    bus:
      enabled: true
    config:
      monitor:
        github:
          enabled: true
```

> [EXAMEN] `/monitor` NO es un endpoint estándar del Config Server base. Requiere la dependencia `spring-cloud-config-monitor` Y un broker (Spring Cloud Bus) para propagar los eventos de refresh a los clientes.

## Tabla de mecanismos de refresh

Esta tabla resume los tres mecanismos de refresh para seleccionar el apropiado según el escenario.

| Mecanismo | Endpoint | Alcance | Requiere | Automático |
|-----------|----------|---------|----------|------------|
| Refresh manual | `POST /actuator/refresh` | Una instancia | Actuator expuesto | No |
| Bus refresh | `POST /actuator/busrefresh` | Todas las instancias | Spring Cloud Bus + broker | No |
| Config Monitor | webhook → `/monitor` | Servicios afectados | Bus + spring-cloud-config-monitor | Sí |

## Buenas y malas prácticas

Hacer:
- Anotar con `@RefreshScope` solo los beans que realmente consumen propiedades dinámicas; anotarlo en todos los beans es un antipatrón que degrada el rendimiento.
- Usar `@ConfigurationProperties` + `@RefreshScope` para grupos de propiedades relacionadas, en lugar de múltiples `@Value` dispersas.
- En producción con múltiples instancias, usar Spring Cloud Bus (busrefresh) en lugar de llamar manualmente a cada instancia.
- Verificar el resultado del refresh con `/actuator/env/{property}` inmediatamente después de llamar a `/actuator/refresh`.

Evitar:
- Olvidar exponer el endpoint `refresh` en `management.endpoints.web.exposure.include`; responde 404 y no da error en el arranque.
- Anotar `@RefreshScope` en beans con estado (`@Singleton` con estado mutable); el bean se destruye y recrea, perdiendo el estado.
- Usar `/monitor` sin Spring Cloud Bus; el endpoint recibe el webhook pero no puede propagar el evento a los clientes.
- Confundir `/actuator/refresh` (cliente) con `/actuator/busrefresh` (propagación Bus); llamar a `busrefresh` en un servicio sin Bus configurado falla con 404.

## Verificación y práctica

```bash
# Verificar qué propiedades cambiaron tras un refresh
curl -X POST http://localhost:8080/actuator/refresh
# Respuesta: lista de propiedades que cambiaron

# Verificar el valor actual de una propiedad
curl http://localhost:8080/actuator/env/app.max-orders

# Disparar busrefresh (requiere Spring Cloud Bus)
curl -X POST http://localhost:8080/actuator/busrefresh
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Qué hace `@RefreshScope`? ¿Qué ocurre si un bean usa `@Value` pero no tiene `@RefreshScope` y se llama a `/actuator/refresh`?
2. ¿Qué diferencia hay entre `POST /actuator/refresh` y `POST /actuator/busrefresh`?
3. ¿Qué dependencia adicional requiere el endpoint `/monitor` del Config Server?
4. El endpoint `/actuator/refresh` devuelve `[]` (lista vacía). ¿Qué significa?
5. ¿Por qué no es recomendable anotar todos los beans con `@RefreshScope`?

---

← [1.4 Resolución de configuración](sc-config-resolucion.md) | [Índice](README.md) | [1.6 Seguridad Config Server](sc-config-security.md) →
