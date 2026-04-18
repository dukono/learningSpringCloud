# 3.2.1 Configuración por clase Java (FeignClientConfiguration)

← [3.1 Cliente declarativo básico con OpenFeign](sc-feign-cliente-declarativo.md) | [Índice](README.md) | [3.2.2 Configuración por propiedades](sc-feign-configuracion-propiedades.md) →

---

## Introducción

Spring Cloud OpenFeign crea un sub-contexto (ApplicationContext hijo) independiente por cada cliente `@FeignClient`. La configuración Java por cliente permite registrar beans específicos — `Encoder`, `Decoder`, `Contract`, `Logger`, `ErrorDecoder`, `Retryer`, `RequestInterceptor` — que afectan únicamente a ese cliente, sin modificar el comportamiento global de otros clientes de la aplicación. Este mecanismo es necesario cuando distintos microservicios remotos requieren configuraciones diferentes: por ejemplo, uno necesita autenticación Bearer y otro Basic Auth, o uno usa JSON y otro XML.

> [PREREQUISITO] Conocer el patrón de sub-contexto de Feign: cada `@FeignClient` tiene su propio `ApplicationContext` hijo. Los beans registrados en la clase de configuración por cliente viven en ese contexto hijo, no en el padre.

## Diagrama de contextos

Feign sigue un modelo de herencia de contextos de Spring donde cada cliente tiene su propio contenedor de beans. Esto es lo que hace posible la configuración independiente por cliente.

```
ApplicationContext raíz (Spring Boot)
│
├── sub-contexto: inventory-service
│   ├── Encoder (personalizado)
│   ├── ErrorDecoder (personalizado)
│   └── RequestInterceptor (solo para este cliente)
│
├── sub-contexto: payment-service
│   ├── Encoder (por defecto)
│   └── RequestInterceptor (diferente)
│
└── sub-contexto: notification-service
    └── (usa todos los defaults)
```

## Ejemplo central

El ejemplo muestra la configuración de dos clientes Feign con configuraciones Java independientes. El primero usa un `RequestInterceptor` para añadir JWT Bearer token; el segundo usa un `ErrorDecoder` personalizado. La clase de configuración NO lleva `@Configuration` a nivel de component scan para evitar que los beans sean compartidos con todos los clientes.

```java
// Configuración para inventory-service — SIN @Configuration a nivel de componente scan
package com.example.orders.feign.config;

import feign.Logger;
import feign.RequestInterceptor;
import feign.RequestTemplate;
import feign.codec.ErrorDecoder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;

// NOTA: NO se anota con @Configuration aquí para evitar que Spring la detecte en el
// component scan raíz y convierta sus @Beans en globales para todos los clientes Feign.
public class InventoryFeignConfig {

    @Value("${services.inventory.api-key:default-key}")
    private String apiKey;

    @Bean
    public RequestInterceptor apiKeyInterceptor() {
        return (RequestTemplate template) ->
            template.header("X-API-Key", apiKey);
    }

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

    @Bean
    public ErrorDecoder inventoryErrorDecoder() {
        return (methodKey, response) -> {
            if (response.status() == 404) {
                return new InventoryItemNotFoundException(
                    "Item not found — method: " + methodKey
                );
            }
            return new feign.codec.ErrorDecoder.Default().decode(methodKey, response);
        };
    }
}
```

```java
// Excepción de dominio para inventario
package com.example.orders.feign.config;

public class InventoryItemNotFoundException extends RuntimeException {
    public InventoryItemNotFoundException(String message) {
        super(message);
    }
}
```

```java
// Configuración para payment-service — también SIN @Configuration
package com.example.orders.feign.config;

import feign.Logger;
import feign.RequestInterceptor;
import feign.RequestTemplate;
import feign.Retryer;
import org.springframework.context.annotation.Bean;

public class PaymentFeignConfig {

    @Bean
    public Retryer paymentRetryer() {
        // period=100ms, maxPeriod=1000ms, maxAttempts=3
        return new Retryer.Default(100, 1000, 3);
    }

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC;
    }
}
```

```java
// Clientes Feign que referencian sus configuraciones
package com.example.orders.clients;

import com.example.orders.dto.InventoryResponse;
import com.example.orders.dto.PaymentRequest;
import com.example.orders.dto.PaymentResponse;
import com.example.orders.feign.config.InventoryFeignConfig;
import com.example.orders.feign.config.PaymentFeignConfig;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(
    name = "inventory-service",
    path = "/api/v1",
    configuration = InventoryFeignConfig.class   // ← configuración específica
)
public interface InventoryClient {

    @GetMapping("/items/{id}")
    InventoryResponse getItem(@PathVariable("id") Long id);
}

@FeignClient(
    name = "payment-service",
    path = "/api/v1",
    configuration = PaymentFeignConfig.class     // ← configuración específica
)
public interface PaymentClient {

    @PostMapping("/payments")
    PaymentResponse processPayment(@RequestBody PaymentRequest request);
}
```

