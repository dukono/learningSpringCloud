# 12.5.2 Despliegue en Azure Functions y Google Cloud Functions

← [12.5.1 Despliegue en AWS Lambda](sc-function-aws.md) | [Índice (README.md)](README.md) | [12.6 Compilación a imagen nativa con GraalVM AOT](sc-function-native.md) →

---

## Introducción

Azure Functions y Google Cloud Functions presentan un desafío diferente al de AWS Lambda: cada plataforma tiene su propio modelo de binding (triggers, inputs y outputs declarativos) que es incompatible con el modelo de invocación directa de Spring Boot. Azure Functions define triggers mediante anotaciones Java propias (`@HttpTrigger`, `@QueueTrigger`) y GCF usa una interfaz `HttpFunction` que el runtime de Google invoca directamente. Sin el adaptador de Spring Cloud Function, el desarrollador debe duplicar el bootstrap del contexto Spring en cada handler específico de plataforma.

El adaptador `spring-cloud-function-adapter-azure` y el launcher `spring-cloud-function-adapter-gcp` resuelven esta incompatibilidad envolviendo el contexto Spring en el handler nativo de cada plataforma, de forma análoga al adaptador AWS pero con las particularidades del modelo de bindings de cada cloud. Este fichero documenta también el adaptador para Apache OpenWhisk / IBM Cloud Functions, que sigue un modelo de invocación HTTP puro más cercano al adaptador web.

> **[PREREQUISITO]** Requiere `sc-function-modelo.md` (12.1) y `sc-function-aws.md` (12.5.1). El mecanismo de arranque del contexto Spring en entorno serverless es el mismo en todas las plataformas; este fichero documenta las particularidades de cada adaptador.

## Representación visual

Los tres adaptadores de plataforma siguen el mismo patrón: el runtime de la plataforma invoca un handler nativo, que inicializa el contexto Spring y delega a la función registrada en `FunctionCatalog`.

```
  AZURE FUNCTIONS                    GOOGLE CLOUD FUNCTIONS
  ┌────────────────────────┐         ┌──────────────────────────┐
  │ Azure Runtime          │         │ GCF Runtime              │
  │   @FunctionName        │         │   HttpFunction.service() │
  │   @HttpTrigger         │         └──────────┬───────────────┘
  └──────────┬─────────────┘                    │
             │                                  ▼
             ▼                      SpringBootHttpRequestHandler
  AzureSpringBootRequestHandler       inicializa ApplicationContext
  inicializa ApplicationContext       delega a Function<HttpRequest,
  delega a Function<T,R>               HttpResponse>
             │                                  │
             └──────────────┬───────────────────┘
                            │
                   FunctionCatalog.lookup()
                            │
                   Function.apply(payload)
                            │
                   Respuesta serializada
```

## Ejemplo central

El ejemplo muestra los dos adaptadores con la mínima configuración necesaria para cada plataforma. La función de negocio es la misma en ambos casos.

### Azure Functions — Dependencias Maven

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
    <artifactId>spring-cloud-function-adapter-azure</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>com.microsoft.azure.functions</groupId>
    <artifactId>azure-functions-java-library</artifactId>
    <version>3.0.0</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### Azure Functions — application.yml

```yaml
spring:
  cloud:
    function:
      definition: greetUser
  main:
    web-application-type: none
```

### Azure Functions — Código Java completo

```java
package com.example.scfunction.azure;

import com.microsoft.azure.functions.ExecutionContext;
import com.microsoft.azure.functions.HttpMethod;
import com.microsoft.azure.functions.HttpRequestMessage;
import com.microsoft.azure.functions.HttpResponseMessage;
import com.microsoft.azure.functions.HttpStatus;
import com.microsoft.azure.functions.annotation.AuthorizationLevel;
import com.microsoft.azure.functions.annotation.FunctionName;
import com.microsoft.azure.functions.annotation.HttpTrigger;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.adapter.azure.AzureSpringBootRequestHandler;
import org.springframework.context.annotation.Bean;

import java.util.Optional;
import java.util.function.Function;

@SpringBootApplication
public class AzureFunctionApplication {

    public static void main(String[] args) {
        SpringApplication.run(AzureFunctionApplication.class, args);
    }

    public record UserRequest(String name) {}
    public record GreetingResponse(String message) {}

    @Bean
    public Function<UserRequest, GreetingResponse> greetUser() {
        return request -> new GreetingResponse("Hola, " + request.name() + "!");
    }
}

// Handler de Azure Functions que envuelve el contexto Spring
class AzureGreetHandler extends AzureSpringBootRequestHandler<UserRequest, GreetingResponse> {

    @FunctionName("greetUser")
    public HttpResponseMessage run(
        @HttpTrigger(
            name = "req",
            methods = {HttpMethod.POST},
            authLevel = AuthorizationLevel.ANONYMOUS
        ) HttpRequestMessage<Optional<UserRequest>> request,
        ExecutionContext context
    ) {
        context.getLogger().info("Invocando greetUser en Azure Functions");
        return handleRequest(request.getBody().orElse(new UserRequest("World")), context);
    }
}
```

