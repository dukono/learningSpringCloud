# Parte 9 — Observabilidad (Trazabilidad distribuida)

← [Parte 8 — Comunicación](./08-comunicacion.md) | [Volver al índice](./README.md) | Siguiente: [Parte 10 — Seguridad](./10-seguridad.md) →

---

## 9.1 El problema: rastrear peticiones entre microservicios

En un monolito, un error tiene un stack trace completo en un solo log. En microservicios, una sola petición del usuario pasa por múltiples servicios:

```
Usuario → Gateway → Auth → Pedidos → Productos → Inventario
```

Si hay un error, los logs están en **5 máquinas distintas**, en **5 ficheros distintos**, con **timestamps que no coinciden** exactamente. Sin una forma de correlacionar estos logs, depurar es prácticamente imposible.

```
Log de Pedidos:   ERROR procesando pedido 42
Log de Productos: WARN timeout al consultar inventario
Log de Gateway:   ERROR 500 /api/pedidos/42

¿Estas tres líneas pertenecen a la misma petición? Imposible saberlo sin trazas.
```

**Distributed Tracing** soluciona esto inyectando un identificador único en cada petición que viaja a través de todos los servicios.

---

## 9.2 Conceptos: Trace ID, Span ID, correlación

### Trace ID

Un **Trace ID** es un identificador único que se asigna cuando la petición entra al sistema y se propaga a través de **todos los servicios** que participan en esa petición.

```
Petición del usuario recibe traceId: abc-123-xyz

Gateway    → logs con traceId: abc-123-xyz
Pedidos    → logs con traceId: abc-123-xyz
Productos  → logs con traceId: abc-123-xyz
Inventario → logs con traceId: abc-123-xyz
```

Con el `traceId` se pueden encontrar todos los logs de una petición en cualquier servicio.

### Span ID

Un **Span ID** identifica una **unidad de trabajo** dentro de una traza. Cada llamada a un servicio es un span:

```
Trace: abc-123-xyz
├── Span 1: Gateway → Pedidos           (spanId: span-001)
│   ├── Span 2: Pedidos → Productos     (spanId: span-002, parentSpanId: span-001)
│   │   └── Span 3: Productos → Inventario (spanId: span-003, parentSpanId: span-002)
│   └── Span 4: Pedidos → DB           (spanId: span-004, parentSpanId: span-001)
```

### Formato del log con trazas

```
2024-01-15 10:23:45 [pedidos-service,abc-123-xyz,span-001] INFO - Procesando pedido 42
                     ─────────────  ────────────  ────────
                     Nombre app     Trace ID      Span ID
```

### Cabeceras HTTP de propagación

Las trazas se propagan entre servicios mediante cabeceras HTTP estándar (B3 Propagation de Zipkin o W3C TraceContext):

```
X-B3-TraceId: abc123xyz...
X-B3-SpanId:  span001...
X-B3-ParentSpanId: parent001...
X-B3-Sampled: 1
```

---

## 9.3 Spring Cloud Sleuth (inyección automática de trazas)

> **Nota:** Spring Cloud Sleuth fue el componente original de trazas en Spring Cloud hasta Spring Boot 2.7. En **Spring Boot 3.x está reemplazado por Micrometer Tracing**. Esta sección es relevante para proyectos con Spring Boot 2.x.

### Qué hacía Sleuth automáticamente

- Generaba `traceId` y `spanId` en cada petición entrante
- Los añadía al contexto de log (MDC) para que aparecieran en cada línea de log
- Propagaba los IDs en peticiones HTTP salientes (RestTemplate, Feign, WebClient)
- Enviaba los spans a Zipkin para visualización

### Dependencia (Spring Boot 2.x)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

---

## 9.4 Micrometer Tracing (sucesor de Sleuth en Spring Boot 3)

**Micrometer Tracing** es el reemplazo de Spring Cloud Sleuth desde Spring Boot 3.x. Está integrado directamente en Spring Boot 3 y no necesita Spring Cloud para funcionar.

### Dependencias Maven (Spring Boot 3.x)

