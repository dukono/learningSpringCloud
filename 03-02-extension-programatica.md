# Parte 3.2 — Config Backends: Extensión Programática

← [Backends (YAML)](./03-02-config-backends.md) | [Volver al índice](./README.md)

---

## Por qué la configuración YAML no es suficiente para algunos casos

El YAML del Config Server configura backends predefinidos (Git, Vault, JDBC) con los parámetros que Spring Boot autoconfigura. Pero hay escenarios donde esa capa no alcanza:

- Los repositorios Git no están definidos en tiempo de despliegue sino que se crean dinámicamente (p.ej. cada cliente del sistema tiene su propio repo de configuración generado en onboarding)
- La fuente de configuración no es ninguno de los backends existentes (una API interna, un sistema legacy, un servicio de configuración propietario)
- Se necesita lógica de enrutamiento más compleja que los patrones de nombre que soporta YAML
- Se quiere combinar backends con lógica condicional en tiempo de arranque

La capa programática resuelve todos estos casos: los backends de Spring Cloud Config son beans de Spring, y cualquier bean puede ser reemplazado, extendido o creado con lógica Java arbitraria.

---

## La interfaz central: `EnvironmentRepository`

Todo backend de Spring Cloud Config implementa esta interfaz. Es el único punto de extensión necesario para crear un backend completamente nuevo:

```java
public interface EnvironmentRepository {

    /**
     * Devuelve el entorno de configuración para un servicio dado.
     * @param application  spring.application.name del cliente (ej: "pedidos-service")
     * @param profile      perfil activo del cliente (ej: "prod", "dev")
     * @param label        rama/tag/commit de Git, o cualquier discriminador del backend
     */
    Environment findOne(String application, String profile, String label);
}
```

`Environment` es el objeto que el Config Server devuelve al cliente. Contiene una lista de `PropertySource`, cada uno con un nombre y un mapa de clave-valor:

```java
// Anatomía de Environment
Environment env = new Environment("pedidos-service", "prod");

Map<String, Object> propiedades = new HashMap<>();
propiedades.put("database.url", "jdbc:postgresql://prod-db:5432/pedidos");
propiedades.put("pedidos.max-items", "1000");

env.add(new PropertySource("mi-backend:pedidos-service-prod", propiedades));
// Se pueden añadir múltiples PropertySource — el orden determina la precedencia
```

---

## Configurar un `JGitEnvironmentRepository` como bean (sin YAML)

La configuración YAML de Git instancia internamente un `JGitEnvironmentRepository`. Se puede crear directamente como bean para añadir lógica que YAML no expresa — por ejemplo, leer la URL del repo desde una base de datos o aplicar lógica condicional según el entorno activo:

```java
@Configuration
public class GitBackendConfig {

    @Autowired
    private ConfigurableEnvironment environment;

    @Bean
    @Primary  // sustituye el JGitEnvironmentRepository que Spring Boot crearía desde YAML
    public JGitEnvironmentRepository gitEnvironmentRepository(
            JGitEnvironmentRepositoryFactory factory) {

        JGitEnvironmentProperties props = new JGitEnvironmentProperties();

        // La URL se lee desde una fuente externa, no del YAML
        props.setUri(resolverUrlDelRepo());
        props.setDefaultLabel("main");
        props.setCloneOnStart(true);
        props.setForcePull(true);
        props.setTimeout(10);

        // Añadir credenciales desde un gestor de secretos
        props.setUsername(secretManager.get("git.username"));
        props.setPassword(secretManager.get("git.password"));

        // search-paths dinámico según el entorno activo
        if (environment.acceptsProfiles(Profiles.of("cloud"))) {
            props.setSearchPaths(new String[]{"cloud/{application}"});
        } else {
            props.setSearchPaths(new String[]{"{application}"});
        }

        return factory.build(props);
    }

    private String resolverUrlDelRepo() {
        // Podría leerlo de una BD, de Consul, de una variable de entorno derivada, etc.
        return System.getenv().getOrDefault("CONFIG_REPO_URL",
            "https://github.com/mi-org/config-repo");
    }
}
```

> `JGitEnvironmentRepositoryFactory` es el bean que Spring Boot autoconfigura para crear instancias de `JGitEnvironmentRepository`. Inyectarlo como dependencia permite construir la instancia con la misma configuración de conexión (pool, retries) que Spring gestionaría internamente.

