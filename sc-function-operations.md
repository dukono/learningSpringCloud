# 12.9 Observabilidad y operación de funciones expuestas

← [12.8 Puntos de extensión avanzados: FunctionAroundWrapper y FunctionRegistration](sc-function-extension.md) | [Índice (README.md)](README.md) | [12.10 Testing y verificación de Spring Cloud Function](sc-function-testing.md) →

---

## Introducción

El problema que cubre este fichero es la opacidad operacional de las funciones serverless y de mensajería: una función Spring Cloud Function expuesta vía HTTP o Stream no tiene por defecto ningún endpoint de introspección que permita al operador saber qué funciones están registradas, si están disponibles o cuántas veces han sido invocadas. Sin observabilidad, cuando una función no responde es imposible distinguir entre un error en la lógica, un problema de configuración de binding o una función no registrada en el catálogo.

Spring Cloud Function provee tres mecanismos de observabilidad: el endpoint Actuator `/actuator/functions` que lista todas las funciones registradas, la integración con Micrometer para métricas de invocación (`function.invocations.count` y `function.invocations.duration`), y el logging estándar de Spring que puede habilitarse por paquete. Este fichero depende directamente de la configuración de `sc-function-config.md` (12.2): si `spring-cloud-function-web` no está en el classpath o el endpoint `functions` no está incluido en `management.endpoints.web.exposure.include`, los endpoints Actuator documentados aquí no estarán disponibles en runtime.

> **[PREREQUISITO]** Requiere `sc-function-config.md` (12.2). La infraestructura base de Spring Boot Actuator está fuera del alcance de este módulo; ver la [documentación de Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html).

> **[CONCEPTO]** `management.endpoints.web.exposure.include=functions` habilita el endpoint `/actuator/functions`. Este endpoint lista las funciones registradas en `FunctionCatalog` con sus tipos de entrada y salida, permitiendo verificar en runtime qué funciones están disponibles sin acceso al código fuente.

## Representación visual

Los tres niveles de observabilidad se aplican en distintas capas del pipeline de invocación de Spring Cloud Function.

```
  Invocación HTTP / Stream
          │
          ▼
  Spring Cloud Function Pipeline
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  NIVEL 1 — Logging (org.springframework.cloud.fn)   │
  │  Trazas de invocación, resolución de función,       │
  │  conversión de tipos                                │
  │                                                     │
  │  NIVEL 2 — Micrometer Metrics                       │
  │  function.invocations.count (contador)              │
  │  function.invocations.duration (timer histograma)   │
  │  Tags: functionName, outcome (success/error)        │
  │                                                     │
  │  NIVEL 3 — Actuator /actuator/functions             │
  │  Lista de funciones registradas + tipos             │
  │  Verificación de disponibilidad en runtime          │
  │                                                     │
  └─────────────────────────────────────────────────────┘
          │
          ▼
  Prometheus / Grafana / Datadog (vía Micrometer)
```

## Ejemplo central

El ejemplo configura los tres niveles de observabilidad en un único servicio y muestra el formato de las métricas expuestas.

### Dependencias Maven

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2025.1.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <!-- Micrometer Prometheus registry para scraping -->
  <dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      definition: processOrder

management:
  endpoints:
    web:
      exposure:
        # Expone /actuator/functions, /actuator/metrics y /actuator/prometheus
        include: functions, metrics, prometheus, health, info
  endpoint:
    functions:
      enabled: true
  metrics:
    tags:
      # Tag global añadido a todas las métricas (útil para diferenciar servicios en Grafana)
      application: order-function-service

logging:
  level:
    # Habilita DEBUG para ver resolución de funciones, conversión de tipos e invocaciones
    org.springframework.cloud.function: DEBUG
```

### Código Java completo

```java
package com.example.scfunction.operations;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.util.function.Function;

