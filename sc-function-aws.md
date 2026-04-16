# 12.5.1 Despliegue en AWS Lambda con SpringBootRequestHandler y FunctionInvoker

← [12.4 Routing dinámico con RoutingFunction y expresiones SpEL](sc-function-routing.md) | [Índice (README.md)](README.md) | [12.5.2 Despliegue en Azure Functions y Google Cloud Functions](sc-function-cloud.md) →

---

## Introducción

El problema concreto que resuelve este adaptador es la incompatibilidad entre el modelo de arranque de Spring (que levanta un `ApplicationContext` completo) y el modelo de invocación de AWS Lambda (que ejecuta un método estático sin servidor persistente). Sin el adaptador, el desarrollador debe gestionar manualmente el ciclo de vida del contexto Spring dentro del handler Lambda, lo que produce inicializaciones duplicadas, memory leaks por contextos no cerrados y cold starts de 10-15 segundos en funciones con dependencias pesadas.

Spring Cloud Function provee el adaptador `spring-cloud-function-adapter-aws` que envuelve el contexto Spring en un `SpringBootRequestHandler` o `FunctionInvoker` y lo inicializa una vez en la primera invocación (o en el init method del Lambda), reutilizándolo en invocaciones subsiguientes (warm Lambda). El adaptador también convierte automáticamente entre los tipos AWS API Gateway Proxy Request/Response y los POJOs de la función.

> **[PREREQUISITO]** Requiere `sc-function-modelo.md` (12.1) y `sc-function-config.md` (12.2). Para entender la motivación del cold start, ver también `sc-function-native.md` (12.6).

> **[CONCEPTO]** La configuración de memoria (mínimo 512 MB recomendado para Spring) y el timeout se gestionan en la consola AWS o vía SAM/CDK, fuera del scope de Spring Cloud Function. Ver [documentación AWS Lambda de configuración de memoria](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html).

## Representación visual

El flujo de invocación en AWS Lambda con el adaptador Spring Cloud Function tiene dos fases: la inicialización del contexto (cold start) y la invocación recurrente (warm).

```
  AWS Lambda Runtime
  ┌──────────────────────────────────────────────────────────┐
  │  COLD START (primera invocación)                         │
  │                                                          │
  │  Lambda Runtime → FunctionInvoker.handleRequest()        │
  │                          │                               │
  │                          ▼                               │
  │              SpringApplication.run()                     │
  │              ApplicationContext init (~2-8s)             │
  │              FunctionCatalog populated                   │
  │                          │                               │
  │                          ▼                               │
  │              Deserializa APIGatewayRequest               │
  │              → POJO de entrada                           │
  │                          │                               │
  │                          ▼                               │
  │              Function<POJO,POJO>.apply()                 │
  │                          │                               │
  │                          ▼                               │
  │              Serializa POJO → APIGatewayResponse         │
  └──────────────────────────────────────────────────────────┘

  WARM (invocaciones siguientes)
  Lambda Runtime → FunctionInvoker.handleRequest()
                          │
                  Contexto ya iniciado (reutilizado)
                          │
                  Function.apply() directo
```

## Ejemplo central

El ejemplo implementa una función que procesa un pedido y devuelve una factura, empaquetada como Fat JAR para despliegue en AWS Lambda con API Gateway Proxy.

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
    <artifactId>spring-cloud-function-adapter-aws</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>

<build>
  <plugins>
    <!-- Fat JAR para AWS Lambda -->
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
        <!-- AWS Lambda no soporta el layout Spring Boot por defecto -->
        <layout>ZIP</layout>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>repackage</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      definition: processOrder
  main:
    # Deshabilita el servidor web embebido; Lambda no necesita servidor HTTP
    web-application-type: none
```

### Código Java completo

```java
package com.example.scfunction.aws;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.adapter.aws.FunctionInvoker;
import org.springframework.context.annotation.Bean;

import java.util.function.Function;

@SpringBootApplication
public class OrderLambdaApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderLambdaApplication.class, args);
    }

    // POJO de entrada (deserializado desde el body del API Gateway Proxy Request)
    public record Order(Long id, String item, double price) {}

    // POJO de salida (serializado al body del API Gateway Proxy Response)
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

