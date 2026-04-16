# 2.7 Dashboard y API REST del Eureka Server

← [2.6 Descubrimiento de servicios desde el cliente](sc-eureka-descubrimiento.md) | [Índice (README.md)](README.md) | [2.8 Alta disponibilidad: Eureka Server en cluster →](sc-eureka-cluster.md)

---

Además de su función de registro, el Eureka Server expone una interfaz web y una API REST HTTP que permiten operar el registro directamente: consultar qué instancias están registradas, ver su estado y metadata, cambiar su estado manualmente o darlas de baja sin esperar a que expire su lease. Esta API es la herramienta de diagnóstico y control operativo más directa que tiene un equipo de operaciones durante un incidente. Un senior debe conocerla para diagnosticar problemas sin necesidad de acceder a los logs de las instancias individuales.

> [ADVERTENCIA] En entornos Kubernetes, el mecanismo de discovery cambia radicalmente: Kubernetes Service y el DNS nativo del clúster sustituyen a Eureka. La API REST del Eureka Server descrita en esta sección no es relevante en despliegues nativos Kubernetes. Consulta el módulo Spring Cloud Kubernetes para ese escenario.

## Diagrama: endpoints del API REST de Eureka

El siguiente diagrama muestra la estructura de la API REST del Eureka Server y las operaciones disponibles sobre cada recurso.

```
http://localhost:8761

├── /                              ← Dashboard web (HTML)
│
└── /eureka/
    ├── apps/                      GET  → lista todas las aplicaciones
    │   │
    │   └── {APP_NAME}/            GET  → todas las instancias de la app
    │       │
    │       └── {INSTANCE_ID}/     GET  → detalle de la instancia
    │           │
    │           ├── status         PUT  → cambiar estado (OUT_OF_SERVICE/UP)
    │           └── (DELETE)       DEL  → dar de baja la instancia manualmente
    │
    └── vips/{vipAddress}          GET  → instancias por vipAddress
```

## Implementación: operaciones comunes con la API REST

El siguiente conjunto de comandos curl cubre las operaciones más frecuentes en producción: consultar, cambiar estado y dar de baja instancias.

**Consultar todas las aplicaciones registradas (JSON)**

```bash
curl -s -H "Accept: application/json" \
  http://localhost:8761/eureka/apps | python3 -m json.tool
```

**Respuesta típica (fragmento)**

```json
{
  "applications": {
    "versions__delta": "1",
    "apps__hashcode": "UP_3_",
    "application": [
      {
        "name": "SERVICIO-CATALOGO",
        "instance": [
          {
            "instanceId": "servicio-catalogo:10.0.2.4:8082",
            "hostName": "10.0.2.4",
            "app": "SERVICIO-CATALOGO",
            "ipAddr": "10.0.2.4",
            "status": "UP",
            "port": { "$": 8082, "@enabled": "true" },
            "vipAddress": "servicio-catalogo",
            "leaseInfo": {
              "renewalIntervalInSecs": 30,
              "durationInSecs": 90,
              "registrationTimestamp": 1704067200000,
              "lastRenewalTimestamp": 1704067500000
            },
            "metadata": {
              "zona": "eu-west-1a",
              "version": "2.1.0"
            }
          }
        ]
      }
    ]
  }
}
```

**Consultar instancias de una aplicación específica**

```bash
# APP_NAME en mayúsculas (Eureka normaliza los nombres a mayúsculas)
curl -s -H "Accept: application/json" \
  http://localhost:8761/eureka/apps/SERVICIO-CATALOGO | python3 -m json.tool
```

**Consultar el detalle de una instancia específica**

```bash
# INSTANCE_ID es el valor de eureka.instance.instance-id configurado en el cliente
curl -s -H "Accept: application/json" \
  http://localhost:8761/eureka/apps/SERVICIO-CATALOGO/servicio-catalogo:10.0.2.4:8082 \
  | python3 -m json.tool
```

**Cambiar el estado de una instancia a OUT_OF_SERVICE (retirar del tráfico)**

```bash
# Operación útil durante mantenimiento: retira la instancia del balanceo
# sin matarla. El @LoadBalanced no enviará tráfico a instancias OUT_OF_SERVICE.
curl -X PUT \
  "http://localhost:8761/eureka/apps/SERVICIO-CATALOGO/servicio-catalogo:10.0.2.4:8082/status?value=OUT_OF_SERVICE"
```

**Restaurar el estado a UP**

```bash
# Restaurar después del mantenimiento
curl -X PUT \
  "http://localhost:8761/eureka/apps/SERVICIO-CATALOGO/servicio-catalogo:10.0.2.4:8082/status?value=UP"
```

**Dar de baja una instancia manualmente (DELETE)**

```bash
# Elimina la instancia del registro inmediatamente, sin esperar expiración del lease.
# Útil cuando una instancia ha caído de forma abrupta y hay que limpiar el registro
# antes de que expire el lease (hasta 90 segundos por defecto).
curl -X DELETE \
  "http://localhost:8761/eureka/apps/SERVICIO-CATALOGO/servicio-catalogo:10.0.2.4:8082"
```

**Cliente Java para la API REST de Eureka via Spring Cloud**

