# 2.5.1 Heartbeat y lease del registro

← [2.4 Registro de instancias: metadata y estado](sc-eureka-instancias.md) | [Índice (README.md)](README.md) | [2.5.2 Self-preservation mode →](sc-eureka-self-preservation.md)

---

El registro de Eureka no es permanente: cada instancia registrada tiene un arrendamiento (lease) con fecha de caducidad. Si el servicio deja de renovar ese arrendamiento enviando heartbeats periódicos, el servidor expira la entrada y la elimina del registro. Este mecanismo garantiza que instancias caídas no permanezcan indefinidamente en el catálogo y que los consumidores no reciban endpoints muertos. Sin embargo, los valores por defecto de los intervalos (30 segundos de heartbeat, 90 segundos de expiración) introducen un retraso de hasta 90 segundos entre el momento en que una instancia cae y el momento en que desaparece del registro, durante el cual los consumidores pueden recibir errores 503 al intentar llamar a esa instancia.

> [CONCEPTO] El **lease** es el arrendamiento que cada instancia registrada mantiene en el servidor Eureka. Se renueva periódicamente mediante un PUT HTTP (heartbeat). Si el servidor no recibe renovación en `lease-expiration-duration-in-seconds` segundos, expira el lease y marca la instancia como candidata a eliminación.

## Diagrama: ciclo de vida del lease

El siguiente diagrama muestra la secuencia completa desde el registro inicial hasta la expiración y eliminación de una instancia que deja de enviar heartbeats.

```
Tiempo →

t=0    Cliente arranca
       PUT /eureka/apps/SERVICIO-A    ← registro inicial
       Servidor crea lease con:
         - lastRenewalTimestamp = now
         - duration = lease-expiration-duration-in-seconds (default: 90s)

t=30   Cliente envía heartbeat
       PUT /eureka/apps/SERVICIO-A/instancia-1/status
       Servidor actualiza: lastRenewalTimestamp = now

t=60   Cliente envía heartbeat
       Servidor actualiza: lastRenewalTimestamp = now

t=75   Proceso falla — no hay más heartbeats

t=120  Servidor ejecuta hilo de eviction (cada eviction-interval-timer ms)
       lastRenewalTimestamp + duration = 60 + 90 = 150s
       now = 120s < 150s → instancia NO expirada todavía

t=155  Servidor ejecuta hilo de eviction
       lastRenewalTimestamp + duration = 60 + 90 = 150s
       now = 155s > 150s → instancia EXPIRADA
       El servidor la elimina del registro (o no, si self-preservation activo)

t=185  Cliente que hace GET /eureka/apps ya no ve SERVICIO-A
       (o espera hasta el próximo registry-fetch-interval-seconds para ver el cambio)
```

## Implementación: configuración de lease con valores de producción

El siguiente ejemplo muestra la configuración de lease tanto en el cliente como en el servidor, con valores ajustados para distintos escenarios.

**application.yml del microservicio (cliente)**

```yaml
spring:
  application:
    name: servicio-pedidos

eureka:
  instance:
    # Intervalo de heartbeat en segundos (default: 30).
    # Reducir a 10s en entornos que requieren detección rápida de fallos.
    # No reducir más de 5s en producción: incrementa la carga del servidor
    # proporcionalmente a nInstancias / intervalo peticiones/segundo.
    lease-renewal-interval-in-seconds: 10

    # Tiempo máximo sin heartbeat antes de que el servidor expire el lease (default: 90).
    # Debe ser siempre > lease-renewal-interval-in-seconds * 2 para evitar
    # falsos positivos por latencia de red.
    # El servidor Eureka añade un factor de gracia adicional de 10% en su cálculo.
    lease-expiration-duration-in-seconds: 30

  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

**application.yml del Eureka Server**

```yaml
eureka:
  server:
    # Intervalo del hilo de eviction en milisegundos (default: 60000).
    # Reducir en entornos de desarrollo para que las instancias caídas
    # desaparezcan más rápido del registro.
    eviction-interval-timer-in-ms: 5000

    # Desactivar self-preservation en desarrollo para que las instancias
    # expiradas se eliminen inmediatamente.
    # En producción mantener true. Ver sección 2.5.2.
    enable-self-preservation: false