---

## Múltiples repos Git dinámicos desde base de datos

Este es el caso que YAML no puede resolver: la lista de repositorios no se conoce en tiempo de despliegue. Un ejemplo real: una plataforma SaaS donde cada tenant tiene su propio repositorio de configuración y los nuevos tenants se registran en runtime.

```java
@Component
public class MultiTenantGitRepositoryLocator implements EnvironmentRepository, Ordered {

    private final TenantRepository tenantRepository;       // JPA/JDBC propio
    private final JGitEnvironmentRepositoryFactory factory;
    private final ConfigurableEnvironment environment;

    // Cache de repos ya inicializados, para no clonar en cada petición
    private final Map<String, JGitEnvironmentRepository> repoCache =
        new ConcurrentHashMap<>();

    @Override
    public Environment findOne(String application, String profile, String label) {
        // El tenant se extrae del nombre del servicio: "tenant-a:pedidos-service"
        String[] partes = application.split(":", 2);
        if (partes.length < 2) {
            return new Environment(application, profile);   // sin tenant → vacío
        }

        String tenantId = partes[0];
        String serviceName = partes[1];

        // Obtener (o crear y cachear) el repo para este tenant
        JGitEnvironmentRepository repo = repoCache.computeIfAbsent(
            tenantId,
            id -> crearRepoParaTenant(id)
        );

        // Delegar la búsqueda al repo JGit ya configurado
        return repo.findOne(serviceName, profile, label);
    }

    private JGitEnvironmentRepository crearRepoParaTenant(String tenantId) {
        // Consultar la URL del repo del tenant en la base de datos
        Tenant tenant = tenantRepository.findById(tenantId)
            .orElseThrow(() -> new IllegalArgumentException("Tenant no encontrado: " + tenantId));

        JGitEnvironmentProperties props = new JGitEnvironmentProperties();
        props.setUri(tenant.getConfigRepoUrl());
        props.setDefaultLabel(tenant.getConfigRepoBranch());
        props.setCloneOnStart(false);   // clone lazy — no todos los repos al arrancar

        JGitEnvironmentRepository repo = factory.build(props);
        repo.afterPropertiesSet();   // inicializar el repo (equivalente a @PostConstruct)
        return repo;
    }

    @Override
    public int getOrder() { return 0; }   // mayor prioridad que el repo global
}
```

```java
// Entidad JPA que almacena los repos de configuración de cada tenant
@Entity
public class Tenant {
    @Id
    private String tenantId;
    private String configRepoUrl;
    private String configRepoBranch;
    // ...
}
```

> **[ADVERTENCIA]** El `repoCache` crece indefinidamente con el número de tenants. En producción, usar una caché con expiración (`Caffeine`, `Guava Cache`) para evitar acumular repos de tenants inactivos. El clon local de Git de cada repo ocupa disco.

---

## `CompositeEnvironmentRepository` programático

El backend Composite en YAML es fijo en tiempo de despliegue. Programáticamente se puede construir la lista de backends con lógica condicional — por ejemplo, añadir Vault solo si el perfil `cloud` está activo:

```java
@Configuration
public class CompositeBackendConfig {

    @Bean
    @Primary
    public EnvironmentRepository compositeEnvironmentRepository(
            JGitEnvironmentRepositoryFactory gitFactory,
            ApplicationContext context) {

        List<EnvironmentRepository> repos = new ArrayList<>();

        // Siempre incluir el repo Git principal
        JGitEnvironmentProperties gitProps = new JGitEnvironmentProperties();
        gitProps.setUri("https://github.com/mi-org/config-repo");
        gitProps.setCloneOnStart(true);
        repos.add(gitFactory.build(gitProps));

        // Añadir Vault solo si el perfil "cloud" está activo
        if (context.getEnvironment().acceptsProfiles(Profiles.of("cloud"))) {
            repos.add(context.getBean(VaultEnvironmentRepository.class));
        }

        // Añadir un repositorio JDBC con menor prioridad (fallback)
        if (context.getEnvironment().acceptsProfiles(Profiles.of("jdbc-fallback"))) {
            repos.add(context.getBean(JdbcEnvironmentRepository.class));
        }

        // true = no fallar si algún repositorio falla (tolerante a fallos)
        return new CompositeEnvironmentRepository(repos, true);
    }
}
```