```java
package com.ejemplo.admin.service;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.List;
import java.util.Map;

@Service
public class EurekaAdminService {

    private final DiscoveryClient discoveryClient;
    private final RestTemplate restTemplate;

    // Este RestTemplate NO tiene @LoadBalanced: hace llamadas directas al servidor Eureka
    public EurekaAdminService(DiscoveryClient discoveryClient,
                               RestTemplate restTemplate) {
        this.discoveryClient = discoveryClient;
        this.restTemplate = restTemplate;
    }

    /**
     * Retira una instancia del tráfico para mantenimiento.
     * Equivale al PUT /eureka/apps/{appId}/{instanceId}/status?value=OUT_OF_SERVICE
     */
    public void retirarInstancia(String eurekaServerUrl,
                                  String appId,
                                  String instanceId) {
        String url = eurekaServerUrl + "/eureka/apps/" + appId + "/" +
                     instanceId + "/status?value=OUT_OF_SERVICE";
        restTemplate.put(url, null);
    }

    /**
     * Lista las instancias de un servicio usando la abstracción DiscoveryClient.
     * Más portable que llamar directamente a la API REST de Eureka.
     */
    public List<ServiceInstance> listarInstancias(String servicioNombre) {
        return discoveryClient.getInstances(servicioNombre);
    }

    /**
     * Extrae la metadata de zona de cada instancia disponible.
     */
    public void mostrarMetadataZona(String servicioNombre) {
        discoveryClient.getInstances(servicioNombre).forEach(inst -> {
            Map<String, String> metadata = inst.getMetadata();
            String zona = metadata.getOrDefault("zona", "sin zona");
            System.out.printf("Instancia %s — zona: %s%n",
                inst.getInstanceId(), zona);
        });
    }
}
```

## Tabla de endpoints del API REST

La siguiente tabla resume los endpoints del API REST de Eureka con su semántica y uso operativo.

| Método | Endpoint | Descripción | Uso típico |
|---|---|---|---|
| GET | `/eureka/apps` | Lista todas las aplicaciones con sus instancias | Diagnóstico general del registro |
| GET | `/eureka/apps/{appId}` | Instancias de una aplicación específica | Verificar cuántas instancias activas hay |
| GET | `/eureka/apps/{appId}/{instanceId}` | Detalle completo de una instancia | Inspeccionar metadata, estado y lease info |
| PUT | `/eureka/apps/{appId}/{instanceId}/status?value=OUT_OF_SERVICE` | Retira la instancia del tráfico | Mantenimiento sin matar el proceso |
| PUT | `/eureka/apps/{appId}/{instanceId}/status?value=UP` | Restaura la instancia al tráfico | Fin del mantenimiento |
| DELETE | `/eureka/apps/{appId}/{instanceId}` | Baja manual inmediata de la instancia | Limpieza urgente de instancias huérfanas |
| GET | `/eureka/vips/{vipAddress}` | Instancias por vipAddress lógico | Descubrimiento por virtual host name |

**Formatos de respuesta**

| Cabecera `Accept` | Formato de respuesta |
|---|---|
| `application/json` | JSON con estructura `applications.application[].instance[]` |
| `application/xml` (default si omitida) | XML equivalente |

## Buenas y malas prácticas

Hacer:
- Usar el endpoint `GET /eureka/apps/{appId}` como primer paso en cualquier diagnóstico de "el servicio X no responde": verifica cuántas instancias están registradas, cuál es su estado y cuándo fue el último heartbeat (`lastRenewalTimestamp` en la respuesta).
- Incluir la operación `PUT /status?value=OUT_OF_SERVICE` en los scripts de despliegue antes de detener una instancia: permite que el balanceador deje de enviar tráfico antes del shutdown, reduciendo errores 503 durante el rolling update.
- Añadir autenticación HTTP Basic al servidor Eureka en producción (ver sección 2.9) antes de exponer la API REST: sin autenticación, cualquiera con acceso a la red interna puede dar de baja instancias o leer el mapa completo de topología del sistema.

Evitar:
- Usar el DELETE de la API REST para eliminar instancias activas en producción sin un incidente justificado: si la instancia sigue ejecutándose, se reregistrará en el siguiente heartbeat (30 segundos por defecto), haciendo la operación inefectiva y generando confusión en el registro.
- Confiar en la visibilidad del dashboard web de Eureka como única fuente de verdad del estado del sistema: el dashboard muestra el estado del registro, no el estado real de la aplicación. Una instancia UP en Eureka puede tener la base de datos caída si `healthcheck.enabled` no está activo.
- Llamar a los endpoints de la API REST de Eureka directamente desde la lógica de negocio de los microservicios: usa `DiscoveryClient` en su lugar, que es portable, cachea localmente y funciona con cualquier backend de discovery.

## Comparación: dashboard web vs API REST

El dashboard web y la API REST ofrecen vistas del mismo registro, pero con propósitos diferentes.

| Aspecto | Dashboard web | API REST |
|---|---|---|
| URL base | `http://server:8761/` | `http://server:8761/eureka/` |
| Formato | HTML renderizado | JSON o XML |
| Uso | Visualización rápida durante incidentes | Automatización, scripts, administración |
| Operaciones disponibles | Solo lectura (visualización) | Lectura y escritura (cambio de estado, baja) |
| Autenticación | Misma que el servidor (HTTP Basic si configurada) | Misma que el servidor |

---

← [2.6 Descubrimiento de servicios desde el cliente](sc-eureka-descubrimiento.md) | [Índice (README.md)](README.md) | [2.8 Alta disponibilidad: Eureka Server en cluster →](sc-eureka-cluster.md)
