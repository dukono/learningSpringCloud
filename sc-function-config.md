# 12.2 Configuración de funciones y exposición HTTP

← [12.1 Modelo de programación funcional](sc-function-modelo.md) | [Índice (README.md)](README.md) | [12.3 Manejo de tipos, inferencia y serialización](sc-function-tipos.md) →

---

## Introducción

Una vez definidas las funciones como beans, el problema operativo es controlar cuál de ellas está activa, cómo se componen, cuáles no deben exponerse y cómo se mapean a endpoints HTTP. Sin configuración explícita, el framework lanza `IllegalStateException` cuando detecta más de un bean funcional elegible y no hay `spring.cloud.function.definition` configurado. Este fichero cubre las propiedades `spring.cloud.function.*` que gobiernan ese ciclo de activación, composición y exclusión, así como el starter `spring-cloud-function-web` que añade la exposición HTTP automática sobre Spring MVC o WebFlux.

El módulo de configuración HTTP se necesita en servicios que deben exponer funciones como REST sin escribir controladores: el path `/nombre-de-funcion` se genera automáticamente a partir del nombre del bean, y el método HTTP (GET para `Supplier`, POST para `Function` y `Consumer`) se deduce del contrato.

> **[PREREQUISITO]** Este fichero presupone conocimiento de `sc-function-modelo.md` (12.1): el lector debe saber qué es `FunctionCatalog` y cómo se registran los beans funcionales antes de configurar cuál está activo.

> **[CONCEPTO]** `spring-cloud-function-web` delega el routing HTTP a Spring MVC o WebFlux según el starter presente en el classpath. La configuración avanzada de MVC/WebFlux está fuera del alcance de este módulo; consultar la documentación oficial de Spring Framework.

## Representación visual

El flujo de activación de una función parte de la propiedad `spring.cloud.function.definition` y termina en el endpoint HTTP o en el adaptador de mensajería. El siguiente diagrama muestra las capas implicadas.

```
  application.yml
  spring.cloud.function.definition: trim|uppercase
                │
                ▼
  FunctionCatalog.lookup("trim|uppercase")
  ┌──────────────────────────────────────────┐
  │  Bean "trim": Function<String,String>    │
  │  Bean "uppercase": Function<String,String>│
  │  Composed: trim → uppercase              │
  └───────────────────┬──────────────────────┘
                      │
          spring-cloud-function-web
                      │
          ┌───────────┴──────────────┐
          ▼                          ▼
  POST /trim|uppercase          GET /greeter
  (Function/Consumer)           (Supplier)
  Spring MVC / WebFlux
```

La exclusión de beans se configura con `spring.cloud.function.ineligible-definitions`, que evita que beans de infraestructura expuestos como `Function` (p. ej., convertidores internos) sean registrados en el catálogo.

## Ejemplo central

El ejemplo configura tres funciones, una composición declarativa y la exclusión de un bean interno. La clase `InternalConverter` simula un bean funcional que no debe exponerse vía HTTP.

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
</dependencies>
```

### application.yml

```yaml
spring:
  cloud:
    function:
      # Activa la composición trim → uppercase como función por defecto
      definition: trim|uppercase
      # Excluye el bean "internalConverter" del catálogo HTTP
      ineligible-definitions: internalConverter
```

### Código Java completo

```java
package com.example.scfunction.config;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

@SpringBootApplication
public class FunctionConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(FunctionConfigApplication.class, args);
    }

    // Función 1: elimina espacios
    @Bean
    public Function<String, String> trim() {
        return String::trim;
    }

    // Función 2: convierte a mayúsculas
    @Bean
    public Function<String, String> uppercase() {
        return String::toUpperCase;
    }

    // Supplier expuesto como GET /greeter
    @Bean
    public Supplier<String> greeter() {
        return () -> "Hello from Spring Cloud Function!";
    }

    // Consumer expuesto como POST /auditLogger (204 No Content)
    @Bean
    public Consumer<String> auditLogger() {
        return msg -> System.out.println("[AUDIT] " + msg);
    }

    // Bean funcional excluido del catálogo via ineligible-definitions
    @Bean
    public Function<String, String> internalConverter() {
        return s -> s.replace("-", "_");
    }
}
```

Con esta configuración, los endpoints disponibles son:
- `POST /trim|uppercase` → acepta texto plano, devuelve texto en mayúsculas sin espacios
- `GET /greeter` → devuelve `"Hello from Spring Cloud Function!"`
- `POST /auditLogger` → acepta texto plano, devuelve 204
- `/internalConverter` → **no disponible** (excluido por `ineligible-definitions`)

Para invocar la función compuesta:
```bash
curl -X POST http://localhost:8080/trim|uppercase \
  -H "Content-Type: text/plain" \
  -d "  hello world  "