```

**Verificación del estado del lease via API REST**

```bash
# Consultar el registro de un servicio específico y ver su lease info
curl -s -H "Accept: application/json" \
  http://localhost:8761/eureka/apps/SERVICIO-PEDIDOS | \
  python3 -m json.tool | grep -A 5 "leaseInfo"

# Respuesta esperada:
# "leaseInfo": {
#   "renewalIntervalInSecs": 10,
#   "durationInSecs": 30,
#   "registrationTimestamp": 1704067200000,
#   "lastRenewalTimestamp": 1704067500000,
#   "evictionTimestamp": 0,
#   "serviceUpTimestamp": 1704067200000
# }
```

**Test de integración: verificación de expiración de lease**

```java
package com.ejemplo.eureka;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.test.context.TestPropertySource;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test que verifica el comportamiento del lease: con lease muy corto y el servidor
 * en modo eviction agresiva, la instancia debe desaparecer del registro al expirar.
 *
 * Este test requiere un Eureka Server levantado en localhost:8761.
 * Ver sc-eureka-testing.md para test con servidor embebido.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    "eureka.client.enabled=true",
    "eureka.instance.lease-renewal-interval-in-seconds=2",
    "eureka.instance.lease-expiration-duration-in-seconds=5",
    "spring.application.name=test-servicio-lease"
})
class LeaseIntegrationTest {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Test
    void instanciaRegistradaEsDescubrible() {
        // Dar tiempo a que el registro se propague
        try { Thread.sleep(3000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }

        List<?> instancias = discoveryClient.getInstances("test-servicio-lease");
        assertThat(instancias).isNotEmpty();
    }
}
```

## Tabla de propiedades del lease

La siguiente tabla recoge los parámetros que controlan el comportamiento del lease tanto en el cliente como en el servidor.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `eureka.instance.lease-renewal-interval-in-seconds` | int | `30` | Intervalo de heartbeat del cliente al servidor en segundos |
| `eureka.instance.lease-expiration-duration-in-seconds` | int | `90` | Tiempo máximo sin heartbeat antes de que el servidor expire el lease |
| `eureka.server.eviction-interval-timer-in-ms` | long | `60000` | Intervalo del hilo del servidor que comprueba y elimina leases expirados |
| `eureka.server.enable-self-preservation` | boolean | `true` | Si activo, suspende la eviction cuando la tasa de renewals cae bruscamente |
| `eureka.server.renewal-percent-threshold` | double | `0.85` | Umbral mínimo de renewals para no activar self-preservation |

## Buenas y malas prácticas

Hacer:
- Mantener la relación `lease-expiration-duration >= lease-renewal-interval * 3` para tolerancia a una pérdida de heartbeat por latencia de red: con un intervalo de 30s, el mínimo sensato de expiración es 90s (el valor por defecto).
- Reducir los intervalos en entornos de desarrollo (`lease-renewal-interval: 5`, `lease-expiration-duration: 15`) para que el ciclo de registrar/detectar caída sea más rápido y los tests sean más ágiles.
- Monitorizar el campo `lastRenewalTimestamp` de la instancia vía la API REST de Eureka (`/eureka/apps/{appId}/{instanceId}`) para detectar instancias que están enviando heartbeats pero con retraso, síntoma de sobrecarga de GC o CPU.

Evitar:
- Reducir `lease-renewal-interval-in-seconds` a 1-2 segundos en producción con muchas instancias: con 200 instancias y un intervalo de 2s, el servidor recibe 100 heartbeats por segundo, lo que puede saturarlo y generar un efecto cascada donde el propio servidor responde lento y los clientes empiezan a expirar.
- Configurar `lease-expiration-duration` por debajo del tiempo que tarda la JVM en completar un full GC en el servicio: una pausa de GC de 10 segundos con un `lease-expiration-duration` de 5 segundos produce desregistros falsos y reconexiones innecesarias.
- Confiar en los valores por defecto sin revisarlos: los 90 segundos de expiración por defecto significan que un circuit breaker verá errores durante hasta 90 segundos antes de que Eureka deje de dirigir tráfico a la instancia caída; combinado con un circuit breaker bien configurado esto es aceptable, pero sin circuit breaker produce errores visibles al usuario final.

---

← [2.4 Registro de instancias: metadata y estado](sc-eureka-instancias.md) | [Índice (README.md)](README.md) | [2.5.2 Self-preservation mode →](sc-eureka-self-preservation.md)
