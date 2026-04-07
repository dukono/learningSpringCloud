# Parte 3.6 — Spring Cloud Config: Cifrado y Alta Disponibilidad

← [Refresco de Configuración](./03-05-config-refresh.md) | [Volver al índice](./README.md) | Siguiente: [Parte 4 — Service Discovery →](./04-service-discovery.md)

---

## 3.8 Cifrado y descifrado de propiedades sensibles

El Config Server puede almacenar propiedades cifradas en el repo Git y descifrarlas antes de entregarlas a los clientes. Esto resuelve el dilema entre dos necesidades opuestas: el repo Git debe ser visible para el equipo (auditoría, historial, PRs), pero las contraseñas y API keys no deben ser legibles para cualquiera con acceso de lectura al repo.

> **[CONCEPTO]** Las propiedades sensibles en el repositorio Git se almacenan con el prefijo `{cipher}`. El Config Server detecta ese prefijo, descifra el valor antes de entregarlo al cliente, y el cliente recibe el texto plano — nunca el valor cifrado.

**Clave simétrica (AES):** una única clave se usa tanto para cifrar como para descifrar. Es simple de gestionar, válida para equipos pequeños o entornos con un único Config Server. El problema es que quien tiene la clave puede tanto cifrar como descifrar: si la clave se filtra, todos los secretos quedan expuestos.

**Clave asimétrica (RSA):** hay dos claves distintas — la pública cifra, la privada descifra. La clave pública puede distribuirse libremente (cualquier desarrollador puede cifrar un nuevo secreto para añadirlo al repo), pero solo el Config Server tiene la clave privada y puede descifrar. Más segura para equipos grandes o entornos con múltiples personas que necesitan añadir secretos.

### Clave simétrica (AES) — sencilla, válida para equipos pequeños

```yaml
# application.yml del Config Server (NO bootstrap.yml en Spring Boot 3.x)
encrypt:
  key: ${ENCRYPT_KEY}   # clave de al menos 32 caracteres; nunca hardcodeada
```

```bash
# Cifrar un valor usando el endpoint del Config Server
curl -X POST http://localhost:8888/encrypt -d "mi-password-secreto"
# responde: AQA8bX3+kY...texto-cifrado...
```

```yaml
# pedidos-service-prod.yml en el repo Git
spring:
  datasource:
    password: '{cipher}AQA8bX3+kY...texto-cifrado...'
    # El prefijo {cipher} indica que el valor está cifrado
    # El Config Server lo descifra antes de entregarlo al cliente
```

---

### Clave asimétrica RSA — recomendada en producción

La clave asimétrica es más segura porque la clave de cifrado (pública) puede distribuirse, pero solo el Config Server tiene la clave privada para descifrar.

```bash
# Generar keystore con par RSA
keytool -genkeypair -alias config-server-key \
  -keyalg RSA -keysize 2048 \
  -dname "CN=Config Server,OU=Engineering,O=MiEmpresa" \
  -keystore config-server.jks \
  -storepass ${KEYSTORE_PASSWORD}
```

```yaml
# application.yml del Config Server
encrypt:
  key-store:
    location: classpath:/config-server.jks
    password: ${KEYSTORE_PASSWORD}
    alias: config-server-key
    secret: ${KEY_PASSWORD}
```

---

### Vault Transit como backend de cifrado

Las opciones AES y RSA almacenan la clave de cifrado en el propio Config Server (en variables de entorno o en un keystore). Si el servidor o su fichero de configuración se ven comprometidos, los secretos quedan expuestos. La alternativa es delegar el cifrado al motor **Transit** de HashiCorp Vault: el Config Server nunca posee la clave en local — solo envía el texto plano a Vault y recibe el texto cifrado, y viceversa. La clave vive exclusivamente en Vault, con soporte nativo de rotación sin ventana de interrupción.

```
# Flujo con Vault Transit:
Config Server → POST /v1/transit/encrypt/config-key (texto plano)
             ← vault:v1:AQABBYl...texto-cifrado-por-vault...
Config Server → almacena {cipher}vault:v1:AQA... en Git

# Al entregar la config al cliente:
Config Server → POST /v1/transit/decrypt/config-key (texto cifrado)
             ← texto plano original
Config Server → entrega al cliente como propiedad descifrada
```

Vault Transit permite versionar claves (`v1:`, `v2:`): cuando se rota la clave, los valores cifrados con versiones anteriores siguen siendo descifrables porque Vault mantiene las versiones antiguas hasta que se hace una migración explícita. Esto elimina la ventana de mantenimiento del procedimiento de rotación manual.

> La integración con Vault Transit requiere implementar un `TextEncryptor` o `EnvironmentEncryptor` personalizado que llame a la API de Vault. Ver implementación en el companion programático.

