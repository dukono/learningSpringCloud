# 4.7 Logging — Logger.Level, diagnóstico de tráfico HTTP y uso seguro en producción

<- [4.6 Interceptores y observabilidad](sc-feign-interceptores.md) | [Índice](README.md) | [4.8 Manejo de errores HTTP](sc-feign-errores.md) ->

---

## Introducción

El logging de Feign registra el tráfico HTTP de cada cliente: URL, método, cabeceras y cuerpo de la petición y la respuesta. Es la herramienta de diagnóstico más directa cuando un cliente Feign se comporta de forma inesperada: un timeout silencioso, una cabecera que no llega al servidor o una deserialización fallida se hacen visibles en los logs de Feign antes de que aparezcan como errores de negocio.

El control del nivel de log es esencial porque el nivel máximo (`FULL`) loggea el cuerpo completo de todas las peticiones y respuestas. En producción, esto puede exponer tokens de autenticación, datos personales y secretos de negocio en los sistemas de log. La configuración debe ajustarse por entorno: `FULL` en desarrollo y staging, `BASIC` o `NONE` en producción.

---

## Diagrama: niveles de Logger.Level y lo que registran

La siguiente tabla muestra exactamente qué información registra cada nivel.

```
 Logger.Level.NONE    → sin output (default)

 Logger.Level.BASIC   → [método] [URL] → [status] [ms]
   ---> GET http://catalog-service/api/products/123
   <--- 200 (45ms)

 Logger.Level.HEADERS → BASIC + cabeceras de petición y respuesta
   ---> GET http://catalog-service/api/products/123
   Accept: application/json
   X-Correlation-Id: abc-123
   <--- 200 (45ms)
   Content-Type: application/json
   X-Request-Id: srv-456

 Logger.Level.FULL    → HEADERS + cuerpo de petición y respuesta
   ---> GET http://catalog-service/api/products/123
   Accept: application/json
   [sin cuerpo en GET]
   <--- 200 (45ms)
   Content-Type: application/json
   {"productId":"123","name":"Widget","price":9.99}
```

---

## Ejemplo central

El siguiente ejemplo muestra las dos formas de configurar el nivel de log: mediante propiedades YAML (recomendado para poder cambiarla sin recompilar) y mediante clase `@Configuration` Java (para control programático).

**Configuración por YAML (recomendada en producción):**

```yaml
# application.yml
feign:
  client:
    config:
      default:
        logger-level: BASIC        # Nivel global para todos los clientes

      payment-service:
        logger-level: HEADERS      # Más detalle en el cliente crítico de pagos

# IMPORTANTE: Feign escribe sus logs a nivel DEBUG del logger de la interfaz.
# Sin esta configuración del logger SLF4J, los logs de Feign no aparecen
# aunque logger-level sea FULL.
logging:
  level:
    com.example.orderservice.client: DEBUG   # nivel DEBUG para el paquete de los clientes Feign
    # O por cliente individual:
    # com.example.orderservice.client.CatalogReadClient: DEBUG
```

**Configuración por clase @Configuration Java (para control programático):**

```java
package com.example.orderservice.config;

import feign.Logger;
import org.springframework.context.annotation.Bean;

/**
 * Configuración local: aplica solo al cliente que declare
 * configuration = PaymentFeignConfig.class
 */
public class PaymentFeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        // Leer de propiedades de entorno para poder cambiar sin recompilar
        // En producción, devolver Logger.Level.BASIC o NONE
        return Logger.Level.HEADERS;
    }

    /**
     * Logger que redirige el output de Feign al sistema de logging SLF4J.
     * Spring Cloud registra automáticamente Slf4jLogger si SLF4J está en el classpath.
     * Solo es necesario sobreescribirlo para usar un logger personalizado.
     */
    @Bean
    public Logger feignLogger() {
        return new feign.slf4j.Slf4jLogger(PaymentFeignConfig.class);
    }
}
```