> `CompositeEnvironmentRepository(repos, failOnError)`: si `failOnError=true`, cualquier excepción en un backend se propaga y la petición falla. Si `failOnError=false`, el backend con error se salta y se consulta el siguiente. En producción, `false` es la configuración segura si el backend con error es un fallback, pero `true` si todos los backends son obligatorios.

---

## Backend completamente custom (fuente propia)

Implementar `EnvironmentRepository` directamente para leer la configuración desde una fuente arbitraria — una API REST interna, un sistema de configuración legacy, un motor de reglas:

```java
@Component
@Order(Ordered.LOWEST_PRECEDENCE)   // último recurso, solo si los otros no tienen la propiedad
public class ApiInternBackend implements EnvironmentRepository {

    private final RestTemplate restTemplate;
    private final String apiBaseUrl;

    public ApiInternBackend(RestTemplate restTemplate,
                             @Value("${config.api.base-url}") String apiBaseUrl) {
        this.restTemplate = restTemplate;
        this.apiBaseUrl = apiBaseUrl;
    }

    @Override
    public Environment findOne(String application, String profile, String label) {
        // Llamar a la API interna con los parámetros del cliente
        String url = apiBaseUrl + "/config/{app}/{profile}";

        try {
            Map<String, Object> response = restTemplate.getForObject(
                url, Map.class, application, profile
            );

            Environment env = new Environment(application, profile);
            if (response != null && !response.isEmpty()) {
                env.add(new PropertySource(
                    "api-intern:" + application + "-" + profile,
                    response
                ));
            }
            return env;

        } catch (RestClientException e) {
            // Si la API no responde, devolver entorno vacío — el Composite continuará con el siguiente backend
            log.warn("API interna no disponible para {}/{}: {}", application, profile, e.getMessage());
            return new Environment(application, profile);
        }
    }
}
```

---

## Antipatrones frecuentes

> **[ADVERTENCIA]** Registrar un bean `EnvironmentRepository` custom sin `@Primary` cuando el Config Server ya tiene un backend configurado en YAML. Spring Boot autoconfiguró ya un `JGitEnvironmentRepository` (o el backend declarado en YAML). Sin `@Primary`, Spring lanza `NoUniqueBeanDefinitionException` al arrancar, o el backend custom nunca se usa si hay ambigüedad de prioridad. Si el bean custom debe **reemplazar** el backend YAML, usar `@Primary`. Si debe **añadirse** al pipeline junto con el YAML, usar `CompositeEnvironmentRepository`.

> **[ADVERTENCIA]** Crear un `JGitEnvironmentRepository` como bean sin gestionar el ciclo de vida del directorio temporal de clon. Si se crean múltiples repositorios dinámicamente (p.ej. uno por tenant) sin limpiar los clones locales, el disco del Config Server se llena gradualmente. Siempre configurar `basedir` explícito y añadir una política de limpieza cuando el tenant se elimina.

> **[ADVERTENCIA]** Lanzar excepciones en `EnvironmentRepository.findOne()` cuando el servicio externo no está disponible. Una excepción no controlada en `findOne()` hace que el Config Server devuelva un error 500 al cliente, que interpreta esto como un fallo crítico y puede abortar el arranque (si `fail-fast=true`). Siempre devolver un `Environment` vacío cuando la fuente no está disponible — igual que el ejemplo de `ApiInternBackend` arriba.

---

## Tabla resumen: YAML vs programático por caso de uso

| Caso de uso | YAML suficiente | Requiere Java |
|---|---|---|
| Un repo Git fijo conocido en despliegue | Sí | No |
| Múltiples repos según patrón de nombre | Sí (`repos:` + `pattern:`) | No |
| Repos creados dinámicamente en runtime | No | Sí — `JGitEnvironmentRepository` en bean |
| Backend personalizado (API propia, legacy) | No | Sí — implementar `EnvironmentRepository` |
| Composite con lógica condicional por perfil | No | Sí — `CompositeEnvironmentRepository` en bean |
| Vault con lógica de autenticación custom | No | Sí — subclase de `VaultEnvironmentRepository` |
| Routing tenant-aware (multi-tenant) | No | Sí — `EnvironmentRepository` con cache de repos |

---

← [Backends (YAML)](./03-02-config-backends.md) | [Volver al índice](./README.md)