// Handler registrado en AWS Lambda como:
// com.example.scfunction.aws.OrderLambdaHandler::handleRequest
class OrderLambdaHandler extends FunctionInvoker {
    // FunctionInvoker inicializa el ApplicationContext de Spring en el primer handleRequest
    // y lo reutiliza en invocaciones warm. No se necesita código adicional.
}
```

Configuración del handler en AWS Lambda (consola o SAM):
```yaml
# template.yaml (AWS SAM)
Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.scfunction.aws.OrderLambdaHandler::handleRequest
      Runtime: java21
      MemorySize: 512
      Timeout: 30
      CodeUri: target/order-lambda-0.0.1-SNAPSHOT-aws.jar
      Environment:
        Variables:
          SPRING_CLOUD_FUNCTION_DEFINITION: processOrder
```

> **[EXAMEN]** La diferencia entre `SpringBootRequestHandler` (deprecated desde SCF 4.x) y `FunctionInvoker` es que `FunctionInvoker` no requiere especificar los tipos genéricos en la subclase y resuelve la función mediante `spring.cloud.function.definition`. `SpringBootRequestHandler` requería `class MyHandler extends SpringBootRequestHandler<Order,Invoice>` con los tipos explícitos. En Spring Cloud 2025.1.1, usar siempre `FunctionInvoker`.

## Tabla de elementos clave

La siguiente tabla recoge los elementos del adaptador AWS y sus configuraciones críticas.

| Elemento | Tipo | Descripción |
|---|---|---|
| `FunctionInvoker` | Clase adaptador SCF | Handler Lambda que inicializa ApplicationContext y delega a la función activa |
| `SpringBootRequestHandler` | Clase adaptador (deprecated) | Handler legacy con tipos genéricos explícitos; evitar en nuevos proyectos |
| `spring-cloud-function-adapter-aws` | Dependencia Maven | Starter que provee `FunctionInvoker` y conversión API Gateway ↔ POJO |
| `layout: ZIP` | Propiedad Maven plugin | Formato de empaquetado requerido por Lambda; el layout por defecto de Spring Boot no es compatible |
| `spring.main.web-application-type: none` | Propiedad | Deshabilita servidor web embebido; obligatorio en Lambda (no hay puerto abierto) |
| `SPRING_CLOUD_FUNCTION_DEFINITION` | Variable de entorno Lambda | Alternativa a `application.yml` para especificar la función activa en Lambda |
| Cold start | Concepto | Primera invocación Lambda; incluye `SpringApplication.run()` (~2-8s con Spring) |
| Warm invocation | Concepto | Invocaciones subsiguientes con contexto ya iniciado; latencia < 100ms |
| `MemorySize: 512` | Configuración SAM/consola | Mínimo recomendado para Spring; aumentar para reducir cold start |

## Buenas y malas prácticas

**Hacer:**
- Usar `spring.main.web-application-type=none` en `application.yml` para evitar que Spring intente arrancar un servidor Tomcat/Netty dentro de Lambda, lo que aumenta el cold start innecesariamente.
- Configurar la función activa vía variable de entorno `SPRING_CLOUD_FUNCTION_DEFINITION` en Lambda en lugar de hardcodearla en `application.yml`, para poder reutilizar el mismo artefacto con distintas funciones en distintos lambdas.
- Usar `MemorySize` de al menos 512 MB: Lambda asigna CPU proporcional a la memoria, y un JVM Spring con 128 MB puede tardar más de 30 segundos en el cold start, superando el timeout típico.
- Empaquetar con `layout: ZIP` en `spring-boot-maven-plugin`; el layout `JAR` por defecto pone las clases en `BOOT-INF/classes`, que Lambda no puede encontrar directamente.

**Evitar:**
- Evitar el uso de `SpringBootRequestHandler` en proyectos nuevos: está deprecated y requiere duplicar los tipos genéricos en la subclase del handler, lo que fuerza un nuevo artefacto por cada función con tipos distintos.
- Evitar inicializar beans pesados (conexiones a base de datos, clientes HTTP) en el constructor de la función: el cold start ya es costoso; diferir la inicialización hasta la primera invocación o usar `@Lazy`.
- Evitar Fat JARs superiores a 50 MB sin evaluar el impacto en cold start: cada MB adicional en el JAR añade tiempo de carga en el cold start de Lambda. Evaluar Thin JAR o imagen nativa (ver `sc-function-native.md`) para funciones críticas en latencia.
- Evitar asumir que el contexto Spring se reinicia en cada invocación: Lambda reutiliza el contexto en invocaciones warm, lo que significa que los beans son stateful entre invocaciones. No usar campos de instancia como estado de ejecución.

---

← [12.4 Routing dinámico con RoutingFunction y expresiones SpEL](sc-function-routing.md) | [Índice (README.md)](README.md) | [12.5.2 Despliegue en Azure Functions y Google Cloud Functions](sc-function-cloud.md) →
