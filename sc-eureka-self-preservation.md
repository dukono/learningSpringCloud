# 2.5.2 Self-preservation mode

← [2.5.1 Heartbeat y lease del registro](sc-eureka-lease.md) | [Índice (README.md)](README.md) | [2.6 Descubrimiento de servicios desde el cliente →](sc-eureka-descubrimiento.md)

---

El self-preservation mode es el mecanismo de defensa del Eureka Server ante fallos de red transitorios que afectan a múltiples instancias simultáneamente. Sin él, un fallo de red parcial que impida a un grupo de servicios enviar heartbeats durante unos minutos provocaría que el servidor elimine todas esas instancias del registro, dejando a los consumidores sin instancias para llamar aunque los servicios sigan ejecutándose correctamente. El self-preservation suspende la eviction de instancias cuando detecta que la tasa de renovaciones cae por debajo de un umbral, bajo la premisa de que es más probable que el servidor esté sufriendo un problema de red que que todas las instancias hayan caído a la vez. Este comportamiento es correcto en producción pero perjudicial en desarrollo, donde las instancias se detienen frecuentemente y el registro se llena de entradas obsoletas.

> [CONCEPTO] Cuando la tasa real de heartbeats recibidos en el último minuto cae por debajo del `renewal-percent-threshold` (85% por defecto) de los heartbeats esperados, Eureka activa self-preservation y suspende la eliminación de instancias con lease expirado hasta que la tasa se recupere.

## Diagrama: activación y desactivación del self-preservation

El siguiente diagrama ilustra la lógica de umbral que decide si Eureka evicta instancias o suspende la operación en modo self-preservation.

```
Cada minuto, el servidor calcula:
  heartbeats_esperados = nInstancias * (60 / lease-renewal-interval)
  heartbeats_recibidos = renewals en el último minuto

  ratio = heartbeats_recibidos / heartbeats_esperados

  ratio >= renewal-percent-threshold (0.85) ?
  │
  ├── SÍ ──► Modo NORMAL
  │            El hilo de eviction elimina instancias con lease expirado
  │
  └── NO ──► SELF-PRESERVATION ACTIVADO
               El hilo de eviction se suspende
               El dashboard muestra: "EMERGENCY! EUREKA MAY BE INCORRECTLY
               CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT."
               Las instancias con lease expirado permanecen en el registro

Cuando ratio vuelve a superar el umbral:
  └──► SELF-PRESERVATION DESACTIVADO
        El hilo de eviction se reanuda
        Las instancias con lease expirado se eliminan en el próximo ciclo
```

## Implementación: configuración de self-preservation por entorno

La configuración óptima difiere significativamente entre desarrollo y producción. El siguiente ejemplo muestra los perfiles de Spring que implementan esta diferencia.

**application.yml (configuración base)**

```yaml
spring:
  application:
    name: eureka-server

server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

**application-dev.yml (desarrollo — self-preservation desactivado)**

```yaml
eureka:
  server:
    # Desactivar en desarrollo: las instancias se detienen con frecuencia
    # y queremos que desaparezcan del registro inmediatamente.
    enable-self-preservation: false
    # Eviction más frecuente en desarrollo para feedback rápido
    eviction-interval-timer-in-ms: 5000
```

**application-prod.yml (producción — self-preservation activo con umbral ajustado)**

```yaml
eureka:
  server:
    # Activo en producción: protege contra fallos de red transitorios
    enable-self-preservation: true
    # Umbral: si cae por debajo del 75% de heartbeats esperados, activa self-preservation.
    # Default: 0.85. Reducir ligeramente si los deployments rolling frecuentes
    # disparan falsos self-preservations (al parar instancias en el rolling update).
    renewal-percent-threshold: 0.75
    # Eviction estándar en producción
    eviction-interval-timer-in-ms: 60000
```

> [ADVERTENCIA] En el dashboard de Eureka, el mensaje en rojo "EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT" indica que self-preservation está activo. En desarrollo, esto significa que hay instancias obsoletas en el registro. En producción, indica un posible problema de red. No ignorar este mensaje en producción.

## Tabla de propiedades del self-preservation

La siguiente tabla recoge los parámetros que controlan el umbral y el comportamiento del self-preservation.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `eureka.server.enable-self-preservation` | boolean | `true` | Activa o desactiva el modo de auto-preservación |
| `eureka.server.renewal-percent-threshold` | double | `0.85` | Umbral mínimo de renovaciones para no activar self-preservation |
| `eureka.server.eviction-interval-timer-in-ms` | long | `60000` | Intervalo del hilo de eviction; en self-preservation, este hilo se suspende |

## Buenas y malas prácticas

Hacer:
- Desactivar `enable-self-preservation` en entornos de desarrollo con `false`: en local, los servicios se detienen y arrancan con frecuencia; con self-preservation activo, el registro se llena de instancias obsoletas que no desaparecen y los tests de integración fallan al encontrar instancias caídas.
- Ajustar `renewal-percent-threshold` a `0.75` en entornos con deployments rolling frecuentes: durante un rolling update, el 25% de las instancias se reinician simultáneamente, lo que puede activar self-preservation con el umbral de 0.85 por defecto, generando alarmas falsas.
- Monitorizar el ratio de renovaciones en producción con métricas de Actuator y alertar cuando se acerque al umbral: el self-preservation debería ser la excepción, no el estado habitual del servidor.

Evitar:
- Desactivar self-preservation en producción como "solución" a instancias obsoletas en el registro: el problema real son las instancias que no envían heartbeats; desactivar self-preservation elimina síntomas pero deja el sistema vulnerable a fallos de red transitorios que eliminarían instancias sanas del registro.
- Reducir `renewal-percent-threshold` a valores muy bajos (0.3 o menos): un umbral tan bajo hace que self-preservation no se active nunca, incluso ante un fallo de red real que afecte al 60% de las instancias, anulando la protección que el mecanismo ofrece.

> [EXAMEN] Una pregunta clásica de entrevista es: "¿Por qué en desarrollo las instancias caídas permanecen en el registro de Eureka?". La respuesta correcta menciona self-preservation: si el servidor tiene pocas instancias registradas y se detienen más del 15% en poco tiempo (umbral por defecto 0.85), activa self-preservation y deja de evictar. La solución es `eureka.server.enable-self-preservation: false` en el perfil de desarrollo.

---

← [2.5.1 Heartbeat y lease del registro](sc-eureka-lease.md) | [Índice (README.md)](README.md) | [2.6 Descubrimiento de servicios desde el cliente →](sc-eureka-descubrimiento.md)