### Google Cloud Functions — Dependencias Maven

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-gcp</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>com.google.cloud.functions</groupId>
    <artifactId>functions-framework-api</artifactId>
    <version>1.1.0</version>
    <scope>provided</scope>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### Google Cloud Functions — application.yml

```yaml
spring:
  cloud:
    function:
      definition: greetUser
  main:
    web-application-type: none
```

### Google Cloud Functions — Código Java completo

```java
package com.example.scfunction.gcp;

import com.google.cloud.functions.HttpFunction;
import com.google.cloud.functions.HttpRequest;
import com.google.cloud.functions.HttpResponse;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.function.adapter.gcp.GoogleCloudFunctionJarLauncher;
import org.springframework.context.annotation.Bean;

import java.util.function.Function;

@SpringBootApplication
public class GcpFunctionApplication {

    public static void main(String[] args) {
        SpringApplication.run(GcpFunctionApplication.class, args);
    }

    public record UserRequest(String name) {}
    public record GreetingResponse(String message) {}

    @Bean
    public Function<UserRequest, GreetingResponse> greetUser() {
        return request -> new GreetingResponse("Hola desde GCP, " + request.name() + "!");
    }
}

// Launcher de GCP: registrado como --target en el despliegue de GCF
// gcloud functions deploy greetUser \
//   --entry-point com.example.scfunction.gcp.GcpGreetLauncher \
//   --runtime java21
class GcpGreetLauncher extends GoogleCloudFunctionJarLauncher implements HttpFunction {

    @Override
    public void service(HttpRequest request, HttpResponse response) throws Exception {
        super.service(request, response);
    }
}
```

> **[CONCEPTO]** `GoogleCloudFunctionJarLauncher` implementa `HttpFunction` de GCF y en su método `service()` inicializa el `ApplicationContext` de Spring y delega a la función definida en `spring.cloud.function.definition`. El entry-point que se registra en GCF es la clase que extiende `GoogleCloudFunctionJarLauncher`.

## Tabla de elementos clave

La siguiente tabla compara los adaptadores de las tres plataformas no-AWS.

| Plataforma | Adaptador Maven | Clase handler | Mecanismo de trigger |
|---|---|---|---|
| Azure Functions | `spring-cloud-function-adapter-azure` | `AzureSpringBootRequestHandler<I,O>` | `@HttpTrigger`, `@QueueTrigger`, etc. |
| Google Cloud Functions | `spring-cloud-function-adapter-gcp` | `GoogleCloudFunctionJarLauncher` | HTTP (implements `HttpFunction`) |
| Apache OpenWhisk / IBM | `spring-cloud-function-adapter-openwhisk` | `OpenWhiskSpringBootRequestHandler` | HTTP JSON con formato OpenWhisk |
| AWS Lambda | `spring-cloud-function-adapter-aws` | `FunctionInvoker` | API Gateway Proxy, SQS, SNS, etc. |

Los parámetros de configuración comunes a todos los adaptadores son los siguientes.

| Parámetro | Descripción |
|---|---|
| `spring.cloud.function.definition` | Nombre de la función activa; obligatorio con múltiples beans |
| `spring.main.web-application-type=none` | Desactiva servidor web embebido en todos los adaptadores serverless |
| `layout: ZIP` (Maven plugin) | Formato de JAR compatible con Lambda y GCF |

## Buenas y malas prácticas

**Hacer:**
- Usar `spring.main.web-application-type=none` en todos los despliegues serverless (Azure, GCP, OpenWhisk) para evitar el arranque del servidor web embebido que añade tiempo al cold start sin utilidad en esos entornos.
- Separar la lógica de negocio (función Spring) del handler de la plataforma (clase que extiende el adaptador) en clases distintas: permite testear la función como POJO sin depender del SDK de la plataforma.
- En Azure Functions, gestionar la autenticación con `AuthorizationLevel.FUNCTION` o `ADMIN` en lugar de `ANONYMOUS` en entornos de producción; la configuración de seguridad de la plataforma es independiente de la lógica Spring.
- En GCF, empaquetar la aplicación como Fat JAR con el launcher como entry-point; GCF no soporta el classpath del Fat JAR de Spring Boot por defecto y requiere que todas las clases sean accesibles.

**Evitar:**
- Evitar usar bindings de Azure (Queue, ServiceBus, CosmosDB) directamente en el bean funcional de Spring: acopla la función al SDK de Azure y rompe la portabilidad a otras plataformas. Usar en su lugar el trigger de Azure para recibir el payload y delegarlo a la función Spring como POJO.
- Evitar heredar múltiples adaptadores en la misma clase: cada plataforma requiere su propio handler. Un artefacto puede tener múltiples handlers para distintas plataformas, pero cada handler es específico de su plataforma.
- Evitar ignorar el cold start en plataformas con facturación por invocación: en GCF y Azure Functions tier de consumo, un cold start de 8 segundos puede provocar timeouts en integraciones síncronas. Evaluar imagen nativa (ver `sc-function-native.md`) para funciones críticas.

---

← [12.5.1 Despliegue en AWS Lambda](sc-function-aws.md) | [Índice (README.md)](README.md) | [12.6 Compilación a imagen nativa con GraalVM AOT](sc-function-native.md) →
