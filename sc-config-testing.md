# 1.9 Testing / Verificación de Spring Cloud Config

← [1.8 Operación del Config Server en producción](sc-config-operacion.md) | [Índice (README.md)](README.md) | [2.1 Arquitectura y rol de Eureka en el ecosistema de microservicios →](sc-eureka-arquitectura.md)

---

## Introducción

El módulo Spring Cloud Config introduce una dependencia de arranque en todos los microservicios que lo usan: sin el Config Server disponible, los clientes no arrancan (si `fail-fast: true`) o arrancan con configuración vacía (si `fail-fast: false`). Esto hace que los tests de integración que ignoran el Config Server fallen por razones ajenas al código de negocio.

La estrategia de testing del módulo tiene tres objetivos: verificar que el Config Server resuelve correctamente las propiedades (test del servidor), verificar que el Config Client carga la configuración esperada al arrancar (test del cliente), y verificar que el mecanismo de refresco funciona sin necesidad de un servidor real (test de `@RefreshScope`). Cada objetivo usa una estrategia diferente.

> [PREREQUISITO] Los tests de integración del Config Server con repositorio Git local embebido requieren `JGit` en el classpath de test, que ya viene incluido en `spring-cloud-config-server`.

## Estrategia 1: Test del Config Server con repositorio Git local

Esta estrategia despliega un Config Server real en el test con un repositorio Git local inicializado programáticamente. Verifica que el servidor resuelve correctamente las propiedades para una aplicación y perfil dados.

```xml
<!-- pom.xml — dependencias de test del Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
    <scope>test</scope>  <!-- solo en test si es un módulo cliente que prueba el servidor -->
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
package com.example.configserver;

import org.eclipse.jgit.api.Git;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.cloud.config.environment.Environment;
import org.springframework.test.context.TestPropertySource;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    classes = ConfigServerApplication.class,
    webEnvironment = WebEnvironment.RANDOM_PORT
)
@TestPropertySource(properties = {
    "spring.cloud.config.server.git.uri=file:${java.io.tmpdir}/test-config-repo",
    "spring.cloud.config.server.git.clone-on-start=true",
    "spring.cloud.config.server.git.default-label=main"
})
class ConfigServerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @BeforeAll
    static void initGitRepo() throws Exception {
        // Crear un repositorio Git local con ficheros de configuración de prueba
        Path repoPath = Path.of(System.getProperty("java.io.tmpdir"), "test-config-repo");
        Files.createDirectories(repoPath);

        try (Git git = Git.init().setDirectory(repoPath.toFile()).call()) {
            // Crear application.yml base
            File appYml = repoPath.resolve("application.yml").toFile();
            Files.writeString(appYml.toPath(), "global.timeout: 5000\n");

            // Crear order-service.yml
            File orderYml = repoPath.resolve("order-service.yml").toFile();
            Files.writeString(orderYml.toPath(),
                    "order.payment-url: http://payment:8080\norder.retries: 3\n");

            // Crear order-service-prod.yml
            File orderProdYml = repoPath.resolve("order-service-prod.yml").toFile();
            Files.writeString(orderProdYml.toPath(),
                    "order.payment-url: https://payment.prod:8443\norder.retries: 5\n");

            git.add().addFilepattern(".").call();
            git.commit()
               .setAuthor("test", "test@test.com")
               .setMessage("Initial config")
               .call();
        }
    }

    @Test
    void resolvesProdPropertiesWithCorrectPrecedence() {
        // Petición al Config Server para order-service en perfil prod
        Environment env = restTemplate.getForObject(
                "/order-service/prod/main",
                Environment.class
        );

        assertThat(env).isNotNull();
        assertThat(env.getPropertySources()).hasSize(3); // order-service-prod, order-service, application

        // La propiedad más específica (prod) tiene precedencia
        String paymentUrl = (String) env.getPropertySources().get(0)
                .getSource().get("order.payment-url");
        assertThat(paymentUrl).isEqualTo("https://payment.prod:8443");
    }

    @Test
    void returnsGlobalPropertyFromApplicationYml() {
        Environment env = restTemplate.getForObject(
                "/order-service/prod/main",
                Environment.class
        );

        // Buscar la propiedad global en el PropertySource de application.yml (el último)
        Object timeout = env.getPropertySources().stream()
                .flatMap(ps -> ps.getSource().entrySet().stream())
                .filter(e -> e.getKey().equals("global.timeout"))
                .map(e -> e.getValue())
                .findFirst()
                .orElse(null);

        assertThat(timeout).isEqualTo(5000);
    }
}
```

## Estrategia 2: Test del Config Client con backend native

Esta estrategia testea un microservicio Config Client usando el backend `native` del Config Server, con los ficheros de configuración en `src/test/resources`. No requiere Git ni servidor remoto.

```java
package com.example.orderservice;

import com.example.orderservice.config.OrderProperties;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@TestPropertySource(properties = {
    // Desactiva la importación del Config Server remoto en tests
    "spring.config.import=optional:configserver:",
    // Activa el backend native apuntando a src/test/resources/config/
    "spring.profiles.active=test"
})
class OrderServiceConfigTest {

    @Autowired
    private OrderProperties orderProperties;

    @Test
    void loadsPropertiesFromLocalConfig() {
        assertThat(orderProperties.getPaymentServiceUrl())
                .isEqualTo("http://payment-mock:8080");
        assertThat(orderProperties.getMaxRetries()).isEqualTo(1);
    }
}
```