```xml
<!-- Micrometer Tracing con bridge a Brave (compatible con Zipkin) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Exportar spans a Zipkin -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<!-- O usar OpenTelemetry bridge (para Jaeger, Grafana Tempo, etc.) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Configuración en application.yml

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # 1.0 = muestrea el 100% de peticiones
                         # 0.1 = muestrea el 10% (recomendado en producción)

  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

### Configuración del formato de log con trazas

```yaml
# logback-spring.xml o application.yml
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

Con esta configuración, cada línea de log tendrá el formato:
```
INFO [pedidos-service,abc123,span001] - Procesando pedido 42
```

### Crear spans personalizados

```java
@Service
public class InventarioService {

    @Autowired
    private Tracer tracer;

    public void verificarStock(Long productoId) {
        // Crear un span personalizado para medir esta operación específica
        Span span = tracer.nextSpan().name("verificar-stock").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("producto.id", String.valueOf(productoId));

            // lógica de negocio
            int stock = inventarioRepository.getStock(productoId);
            span.tag("stock.cantidad", String.valueOf(stock));

        } catch (Exception ex) {
            span.error(ex);
            throw ex;
        } finally {
            span.end();   // ← siempre cerrar el span
        }
    }
}
```

### Propagación automática en Feign y RestTemplate

Micrometer Tracing propaga los IDs automáticamente en:
- Llamadas HTTP con `@FeignClient`
- `RestTemplate` con interceptores registrados
- `WebClient` con filtros registrados
- Mensajería con Spring Cloud Stream (en los headers del mensaje)

No se necesita código adicional para la propagación.

---

## 9.5 Zipkin: servidor de trazas distribuidas

**Zipkin** es un sistema de trazas distribuidas que recibe los spans de todos los microservicios y permite visualizarlos como un árbol de llamadas.

### Arrancar Zipkin con Docker

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
# UI disponible en http://localhost:9411
```

### O con Docker Compose

```yaml
# docker-compose.yml
services:
  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"
```

### Qué muestra la UI de Zipkin

1. **Lista de trazas:** todas las peticiones recientes, ordenadas por tiempo o duración
2. **Detalle de traza:** árbol visual de spans con duración de cada uno
3. **Dependencias:** mapa automático de qué servicios llaman a cuáles
4. **Búsqueda:** filtrar por traceId, servicio, nombre de span, tiempo, etc.

### Ejemplo de traza en Zipkin

```
Trace abc-123-xyz  (total: 150ms)
├── [gateway-service]       GET /api/pedidos/42          145ms
│   ├── [pedidos-service]   GET /pedidos/42               140ms
│   │   ├── [productos-service] GET /productos/101        50ms
│   │   │   └── [inventario-service] GET /stock/101       45ms ← el más lento
│   │   └── [DB pedidos]    SELECT * FROM pedidos          5ms
│   └── [auth-service]      POST /validate-token          3ms
```

---

## 9.6 Integración con Grafana y Prometheus

Para observabilidad completa, se combina:
- **Micrometer Tracing + Zipkin/Tempo:** trazas distribuidas
- **Micrometer Metrics + Prometheus:** métricas de rendimiento
- **Loki:** agregación de logs
- **Grafana:** visualización unificada de los tres pilares

### Dependencias para métricas con Prometheus

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### Configuración

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      # Tags globales que aparecerán en todas las métricas
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}
```

### Métricas automáticas disponibles

| Métrica | Descripción |
|---|---|
| `http_server_requests_seconds` | Latencia y tasa de peticiones HTTP |
| `jvm_memory_used_bytes` | Uso de memoria JVM |
| `process_cpu_usage` | Uso de CPU |
| `hikaricp_connections_active` | Conexiones activas a BD |
| `spring_cloud_gateway_requests_seconds` | Peticiones procesadas por el Gateway |
| `resilience4j_circuitbreaker_state` | Estado de los circuit breakers |

### Stack de observabilidad completo con Docker Compose

```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"

  loki:
    image: grafana/loki
    ports:
      - "3100:3100"
```

```yaml
# prometheus.yml — scraping de todos los microservicios
scrape_configs:
  - job_name: 'spring-cloud-apps'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'gateway-service:8080'
          - 'pedidos-service:8083'
          - 'productos-service:8082'
```

---

← [Parte 8 — Comunicación](./08-comunicacion.md) | [Volver al índice](./README.md) | Siguiente: [Parte 10 — Seguridad](./10-seguridad.md) →