# Respuesta: HELLO WORLD
```

> **[EXAMEN]** Cuando hay más de un bean `Function`/`Consumer`/`Supplier` en el contexto y `spring.cloud.function.definition` no está configurado, Spring Cloud Function lanza `IllegalStateException` en el arranque con el mensaje "Multiple functions detected". Esta es una pregunta frecuente en entrevistas sobre el comportamiento del framework ante ambigüedad de beans.

## Tabla de elementos clave

Las propiedades de configuración que un profesional senior debe conocer de memoria se recogen en la siguiente tabla.

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `spring.cloud.function.definition` | `String` | `null` | Nombre del bean activo o composición `a\|b\|c`; obligatorio con múltiples beans |
| `spring.cloud.function.ineligible-definitions` | `String` (lista CSV) | `null` | Beans excluidos del catálogo; evita exponer funciones de infraestructura |
| `spring.cloud.function.routing-expression` | `String` (SpEL) | `null` | Expresión SpEL para routing dinámico; activa `RoutingFunction` |
| `spring.cloud.function.web.path` | `String` | `/` | Prefijo base de los endpoints HTTP generados |
| `spring.cloud.function.compile.timeout` | `Duration` | — | Timeout para compilación de funciones registradas programáticamente |
| `management.endpoints.web.exposure.include` | `String` | `health` | Debe incluir `functions` para exponer el endpoint `/actuator/functions` |

## Buenas y malas prácticas

**Hacer:**
- Especificar siempre `spring.cloud.function.definition` en `application.yml` cuando el servicio tiene más de un bean funcional; resolver la ambigüedad en configuración, no en código.
- Usar `spring.cloud.function.ineligible-definitions` para excluir funciones utilitarias internas (convertidores, validadores) que no deben ser alcanzables vía HTTP.
- Separar las funciones de negocio en una clase `@Configuration` propia para que sean detectables en test sin levantar el contexto web completo.
- Configurar `spring.cloud.function.web.path=/api/fn` en producción para diferenciar los endpoints de función de los endpoints del dominio y evitar colisiones de path.

**Evitar:**
- Evitar confiar en la resolución automática cuando hay un solo bean funcional en proyectos que con el tiempo crecerán: añadir `spring.cloud.function.definition` desde el inicio para que el comportamiento sea explícito y no cambie al agregar nuevos beans.
- Evitar usar el nombre `function` o `handler` para los beans funcionales: colisionan con nombres internos del framework y pueden producir comportamientos inesperados en el catálogo.
- Evitar exponer todas las funciones sin revisar `ineligible-definitions` en servicios con beans de infraestructura que implementen interfaces funcionales (convertidores de mensajes, transformadores de serialización).
- Evitar confiar en el path por defecto (`/`) en entornos con API Gateway o Service Mesh: el prefijo vacío dificulta el enrutamiento por prefijo y los health checks.

## Comparación: exposición HTTP vs adaptador de mensajería

Spring Cloud Function soporta dos modos de exposición que coexisten en el mismo artefacto. La diferencia no está en el bean funcional sino en el starter presente en el classpath.

| Aspecto | HTTP (`spring-cloud-function-web`) | Stream (`spring-cloud-stream` + binder) |
|---|---|---|
| Starter requerido | `spring-cloud-function-web` + `spring-boot-starter-web/webflux` | `spring-cloud-stream` + binder Kafka/Rabbit |
| Protocolo | HTTP REST (MVC o WebFlux) | Mensajería asíncrona |
| Método HTTP | GET (Supplier), POST (Function/Consumer) | N/A |
| Path generado | `/<nombre-función>` | N/A — binding `<nombre>-in-0` |
| `definition` requerido | Con múltiples beans | Con múltiples beans |
| Coexistencia | Sí — ambos starters pueden estar activos | Sí — mismo bean funcional es invocado por ambos |

---

← [12.1 Modelo de programación funcional](sc-function-modelo.md) | [Índice (README.md)](README.md) | [12.3 Manejo de tipos, inferencia y serialización](sc-function-tipos.md) →