---

### Verificar cifrado: endpoint `/decrypt`

El endpoint complementario a `/encrypt` permite verificar que un valor cifrado almacenado en Git se puede descifrar correctamente:

```bash
# Verificar que el valor cifrado es correcto
curl -X POST http://localhost:8888/decrypt \
  -d "AQA8bX3+kY...texto-cifrado..."
# responde: mi-password-secreto
```

Útil para diagnosticar errores del tipo `Cannot decrypt: ...` que aparecen en los clientes cuando el valor cifrado está malformado o fue cifrado con una clave distinta.

---

### Rotación de la clave de cifrado

Cuando se cambia la clave de cifrado (`encrypt.key`), todos los valores `{cipher}` almacenados en el repo Git quedan inválidos porque fueron cifrados con la clave anterior.

**Procedimiento de rotación:**

```bash
# 1. Tener la clave antigua y la nueva disponibles temporalmente

# 2. Descifrar cada valor con la clave ANTIGUA
curl -X POST http://localhost:8888/decrypt \
  -H "encrypt.key: CLAVE_ANTIGUA" \
  -d "AQA8bX3+...valor-cifrado-antiguo..."
# → mi-password-secreto

# 3. Volver a cifrar con la clave NUEVA
curl -X POST http://localhost:8888/encrypt \
  -d "mi-password-secreto"
# → XB9kQ2+...nuevo-valor-cifrado...

# 4. Actualizar el repo Git con los nuevos valores {cipher}
# 5. Una vez migrados todos los valores, activar la nueva clave en el Config Server
```

> **[ADVERTENCIA]** Durante la rotación hay una ventana en la que el servidor tiene la nueva clave pero el repo aún tiene valores cifrados con la antigua. En ese período los clientes no pueden arrancar. Planificar la rotación en ventana de mantenimiento o usar Vault (que gestiona la rotación de forma automática y transparente).

**Limitación de la rotación con AES/RSA nativo:** Spring Cloud Config solo soporta una clave de cifrado activa en cada momento. No es posible descifrar valores con clave antigua mientras se activa la clave nueva — hay que migrar todos los valores de golpe. Para evitar esta limitación sin ventana de mantenimiento hay tres opciones:

| Opción | Descripción | Ventana de mantenimiento |
|---|---|---|
| Rotación manual (procedimiento anterior) | Descifrar + volver a cifrar todos los valores | Sí |
| Vault Transit | Vault mantiene versiones anteriores de la clave | No |
| `TextEncryptor` custom multiversión | Implementar un encriptador que prueba claves en orden | No (requiere código) |

---

### Descifrado en el cliente (en lugar del servidor)

Por defecto el servidor descifra antes de entregar. Para que sea el cliente quien descifre:

```yaml
# application.yml del Config Server
spring:
  cloud:
    config:
      server:
        encrypt:
          enabled: false   # el servidor entrega el valor cifrado tal cual
```

```yaml
# application.yml del cliente (con la misma clave)
encrypt:
  key: ${ENCRYPT_KEY}
```

---

## 3.9 Alta disponibilidad del Config Server

> **[CONCEPTO]** Un **punto único de fallo** (SPOF) es un componente cuya caída detiene el sistema completo. El Config Server es un SPOF si solo hay una instancia: ningún microservicio nuevo puede arrancar hasta que se recupere. Los microservicios ya en ejecución sobreviven, pero cualquier nuevo arranque o refresco fallará.

El Config Server es un **punto único de fallo** si solo hay una instancia: si cae, ningún microservicio puede arrancar ni refrescar su configuración. En producción esto es inaceptable. Los microservicios que ya están en ejecución seguirán funcionando con la configuración en memoria, pero cualquier nuevo arranque fallará. Hay tres estrategias para eliminar este punto único de fallo, con distintos compromisos entre simplicidad operacional y coste de infraestructura.

---

### Estrategia 1: Múltiples instancias detrás de un load balancer

```
[Load Balancer]
      ├── Config Server :8888 (instancia 1)
      ├── Config Server :8888 (instancia 2)
      └── Config Server :8888 (instancia 3)
           (todas apuntan al mismo repo Git)
```

Los Config Clients usan la URL del Load Balancer:
```yaml
spring:
  config:
    import: "configserver:http://config-lb.interno:8888"
```

---

### Estrategia 2: Config Server registrado en Eureka

```yaml
# Config Server — también es Eureka client
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/

spring:
  application:
    name: config-server
```

```yaml
# Config Client — descubre el Config Server por Eureka en lugar de URL fija
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server   # nombre registrado en Eureka
```

