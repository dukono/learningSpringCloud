# 2.3 Configuración y arranque del Eureka Client

← [2.2 Configuración y arranque del Eureka Server](sc-eureka-server.md) | [Índice (README.md)](README.md) | [2.4 Registro de instancias: metadata y estado →](sc-eureka-instancias.md)

---

Cada microservicio que quiera ser descubierto por otros —o que quiera descubrir a sus dependencias— necesita configurarse como Eureka Client. Sin esta configuración, el servicio arranca de forma aislada: no aparece en el registro y no puede resolver nombres lógicos de otros servicios. El cliente cumple dos roles simultáneamente: se anuncia en el servidor (registro) y mantiene una caché local del registro completo (descubrimiento). Entender estas dos responsabilidades por separado es clave para diagnosticar problemas en producción, porque cada una tiene su propio intervalo de refresco y su propio punto de fallo.

> [PREREQUISITO] El Eureka Server debe estar arrancado y accesible en la URL configurada en `eureka.client.serviceUrl.defaultZone` antes de levantar el cliente. Sin servidor disponible, el cliente reintenta en segundo plano y continúa arrancando, pero durante ese tiempo no está registrado.

## Diagrama: ciclo de vida del Eureka Client

El siguiente diagrama muestra las dos fases del cliente (registro y fetch de caché) y sus intervalos de tiempo, que son independientes entre sí.

```
Arranque del microservicio (Eureka Client)
         │
         ▼
@EnableDiscoveryClient (opcional en Spring Boot 4.0.x)
         │
         ▼
EurekaClientAutoConfiguration activa
         │
         ├─────────────────────────────────────────────────────────┐
         │                                                         │
         ▼  FASE 1: REGISTRO                                       ▼  FASE 2: FETCH DEL REGISTRO
         │                                                         │
PUT /eureka/apps/{APP_NAME}                          GET /eureka/apps → caché local
         │                                                         │
         ├── Inicial: al arrancar                   ├── Inicial: al arrancar
         │                                          │
         └── Heartbeat periódico:                   └── Refresco periódico:
             lease-renewal-interval                     registry-fetch-interval
             (default: 30 s)                            (default: 30 s)
         │
         ▼
Instancia visible en el registro con estado UP
```

## Implementación: Eureka Client completo

El siguiente ejemplo es un microservicio completo configurado como Eureka Client, incluyendo la configuración para múltiples instancias del mismo servicio con IDs únicos.

**pom.xml (dependencias relevantes)**

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
  <!-- Eureka Client -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>

  <!-- Spring Boot Web -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- Actuator: expone /actuator/health que Eureka usa para el health check -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>

  <!-- Spring Cloud LoadBalancer: reemplaza a Ribbon para llamadas @LoadBalanced -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>
</dependencies>
```

**ServicioInventarioApplication.java**

```java
package com.ejemplo.inventario;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
// @EnableDiscoveryClient es opcional en Spring Boot 4.0.x;
// la autoconfiguración lo activa automáticamente al detectar Eureka en el classpath.
// Se puede incluir por claridad explícita, pero no es necesario.
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class ServicioInventarioApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServicioInventarioApplication.class, args);
    }
}
```

**application.yml**

```yaml
server:
  port: 8082

spring:
  application:
    # Identificador lógico del servicio en el registro de Eureka
    # Los consumidores usarán este nombre para llamar: http://servicio-inventario/...
    name: servicio-inventario

eureka:
  instance:
    # Genera un instance-id único cuando hay varias instancias del mismo servicio.
    # Sin esta propiedad, la segunda instancia sobreescribe a la primera en el registro.
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
    # Preferir la IP en vez del hostname (útil en contenedores donde el hostname no resuelve)
    prefer-ip-address: true
    # Intervalo de heartbeat (segundos). Default: 30
    lease-renewal-interval-in-seconds: 30
    # Tiempo máximo sin heartbeat antes de considerar la instancia expirada (segundos). Default: 90
    lease-expiration-duration-in-seconds: 90
  client:
    # URL del Eureka Server (o servers separados por coma en cluster)
    service-url:
      defaultZone: http://localhost:8761/eureka/
    # El servicio SÍ se registra en Eureka (default: true)
    register-with-eureka: true
    # El servicio SÍ descarga el registro para poder descubrir a otros (default: true)
    fetch-registry: true
    # Intervalo de refresco de la caché local del registro (segundos). Default: 30
    registry-fetch-interval-seconds: 30
    # Integrar Spring Actuator con Eureka para propagar el estado de salud real
    healthcheck:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

**Configuración del RestTemplate con balanceo de carga**

