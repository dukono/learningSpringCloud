# 2.11 Operación y troubleshooting de Eureka

← [2.10 Integración con Spring Cloud Config](sc-eureka-integracion-config.md) | [Índice (README.md)](README.md) | [2.12 Testing / Verificación de Spring Cloud Netflix Eureka →](sc-eureka-testing.md)

---

Eureka introduce retrasos de propagación, ventanas de staleness y comportamientos de self-preservation que pueden confundir a un equipo cuando algo no funciona como se espera. La mayoría de los problemas de producción relacionados con Eureka se manifiestan de tres formas: instancias que aparecen en el registro pero no responden, instancias que tardan en aparecer después del arranque, y instancias que no desaparecen después de caer. Diagnosticar correctamente cuál de estos tres problemas está ocurriendo requiere entender los tiempos de propagación, los niveles de log y el impacto del self-preservation en cada situación.

## Diagrama: mapa de síntomas y causas raíz en Eureka

El siguiente diagrama conecta los síntomas observables con sus causas raíz más frecuentes, que es el punto de partida para el diagnóstico en producción.

```
SÍNTOMA: llamadas fallan con "No instances available for servicio-X"
         │
         ├── El servicio X está arrancando → esperar 30-60s (delay de registro)
         ├── fetch-registry: false en el consumidor → activar fetch-registry
         ├── registry-fetch-interval demasiado alto → reducir el intervalo
         └── Servicio X no tiene la URL de Eureka correcta → verificar defaultZone

SÍNTOMA: instancias DOWN/caídas siguen en el registro
         │
         ├── Self-preservation activado → verificar dashboard (mensaje de WARNING)
         ├── eviction-interval-timer-in-ms muy alto → reducir a 5000 en dev
         └── lease-expiration-duration-in-seconds muy alto → reducir valores

SÍNTOMA: instancia registrada como UP pero las llamadas fallan con 503/timeout
         │
         ├── healthcheck.enabled: false → el health del actuator no se propaga
         ├── La app está UP pero la BD está caída → activar healthcheck.enabled
         └── prefer-ip-address: false con hostname no resolvible → usar prefer-ip-address: true

SÍNTOMA: el registro tarda en propagarse entre peers del cluster
         │
         ├── peer-eureka-nodes-update-interval-ms muy alto → reducir
         └── Replicación asíncrona: esperar 5-10s para convergencia entre peers
```

## Implementación: diagnóstico y resolución de los problemas más frecuentes

El siguiente conjunto de diagnósticos cubre los seis patrones de fallo más comunes en producción.

**Problema 1: delay de propagación del registro**

Cuando un servicio arranca, no es inmediatamente visible para los consumidores. El retraso se compone de múltiples intervalos acumulados:

```
Tiempo mínimo para que un servicio sea visible tras arrancar:
 lease-renewal-interval-in-seconds (30s)  ← primer heartbeat
 + registry-fetch-interval-seconds (30s)  ← caché del consumidor se refresca
 + response-cache-update-interval-ms (30s) ← caché de respuestas del servidor
 ─────────────────────────────────────────
 = hasta 90 segundos en el peor caso con valores por defecto
```

La mitigación en desarrollo es reducir todos estos intervalos:

```yaml
# application.yml del servidor Eureka (desarrollo)
eureka:
  server:
    response-cache-update-interval-ms: 5000
    eviction-interval-timer-in-ms: 5000

# application.yml del microservicio cliente (desarrollo)
eureka:
  instance:
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 15
  client:
    registry-fetch-interval-seconds: 5
```

**Problema 2: instancias obsoletas (stale instances) en el registro**

Se diagnostican con una llamada directa a la API REST y comparando el `lastRenewalTimestamp` con la hora actual:

```bash
# Obtener todas las instancias de un servicio y mostrar su último heartbeat
curl -s -H "Accept: application/json" \
  http://localhost:8761/eureka/apps/SERVICIO-PEDIDOS | \
  python3 -c "
import json, sys, datetime
data = json.load(sys.stdin)
apps = data['applications']['application']
for app in apps:
    for inst in app['instance']:
        ts = inst['leaseInfo']['lastRenewalTimestamp'] / 1000
        dt = datetime.datetime.fromtimestamp(ts)
        age = datetime.datetime.now() - dt
        print(f\"{inst['instanceId']} | último heartbeat: {dt} | hace {age.seconds}s | estado: {inst['status']}\")
"
```

Si el `lastRenewalTimestamp` tiene más de `lease-expiration-duration-in-seconds` segundos de antigüedad y la instancia sigue en el registro, self-preservation está activo. Verificar en el dashboard.

**Problema 3: instancia UP pero inaccesible**

El problema es que Eureka marca como UP solo en base al heartbeat de red, sin conocer el estado real de la aplicación. La solución es activar la integración con Actuator:

```yaml
# application.yml del microservicio
eureka:
  client:
    healthcheck:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      # Exponer detalles para diagnóstico; en producción puede limitarse
      show-details: always
```

Con esta configuración, si el health check del Actuator devuelve DOWN (por ejemplo, porque la base de datos no responde), Eureka marca la instancia como DOWN y Spring Cloud LoadBalancer no la selecciona para nuevas peticiones.

