# 7.8 Spring Cloud Bus — Endpoint busenv y cambio en caliente

← [7.7 Spring Cloud Bus — Seguridad de endpoints del Bus](sc-bus-seguridad-endpoint.md) | [Índice](README.md) | [7.9 Spring Cloud Bus — Testing y Troubleshooting](sc-bus-testing.md) →

---

## Introducción

El endpoint `POST /actuator/bus-env` permite modificar propiedades del `Environment` de Spring en todos los nodos del Bus simultáneamente, sin reiniciar ningún servicio. Es el equivalente distribuido del endpoint local `/actuator/env` de Spring Boot Actuator. La diferencia clave con `/actuator/bus-refresh` es que `bus-env` inyecta valores directamente en el `Environment` sin consultar al Config Server, mientras que `bus-refresh` recarga el `Environment` desde el Config Server.

> [CONCEPTO] `POST /actuator/bus-env` modifica el `Environment` en memoria de cada nodo, pero esos cambios no son persistentes: si la instancia se reinicia, la propiedad volverá al valor original del Config Server o del `application.yml`. Para cambios permanentes, hay que modificar el repositorio de configuración y usar `bus-refresh`.

## BusEnvironmentManager y BusEnvironmentManagerEndpoint

El endpoint `bus-env` está implementado sobre dos componentes del Bus:

`BusEnvironmentManagerEndpoint` es el endpoint de Actuator que recibe las peticiones HTTP `POST /actuator/bus-env`. Valida el payload JSON y publica un `EnvironmentChangeRemoteApplicationEvent` en el Bus.

`BusEnvironmentManager` es el componente que procesa el `EnvironmentChangeRemoteApplicationEvent` cuando llega a cada nodo. Inyecta la propiedad en el `Environment` local usando un `MapPropertySource` con nombre `manager` que tiene alta prioridad.

El flujo completo del `bus-env` es:

```mermaid
flowchart TD
    HTTP["POST /actuator/bus-env\n{\"name\": \"app.timeout\", \"value\": \"5000\"}"]
    EP["BusEnvironmentManagerEndpoint\nsetProperty(name, value, destination)"]
    PUB["Publica\nEnvironmentChangeRemoteApplicationEvent"]
    BROKER[("Broker\nspringCloudBus")]
    ALL["Todos los nodos\nreciben el evento"]
    MGR["BusEnvironmentManager\nsetEnvironment(name, value)"]
    PS[["PropertySource 'manager'\nactualizad en Environment local"]]
    RS{{"¿Bean tiene @RefreshScope?"}}
    REFRESHED["Bean recrea en próximo acceso\n(valor actualizado)"]
    NOTREFRESHED>Bean NO se recrea\n(aún usa valor anterior)]

    HTTP --> EP --> PUB --> BROKER --> ALL --> MGR --> PS --> RS
    RS -->|sí + bus-refresh| REFRESHED
    RS -->|no / sin bus-refresh| NOTREFRESHED

    classDef root      fill:#1f2328,color:#fff,stroke:#444,font-weight:bold
    classDef primary   fill:#0969da,color:#fff,stroke:#0550ae
    classDef secondary fill:#2da44e,color:#fff,stroke:#1a7f37
    classDef danger    fill:#cf222e,color:#fff,stroke:#a40e26
    classDef warning   fill:#9a6700,color:#fff,stroke:#7d4e00
    classDef storage   fill:#6e40c9,color:#fff,stroke:#5a32a3
    classDef neutral   fill:#e6edf3,color:#1f2328,stroke:#d0d7de

    class HTTP primary
    class EP primary
    class PUB primary
    class BROKER storage
    class ALL neutral
    class MGR primary
    class PS storage
    class RS warning
    class REFRESHED secondary
    class NOTREFRESHED danger
```
*El `bus-env` actualiza el `PropertySource "manager"` en todos los nodos, pero los beans `@Value` sin `@RefreshScope` no se recrean hasta que se dispara también un `bus-refresh`.*

## Ejemplo central — Uso completo del endpoint bus-env

El siguiente ejemplo muestra cómo configurar, invocar y verificar el endpoint `bus-env`. Incluye la configuración YAML, ejemplos de curl y un controller para verificar que la propiedad cambió.

```yaml
# application.yml — habilitar bus-env
spring:
  application:
    name: order-service
  rabbitmq:
    host: localhost
    port: 5672
  cloud:
    bus:
      enabled: true
      env:
        enabled: true    # Habilitar explícitamente bus-env

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env, health, env
  endpoint:
    bus-env:
      enabled: true
    env:
      enabled: true
      show-values: always    # Mostrar valores en /actuator/env para verificación
```

```java
// FeatureFlagConfig.java — bean que consume la propiedad modificable vía bus-env
package com.example.orderservice.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Component
@RefreshScope
public class FeatureFlagConfig {

    @Value("${feature.new-checkout:false}")
    private boolean newCheckoutEnabled;

    @Value("${app.timeout:3000}")
    private int timeoutMs;

    public boolean isNewCheckoutEnabled() {
        return newCheckoutEnabled;
    }

    public int getTimeoutMs() {
        return timeoutMs;
    }
}
```

