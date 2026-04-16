# 4.3 Configuración por clase Java — @Configuration local, global y FeignClientsConfiguration

<- [4.2 Declaración de @FeignClient](sc-feign-cliente-declarativo.md) | [Índice](README.md) | [4.4 Configuración por propiedades YAML](sc-feign-configuracion-propiedades.md) ->

---

## Introducción

Feign crea un child `ApplicationContext` independiente por cada cliente declarado. Ese contexto contiene los beans de infraestructura del cliente: `Encoder`, `Decoder`, `Logger`, `Retryer`, `Contract`, `RequestInterceptor` y otros. La clase `FeignClientsConfiguration` define los beans por defecto; para sobreescribir cualquiera de ellos basta con declarar un bean del mismo tipo en una clase `@Configuration` local.

Este mecanismo permite que dos clientes Feign en la misma aplicación tengan codecs completamente distintos — por ejemplo, uno que serializa con Jackson y otro que usa Protobuf — sin interferirse entre sí. La configuración local afecta solo al cliente que la declara; la configuración global (via `defaultConfiguration`) afecta a todos los clientes del módulo.

> [ADVERTENCIA] La clase `@Configuration` usada como configuración local de un cliente Feign **no debe** ser detectada por el `@ComponentScan` de la aplicación. Si Spring la registra en el contexto padre, los beans que define se aplican a todos los clientes, anulando el aislamiento por cliente y pudiendo causar comportamientos inesperados difíciles de diagnosticar.

---

## Diagrama: jerarquía de contextos en Feign

El siguiente diagrama muestra cómo se relacionan el contexto de aplicación principal, el contexto padre de Feign y los child contexts por cliente.

```
 ApplicationContext (Spring Boot)
 └── FeignContext  (padre compartido de todos los clientes)
      │  beans de FeignClientsConfiguration (defaults)
      │  beans de defaultConfiguration (si se declaró en @EnableFeignClients)
      │
      ├── child context: "catalogRead"
      │    └── beans de FeignReadOnlyConfig  ← sobreescriben los defaults solo aquí
      │
      ├── child context: "catalogWrite"
      │    └── beans de FeignAuthConfig      ← sobreescriben los defaults solo aquí
      │
      └── child context: "paymentFeign"
           └── sin @Configuration local → hereda todos los defaults
```

---

## Ejemplo central

El siguiente ejemplo muestra tres escenarios: configuración local por cliente, configuración global y la sobreescritura de un bean de `FeignClientsConfiguration`.

**Configuración local — solo afecta al cliente que la referencia:**

```java
package com.example.orderservice.config;

// IMPORTANTE: sin @Configuration a nivel de clase si está dentro del @ComponentScan
// La clase es reconocida por Feign porque se pasa en el atributo `configuration`
// de @FeignClient; Spring no la escanea de forma independiente.
public class FeignAuthConfig {

    /**
     * Sobreescribe el Retryer por defecto (Retryer.NEVER_RETRY).
     * Solo afecta al cliente que declara configuration = FeignAuthConfig.class.
     */
    @Bean
    public Retryer feignRetryer() {
        // period=100ms, maxPeriod=1000ms, maxAttempts=3
        return new Retryer.Default(100, 1000, 3);
    }

    /**
     * RequestInterceptor que añade el token Bearer a todas las peticiones
     * del cliente que usa esta configuración.
     */
    @Bean
    public RequestInterceptor authInterceptor(TokenProvider tokenProvider) {
        return requestTemplate ->
            requestTemplate.header("Authorization", "Bearer " + tokenProvider.getToken());
    }
}
```

```java
@FeignClient(
    name          = "catalog-service",
    contextId     = "catalogWrite",
    configuration = FeignAuthConfig.class   // solo este cliente usa FeignAuthConfig
)
public interface CatalogWriteClient {
    @PostMapping("/api/products/reserve")
    void reserve(@RequestBody ReservationRequest request);
}
```

**Configuración global — se aplica a todos los clientes del módulo:**

```java
package com.example.orderservice.config;

// Tampoco lleva @Configuration si está dentro del package escaneado
public class GlobalFeignConfig {

    /**
     * Logger.Level global: BASIC en todos los clientes.
     * Se puede sobreescribir a nivel local declarando otro Logger.Level bean
     * en la configuración del cliente específico.
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC;
    }
}
```

```java
@SpringBootApplication
@EnableFeignClients(
    basePackages      = "com.example.orderservice.client",
    defaultConfiguration = GlobalFeignConfig.class   // aplicado a todos los clientes
)
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

**Sobreescritura segura de FeignClientsConfiguration — beans disponibles para sobreescribir:**

```java
package com.example.orderservice.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import feign.codec.Decoder;
import feign.codec.Encoder;
import feign.form.spring.SpringFormEncoder;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringDecoder;
import org.springframework.cloud.openfeign.support.SpringEncoder;

public class FeignFormConfig {

