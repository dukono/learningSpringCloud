# 6.7 Spring Cloud Stream — Consumer Groups y competing consumers

← [6.6 RabbitMQ binder](sc-stream-rabbit-binder.md) | [Índice](README.md) | [6.8 Particionado](sc-stream-particionado.md) →

---

## Introducción

Los consumer groups son el mecanismo que permite escalar horizontalmente microservicios basados en mensajería sin duplicar el procesamiento. Resuelven el problema de garantizar que cada mensaje sea procesado exactamente una vez cuando múltiples instancias de un servicio compiten por los mensajes del mismo topic o queue. Existen porque el modelo publish-subscribe por defecto entrega cada mensaje a todos los suscriptores, lo cual es indeseable cuando se escala un consumer. Se necesitan en cualquier despliegue con más de una instancia del mismo microservicio consumidor.

## Modelo de consumer groups — broadcast vs competing consumers

La diferencia clave entre un anonymous consumer y un consumer con grupo determina el comportamiento de entrega de mensajes. El diagrama siguiente ilustra los dos modelos:

```
SIN group (anonymous consumers) — cada instancia recibe todos los mensajes:
  Topic: orders-topic
    Mensaje M1 → Instancia A  ✓
    Mensaje M1 → Instancia B  ✓  (duplicado — indeseable para procesamiento)
    Mensaje M1 → Instancia C  ✓  (duplicado — indeseable para procesamiento)

CON group: order-service — competing consumers:
  Topic: orders-topic
    Mensaje M1 → Instancia A  ✓
    Mensaje M2 → Instancia B  ✓
    Mensaje M3 → Instancia C  ✓  (load balancing — deseable)
```

## Ejemplo central — configuración de consumer group

El siguiente ejemplo muestra tres instancias del mismo microservicio configuradas con el mismo consumer group. Solo cambia el `instance-index` en cada instancia para el soporte de particionado:

```java
package com.example.stream;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.util.function.Consumer;

@SpringBootApplication
public class OrderConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderConsumerApplication.class, args);
    }

    // Con group configurado: solo UNA de las tres instancias procesa cada mensaje
    @Bean
    public Consumer<String> processOrder() {
        return order -> {
            System.out.println("Processing order on instance: " + order);
            // lógica de negocio
        };
    }
}
```

```yaml
# application.yml — instancia 0 de 3
spring:
  cloud:
    function:
      definition: processOrder

    stream:
      bindings:
        processOrder-in-0:
          destination: orders-topic
          # CON group: competing consumers (una instancia por mensaje)
          group: order-service
          # SIN group: anonymous consumer (todas reciben todos — NO escalar así)
          # group: (ausente)

      kafka:
        binder:
          brokers: localhost:9092
```

```yaml
# application-instance1.yml — variables que cambian por instancia en Kubernetes/PaaS
# (instanceIndex se usa para particionado, no para consumer groups básicos)
spring:
  cloud:
    stream:
      instance-index: 0   # 0, 1 o 2 según la instancia
      instance-count: 3   # Total de instancias
```

## Tabla — comportamiento por configuración de group

| Configuración | Comportamiento | Cuándo usar |
|---------------|---------------|-------------|
| Sin `group` (anonymous) | Todas las instancias reciben todos los mensajes | Fan-out intencional: notificaciones, caché invalidation |
| Con `group` (competing consumers) | Solo una instancia del grupo procesa cada mensaje | Procesamiento escalado horizontalmente |
| Mismo `group`, distintas apps | Apps distintas que compiten por los mismos mensajes | Balanceo entre microservicios distintos |
| Distintos `group`, mismo topic | Cada grupo recibe todos los mensajes del topic | Múltiples consumidores independientes (log + procesador) |

> [CONCEPTO] En Kafka, el consumer group se mapea directamente al concepto nativo de Kafka consumer group. Los mensajes de cada partición son asignados a un solo member del grupo. Esto significa que el número de instancias del grupo no puede superar el número de particiones del topic para tener beneficio real del paralelismo.

> [CONCEPTO] En RabbitMQ, el consumer group se implementa mediante una queue compartida: todos los consumers del grupo se suscriben a la misma queue y RabbitMQ distribuye los mensajes usando round-robin. La queue se nombra `[destination].[group]` y es durable si `durable-subscription: true`.

> [EXAMEN] Si se despliegan 3 instancias de un microservicio con `group: order-service` y llegan 9 mensajes al topic `orders-topic`, cada instancia procesará aproximadamente 3 mensajes (round-robin / Kafka partition assignment). Si no se configura `group`, las 3 instancias procesan los 9 mensajes cada una, resultando en 27 procesados — triplicando el trabajo con resultados incorrectos.

> [ADVERTENCIA] Las suscripciones anónimas (sin `group`) son efímeras: en RabbitMQ, la queue desaparece cuando la aplicación se detiene. Los mensajes enviados mientras la aplicación estaba caída se pierden. Las suscripciones con `group` son durables y los mensajes quedan encolados hasta que la aplicación vuelve.

## Comparación — durable vs anonymous consumer

| Aspecto | Con group | Sin group (anonymous) |
|---------|-----------|-----------------------|
| Tipo de suscripción | Durable | Efímera |
| Competing consumers | Sí (solo uno procesa) | No (todos reciben) |
| Mensajes durante caída | Encolados (se procesan al volver) | Perdidos |
| Escalado horizontal | Correcto | Genera duplicados |
| Caso de uso | Microservicio escalado | Fan-out / notificaciones |

## Buenas y malas prácticas

**Buenas prácticas:**
- Siempre configurar `group` en microservicios que escalan horizontalmente.
- Usar el mismo `group` en todas las instancias del mismo microservicio.
- Nombrar el grupo con el nombre del servicio para facilitar el diagnóstico: `order-service`, `payment-processor`.

**Malas prácticas:**
- Omitir `group` en microservicios con múltiples instancias (genera duplicación de procesamiento).
- Usar grupos distintos para instancias del mismo servicio (cada instancia recibe todos los mensajes).
- Usar `group` cuando se necesita fan-out intencional a múltiples servicios independientes.

## Verificación y práctica

1. Si se despliegan 3 instancias de `order-service` todas con `spring.cloud.stream.bindings.processOrder-in-0.group=order-service`, ¿cuántas instancias procesarán cada mensaje publicado en `orders-topic`?

2. ¿Qué ocurre con los mensajes publicados en un topic mientras un anonymous consumer está caído? ¿Y con un consumer que tiene `group` configurado?

3. En Kafka, ¿cuál es el límite práctico del número de instancias del consumer group respecto al topic? ¿Qué pasa si hay más instancias que particiones?

4. ¿Cómo se nombra la queue en RabbitMQ para un binding con `destination=alerts` y `group=notification-service`?

5. ¿En qué escenario es correcto NO configurar `group` en un consumer de Spring Cloud Stream?

---

← [6.6 RabbitMQ binder](sc-stream-rabbit-binder.md) | [Índice](README.md) | [6.8 Particionado](sc-stream-particionado.md) →
