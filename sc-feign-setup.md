# 4.1 Setup y bootstrap — dependencias, @EnableFeignClients y descubrimiento de servicios

<- [3.15 Testing / Verificación de Gateway](sc-gateway-testing.md) | [Índice](README.md) | [4.2 Declaración de @FeignClient](sc-feign-cliente-declarativo.md) ->

---

## Introducción

Cuando un microservicio necesita invocar a otro sobre HTTP, la alternativa manual implica instanciar `RestClient` o `WebClient`, construir la URL, serializar el cuerpo, deserializar la respuesta y gestionar errores HTTP, repitiendo ese código en cada llamada. Spring Cloud OpenFeign elimina toda esa infraestructura: el desarrollador declara una interfaz Java anotada y Feign genera el cliente en tiempo de arranque. El resultado es un cliente HTTP con la misma ergonomía que un repositorio Spring Data: solo la interfaz, sin implementación manual.

El bootstrap requiere tres pasos: añadir el starter al classpath, habilitar el escaneo de clientes con `@EnableFeignClients` y, en entornos con service discovery, asegurarse de que LoadBalancer puede resolver el nombre lógico del servicio. Sin LoadBalancer o Eureka en el classpath, `@FeignClient(name = "inventory-service")` fallará en arranque porque Feign no sabe a qué IP dirigir la llamada.

> [PREREQUISITO] Spring Cloud LoadBalancer (`spring-cloud-starter-loadbalancer`) o Eureka Client (`spring-cloud-starter-netflix-eureka-client`) deben estar en el classpath cuando se usa el atributo `name` en `@FeignClient`. Sin ellos, usar el atributo `url` con dirección fija.

---

## Diagrama: flujo de arranque de Feign

El siguiente diagrama muestra cómo Spring Boot autoconfigura los clientes Feign durante el arranque y cómo se resuelve el nombre lógico en cada petición.

```
 @SpringBootApplication
 + @EnableFeignClients
        │
        ▼
 FeignClientsRegistrar                ← escanea interfaces @FeignClient
        │  crea BeanDefinition por cada interfaz
        ▼
 FeignClientFactoryBean               ← child ApplicationContext por cliente
        │  instancia Encoder, Decoder, Logger, Retryer …
        ▼
 ReflectiveFeign.newInstance()        ← genera proxy JDK para la interfaz
        │
   en cada llamada al proxy:
        │
        ▼
 LoadBalancerFeignClient              ← extrae el nombre lógico
        │
        ▼
 Spring Cloud LoadBalancer            ← consulta Eureka / registry
        │  selecciona instancia concreta (host:port)
        ▼
 HTTP Client subyacente               ← HttpURLConnection / HC5 / OkHttp
        │
        ▼
   Servicio remoto
```

---

## Ejemplo central

El siguiente ejemplo muestra el mínimo necesario para activar Feign con Eureka en un proyecto Spring Boot 4.0.x. Incluye la dependencia Maven, la configuración YAML y la clase principal.

**pom.xml** (fragmento de dependencias — usar dentro del BOM `spring-cloud-dependencies`):

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
  <!-- Starter de OpenFeign -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>

  <!-- LoadBalancer: necesario para resolución de nombre lógico -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>

  <!-- Eureka Client: registro y descubrimiento (opcional si se usa url fija) -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
</dependencies>
```

**application.yml**:

```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true

# Desactivar Ribbon si viene de una migración pre-2020.0
# spring.cloud.loadbalancer.ribbon.enabled=false  ← ya no es necesario en 2025.1.1
```

**OrderServiceApplication.java**:

```java
package com.example.orderservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.orderservice.client")
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

**InventoryClient.java** (interfaz de cliente — la implementación la genera Feign):

```java
package com.example.orderservice.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "inventory-service")   // "inventory-service" → resuelto por Eureka + LoadBalancer
public interface InventoryClient {

    @GetMapping("/api/inventory/{productId}")
    InventoryResponse getInventory(@PathVariable("productId") String productId);
}
```

> [CONCEPTO] `@EnableFeignClients` es el punto de entrada del módulo. Sin esta anotación, ninguna interfaz `@FeignClient` del classpath es registrada como bean de Spring. El atributo `basePackages` limita el escaneo; sin él, Feign escanea desde el paquete de la clase anotada hacia abajo.

---

## Tabla de elementos clave

Los siguientes parámetros y propiedades controlan el bootstrap del módulo. Un senior debe conocerlos de memoria.