    /**
     * Sobreescribe el Encoder por defecto para soportar multipart/form-data.
     * SpringFormEncoder delega en SpringEncoder para los tipos no-form.
     */
    @Bean
    public Encoder feignEncoder(ObjectFactory<HttpMessageConverters> messageConverters) {
        return new SpringFormEncoder(new SpringEncoder(messageConverters));
    }
}
```

> [CONCEPTO] `FeignClientsConfiguration` define los beans que Feign usa por defecto: `SpringEncoder`, `SpringDecoder`, `SpringMvcContract`, `Retryer.NEVER_RETRY`, `Logger.NoOpLogger` y otros. Para sobreescribir uno, basta con declarar un `@Bean` del mismo tipo en la clase de configuración local. Los beans no sobreescritos se heredan del contexto padre.

---

## Tabla de elementos clave

Los beans de `FeignClientsConfiguration` que pueden sobreescribirse mediante configuración local o global.

| Bean / Tipo | Implementación por defecto | Para qué se sobreescribe |
|---|---|---|
| `Encoder` | `SpringEncoder` | Soportar multipart, Protobuf, formato no-JSON |
| `Decoder` | `SpringDecoder` | Deserialización personalizada, tipos genéricos complejos |
| `ErrorDecoder` | `ErrorDecoder.Default` | Convertir códigos HTTP de error a excepciones de dominio |
| `Retryer` | `Retryer.NEVER_RETRY` | Habilitar reintentos nativos de Feign |
| `Logger` | `Logger.NoOpLogger` | Activar logging a SLF4J (`Slf4jLogger`) |
| `Logger.Level` | `Logger.Level.NONE` | Controlar verbosidad del log HTTP |
| `Contract` | `SpringMvcContract` | Usar anotaciones Feign nativas en lugar de Spring MVC |
| `RequestInterceptor` | ninguno | Propagación de cabeceras, auth, tracing |
| `feign.Client` | `LoadBalancerFeignClient` | Cambiar transporte HTTP (HC5, OkHttp, reactivo) |

---

## Buenas y malas prácticas

**Hacer:**
- Colocar las clases de configuración local de Feign en un subpaquete que esté **fuera** del `basePackages` del `@ComponentScan` de la aplicación (p. ej. `config.feign` y excluirlo explícitamente), o bien no anotar la clase con `@Configuration` para que Spring no la registre de forma autónoma.
- Usar `defaultConfiguration` en `@EnableFeignClients` para los beans transversales (Logger, RequestInterceptor de correlación) en lugar de repetirlos en cada configuración de cliente. Reduce la duplicación y garantiza coherencia en el comportamiento observable de todos los clientes.
- Documentar explícitamente en el Javadoc de cada clase de configuración si es local (un cliente) o global (todos los clientes), porque el código no lo expresa visualmente.

**Evitar:**
- Anotar la clase de configuración local con `@Configuration` **y** dejarla dentro del paquete escaneado. Es el error de configuración más frecuente en Feign. El síntoma: un `RequestInterceptor` "local" que aparece en todos los clientes del módulo.
- Definir el mismo tipo de bean en la configuración local y en `defaultConfiguration` sin entender el orden de precedencia. La configuración local tiene precedencia sobre la global, pero si ambas están activas, el bean local del child context gana. Si el bean local declara dependencias no disponibles en ese child context, el arranque falla con `NoSuchBeanDefinitionException`.
- Inyectar beans del contexto padre (p. ej. `@Autowired SecurityContext`) directamente en la clase de configuración local de Feign. El child context tiene visibilidad del padre, pero la inicialización del child puede ocurrir antes de que ciertos beans del padre estén listos, causando dependencias circulares sutiles.

---

## Comparación: configuración Java vs configuración por propiedades YAML

La configuración por clase Java y la configuración por propiedades son complementarias pero tienen semánticas distintas. El siguiente fichero (4.4) profundiza en la configuración YAML.

| Aspecto | `@Configuration` Java | `feign.client.config.*` YAML |
|---|---|---|
| Alcance | Local (un cliente) o global (`defaultConfiguration`) | Por cliente (`[clientName]`) o global (`default`) |
| Precedencia (SC 2021+) | Menor si `feign.client.default-to-properties=true` (default) | Mayor por defecto |
| Qué se puede configurar | Cualquier bean de Feign (Encoder, Decoder, Interceptor…) | Timeouts, Logger.Level, ErrorDecoder, RequestInterceptor |
| Recompilación necesaria | Sí | No (cambio de propiedad en ConfigMap o Config Server) |
| Legibilidad para Ops | Baja | Alta |

> [EXAMEN] Pregunta frecuente: "Tengo un `Logger.Level.FULL` en la clase `@Configuration` del cliente y `NONE` en `application.yml`. ¿Cuál gana?" Respuesta: el YAML gana porque `feign.client.default-to-properties=true` por defecto en Spring Cloud 2021+. Para invertir la precedencia, establecer `feign.client.default-to-properties=false`.

---

<- [4.2 Declaración de @FeignClient](sc-feign-cliente-declarativo.md) | [Índice](README.md) | [4.4 Configuración por propiedades YAML](sc-feign-configuracion-propiedades.md) ->
