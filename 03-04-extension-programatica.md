# 3.4.15 Extensión Programática: Config Client

← [3.4 Config Client](./03-04-config-client.md) | [Índice](./README.md) | [3.5 Refresco →](./03-05-config-refresh.md)

---

## Puntos de extensión del Config Client

La capa YAML del Config Client configura qué servidor usar, el perfil activo y los reintentos. Pero no permite añadir fuentes de configuración adicionales a las predefinidas, ni modificar cómo el cliente construye las peticiones HTTP al Config Server. Los casos que requieren Java son: añadir una fuente de propiedades de un sistema interno junto al Config Server (sin reemplazarlo), inyectar headers de autenticación propietarios en las peticiones del cliente, o diagnosticar programáticamente qué configuración cargó el cliente y desde qué fuente. Todos estos casos usan la capa `PropertySourceLocator`, que Spring Cloud invoca durante la fase de bootstrap — antes de que el contexto principal de Spring Boot se inicialice.

| Interface / Clase | Para qué |
|---|---|
| `PropertySourceLocator` | Añadir una fuente de propiedades custom en el arranque del cliente |
| `ConfigServicePropertySourceLocator` | Extender el localizador por defecto (añadir headers, lógica custom) |
| `ConfigClientProperties` | Acceder y modificar la configuración del cliente programáticamente |
| `BootstrapConfiguration` | Registrar beans que se crean antes del contexto principal |

---

## 3.4.15.1 Ciclo de vida Bootstrap con `PropertySourceLocator`

```
Spring Boot arranque — fase Bootstrap (antes del contexto principal)
  │
  ├── 1. Se crea el Bootstrap ApplicationContext
  │        Lee META-INF/spring/...BootstrapConfiguration.imports
  │
  ├── 2. Se instancian los BootstrapConfiguration beans
  │
  ├── 3. Se invoca PropertySourceLocator.locate(environment) para CADA locator registrado
  │        ├── ConfigServicePropertySourceLocator (built-in) → llama al Config Server HTTP
  │        └── InternalApiPropertySourceLocator (custom) → llama a API interna
  │
  ├── 4. Las PropertySources resultantes se añaden al Environment principal
  │        Orden: primero los locators de mayor precedencia
  │
  └── 5. Bootstrap Context cierra — el Environment enriquecido pasa al contexto principal

Spring Boot arranque — contexto principal
  │
  ├── 6. @Value, @ConfigurationProperties se resuelven con las propiedades del paso 4
  └── 7. Beans de la aplicación se crean y se inyectan las propiedades
```

> Si un `PropertySourceLocator` no está registrado en `META-INF/...BootstrapConfiguration.imports`, no se invoca en el paso 3 — el bean existe en el contexto pero nunca aporta propiedades al `Environment`. Ver antipatrones al final del fichero.

---

## 3.4.15.2 `PropertySourceLocator` completamente custom

`PropertySourceLocator` es la interfaz que Spring Cloud Config usa internamente para conectar con el Config Server. Implementarla directamente permite añadir una fuente de propiedades de cualquier origen (API interna, base de datos, servicio de configuración corporativo) que se comporta exactamente igual que el Config Server: se carga en el arranque, participa en la fusión de propiedades, y se refresca con `@RefreshScope`.

```java
@Component
public class InternalApiPropertySourceLocator implements PropertySourceLocator {

    private final RestTemplate restTemplate;
    private final String apiUrl;

    public InternalApiPropertySourceLocator(RestTemplate restTemplate,
                                             @Value("${config.internal.api-url}") String apiUrl) {
        this.restTemplate = restTemplate;
        this.apiUrl = apiUrl;
    }

    @Override
    public PropertySource<?> locate(Environment environment) {
        String appName = environment.getProperty("spring.application.name");
        String profile = environment.getProperty("spring.profiles.active", "default");

        try {
            // Llamar a la API interna con los parámetros del servicio
            Map<String, Object> props = restTemplate.getForObject(
                apiUrl + "/config/{app}/{profile}",
                Map.class, appName, profile
            );

            if (props == null || props.isEmpty()) {
                return new MapPropertySource("internal-api-empty", Map.of());
            }

            return new MapPropertySource("internal-api:" + appName + "-" + profile, props);

        } catch (RestClientException e) {
            log.warn("API interna no disponible — usando solo Config Server: {}", e.getMessage());
            // Devolver fuente vacía — no falla el arranque
            return new MapPropertySource("internal-api-unavailable", Map.of());
        }
    }
}
```