| Parámetro / Atributo | Tipo | Default | Descripción |
|---|---|---|---|
| `@EnableFeignClients` | anotación | — | Activa el escaneo y registro de clientes Feign |
| `basePackages` | `String[]` | paquete de la clase anotada | Limita el escaneo de interfaces `@FeignClient` |
| `basePackageClasses` | `Class[]` | — | Alternativa type-safe a `basePackages` |
| `defaultConfiguration` | `Class[]` | `FeignClientsConfiguration` | Clase(s) `@Configuration` aplicadas a todos los clientes |
| `spring-cloud-starter-openfeign` | dependencia | — | Starter que activa la autoconfiguración de Feign |
| `spring-cloud-starter-loadbalancer` | dependencia | — | Necesario para resolución de `name` sin Eureka |
| `spring-cloud-starter-netflix-eureka-client` | dependencia | — | Activa descubrimiento; LoadBalancer lo usa automáticamente |
| `spring.cloud.loadbalancer.ribbon.enabled` | `Boolean` | `false` en 2025.1.1 | [LEGACY] Solo relevante en migraciones desde pre-2020.0 |

---

## Buenas y malas prácticas

**Hacer:**
- Especificar siempre `basePackages` o `basePackageClasses` en `@EnableFeignClients`. Sin restricción, el escaneo recorre todo el classpath de la aplicación en arranque, lo que ralentiza el inicio en proyectos grandes.
- Usar el BOM `spring-cloud-dependencies` para gestionar todas las versiones de Spring Cloud. La compatibilidad entre `spring-cloud-starter-openfeign` y `spring-boot-starter` no está garantizada si se declaran versiones manuales.
- Mantener `spring-cloud-starter-loadbalancer` junto a `spring-cloud-starter-openfeign` como par inseparable cuando se usa `name`. Si LoadBalancer no está, el proxy arranca pero lanza `IllegalStateException` en la primera llamada.

**Evitar:**
- Poner `@EnableFeignClients` en una clase `@Configuration` separada que también lleva `@ComponentScan`. Esa combinación puede causar que el child context de Feign se registre en el contexto padre, duplicando beans y produciendo `BeanDefinitionOverrideException` en arranque.
- Usar el atributo `url` en producción con dirección IP hardcodeada. Rompe el balanceo de carga y hace imposible el despliegue multi-instancia sin recompilar.
- Combinar `spring-cloud-starter-netflix-eureka-client` con configuración de Ribbon en migraciones sin eliminar explícitamente las propiedades `ribbon.*`. El comportamiento undefined puede dar lugar a balanceo inconsistente entre reintentos.

---

## Comparación: `name` vs `url` en `@FeignClient`

Ambos atributos permiten declarar a qué servicio apunta el cliente, pero tienen semánticas muy diferentes. La elección afecta el comportamiento en producción.

| Aspecto | `name` (nombre lógico) | `url` (dirección fija) |
|---|---|---|
| Requiere LoadBalancer/Eureka | Sí | No |
| Balanceo de carga | Automático (round-robin por defecto) | No hay balanceo |
| Usado en | Producción / entornos con service discovery | Tests, prototipos, servicios externos sin registry |
| Ejemplo | `@FeignClient(name = "inventory-service")` | `@FeignClient(name = "ext", url = "https://api.external.com")` |
| Cambio de instancia sin recompilar | Sí | No |
| Fallo si registry no disponible | Sí (en arranque o en llamada según config) | No |

> [EXAMEN] Una pregunta frecuente en entrevista es: "¿Qué ocurre si declaro `@FeignClient(name = "x")` y no hay Eureka ni LoadBalancer en el classpath?" La respuesta: la aplicación arranca pero la primera llamada lanza `java.lang.IllegalStateException: No instances available for x` porque no hay `ServiceInstanceListSupplier` capaz de resolver el nombre.

---

## Referencias externas (grupo2)

**Spring Cloud LoadBalancer** — Feign delega la resolución de `name` a `spring-cloud-starter-loadbalancer`. Con Eureka activo, el registro de instancias es automático. Sin Eureka, se puede configurar una lista estática con `spring.cloud.loadbalancer.configurations=default` y propiedades `[service-id].ribbon.listOfServers` [LEGACY] o la propiedad moderna `loadbalancer.[service-id].instances`. Ver módulo sc-loadbalancer para algoritmos y configuración avanzada.

**Eureka** — cuando `spring-cloud-starter-netflix-eureka-client` está en el classpath, Feign resuelve `name` consultando el registry de Eureka. Sin Eureka, usar el atributo `url`. Ver módulo sc-eureka para registro y configuración detallada.

---

<- [3.15 Testing / Verificación de Gateway](sc-gateway-testing.md) | [Índice](README.md) | [4.2 Declaración de @FeignClient](sc-feign-cliente-declarativo.md) ->
