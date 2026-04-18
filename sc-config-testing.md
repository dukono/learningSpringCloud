# 1.8 Testing / Verificación de Config Server

← [1.7 Operación y alta disponibilidad](sc-config-operacion.md) | [Índice](README.md) | [3.1 Cliente declarativo con OpenFeign](sc-feign-cliente-declarativo.md) →

---

## Introducción

El testing del Config Server consolida todos los conceptos del módulo. El principal problema que resuelven las estrategias de testing aquí descritas es la dependencia de infraestructura externa: los tests de integración no deben necesitar un repositorio Git real ni un Config Server en producción para ejecutarse. Spring Cloud Config proporciona el backend `native` como herramienta principal de testing, permitiendo que un Config Server embebido sirva ficheros desde el classpath de tests. Para los clientes, WireMock permite simular un Config Server sin levantarlo realmente.

> [CONCEPTO] El patrón de testing más importante del módulo: usar un `@SpringBootTest` que levanta el Config Server con perfil `native` y `search-locations=classpath:/config`, con los ficheros de configuración en `src/test/resources/config/`. Este patrón no requiere Git, no requiere red, y es reproducible en cualquier entorno CI/CD.

> [PREREQUISITO] Tener completo el conocimiento de los backends (1.3), la resolución de configuración (1.4) y el bootstrap/import del cliente (1.2) antes de estos tests.

## Estrategias de testing

Existen tres estrategias principales para testear la integración con Config Server, con diferentes trade-offs entre fidelidad y complejidad.

```
ESTRATEGIAS DE TESTING — COMPARACIÓN
═══════════════════════════════════════════════════════════════
  ESTRATEGIA 1: Config Server embebido (nativo)
  ┌─────────────────────────────────────────────┐
  │  @SpringBootTest + perfil native             │
  │  ├── Config Server embebido en el test       │
  │  ├── Ficheros en src/test/resources/config/  │
  │  └── Sin Git, sin red externa               │
  │  VENTAJA: máxima fidelidad, sin dependencias │
  │  USO: tests de integración del módulo        │
  └─────────────────────────────────────────────┘

  ESTRATEGIA 2: WireMock simula el Config Server
  ┌─────────────────────────────────────────────┐
  │  WireMock HTTP stub para /app/profile/label  │
  │  ├── Cliente Spring Boot apunta a WireMock   │
  │  ├── Sin Config Server real                  │
  │  └── Respuesta JSON personalizada por test   │
  │  VENTAJA: control total de respuestas        │
  │  USO: tests unitarios del cliente            │
  └─────────────────────────────────────────────┘

  ESTRATEGIA 3: Perfil "test" con override de importación
  ┌─────────────────────────────────────────────┐
  │  spring.config.import=optional:configserver: │
  │  ├── Importación opcional: si falla, continúa│
  │  ├── Valores por defecto en application.yml  │
  │  └── Sin Config Server en tests             │
  │  VENTAJA: más simple, no requiere stub       │
  │  USO: tests que no necesitan config externa  │
  └─────────────────────────────────────────────┘
```

## Ejemplo central — Config Server embebido con backend native

El siguiente ejemplo es el test de integración más completo: levanta un Config Server real con backend native y verifica que sirve la configuración correcta.

**Estructura de ficheros del test**:

```
src/
├── test/
│   ├── java/
│   │   └── com/example/configserver/
│   │       ├── ConfigServerIntegrationTest.java
│   │       └── ConfigClientIntegrationTest.java
│   └── resources/
│       └── config/
│           ├── application.yml              ← Config global de test
│           ├── order-service.yml            ← Config de test para order-service
│           └── order-service-test.yml       ← Config de test para perfil test
```

**src/test/resources/config/application.yml**:

```yaml
app:
  name: "Test Application"
  timeout: 5
  feature-flag: false
```

**src/test/resources/config/order-service.yml**:

```yaml
app:
  name: "Order Service Test"
  max-orders: 10
  payment-url: "http://mock-payment:9090"
```

**ConfigServerIntegrationTest.java — testa el servidor directamente**:

```java
package com.example.configserver;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    webEnvironment = WebEnvironment.RANDOM_PORT,
    properties = {
        "spring.profiles.active=native",
        "spring.cloud.config.server.native.search-locations=classpath:/config"
    }
)
@ActiveProfiles("native")
class ConfigServerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldServeOrderServiceConfig() {
        // El servidor debe resolver /order-service/default
        var response = restTemplate.getForObject(
            "/order-service/default",
            String.class
        );

        assertThat(response)
            .contains("order-service")
            .contains("max-orders");
    }

    @Test
    void shouldFusionGlobalAndServiceSpecificConfig() {
        // Verifica que application.yml y order-service.yml se fusionan
        var response = restTemplate.getForObject(
            "/order-service/default",
            java.util.Map.class
        );

        @SuppressWarnings("unchecked")
        var propertySources = (java.util.List<java.util.Map<String, Object>>)
            response.get("propertySources");

        // Debe haber al menos dos PropertySources: order-service.yml y application.yml
        assertThat(propertySources).hasSizeGreaterThanOrEqualTo(2);
    }
}
```

