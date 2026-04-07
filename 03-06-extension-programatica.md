# 3.6.9 Extensión Programática: Cifrado y Alta Disponibilidad

← [3.6 Cifrado y HA](./03-06-config-avanzado.md) | [Índice](./README.md) | [4 — Service Discovery →](./04-service-discovery.md)

---

## Puntos de extensión del cifrado

La capa YAML del cifrado configura la clave AES o el keystore RSA para descifrar valores `{cipher}`. Pero hay casos donde esa capa no alcanza: la clave no puede residir en el Config Server (porque un compromiso del servidor expone todos los secretos), el algoritmo de cifrado corporativo no es AES ni RSA, o se necesita rotar claves sin ventana de mantenimiento. Todos estos casos requieren reemplazar los beans de cifrado que Spring Cloud Config autoconfiguró: `TextEncryptor` para el algoritmo, `EnvironmentEncryptor` para el pipeline completo, o `EncryptionController` para el endpoint REST de cifrado. En todos los casos, el bean de reemplazo debe estar marcado con `@Primary`.

| Interface | Para qué |
|---|---|
| `TextEncryptor` | Cifrar y descifrar un valor de texto |
| `EnvironmentEncryptor` | Procesar un `Environment` completo buscando valores `{cipher}` |
| `KeyStoreKeyFactory` | Fábrica de claves para acceder a keystores JKS/PKCS12 |
| `EncryptionController` | Endpoint REST `/encrypt` y `/decrypt` — reemplazable |

La relación entre ellas: `EnvironmentEncryptor` usa `TextEncryptor` internamente para descifrar cada valor individual. Reemplazando `TextEncryptor` se cambia el algoritmo de cifrado sin tocar el resto del pipeline.

---

## 3.6.9.1 `TextEncryptor` custom — algoritmo de cifrado propio

Implementar `TextEncryptor` permite reemplazar completamente el algoritmo de cifrado. Spring Cloud Config dejará de usar AES/RSA y usará la implementación que se proporcione:

```java
@Configuration
public class EncryptionConfig {

    /**
     * Reemplaza el TextEncryptor por defecto (AES/RSA) con una implementación
     * basada en Google Tink — una librería de criptografía de alto nivel.
     */
    @Bean
    @Primary
    public TextEncryptor tinkTextEncryptor() throws GeneralSecurityException {
        // Tink simplifica la gestión de claves y algoritmos
        AeadConfig.register();

        // La clave se almacena en un fichero JSON (puede ser un keyring de Google Cloud KMS)
        KeysetHandle keysetHandle = KeysetHandle.generateNew(AeadKeyTemplates.AES256_GCM);

        Aead aead = keysetHandle.getPrimitive(Aead.class);

        return new TextEncryptor() {
            @Override
            public String encrypt(String text) {
                try {
                    byte[] ciphertext = aead.encrypt(
                        text.getBytes(StandardCharsets.UTF_8),
                        null   // additional data (AAD) — puede ser el nombre del servicio
                    );
                    return Base64.getEncoder().encodeToString(ciphertext);
                } catch (GeneralSecurityException e) {
                    throw new EncryptionException("Error cifrando con Tink", e);
                }
            }

            @Override
            public String decrypt(String encryptedText) {
                try {
                    byte[] ciphertext = Base64.getDecoder().decode(encryptedText);
                    byte[] plaintext = aead.decrypt(ciphertext, null);
                    return new String(plaintext, StandardCharsets.UTF_8);
                } catch (GeneralSecurityException e) {
                    throw new EncryptionException("Error descifrando con Tink", e);
                }
            }
        };
    }
}
```

---

## 3.6.9.2 `TextEncryptor` con Vault Transit — sin claves locales

La implementación más segura para producción: el Config Server nunca tiene la clave en local. Solo envía el texto a Vault y recibe el resultado:

```java
@Component
@Primary
public class VaultTransitTextEncryptor implements TextEncryptor {

    private static final String TRANSIT_KEY = "config-server-key";

    private final VaultTransitOperations transitOps;

    public VaultTransitTextEncryptor(VaultTemplate vaultTemplate) {
        this.transitOps = vaultTemplate.opsForTransit();
    }

    @Override
    public String encrypt(String text) {
        // Vault devuelve: "vault:v1:ciphertext..." — incluye la versión de la clave
        Ciphertext ciphertext = transitOps.encrypt(
            TRANSIT_KEY,
            Plaintext.of(text)
        );
        return ciphertext.getCiphertext();   // "vault:v1:AQIB..."
    }

    @Override
    public String decrypt(String encryptedText) {
        // Vault desencripta usando la versión correcta de la clave automáticamente
        // (vault:v1:... usa v1, vault:v2:... usa v2, etc.)
        Plaintext plaintext = transitOps.decrypt(
            TRANSIT_KEY,
            Ciphertext.of(encryptedText)
        );
        return plaintext.asString();
    }
}
```

```yaml
# Config del Config Server para conectar con Vault Transit
spring:
  cloud:
    vault:
      uri: https://vault.miempresa.com:8200
      authentication: KUBERNETES
      kubernetes:
        role: config-server
  # Deshabilitar el cifrado built-in (AES/RSA) — lo gestiona el TextEncryptor custom
  # encrypt:
  #   key: (no configurar — se usa el bean TextEncryptor)
```

> Con Vault Transit, la rotación de claves es transparente: Vault mantiene todas las versiones anteriores. Los valores cifrados con `vault:v1:` siguen siendo descifrables cuando se rota a `v2:`. No hay ventana de mantenimiento.

---

## 3.6.9.3 `TextEncryptor` multiversión — rotación sin ventana de mantenimiento

Para el caso de AES/RSA local (sin Vault), se puede implementar un `TextEncryptor` que prueba múltiples claves en orden, permitiendo que valores cifrados con la clave antigua sigan funcionando mientras se migra:

```java
@Bean
@Primary
public TextEncryptor multiVersionTextEncryptor(
        @Value("${encrypt.key.current}") String currentKey,
        @Value("${encrypt.key.previous:#{null}}") String previousKey) {

    TextEncryptor current = new HexEncodingTextEncryptor(
        new AesBytesEncryptor(currentKey, KeyGenerators.string().generateKey())
    );

    List<TextEncryptor> fallbacks = new ArrayList<>();
    if (previousKey != null) {
        fallbacks.add(new HexEncodingTextEncryptor(
            new AesBytesEncryptor(previousKey, KeyGenerators.string().generateKey())
        ));
    }

    return new TextEncryptor() {
        @Override
        public String encrypt(String text) {
            // Siempre cifrar con la clave actual
            return current.encrypt(text);
        }

        @Override
        public String decrypt(String encryptedText) {
            // Intentar con la clave actual primero
            try {
                return current.decrypt(encryptedText);
            } catch (Exception e) {
                // Si falla, intentar con las claves anteriores
                for (TextEncryptor fallback : fallbacks) {
                    try {
                        return fallback.decrypt(encryptedText);
                    } catch (Exception ignored) {}
                }
                throw new EncryptionException(
                    "No se pudo descifrar con ninguna clave disponible", e);
            }
        }
    };
}
```

```yaml
# Configuración durante la rotación:
encrypt:
  key:
    current: ${ENCRYPT_KEY_V2}    # clave nueva — usada para cifrar
    previous: ${ENCRYPT_KEY_V1}   # clave antigua — usada como fallback al descifrar
# Una vez migrados todos los valores al nuevo cifrado:
# eliminar encrypt.key.previous
```

---

## 3.6.9.4 Alta disponibilidad programática — failover entre Config Servers

En lugar de confiar en un load balancer externo, se puede implementar failover directo en el cliente:

```java
@Component
@Primary
public class FailoverConfigServiceLocator extends ConfigServicePropertySourceLocator {

    private static final List<String> CONFIG_SERVERS = List.of(
        "http://config-server-1:8888",
        "http://config-server-2:8888",
        "http://config-server-3:8888"
    );

    public FailoverConfigServiceLocator(ConfigClientProperties properties) {
        super(properties);
    }

    @Override
    public PropertySource<?> locate(Environment environment) {
        List<String> shuffled = new ArrayList<>(CONFIG_SERVERS);
        Collections.shuffle(shuffled);   // distribución aleatoria de carga

        for (String serverUrl : shuffled) {
            try {
                // Intentar con este Config Server
                super.getProperties().setUri(new String[]{serverUrl});
                PropertySource<?> result = super.locate(environment);
                if (result != null) {
                    log.debug("Configuración obtenida de: {}", serverUrl);
                    return result;
                }
            } catch (Exception e) {
                log.warn("Config Server {} no disponible: {} — intentando siguiente",
                    serverUrl, e.getMessage());
            }
        }

        throw new IllegalStateException(
            "Ninguno de los Config Servers disponibles respondió: " + CONFIG_SERVERS);
    }
}
```

> Esta implementación complementa (no reemplaza) la opción de múltiples URIs en YAML (`spring.cloud.config.uri: [url1, url2, url3]`). La diferencia: la opción YAML intenta las URIs en orden fijo. Esta implementación las mezcla aleatoriamente para distribuir la carga entre instancias, y tiene lógica de fallback con logging explícito.

---

## 3.6.9.5 Antipatrones

> **[ADVERTENCIA]** Registrar un `TextEncryptor` custom sin `@Primary`. Spring Boot autoconfiguró un `TextEncryptor` basado en `encrypt.key` o `encrypt.key-store`. Sin `@Primary`, el bean custom existe en el contexto pero el autoconfigurdo toma precedencia. El síntoma: el cifrado YAML sigue funcionando (con la clave AES/RSA), pero el cifrado custom nunca se invoca, sin ningún error ni advertencia.

> **[ADVERTENCIA]** No gestionar la indisponibilidad del servicio de cifrado externo (Vault, KMS) en `TextEncryptor.decrypt()`. Si el servicio externo no responde y el método lanza una excepción no controlada, el Config Server devuelve un error 500 para **todos** los clientes — incluyendo los que no usan propiedades cifradas. Siempre implementar un fallback explícito: reintentos con backoff, circuit breaker, o un mensaje de error controlado que identifique el problema.

> **[ADVERTENCIA]** En el `TextEncryptor` multiversión, iterar las claves en orden aleatorio en lugar de "clave nueva primero". Si la clave antigua se prueba antes que la nueva, valores recién cifrados con la clave nueva fallarán el primer intento y solo tendrán éxito en el segundo. El orden debe ser siempre: clave vigente primero, claves anteriores como fallback en orden de creación descendente.

---

### Tabla resumen: cuándo YAML es suficiente vs cuándo se necesita Java para cifrado y HA

| Caso | YAML suficiente | Requiere Java |
|---|---|---|
| Cifrado AES con clave fija | Sí | No |
| Cifrado RSA con keystore | Sí | No |
| Rotación de clave (con ventana de mantenimiento) | Sí | No |
| Rotación sin ventana (múltiples claves activas) | No | Sí — `TextEncryptor` multiversión |
| Cifrado con Vault Transit | No | Sí — `TextEncryptor` con VaultTemplate |
| Cifrado con Google Tink / HSM | No | Sí — `TextEncryptor` custom |
| HA con load balancer externo | Sí | No |
| HA con múltiples URIs fijas | Sí | No |
| HA con failover y balanceo propio | No | Sí — extender `ConfigServicePropertySourceLocator` |

---

← [3.6 Cifrado y HA](./03-06-config-avanzado.md) | [Índice](./README.md) | [4 — Service Discovery →](./04-service-discovery.md)