```java
// Configuración de arranque con defaultConfiguration para aplicar a TODOS los clientes
package com.example.orders;

import com.example.orders.feign.config.GlobalFeignConfig;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients(
    basePackages = "com.example.orders.clients",
    defaultConfiguration = GlobalFeignConfig.class  // ← aplica a todos los clientes
)
public class OrdersApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApplication.class, args);
    }
}
```

```java
// Configuración global (defaultConfiguration) — PUEDE llevar @Configuration porque
// se registra explícitamente como defaultConfiguration, no se auto-detecta por scan
package com.example.orders.feign.config;

import feign.Logger;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GlobalFeignConfig {

    @Bean
    public Logger.Level defaultFeignLoggerLevel() {
        return Logger.Level.BASIC;
    }
}
```

## Tabla de beans configurables

La clase de configuración Java puede registrar los siguientes beans para personalizar el comportamiento del cliente:

| Bean | Interfaz/Clase | Efecto |
|---|---|---|
| `Encoder` | `feign.codec.Encoder` | Serializa el cuerpo de la petición |
| `Decoder` | `feign.codec.Decoder` | Deserializa el cuerpo de la respuesta |
| `Contract` | `feign.Contract` | Define qué anotaciones de mapeo se reconocen |
| `Logger` | `feign.Logger` | Destino de los logs de Feign |
| `Logger.Level` | `feign.Logger.Level` | Verbosidad del logging (NONE/BASIC/HEADERS/FULL) |
| `ErrorDecoder` | `feign.codec.ErrorDecoder` | Transforma respuestas de error en excepciones |
| `Retryer` | `feign.Retryer` | Política de reintentos ante fallos |
| `RequestInterceptor` | `feign.RequestInterceptor` | Modifica `RequestTemplate` antes de enviar |
| `Client` | `feign.Client` | Cliente HTTP subyacente |

## El problema crítico de @Configuration

El problema más importante de la configuración Java de Feign es la visibilidad de la clase. Si la clase de configuración de un cliente específico lleva `@Configuration` y está en un paquete que el component scan raíz detecta, sus `@Bean` se registran en el contexto padre y quedan disponibles para **todos** los clientes Feign, eliminando el efecto de configuración por cliente.

La solución oficial es no anotar la clase con `@Configuration` cuando sea configuración por cliente. Si se necesita inyectar valores con `@Value` o usar `@Autowired`, eso funciona igual sin `@Configuration` porque Feign invoca los métodos `@Bean` de la clase instanciándola directamente en el sub-contexto.

> [ADVERTENCIA] Si la clase de configuración específica de un cliente tiene `@Configuration` Y está dentro del basePackage del component scan, sus beans se volverán globales para todos los clientes Feign. Este es uno de los errores más frecuentes en la certificación Spring Professional.

## Buenas y malas prácticas

**Buenas prácticas:**
- Colocar las clases de configuración por cliente en un subpaquete separado del basePackages de `@EnableFeignClients` para evitar que sean detectadas por el scan.
- Usar `defaultConfiguration` en `@EnableFeignClients` para comportamiento común a todos los clientes.
- Nombrar las clases claramente: `InventoryFeignConfig`, `PaymentFeignConfig`, no `FeignConfig`.

**Malas prácticas:**
- Añadir `@Configuration` a una clase de configuración específica de cliente si está dentro del basePackage de scan.
- Registrar un `RequestInterceptor` en la configuración global cuando solo aplica a un cliente.

## Verificación y práctica

> [EXAMEN] **1.** ¿Por qué la clase de configuración específica de un cliente Feign NO debe llevar `@Configuration` si está dentro del paquete escaneado por Spring?

> [EXAMEN] **2.** ¿Cómo se aplica una configuración por defecto a todos los clientes Feign registrados con `@EnableFeignClients`?

> [EXAMEN] **3.** Un `Retryer` registrado en `InventoryFeignConfig` (configuración de `inventory-service`), ¿afectará también al `PaymentClient`? Justifica la respuesta.

> [EXAMEN] **4.** Enumera al menos 5 tipos de beans que se pueden registrar en una clase de configuración Java de Feign y describe el efecto de cada uno.

> [EXAMEN] **5.** ¿Qué ocurre si tanto la `defaultConfiguration` de `@EnableFeignClients` como la `configuration` específica del cliente definen un bean del mismo tipo (por ejemplo, `Logger.Level`)? ¿Cuál tiene precedencia?

---

← [3.1 Cliente declarativo básico con OpenFeign](sc-feign-cliente-declarativo.md) | [Índice](README.md) | [3.2.2 Configuración por propiedades](sc-feign-configuracion-propiedades.md) →
