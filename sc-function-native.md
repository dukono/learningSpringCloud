# 12.6 Compilación a imagen nativa con GraalVM AOT

← [12.5.2 Despliegue en Azure Functions y Google Cloud Functions](sc-function-cloud.md) | [Índice (README.md)](README.md) | [12.7 Integración con Spring Cloud Stream](sc-function-stream.md) →

---

## Introducción

El problema que resuelve la compilación nativa es el cold start de las funciones Spring en entornos serverless. Una función Spring empaquetada como Fat JAR requiere que la JVM arranque, cargue el bytecode y el framework inicialice el `ApplicationContext`: en Lambda o GCF esto puede durar entre 3 y 15 segundos dependiendo del número de beans. En arquitecturas serverless con invocaciones esporádicas, este tiempo de arranque es inaceptable porque excede los timeouts de los clientes síncronos.

GraalVM Native Image compila el código Java a un ejecutable binario en tiempo de compilación (Ahead-Of-Time, AOT), eliminando la JVM en runtime. El arranque de un ejecutable nativo tarda entre 50 y 200 ms. Spring Boot 4.0.x y Spring Cloud Function incluyen soporte AOT completo: el goal `process-aot` del plugin Maven genera los hints de reflexión, proxies y recursos necesarios para que GraalVM pueda compilar el contexto Spring sin depender de reflexión dinámica en runtime.

> **[PREREQUISITO]** Requiere `sc-function-aws.md` (12.5.1) para entender la motivación del cold start. La compilación se delega a GraalVM; Spring Cloud Function provee hints AOT. Para el compilador nativo, consultar la [documentación de GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/).

> **[ADVERTENCIA]** La compilación nativa tiene restricciones: reflexión dinámica, proxies JDK y carga dinámica de clases requieren hints explícitos. Spring Boot 4.0.x genera la mayoría automáticamente, pero las librerías de terceros sin soporte AOT pueden fallar en runtime con `ClassNotFoundException` o `NoSuchMethodException`.

## Representación visual

El flujo de compilación nativa tiene dos fases: la fase de build (generación de hints AOT + compilación nativa) y la fase de despliegue (ejecutable binario sin JVM).

```
  FASE DE BUILD
  ┌─────────────────────────────────────────────────────────┐
  │ mvn spring-boot:process-aot                             │
  │    │                                                    │
  │    ▼                                                    │
  │ SpringAOT analiza contexto                              │
  │ Genera:                                                 │
  │   - reflect-config.json (hints de reflexión)            │
  │   - resource-config.json (recursos en classpath)        │
  │   - proxy-config.json (interfaces de proxy JDK)         │
  │   - serialization-config.json (serialización)           │
  │    │                                                    │
  │    ▼                                                    │
  │ mvn -Pnative native:compile                             │
  │    │                                                    │
  │    ▼                                                    │
  │ GraalVM native-image                                    │
  │   Lee hints + bytecode                                  │
  │   Compila a ejecutable binario (~5-10 min)              │
  └────────────────────┬────────────────────────────────────┘
                       │
  FASE DE DESPLIEGUE   │
  ┌────────────────────▼────────────────────────────────────┐
  │ ./target/order-function (ejecutable Linux/macOS/Windows)│
  │   Arranque: ~50-200 ms (sin JVM)                        │
  │   Memoria: ~30-80 MB (sin heap JVM)                     │
  └─────────────────────────────────────────────────────────┘
```

## Ejemplo central

El ejemplo muestra la configuración Maven completa para compilar una función Spring Cloud Function a imagen nativa, con el perfil `native` y el plugin GraalVM.

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
    <artifactId>spring-cloud-function-context</artifactId>
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
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <executions>
        <execution>
          <id>process-aot</id>
          <goals>
            <!-- Genera los hints AOT necesarios para GraalVM -->
            <goal>process-aot</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>

<profiles>
  <profile>
    <id>native</id>
    <build>
      <plugins>
        <plugin>
          <groupId>org.graalvm.buildtools</groupId>
          <artifactId>native-maven-plugin</artifactId>
          <version>0.10.3</version>
          <extensions>true</extensions>
          <executions>
            <execution>
              <id>build-native</id>
              <goals>
                <goal>compile-no-fork</goal>
              </goals>
              <phase>package</phase>
            </execution>
          </executions>
          <configuration>
            <!-- Nombre del ejecutable resultante -->
            <imageName>order-function</imageName>
            <!-- Opciones adicionales de GraalVM native-image -->
            <buildArgs>
              <buildArg>--initialize-at-build-time=org.slf4j</buildArg>
              <buildArg>-H:+ReportExceptionStackTraces</buildArg>
            </buildArgs>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      definition: processOrder
  main:
    web-application-type: none
```

### Código Java completo

```java
package com.example.scfunction.native_;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.util.function.Function;

@SpringBootApplication
public class NativeFunctionApplication {

    public static void main(String[] args) {
        SpringApplication.run(NativeFunctionApplication.class, args);
    }

    // Record: compatible con GraalVM AOT (no requiere hints de reflexión adicionales)
    public record Order(Long id, String item, double price) {}
    public record Invoice(Long orderId, String description, double total) {}

