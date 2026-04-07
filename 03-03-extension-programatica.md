# Parte 3.3 — Config Server: Extensión Programática

← [Config Server (YAML)](./03-03-config-server.md) | [Volver al índice](./README.md)

---

## Puntos de extensión del Config Server

La capa YAML del Config Server configura el comportamiento estándar: qué backend usar, dónde está el repositorio, cómo autenticarse. Pero hay casos donde hace falta intervenir en la lógica interna: reemplazar el algoritmo de cifrado por uno corporativo (Vault Transit, HSM), auditar qué propiedades se sirven a qué servicios, o filtrar propiedades sensibles antes de entregarlas a clientes no autorizados. Todos estos casos usan los mismos puntos de extensión — interfaces Spring que el Config Server invoca internamente y que se pueden reemplazar registrando un bean `@Primary`.

Los puntos de extensión más relevantes son:

| Interface / Clase | Para qué |
|---|---|
| `EnvironmentEncryptor` | Reemplazar el algoritmo de cifrado del `/encrypt` y `/decrypt` |
| `EnvironmentController` | Extender o interceptar el endpoint REST de propiedades |
| `EnvironmentRepository` | Backend custom (cubierto en `03-02-extension-programatica.md`) |
| `PropertyValueDescriptor` | Controlar cómo se serializa cada propiedad en la respuesta |

---

## `EnvironmentEncryptor` — cifrado custom (Vault Transit, HSM, Tink)

`EnvironmentEncryptor` es la interfaz que el Config Server usa internamente para descifrar los valores `{cipher}` antes de entregarlos al cliente, y también la que expone el endpoint `/encrypt`. Reemplazarla permite delegar el cifrado a cualquier sistema externo: Vault Transit, un HSM corporativo, Google Tink, o cualquier otro proveedor.

```java
@Component
@Primary   // sustituye el EnvironmentEncryptor autoconfigurdo por Spring Boot
public class VaultTransitEncryptor implements EnvironmentEncryptor {

    private final VaultTransitOperations transitOps;

    public VaultTransitEncryptor(VaultTemplate vaultTemplate) {
        this.transitOps = vaultTemplate.opsForTransit();
    }

    /**
     * Descifra todos los valores {cipher} de un Environment antes de entregarlo al cliente.
     * Spring Cloud Config llama a este método automáticamente al servir la configuración.
     */
    @Override
    public Environment decrypt(Environment environment) {
        environment.getPropertySources().forEach(propertySource -> {
            Map<String, Object> decrypted = new LinkedHashMap<>();

            propertySource.getSource().forEach((key, value) -> {
                if (value instanceof String str && str.startsWith("{cipher}")) {
                    String ciphertext = str.substring("{cipher}".length());
                    try {
                        // Llamar a Vault Transit para descifrar
                        String plaintext = transitOps.decrypt("config-encryption-key", ciphertext);
                        decrypted.put(key, plaintext);
                    } catch (VaultException e) {
                        log.error("No se pudo descifrar la propiedad '{}': {}", key, e.getMessage());
                        decrypted.put(key, value);   // mantener el valor cifrado si falla
                    }
                } else {
                    decrypted.put(key, value);
                }
            });

            // Reemplazar el PropertySource con los valores descifrados
            // (PropertySource es inmutable — hay que crear uno nuevo)
        });
        return environment;
    }

    /**
     * Cifra un valor para el endpoint POST /encrypt.
     */
    @Override
    public String encrypt(String text) {
        return transitOps.encrypt("config-encryption-key", Plaintext.of(text)).getCiphertext();
    }
}
```

> Para integrar Vault Transit como dependencia Maven ver `spring-vault-core`. La clave `config-encryption-key` debe existir en el motor `transit/` de Vault antes de usar este encriptador.

---

## `EnvironmentController` — extender el endpoint REST

El endpoint `/{application}/{profile}/{label}` está implementado en `EnvironmentController`. Para añadir lógica transversal (audit log de quién accedió a qué configuración, cabeceras de respuesta personalizadas, transformaciones del entorno antes de enviarlo) la forma más limpia es un `@ControllerAdvice` o reemplazar el bean con `@Primary`.

```java
@RestController
@Primary   // reemplaza el EnvironmentController autoconfigurdo
@RequestMapping(method = RequestMethod.GET,
    value = {"/{name}/{profiles:.*[^-].*}",
             "/{name}/{profiles}/{label:.*}"})
public class AuditableEnvironmentController {

    private final EnvironmentController delegate;   // el controller original
    private final AuditService auditService;

    @GetMapping("/{name}/{profiles:.*[^-].*}")
    public Environment environment(@PathVariable String name,
                                   @PathVariable String profiles,
                                   HttpServletRequest request) {

        // Registrar el acceso: quién pidió qué configuración
        auditService.registrar(AuditEvent.builder()
            .service(name)
            .profiles(profiles)
            .caller(request.getHeader("X-Calling-Service"))
            .timestamp(Instant.now())
            .build());

        // Delegar al controller original
        return delegate.environment(name, profiles);
    }
}
```