@SpringBootApplication
public class FunctionOperationsApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionOperationsApplication.class, args);
    }

    public record Order(Long id, String item, double price) {}
    public record Invoice(Long orderId, String description, double total) {}

    @Bean
    public Function<Order, Invoice> processOrder() {
        return order -> new Invoice(
            order.id(),
            "Factura: " + order.item(),
            order.price() * 1.21
        );
    }
}
```

Verificación del endpoint Actuator de funciones:
```bash
curl http://localhost:8080/actuator/functions
```
Respuesta JSON de ejemplo:
```json
{
  "functions": [
    {
      "name": "processOrder",
      "type": "function",
      "inputType": "com.example.scfunction.operations.FunctionOperationsApplication$Order",
      "outputType": "com.example.scfunction.operations.FunctionOperationsApplication$Invoice"
    }
  ]
}
```

Métricas disponibles en `/actuator/metrics/function.invocations.count`:
```bash
curl http://localhost:8080/actuator/metrics/function.invocations.count
```
Respuesta:
```json
{
  "name": "function.invocations.count",
  "measurements": [
    {"statistic": "COUNT", "value": 42}
  ],
  "availableTags": [
    {"tag": "functionName", "values": ["processOrder"]},
    {"tag": "outcome", "values": ["success", "error"]},
    {"tag": "application", "values": ["order-function-service"]}
  ]
}
```

Métricas Prometheus en `/actuator/prometheus`:
```
# HELP function_invocations_count_total Total de invocaciones de función
# TYPE function_invocations_count_total counter
function_invocations_count_total{application="order-function-service",functionName="processOrder",outcome="success"} 42.0

# HELP function_invocations_duration_seconds Duración de invocaciones de función
# TYPE function_invocations_duration_seconds histogram
function_invocations_duration_seconds_bucket{application="order-function-service",functionName="processOrder",outcome="success",le="0.001"} 38.0
function_invocations_duration_seconds_bucket{application="order-function-service",functionName="processOrder",outcome="success",le="0.01"} 42.0
```

## Tabla de elementos clave

Los elementos de observabilidad que un profesional senior debe conocer son los siguientes.

| Elemento | Tipo | Descripción |
|---|---|---|
| `/actuator/functions` | Endpoint Actuator | Lista funciones registradas en `FunctionCatalog` con tipos I/O |
| `management.endpoints.web.exposure.include=functions` | Propiedad | Habilita el endpoint `/actuator/functions`; no expuesto por defecto |
| `function.invocations.count` | Métrica Micrometer (counter) | Número total de invocaciones por función y resultado (success/error) |
| `function.invocations.duration` | Métrica Micrometer (timer) | Histograma de duración de invocaciones |
| `functionName` tag | Tag Micrometer | Nombre del bean funcional en las métricas |
| `outcome` tag | Tag Micrometer | `success` o `error` según resultado de la invocación |
| `org.springframework.cloud.function: DEBUG` | Logging | Habilita trazas de resolución, lookup y conversión de tipos |
| `management.metrics.tags.*` | Propiedad | Tags globales añadidos a todas las métricas (servicio, entorno, región) |

## Buenas y malas prácticas

**Hacer:**
- Incluir `functions` en `management.endpoints.web.exposure.include` en todos los entornos, incluyendo local, para poder verificar en runtime qué funciones están registradas y con qué tipos: es la primera herramienta de diagnóstico cuando una función no responde.
- Configurar `management.metrics.tags.application` con el nombre del servicio para poder filtrar métricas de función en Grafana o Datadog cuando múltiples servicios publican las mismas métricas Micrometer.
- Habilitar `org.springframework.cloud.function: DEBUG` temporalmente en producción durante incidentes: el log DEBUG de SCF muestra la resolución del nombre de función, el lookup en el catálogo y la conversión de tipos, lo que permite diagnosticar `FunctionNotFoundException` y errores de conversión sin cambiar código.
- Alertar sobre `function.invocations.count{outcome="error"}` en el sistema de monitorización: un incremento sostenido de errores en una función indica un problema de datos o de configuración antes de que los usuarios lo reporten.

**Evitar:**
- Evitar exponer el endpoint `/actuator/functions` en producción sin autenticación: el endpoint revela los nombres de los beans funcionales y sus tipos, información que puede ser útil para un atacante que intente fuzzing de endpoints.
- Evitar dejar `org.springframework.cloud.function: DEBUG` activo en producción de forma permanente: genera un volumen de logs muy alto en servicios con alta concurrencia que puede saturar el sistema de log y ocultar logs de negocio relevantes.
- Evitar ignorar la métrica `function.invocations.duration` en funciones síncronas expuestas vía HTTP: un percentil p99 creciente indica degradación en el tiempo de respuesta antes de que aparezcan errores, permitiendo actuación proactiva.

---

← [12.8 Puntos de extensión avanzados: FunctionAroundWrapper y FunctionRegistration](sc-function-extension.md) | [Índice (README.md)](README.md) | [12.10 Testing y verificación de Spring Cloud Function](sc-function-testing.md) →