    // Función sin reflexión dinámica — compatible con imagen nativa
    @Bean
    public Function<Order, Invoice> processOrder() {
        return order -> new Invoice(
            order.id(),
            "Native: " + order.item(),
            order.price() * 1.21
        );
    }
}
```

Para registrar hints de reflexión adicionales cuando una clase de terceros no tiene soporte AOT:
```java
package com.example.scfunction.native_;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.context.annotation.ImportRuntimeHints;
import org.springframework.stereotype.Component;

// Registrar hints de reflexión para clases sin soporte AOT automático
@Component
@ImportRuntimeHints(CustomRuntimeHints.Registrar.class)
public class CustomRuntimeHints {

    static class Registrar implements RuntimeHintsRegistrar {
        @Override
        public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
            // Registra reflexión para una clase de terceros sin soporte AOT
            hints.reflection()
                .registerType(Order.class, hint -> hint
                    .withMembers()
                    .withConstructors()
                );
        }
    }

    public record Order(Long id, String item, double price) {}
}
```

Comandos de compilación y ejecución:
```bash
# Paso 1: generar hints AOT
mvn spring-boot:process-aot

# Paso 2: compilar imagen nativa (requiere GraalVM JDK 21+ instalado)
mvn -Pnative native:compile

# Paso 3: ejecutar el binario nativo
./target/order-function
# Arranque en ~80 ms vs ~4000 ms del JAR estándar
```

> **[EXAMEN]** La diferencia entre `process-aot` y `native:compile` es que `process-aot` es un goal de Spring Boot Maven Plugin que analiza el contexto Spring y genera los archivos de configuración de GraalVM (JSON de reflexión, recursos, proxies). `native:compile` es un goal del plugin GraalVM que usa esos archivos para compilar el bytecode a ejecutable nativo. Ambos son necesarios: `process-aot` sin `native:compile` solo genera los hints; `native:compile` sin `process-aot` puede producir un binario que falla en runtime por hints incompletos.

## Tabla de elementos clave

Los elementos del flujo de compilación nativa que un profesional senior debe conocer son los siguientes.

| Elemento | Tipo | Descripción |
|---|---|---|
| `spring-boot:process-aot` | Goal Maven | Analiza el contexto Spring y genera hints de reflexión, recursos y proxies para GraalVM |
| `native:compile` | Goal GraalVM plugin | Compila bytecode + hints a ejecutable nativo |
| `native-maven-plugin` | Plugin Maven | Plugin de GraalVM que integra `native-image` en el ciclo de build Maven |
| `RuntimeHintsRegistrar` | Interface Spring AOT | Punto de extensión para registrar hints de reflexión adicionales de forma programática |
| `@ImportRuntimeHints` | Anotación Spring | Asocia un `RuntimeHintsRegistrar` a un bean o clase de configuración |
| `reflect-config.json` | Archivo AOT generado | Lista de clases y miembros accesibles vía reflexión en el ejecutable nativo |
| `--initialize-at-build-time` | Opción native-image | Clases que se inicializan en compilación en lugar de en arranque (mejora arranque) |
| Cold start nativo | Tiempo de arranque | 50-200 ms para ejecutable nativo vs 3-15 s para JAR con JVM en Lambda/GCF |
| Restricción AOT | Limitación | Reflexión dinámica, `Class.forName()` y proxies JDK sin hints fallan en el binario nativo |

## Buenas y malas prácticas

**Hacer:**
- Usar Java `record` para los POJOs de entrada/salida de las funciones: los records generan constructores y getters deterministas que GraalVM puede analizar sin hints adicionales, reduciendo el esfuerzo de configuración AOT.
- Ejecutar `mvn spring-boot:process-aot` en la pipeline CI antes del paso de compilación nativa para detectar incompatibilidades AOT temprano, sin necesidad de esperar los 5-10 minutos de compilación nativa completa.
- Añadir `RuntimeHintsRegistrar` para librerías de terceros que usen reflexión internamente (Jackson modules, validadores custom) en lugar de confiar en la detección automática, que puede fallar silenciosamente.
- Construir la imagen nativa dentro de un contenedor Docker con la imagen oficial de GraalVM para garantizar reproducibilidad del build entre máquinas de desarrollo y CI.

**Evitar:**
- Evitar usar `Class.forName()`, `Method.invoke()` o `Proxy.newProxyInstance()` sin registrar los tipos correspondientes en `RuntimeHintsRegistrar`; el binario nativo lanzará `ClassNotFoundException` en runtime, no en compilación.
- Evitar asumir que todos los starters de Spring Boot tienen soporte AOT completo: en Spring Cloud 2025.1.1 algunos módulos pueden requerir hints adicionales. Verificar en la documentación antes de asumir compatibilidad.
- Evitar compilar imágenes nativas en máquinas con menos de 8 GB de RAM disponible: GraalVM `native-image` requiere entre 4 y 8 GB para proyectos Spring típicos; la compilación puede fallar con `OutOfMemoryError` del compilador.
- Evitar imagen nativa para funciones que cambien frecuentemente (múltiples deploys al día): el ciclo de compilación nativa de 5-10 min alarga el ciclo de desarrollo. Reservar imagen nativa para funciones estables con requisitos de cold start estrictos.

---

← [12.5.2 Despliegue en Azure Functions y Google Cloud Functions](sc-function-cloud.md) | [Índice (README.md)](README.md) | [12.7 Integración con Spring Cloud Stream](sc-function-stream.md) →