```
# Registrar en META-INF/spring/org.springframework.cloud.bootstrap.BootstrapConfiguration.imports
# (Spring Boot 3.x) o en META-INF/spring.factories (Spring Boot 2.x)
com.miempresa.config.InternalApiPropertySourceLocator
```

> `PropertySourceLocator.locate()` se invoca durante el arranque del contexto Bootstrap, antes de que los beans de la aplicación se inicialicen. Por eso **no puede inyectar beans de la aplicación** — solo beans del contexto Bootstrap. Usar `RestTemplate` construido localmente, no el autoconfigurdo por la aplicación.

---

## 3.4.15.3 `ConfigServicePropertySourceLocator` extendido

Para casos donde se necesita modificar cómo el cliente se comunica con el Config Server (añadir cabeceras de autenticación custom, manipular la petición o respuesta, añadir reintentos con lógica propia), se puede extender el localizador por defecto:

```java
@Component
@Primary   // reemplaza el ConfigServicePropertySourceLocator autoconfigurdo
public class AuthenticatedConfigLocator extends ConfigServicePropertySourceLocator {

    private final TokenService tokenService;

    public AuthenticatedConfigLocator(ConfigClientProperties clientProperties,
                                       TokenService tokenService) {
        super(clientProperties);
        this.tokenService = tokenService;
    }

    @Override
    public PropertySource<?> locate(Environment environment) {
        // Añadir el token de autenticación antes de consultar al Config Server
        // El token se obtiene de un servicio de identidad propio (no Spring Security)
        String token = tokenService.getServiceToken("config-client");

        // Inyectar el token en las cabeceras del cliente HTTP
        // ConfigServicePropertySourceLocator expone el RestTemplate para este propósito
        this.setRestTemplate(buildAuthenticatedRestTemplate(token));

        return super.locate(environment);
    }

    private RestTemplate buildAuthenticatedRestTemplate(String token) {
        RestTemplate rt = new RestTemplate();
        rt.setInterceptors(List.of((request, body, execution) -> {
            request.getHeaders().setBearerAuth(token);
            return execution.execute(request, body);
        }));
        return rt;
    }
}
```

---

## 3.4.15.4 `ConfigClientProperties` programático

`ConfigClientProperties` es el bean que contiene toda la configuración del cliente (`spring.cloud.config.*`). Inyectarlo permite leer o modificar la configuración del cliente en runtime desde código Java:

```java
@Service
public class ConfigClientDiagnosticService {

    private final ConfigClientProperties clientProps;

    public ConfigClientDiagnosticService(ConfigClientProperties clientProps) {
        this.clientProps = clientProps;
    }

    public String getConfigServerUrl() {
        return clientProps.getUri()[0];   // URL del Config Server
    }

    public String getAppName() {
        return clientProps.getName();     // spring.cloud.config.name
    }

    public String getActiveLabel() {
        return clientProps.getLabel();    // rama/tag activo
    }

    // Cambiar dinámicamente el label (rama) sin reiniciar
    // (efectivo en el próximo refresco de contexto)
    public void switchToBranch(String branch) {
        clientProps.setLabel(branch);
        log.info("Config Client ahora apunta a la rama: {}", branch);
    }
}
```

---

## 3.4.15.5 `BootstrapConfiguration`