Alternativa más sencilla con `@ControllerAdvice` si solo se necesitan transformar las respuestas:

```java
@ControllerAdvice(assignableTypes = EnvironmentController.class)
public class EnvironmentResponseAdvice implements ResponseBodyAdvice<Environment> {

    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        return Environment.class.isAssignableFrom(returnType.getParameterType());
    }

    @Override
    public Environment beforeBodyWrite(Environment body, MethodParameter returnType,
            MediaType selectedContentType, Class selectedConverterType,
            ServerHttpRequest request, ServerHttpResponse response) {

        // Añadir cabecera con timestamp de la respuesta
        response.getHeaders().add("X-Config-Served-At",
            Instant.now().toString());

        // Filtrar propiedades sensibles antes de enviarlas (doble capa de seguridad)
        if (body != null) {
            body.getPropertySources().forEach(ps ->
                ps.getSource().entrySet().removeIf(e ->
                    e.getKey().contains("password") &&
                    !e.getValue().toString().startsWith("{cipher}")));
        }
        return body;
    }
}
```

---

## `EnvironmentPostProcessor` en el servidor — transformar antes de entregar

`EnvironmentPostProcessor` permite modificar el `Environment` del propio Config Server (no de los clientes) durante el arranque. Útil cuando las propiedades del Config Server deben derivarse de una fuente externa antes de que los beans se inicialicen:

```java
// Registrar en META-INF/spring/org.springframework.boot.env.EnvironmentPostProcessor.imports
public class SecretsManagerEnvironmentPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                        SpringApplication application) {

        // Solo actuar si es el Config Server
        if (!environment.containsProperty("spring.cloud.config.server.git.uri")) {
            return;
        }

        // Leer la contraseña del repo Git desde AWS Secrets Manager
        // antes de que Spring Cloud Config inicialice el JGitEnvironmentRepository
        String gitPassword = awsSecretsManager.getSecret("config-server/git-password");

        Map<String, Object> secretsProps = Map.of("spring.cloud.config.server.git.password",
            gitPassword);

        environment.getPropertySources().addFirst(
            new MapPropertySource("secrets-manager", secretsProps)
        );
    }
}
```

```
# META-INF/spring/org.springframework.boot.env.EnvironmentPostProcessor.imports
com.miempresa.config.SecretsManagerEnvironmentPostProcessor
```

> Este patrón es útil para que el Config Server obtenga sus propias credenciales de un gestor de secretos sin exponerlas en ficheros de configuración. Se ejecuta antes de que cualquier `@Bean` sea creado.

---

## Antipatrones frecuentes

> **[ADVERTENCIA]** No marcar el `EnvironmentEncryptor` custom con `@Primary`. Spring Boot autoconfiguró ya un `EnvironmentEncryptor` basado en la clave `encrypt.key`. Sin `@Primary`, el bean custom y el default coexisten — Spring elige uno arbitrariamente, o puede que intente descifrar con ambos, causando errores de descifrado intermitentes y difíciles de reproducir.

> **[ADVERTENCIA]** Lanzar excepciones no controladas en `EnvironmentEncryptor.decrypt()`. Si el servicio de cifrado externo (Vault, HSM) no está disponible, un `RuntimeException` sin capturar impide que el Config Server entregue **cualquier** configuración, no solo la cifrada. Siempre atrapar las excepciones y decidir explícitamente: devolver el valor original, un placeholder, o un error controlado con mensaje claro.

> **[ADVERTENCIA]** Usar `@ControllerAdvice` sin restringirlo al controlador del Config Server. Un `@ControllerAdvice` sin `assignableTypes` o `basePackages` intercepta también los endpoints de Actuator (`/actuator/health`, `/actuator/info`), lo que puede hacer que los health checks del orquestador (Kubernetes, AWS ALB) empiecen a recibir respuestas inesperadas. Restringir siempre al controlador objetivo:
```java
@ControllerAdvice(assignableTypes = EnvironmentController.class)
```

---

## Resumen: cuándo YAML es suficiente vs cuándo se necesita Java en el servidor

| Caso | YAML suficiente | Requiere Java |
|---|---|---|
| Configurar backend Git/Vault/JDBC | Sí | No |
| Proteger el Config Server con basic auth | Sí | No |
| Proteger con OAuth2 / tokens propios | No | Sí — `SecurityWebFilterChain` |
| Cifrar con AES o RSA local | Sí | No |
| Cifrar con Vault Transit o HSM | No | Sí — `EnvironmentEncryptor` custom |
| Registrar auditoría de accesos | No | Sí — `@ControllerAdvice` o `EnvironmentController` override |
| Leer credenciales del servidor de un gestor de secretos | No | Sí — `EnvironmentPostProcessor` |
| Backend custom (API propia, sistema legacy) | No | Sí — ver `03-02-extension-programatica.md` |

---

← [Config Server (YAML)](./03-03-config-server.md) | [Volver al índice](./README.md)