```java
package com.ejemplo.inventario.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    /**
     * @LoadBalanced indica a Spring Cloud LoadBalancer que intercepte las
     * llamadas de este RestTemplate y resuelva el nombre del servicio
     * (ej: "http://servicio-pedidos/api/pedidos") contra el registro de Eureka.
     *
     * NOTA: Spring Cloud LoadBalancer sustituye a Ribbon desde 2022.x.
     * No añadas spring-cloud-starter-netflix-ribbon: no existe en 2025.1.1.
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

**Uso del RestTemplate con resolución de nombre lógico**

```java
package com.ejemplo.inventario.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class PedidoService {

    private final RestTemplate restTemplate;

    @Autowired
    public PedidoService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public String obtenerPedido(Long id) {
        // Spring Cloud LoadBalancer resuelve "servicio-pedidos"
        // contra el registro de Eureka antes de ejecutar la llamada HTTP
        return restTemplate.getForObject(
            "http://servicio-pedidos/api/pedidos/" + id,
            String.class
        );
    }
}
```

> [ADVERTENCIA] `@LoadBalanced` en el `RestTemplate` delega en Spring Cloud LoadBalancer, no en Ribbon. Si añades `spring-cloud-starter-netflix-ribbon` al pom.xml de un proyecto Spring Cloud 2025.1.1, obtendrás un error de dependencia en tiempo de compilación porque el artefacto fue eliminado del BOM. Ver [Spring Cloud LoadBalancer](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-loadbalancer).

## Tabla de propiedades clave del Eureka Client

La siguiente tabla recoge las propiedades de cliente que un senior debe conocer para configurar correctamente el registro y el descubrimiento.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.application.name` | String | — | Identificador lógico del servicio en el registro; obligatorio |
| `eureka.client.service-url.defaultZone` | String | `http://localhost:8761/eureka/` | URL del Eureka Server; múltiples URLs separadas por coma para cluster |
| `eureka.instance.instance-id` | String | `${hostname}:${appName}:${port}` | ID único de instancia; usar con `${random.value}` para múltiples instancias |
| `eureka.instance.prefer-ip-address` | boolean | `false` | Registra la IP en lugar del hostname; necesario en entornos Docker/K8s |
| `eureka.client.register-with-eureka` | boolean | `true` | Si `false`, el servicio no se registra (útil en clientes puros de discovery) |
| `eureka.client.fetch-registry` | boolean | `true` | Si `false`, el servicio no descarga el registro (útil en servidores puros) |
| `eureka.client.registry-fetch-interval-seconds` | int | `30` | Frecuencia de refresco de la caché local del registro |
| `eureka.instance.lease-renewal-interval-in-seconds` | int | `30` | Intervalo de heartbeat al servidor |
| `eureka.instance.lease-expiration-duration-in-seconds` | int | `90` | Tiempo máximo sin heartbeat antes de expiración |
| `eureka.client.healthcheck.enabled` | boolean | `false` | Integra Actuator health con el estado de Eureka |

## Buenas y malas prácticas

Hacer:
- Configurar `eureka.instance.instance-id` con `${random.value}` cuando se despliegan múltiples instancias del mismo servicio: sin esto, la segunda instancia sobreescribe a la primera en el registro y solo una de ellas estará disponible para el balanceo de carga.
- Activar `eureka.client.healthcheck.enabled: true` junto con Spring Actuator: permite que Eureka marque la instancia como `DOWN` cuando el health check de la aplicación falla, evitando que el balanceador envíe tráfico a una instancia degradada.
- Usar `prefer-ip-address: true` en entornos Docker o Kubernetes donde el hostname del contenedor no resuelve desde fuera: sin esta propiedad, los consumidores intentan conectar al hostname del contenedor, que no es resolvible desde la red del host.

Evitar:
- Dejar el `instance-id` por defecto cuando se usan contenedores efímeros: el ID por defecto incluye el hostname que en Docker suele ser el ID del contenedor, generando instancias huérfanas en el registro cada vez que el contenedor se recrea.
- Reducir `registry-fetch-interval-seconds` a valores inferiores a 5 segundos en sistemas con muchos clientes: cada cliente hace un GET al servidor en ese intervalo, lo que genera una carga de lectura proporcional a `nClientes / intervalo` peticiones por segundo sobre el servidor.
- Desactivar `fetch-registry: false` en un servicio que necesita llamar a otros: el servicio arrancará sin errores pero cualquier intento de resolver un nombre lógico fallará con `No instances available for servicio-X` porque la caché local estará vacía.

## Comparación: @EnableDiscoveryClient vs autoconfiguración

En Spring Boot 4.0.x, la anotación `@EnableDiscoveryClient` es opcional porque la autoconfiguración de Spring Cloud la activa automáticamente al detectar el starter en el classpath.

| Aspecto | Con @EnableDiscoveryClient | Sin @EnableDiscoveryClient |
|---|---|---|
| Comportamiento | Idéntico | Idéntico |
| Necesario en Spring Boot 4.0.x | No | — |
| Valor añadido | Documenta la intención explícitamente | Código más limpio |
| Caso de uso donde sí se necesita | Cuando se desea desactivar la autoconfiguración y controlarlo manualmente | — |

> [EXAMEN] En entrevistas es frecuente preguntar: "¿Es necesario `@EnableDiscoveryClient` en Spring Boot 4.0.x?". La respuesta correcta es: no, la autoconfiguración lo activa al detectar `spring-cloud-starter-netflix-eureka-client` en el classpath. La anotación sigue siendo válida y útil como documentación de intención, pero no cambia el comportamiento.

---

← [2.2 Configuración y arranque del Eureka Server](sc-eureka-server.md) | [Índice (README.md)](README.md) | [2.4 Registro de instancias: metadata y estado →](sc-eureka-instancias.md)