Cuando se necesita que un bean exista **antes** de que Spring procese cualquier `@Value` o `@ConfigurationProperties`, hay que registrarlo como `BootstrapConfiguration`. El contexto Bootstrap es el que carga `PropertySourceLocator` y tiene acceso al `Environment` sin propiedades de la aplicación aún cargadas.

```java
// Bean que debe existir en el contexto Bootstrap (antes que todo)
public class EarlySecretResolverBootstrap {

    @Bean
    public PropertySourceLocator earlySecretsLocator() {
        return environment -> {
            // Resolver secretos de arranque antes de que cualquier @Value se procese
            String dbPassword = resolveFromHsm("db.password");
            String apiKey = resolveFromHsm("external.api.key");

            return new MapPropertySource("early-secrets", Map.of(
                "spring.datasource.password", dbPassword,
                "external.api.key", apiKey
            ));
        };
    }

    private String resolveFromHsm(String keyName) {
        // Llamada al HSM o gestor de secretos en tiempo de bootstrap
        return HsmClient.getInstance().getSecret(keyName);
    }
}
```

```
# Registrar como BootstrapConfiguration (Spring Boot 3.x):
# META-INF/spring/org.springframework.cloud.bootstrap.BootstrapConfiguration.imports
com.miempresa.config.EarlySecretResolverBootstrap
```

> La diferencia con un `@Component` normal: el contexto Bootstrap se inicializa **antes** de que Spring Boot cargue cualquier `application.yml` o consulte al Config Server. Beans registrados aquí tienen acceso al `Environment` vacío (sin propiedades de la aplicación) pero se ejecutan antes que cualquier otra fuente de configuración.

---

## 3.4.15.6 Antipatrones

> **[ADVERTENCIA]** Implementar `PropertySourceLocator` como un `@Component` sin registrarlo en `META-INF/spring/org.springframework.cloud.bootstrap.BootstrapConfiguration.imports`. Spring Cloud no invoca el locator durante el bootstrap — el bean existe en el contexto pero nunca se llama a `locate()`. El servicio arranca sin errores, pero las propiedades del locator están ausentes. El síntoma: `@Value` devuelve el valor por defecto en lugar del valor esperado, y no hay ningún mensaje de error.

> **[ADVERTENCIA]** Sobreescribir `ConfigServicePropertySourceLocator.locate()` sin llamar a `super.locate()`. El método padre es el que establece la conexión HTTP al Config Server, aplica los reintentos, y gestiona el `fail-fast`. Sin `super.locate()`, el cliente pierde toda la integración con el Config Server y funciona solo con las fuentes locales — sin advertencia visible.

> **[ADVERTENCIA]** Registrar un `PropertySourceLocator` que lanza excepciones si el servicio externo no está disponible. Si el locator falla, Spring Cloud puede abortar el bootstrap completo, impidiendo que el servicio arranque incluso cuando el fallo es en una fuente secundaria. Siempre envolver en `try/catch` y devolver un `PropertySource` vacío cuando la fuente no está disponible, de la misma forma que `optional:configserver:` lo hace para el Config Server.

---

### Tabla resumen: cuándo YAML es suficiente vs cuándo se necesita Java en el cliente

| Caso | YAML suficiente | Requiere Java |
|---|---|---|
| Conectar al Config Server estándar | Sí | No |
| Añadir fuente de propiedades adicional (API interna, BD) | No | Sí — `PropertySourceLocator` |
| Modificar las cabeceras HTTP al consultar el Config Server | No | Sí — extender `ConfigServicePropertySourceLocator` |
| Autenticación custom al Config Server | Parcialmente (basic auth y HTTPS en YAML) | Sí para OAuth2 / token propio — extender `ConfigServicePropertySourceLocator` |
| Resolver secretos antes que cualquier `@Value` | No | Sí — `BootstrapConfiguration` |
| Leer la URL del Config Server desde código | No | Sí — inyectar `ConfigClientProperties` |

---

← [3.4 Config Client](./03-04-config-client.md) | [Índice](./README.md) | [3.5 Refresco →](./03-05-config-refresh.md)