> **[ADVERTENCIA] Bootstrap problem:** Si el Config Client usa Eureka para encontrar el Config Server, hay un problema de arranque circular: el servicio necesita el Config Server para arrancar, pero necesita Eureka para encontrar el Config Server.
>
> **Solución:** Configurar la URL de Eureka directamente en el `application.yml` local del cliente (sin esperar al Config Server), y dar prioridad al Config Server en la cadena de configuración.

---

### Estrategia 3: Config Server con múltiples URIs (sin load balancer externo)

Los clientes pueden apuntar a múltiples instancias del Config Server directamente:

```yaml
spring:
  cloud:
    config:
      uri:
        - http://config-server-1:8888
        - http://config-server-2:8888
        - http://config-server-3:8888
      # El cliente intentará cada URI en orden hasta que una responda
```

---

## 3.10 Testing con Spring Cloud Config

### Estrategia 1: deshabilitar el Config Client en tests

La forma más sencilla: el test no necesita el Config Server y usa solo la configuración local.

```yaml
# src/test/resources/application-test.yml
spring:
  cloud:
    config:
      enabled: false   # deshabilita el Config Client en tests
                       # el contexto arranca sin intentar contactar al Config Server
```

```java
@SpringBootTest
@ActiveProfiles("test")
class PedidosServiceTest {
    // arranca sin intentar conectar al Config Server
}
```

---

### Estrategia 2: Config Server opcional en tests

Si el test intenta conectar pero no debe fallar si no hay servidor disponible:

```yaml
# src/test/resources/application-test.yml
spring:
  config:
    import: "optional:configserver:http://localhost:8888"
    # el test usa config local si el servidor no está disponible
```

---

### Estrategia 3: Config Server embebido en el test

Para tests de integración que verifican la interacción real cliente-servidor sin infraestructura externa:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@SpringBootApplication
@EnableConfigServer   // levanta un Config Server real dentro del test
class EmbeddedConfigServerTest {

    @Test
    void clienteRecibeLaConfiguracionCorrecta(@Autowired Environment env) {
        assertThat(env.getProperty("pedidos.max-items")).isEqualTo("100");
    }
}
```

```yaml
# src/test/resources/application.yml del test
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/test-config
```

```
src/test/resources/
├── application.yml          ← config del test (native backend)
└── test-config/
    ├── application.yml      ← config global de prueba
    └── pedidos-service.yml  ← config de prueba para el servicio
```

---

### Estrategia 4: testear que `@RefreshScope` recarga los valores

```java
@SpringBootTest
@ActiveProfiles("test")
class RefreshScopeTest {

    @Autowired
    private FeatureController featureController;   // bean con @RefreshScope

    @Autowired
    private ContextRefresher contextRefresher;     // dispara el refresco programáticamente

    @Test
    void beanSeRecargaTrasRefresh() {
        // valor inicial
        assertThat(featureController.isEnabled()).isFalse();

        // simular cambio de propiedad en el entorno del test
        TestPropertyValues.of("feature.nueva-ui=true")
            .applyTo((ConfigurableApplicationContext) context);

        // disparar refresco — equivale a POST /actuator/refresh
        Set<String> changedKeys = contextRefresher.refresh();

        assertThat(changedKeys).contains("feature.nueva-ui");
        assertThat(featureController.isEnabled()).isTrue();   // bean recargado
    }
}
```

---

### Estrategia 5: testear propiedades cifradas

```java
@SpringBootTest
@ActiveProfiles("test")
class EncryptedPropertiesTest {

    @Value("${database.password}")
    private String password;

    @Test
    void propiedadCifradaSeDescifraCorrectamente() {
        // @Value recibe el valor descifrado, nunca el prefijo {cipher}
        assertThat(password).isEqualTo("mi-password-secreto");
        assertThat(password).doesNotStartWith("{cipher}");
    }
}
```

```yaml
# src/test/resources/application-test.yml
encrypt:
  key: clave-de-prueba-de-al-menos-32-caracteres   # misma clave usada para cifrar
```

---

### Resumen: qué estrategia usar

| Caso | Estrategia |
|---|---|
| Tests unitarios sin config externa | `enabled: false` |
| Tests que usan fichero local como fallback | `optional:configserver:` |
| Tests de integración cliente-servidor | Config Server embebido + backend `native` |
| Verificar que `@RefreshScope` recarga | `ContextRefresher` + `TestPropertyValues` |
| Verificar descifrado de propiedades | `encrypt.key` en `application-test.yml` |

---

→ **Extensión programática:** [03-06-extension-programatica.md](./03-06-extension-programatica.md) — `TextEncryptor` con Vault Transit, rotación sin ventana, failover HA programático

← [Refresco de Configuración](./03-05-config-refresh.md) | [Volver al índice](./README.md) | Siguiente: [Parte 4 — Service Discovery →](./04-service-discovery.md)