**ConfigClientIntegrationTest.java — testa el cliente contra servidor embebido**:

```java
package com.example.configserver;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    classes = { ConfigServerApplication.class },
    webEnvironment = WebEnvironment.RANDOM_PORT,
    properties = {
        "spring.profiles.active=native",
        "spring.cloud.config.server.native.search-locations=classpath:/config",
        "spring.application.name=order-service"
    }
)
class ConfigClientIntegrationTest {

    @Value("${app.max-orders}")
    private int maxOrders;

    @Value("${app.name}")
    private String appName;

    @Test
    void shouldInjectPropertiesFromNativeBackend() {
        assertThat(maxOrders).isEqualTo(10);
        assertThat(appName).isEqualTo("Order Service Test");
    }
}
```

## Test de cliente con WireMock

Cuando se testea solo el cliente (sin querer levantar un Config Server), WireMock simula las respuestas del servidor con JSON predefinido.

**WireMockConfigClientTest.java**:

```java
package com.example.orderservice;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(
    webEnvironment = WebEnvironment.NONE,
    properties = {
        "spring.config.import=configserver:http://localhost:8099",
        "spring.application.name=order-service"
    }
)
class WireMockConfigClientTest {

    static WireMockServer wireMock = new WireMockServer(
        WireMockConfiguration.options().port(8099)
    );

    @BeforeAll
    static void startWireMock() {
        wireMock.start();
        wireMock.stubFor(get(urlEqualTo("/order-service/default"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                      "name": "order-service",
                      "profiles": ["default"],
                      "label": null,
                      "propertySources": [
                        {
                          "name": "classpath:/config/order-service.yml",
                          "source": {
                            "app.max-orders": 99,
                            "app.name": "Order Service WireMock"
                          }
                        }
                      ]
                    }
                    """)
            )
        );
    }

    @AfterAll
    static void stopWireMock() {
        wireMock.stop();
    }

    @Value("${app.max-orders}")
    private int maxOrders;

    @Test
    void shouldUseConfigFromWireMock() {
        assertThat(maxOrders).isEqualTo(99);
    }
}
```

## Tabla de estrategias de testing

| Estrategia | Complejidad | Fidelidad | Dependencias externas | Recomendado para |
|------------|-------------|-----------|----------------------|------------------|
| Config Server embebido (native) | Media | Alta | Ninguna | Tests de integración del módulo Config |
| WireMock | Baja | Media | WireMock library | Tests unitarios del cliente |
| `optional:configserver:` | Muy baja | Baja | Ninguna | Tests que no validan config externa |
| TestContainers | Alta | Máxima | Docker | Tests E2E de entorno completo |

## Buenas y malas prácticas

Hacer:
- Usar `spring.cloud.config.server.native.search-locations=classpath:/config` en todos los tests de integración del Config Server; es rápido y reproducible.
- Tener un fichero `application.yml` de test en `src/test/resources/config/` con valores seguros y controlados (sin credenciales reales).
- Añadir `spring.cloud.config.fail-fast=false` en tests que no necesitan el Config Server; evita fallos de test por el servidor no disponible.
- Usar `spring.config.import=optional:configserver:` en tests de lógica de negocio para desacoplarlos completamente del Config Server.

Evitar:
- Apuntar tests de integración al Config Server de producción o staging; los cambios en el repositorio de configuración romperían los tests de forma no determinista.
- Usar `@SpringBootTest` sin restringir el classpath en tests de módulo pequeño; el tiempo de arranque se dispara.
- Hardcodear valores de configuración esperados en tests sin extraerlos de las mismas fuentes que usa la producción.
- Ignorar el perfil `native` en favor de mocks parciales; el servidor embebido native es más fiel al comportamiento real.

## Verificación y práctica

```bash
# Ejecutar solo los tests del módulo Config
mvn test -pl config-server -Dtest="*ConfigServer*"

# Verificar cobertura de tests de integración
mvn verify -Pintegration-tests

# Ver los PropertySources que entrega el servidor en test
curl http://localhost:8888/order-service/test/main
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Qué perfil de Spring activa el backend native en el Config Server? ¿Cómo se configura la ruta de búsqueda de ficheros?
2. ¿Por qué se recomienda usar `classpath:/config` como `search-locations` en lugar de una ruta absoluta del sistema de archivos en tests?
3. ¿Qué hace `spring.config.import=optional:configserver:`? ¿En qué escenario de test es útil?
4. Un test de integración levanta un Config Server embebido con backend native, pero los beans del cliente no reciben las propiedades de test. ¿Cuáles son las causas más probables?
5. ¿Cómo se puede verificar desde un test que un bean anotado con `@RefreshScope` actualiza sus valores después de un refresh?

---

← [1.7 Operación y alta disponibilidad](sc-config-operacion.md) | [Índice](README.md) | [3.1 Cliente declarativo con OpenFeign](sc-feign-cliente-declarativo.md) →