```java
// FeatureFlagController.java — expone los valores actuales del Environment
package com.example.orderservice.controller;

import com.example.orderservice.config.FeatureFlagConfig;
import org.springframework.core.env.Environment;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class FeatureFlagController {

    private final FeatureFlagConfig featureFlagConfig;
    private final Environment environment;

    public FeatureFlagController(FeatureFlagConfig featureFlagConfig,
                                  Environment environment) {
        this.featureFlagConfig = featureFlagConfig;
        this.environment = environment;
    }

    @GetMapping("/flags")
    public Map<String, Object> getFlags() {
        return Map.of(
            "newCheckout", featureFlagConfig.isNewCheckoutEnabled(),
            "timeout", featureFlagConfig.getTimeoutMs(),
            // Valor directo del Environment (no requiere @RefreshScope)
            "timeoutDirect", environment.getProperty("app.timeout", "3000")
        );
    }
}
```

El uso del endpoint desde la línea de comandos es el siguiente:

```bash
# 1. Verificar el valor actual
curl http://localhost:8080/flags
# → {"newCheckout":false,"timeout":3000,"timeoutDirect":"3000"}

# 2. Cambiar la propiedad en TODOS los nodos vía bus-env
curl -X POST http://localhost:8080/actuator/bus-env \
     -H "Content-Type: application/json" \
     -d '{"name":"feature.new-checkout","value":"true"}'

# 3. Para que los beans @RefreshScope lean el nuevo valor,
#    también es necesario disparar un bus-refresh
curl -X POST http://localhost:8080/actuator/bus-refresh

# 4. Verificar el cambio en ambos nodos
curl http://localhost:8080/flags
# → {"newCheckout":true,"timeout":3000,"timeoutDirect":"3000"}
curl http://localhost:8081/flags
# → {"newCheckout":true,"timeout":3000,"timeoutDirect":"3000"}

# 5. Cambiar solo en un servicio específico usando destination
curl -X POST "http://localhost:8080/actuator/bus-env/order-service:**:**" \
     -H "Content-Type: application/json" \
     -d '{"name":"app.timeout","value":"5000"}'
```

> [ADVERTENCIA] Cambiar una propiedad con `bus-env` actualiza el `Environment` en todos los nodos, pero los beans `@Value` inyectados en beans NO `@RefreshScope` NO se actualizan automáticamente. Para que el cambio sea visible en beans con `@Value`, es necesario que esos beans estén anotados con `@RefreshScope` y también disparar un `bus-refresh` después.

## Diferencia entre bus-env y bus-refresh

Comprender la diferencia entre estos dos endpoints es fundamental para el examen de certificación.

| Aspecto | `POST /actuator/bus-env` | `POST /actuator/bus-refresh` |
|---------|--------------------------|------------------------------|
| Fuente del nuevo valor | Payload del request (`{"name","value"}`) | Config Server (repositorio Git u otro backend) |
| Persistencia | Solo en memoria; se pierde al reiniciar | Depende del Config Server; consistente con el repositorio |
| Evento publicado | `EnvironmentChangeRemoteApplicationEvent` | `BusRefreshEvent` |
| Beans @RefreshScope | No se recrean automáticamente | Se recrean automáticamente |
| Caso de uso típico | Cambios temporales de flags/tuning en caliente | Propagación de cambios del repositorio de configuración |

## Tabla de elementos clave

Los componentes del endpoint `bus-env` y sus roles son:

| Componente | Descripción |
|------------|-------------|
| `BusEnvironmentManagerEndpoint` | Endpoint Actuator que recibe `POST /actuator/bus-env` |
| `BusEnvironmentManager` | Procesa el evento e inyecta la propiedad en el Environment local |
| `EnvironmentChangeRemoteApplicationEvent` | Evento del Bus que transporta el cambio de propiedad al broker |
| `MapPropertySource` `"manager"` | PropertySource de alta prioridad donde se almacena el cambio |

## Buenas y malas prácticas

**Buenas prácticas:**

- Usar `bus-env` solo para cambios temporales de ajuste (timeouts, flags de feature) que no necesitan persistencia entre reinicios.
- Combinar `bus-env` con un posterior `bus-refresh` cuando se necesita que los beans `@RefreshScope` lean el nuevo valor inmediatamente.
- Proteger `/actuator/bus-env` con autenticación estricta, ya que permite modificar cualquier propiedad del `Environment` en todos los nodos.

**Malas prácticas:**

- Usar `bus-env` para cambios de configuración crítica (credenciales de base de datos, URLs de servicios) en lugar de actualizar el Config Server. Los cambios no son persistentes.
- Ignorar que `bus-env` no recrea beans `@RefreshScope` automáticamente. El valor en el `Environment` cambia, pero los beans que ya están instanciados siguen usando el valor anterior.
- Usar `bus-env` sin entender su efecto en el `PropertySource` "manager". Este PropertySource tiene prioridad sobre los demás en el `Environment`.

## Verificación y práctica

> [EXAMEN] **1.** ¿Cuál es el payload JSON que se envía a `POST /actuator/bus-env` para cambiar la propiedad `app.debug` a `true`?

> [EXAMEN] **2.** ¿Qué evento publica el Bus cuando se invoca `POST /actuator/bus-env`?

> [EXAMEN] **3.** ¿Un cambio realizado con `bus-env` persiste si la instancia se reinicia? ¿Por qué?

> [EXAMEN] **4.** ¿Es suficiente llamar a `bus-env` para que los beans con `@Value` y `@RefreshScope` lean el nuevo valor?

> [EXAMEN] **5.** ¿Cuál es la diferencia principal entre `bus-env` y `bus-refresh` en cuanto a la fuente del nuevo valor de la propiedad?

---

← [7.7 Spring Cloud Bus — Seguridad de endpoints del Bus](sc-bus-seguridad-endpoint.md) | [Índice](README.md) | [7.9 Spring Cloud Bus — Testing y Troubleshooting](sc-bus-testing.md) →