**Problema 4: impacto del self-preservation en instancias caídas**

El dashboard muestra el mensaje en rojo cuando self-preservation está activo. Para confirmar si está afectando al registro:

```bash
# Verificar el estado de self-preservation via Actuator del servidor Eureka
curl -s http://localhost:8761/actuator/health | python3 -m json.tool

# Consultar el número de renovaciones esperadas vs recibidas
# (visible en el log del servidor Eureka con nivel DEBUG)
```

Para ver en el log cuándo se activa y desactiva self-preservation:

```yaml
# application.yml del Eureka Server — logging de diagnóstico
logging:
  level:
    com.netflix.eureka: DEBUG
    com.netflix.discovery: DEBUG
```

**Problema 5: timeouts en el cliente para evitar llamadas a instancias caídas**

El balanceo de carga enviará tráfico a instancias caídas durante la ventana de hasta 90 segundos que tarda Eureka en retirarlas. La defensa correcta es configurar timeouts cortos en el cliente HTTP:

```java
package com.ejemplo.pedidos.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        // Timeout de conexión: tiempo máximo para establecer la conexión TCP
        factory.setConnectTimeout(2000);
        // Timeout de lectura: tiempo máximo esperando la respuesta tras conectar
        factory.setReadTimeout(5000);
        return new RestTemplate(factory);
    }
}
```

**Problema 6: diagnóstico con logs del Eureka Client**

Los logs del cliente Eureka son la fuente más directa para diagnosticar problemas de registro y heartbeat:

```yaml
# Activar logs de Eureka Client en el microservicio
logging:
  level:
    com.netflix.discovery: DEBUG
    com.netflix.eureka: DEBUG
```

Patrones de log importantes que hay que saber interpretar:

```
# Registro inicial exitoso:
DiscoveryClient_SERVICIO-PEDIDOS/servicio-pedidos:8081 - registration status: 204

# Heartbeat exitoso:
DiscoveryClient_SERVICIO-PEDIDOS/servicio-pedidos:8081 - Renewing lease, duration 30

# Heartbeat fallido (servidor no responde):
DiscoveryClient_SERVICIO-PEDIDOS/servicio-pedidos:8081 - There is problem renewing the lease for instance

# Refresco de caché del registro:
DiscoveryClient_SERVICIO-PEDIDOS/servicio-pedidos:8081 - Getting all instance registry info from the eureka server

# Self-preservation activado en el servidor (log del servidor):
ENTER: Self preservation is ON
```

## Tabla de diagnóstico rápido

La siguiente tabla es la referencia rápida para diagnosticar los problemas más frecuentes en producción.

| Síntoma | Causa más probable | Verificación | Solución |
|---|---|---|---|
| "No instances available" tras arranque | Delay de propagación (hasta 90s) | Esperar y consultar `/eureka/apps` | Reducir intervalos en dev; añadir retry en prod |
| Instancia caída sigue en registro | Self-preservation activo | Dashboard Eureka (mensaje WARNING) | `enable-self-preservation: false` en dev |
| Instancia UP pero 503 | healthcheck no integrado | Ver `/actuator/health` del servicio | `eureka.client.healthcheck.enabled: true` |
| Registro muy lento (>90s) | Valores por defecto de los intervalos | Sumar `lease-renewal` + `registry-fetch` + `response-cache` | Reducir los tres intervalos en dev |
| Reregistro frecuente sin causa aparente | Pausa de GC supera `lease-expiration-duration` | Logs GC del servicio | Aumentar `lease-expiration-duration` o tunear GC |
| Llamadas fallan a pesar de instancias UP | Hostname no resolvible | `dig` / `nslookup` del hostname en el registro | `prefer-ip-address: true` |

## Buenas y malas prácticas

Hacer:
- Configurar timeouts en el `RestTemplate` o `WebClient` siempre, independientemente de Eureka: los timeouts protegen contra instancias que responden lentamente, que Eureka no puede detectar (una instancia lenta sigue enviando heartbeats).
- Incluir Circuit Breaker (Resilience4j) junto con Eureka: si una instancia está en el registro pero responde con errores, el circuit breaker la aísla antes de que Eureka la retire, minimizando los errores visibles al usuario.
- Revisar el `lastRenewalTimestamp` de las instancias durante un incidente: un timestamp reciente confirma que la instancia está activa a nivel de red, y si aún así las llamadas fallan, el problema está en la capa de aplicación.

Evitar:
- Desactivar self-preservation como respuesta por defecto a instancias obsoletas en producción: el problema real son los heartbeats perdidos; investigar por qué antes de deshabilitar la protección.
- Ignorar el mensaje WARNING del dashboard de Eureka: aunque el sistema siga funcionando con self-preservation activo, indica que hay un problema de red o de carga en el servidor que debe investigarse.
- Confiar en que Eureka eliminará instancias caídas de forma inmediata sin retry en el cliente: el retraso de hasta 90 segundos es inherente al modelo; el retry y el circuit breaker son las herramientas correctas para manejar esa ventana.

---

← [2.10 Integración con Spring Cloud Config](sc-eureka-integracion-config.md) | [Índice (README.md)](README.md) | [2.12 Testing / Verificación de Spring Cloud Netflix Eureka →](sc-eureka-testing.md)