Fichero `src/test/resources/application-test.yml`:

```yaml
order:
  payment-service-url: http://payment-mock:8080
  max-retries: 1
  async-processing: false
```

El prefijo `optional:` en `spring.config.import=optional:configserver:` indica a Spring Boot que si el Config Server no está disponible, continúe el arranque sin error. Es la clave para que los tests de microservicios clientes funcionen sin servidor.

## Estrategia 3: Test de @RefreshScope con ContextRefresher

Esta estrategia verifica que los beans anotados con `@RefreshScope` se actualizan correctamente cuando cambia el `Environment`.

```java
package com.example.orderservice;

import com.example.orderservice.config.OrderProperties;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.context.refresh.ContextRefresher;
import org.springframework.test.context.TestPropertySource;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@TestPropertySource(properties = {
    "spring.config.import=optional:configserver:",
    "order.payment-service-url=http://payment-original:8080",
    "order.max-retries=3"
})
class RefreshScopeTest {

    @Autowired
    private OrderProperties orderProperties;

    @Autowired
    private ContextRefresher contextRefresher;

    @Autowired
    private org.springframework.core.env.ConfigurableEnvironment environment;

    @Test
    void refreshScopeUpdatesPropertiesAfterEnvironmentChange() {
        // Estado inicial
        assertThat(orderProperties.getPaymentServiceUrl())
                .isEqualTo("http://payment-original:8080");

        // Modificar el Environment programáticamente (simula un cambio en el Config Server)
        org.springframework.core.env.MapPropertySource newProps =
                new org.springframework.core.env.MapPropertySource(
                        "test-override",
                        java.util.Map.of("order.payment-service-url", "http://payment-updated:9090")
                );
        environment.getPropertySources().addFirst(newProps);

        // Disparar el refresh — destruye y recrea los beans @RefreshScope
        Set<String> changedKeys = contextRefresher.refresh();

        // Verificar que la propiedad se actualizó
        assertThat(changedKeys).contains("order.payment-service-url");
        assertThat(orderProperties.getPaymentServiceUrl())
                .isEqualTo("http://payment-updated:9090");
    }
}
```

## Tabla resumen: estrategia de testing por situación

La siguiente tabla relaciona las situaciones típicas de testing con la estrategia recomendada.

| Situación | Estrategia recomendada |
|---|---|
| Verificar que el Config Server resuelve propiedades correctamente | Estrategia 1: `@SpringBootTest` del servidor con repositorio Git local |
| Testear un microservicio cliente sin servidor real | Estrategia 2: `optional:configserver:` + ficheros locales en `src/test/resources` |
| Verificar que `@RefreshScope` funciona tras un cambio de propiedades | Estrategia 3: `ContextRefresher` + `MapPropertySource` programático |
| Test de integración con Config Server real en CI/CD | Testcontainers con imagen del Config Server + repositorio Git embebido |
| Test del cifrado/descifrado de propiedades | `@SpringBootTest` del servidor con `encrypt.key` y llamada a `/encrypt` + `/decrypt` |

## Preguntas frecuentes de entrevista

Las siguientes preguntas cubren los puntos más evaluados sobre testing de Spring Cloud Config en entrevistas de nivel senior.

**¿Cómo evitas que los tests de un microservicio fallen porque el Config Server no está disponible?**
Usando `spring.config.import=optional:configserver:`. El prefijo `optional:` hace que Spring Boot ignore la ausencia del Config Server y continúe el arranque con los ficheros locales. Sin `optional:`, `fail-fast` en `false` produce el mismo efecto pero con un warning silencioso, mientras que con `fail-fast: true` el test fallará siempre que no haya servidor.

**¿Qué hace `ConfigServerTestUtils`?**
Es una clase de utilidad de `spring-cloud-config-server` que simplifica la inicialización de repositorios Git locales para tests. Proporciona métodos para crear un repo Git temporal, añadir ficheros y hacer commit, evitando el código JGit manual del ejemplo de la Estrategia 1. Disponible en `spring-cloud-config-server` en el classpath de test.

**¿Por qué un test con `@RefreshScope` puede pasar en local y fallar en CI?**
Porque el refresh destruye y recrea los beans, y el orden de recreación puede variar según la concurrencia del contexto de test. Si dos tests comparten el `ApplicationContext` (comportamiento por defecto de `@SpringBootTest`), un refresh en un test afecta al estado de los beans en el siguiente test. Solución: anotar el test con `@DirtiesContext(classMode = AFTER_EACH_TEST_METHOD)` para forzar la recreación del contexto, o aislar los tests de refresh en su propia clase.

**¿Cuál es la diferencia entre `@SpringBootTest` con el servidor real y el backend native para testear el Config Client?**
Con el servidor real (`@SpringBootTest` del servidor + cliente apuntando a él) se verifica el stack completo incluyendo la resolución HTTP y el JGit. Con `optional:configserver:` + ficheros locales se verifica solo que el cliente carga y enlaza las propiedades correctamente, sin la dependencia de red. La segunda estrategia es más rápida y apta para tests unitarios/componente; la primera es adecuada para tests de integración que verifican la resolución completa.

---

← [1.8 Operación del Config Server en producción](sc-config-operacion.md) | [Índice (README.md)](README.md) | [2.1 Arquitectura y rol de Eureka en el ecosistema de microservicios →](sc-eureka-arquitectura.md)