> [ADVERTENCIA] Feign usa el **nombre completo de la interfaz** como nombre del logger SLF4J, no el nombre de la clase de configuración. Para que aparezcan los logs, el nivel DEBUG debe estar activado en el paquete donde vive la interfaz `@FeignClient`, no en el paquete de configuración.

---

## Tabla de elementos clave

| Elemento | Tipo | Default | Descripción |
|---|---|---|---|
| `Logger.Level.NONE` | enum | valor por defecto | Sin logging; recomendado en producción salvo diagnóstico activo |
| `Logger.Level.BASIC` | enum | — | Método HTTP, URL, código de respuesta y tiempo de ejecución |
| `Logger.Level.HEADERS` | enum | — | BASIC + cabeceras de petición y respuesta |
| `Logger.Level.FULL` | enum | — | HEADERS + cuerpo completo de petición y respuesta |
| `feign.client.config.[c].logger-level` | YAML | heredado de `default` | Nivel de log por cliente vía propiedades |
| `Logger.Level` bean en `@Configuration` | Java | — | Nivel de log por cliente vía clase de configuración |
| `logging.level.[paquete]` | YAML | INFO | Nivel SLF4J del paquete; **debe ser DEBUG** para que Feign emita logs |
| `feign.slf4j.Slf4jLogger` | clase | activo por defecto | Logger que redirige output de Feign a SLF4J |

---

## Buenas y malas prácticas

**Hacer:**
- Mantener `logger-level: BASIC` en producción como nivel mínimo visible. El nivel `BASIC` solo registra método, URL, status y tiempo, suficiente para detectar errores HTTP sin exponer datos sensibles. El coste en rendimiento es despreciable.
- Activar `FULL` solo en entornos no productivos o bajo demanda mediante un Config Server o propiedad de entorno. Con Spring Cloud Config, se puede cambiar el nivel en caliente sin reiniciar la aplicación si el cliente tiene `@RefreshScope`.
- Verificar siempre que el nivel SLF4J del paquete de los clientes Feign está en `DEBUG`. Sin ese ajuste, la configuración de `logger-level` no produce ningún output, lo que lleva a la falsa conclusión de que el logging de Feign "no funciona".

**Evitar:**
- Usar `Logger.Level.FULL` en servicios que manejan datos personales o tokens OAuth2. El cuerpo de la respuesta puede contener PII o credenciales que quedarán permanentemente en los sistemas de log y pueden violar GDPR/PCI-DSS.
- Confiar en el logging de Feign como única estrategia de diagnóstico en producción. Los logs de nivel `FULL` tienen un coste de serialización no trivial en servicios de alto volumen: cada respuesta se convierte a `String` para el log aunque no haya ningún appender configurado a nivel `DEBUG`. En esos casos, preferir el tracing distribuido (Micrometer) que opera con muestreo.

---

## Comparación: niveles de Logger.Level por entorno

La siguiente tabla resume la configuración recomendada según el entorno y el tipo de servicio.

| Entorno | Servicio normal | Servicio con datos sensibles | Diagnóstico activo |
|---|---|---|---|
| Desarrollo local | `FULL` | `HEADERS` | `FULL` |
| Staging / Pre-producción | `HEADERS` | `BASIC` | `FULL` temporal |
| Producción (estado normal) | `BASIC` | `NONE` | — |
| Producción (diagnóstico puntual) | `HEADERS` temporal | `BASIC` temporal | `HEADERS` temporal |

> [EXAMEN] En entrevista: "Tengo `logger-level: FULL` en `application.yml` pero no veo ningún log de Feign. ¿Qué puede estar pasando?" Respuesta modelo: el nivel SLF4J del paquete de los clientes Feign no está en `DEBUG`. Feign escribe sus logs al nivel `DEBUG` del logger de la interfaz; si el logging framework está configurado en `INFO` o superior para ese paquete, los mensajes se descartan antes de llegar al appender.

---

<- [4.6 Interceptores y observabilidad](sc-feign-interceptores.md) | [Índice](README.md) | [4.8 Manejo de errores HTTP](sc-feign-errores.md) ->
