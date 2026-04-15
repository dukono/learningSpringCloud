# 6.4.3 Filtros con lógica de negocio: validación HMAC

← [6.4.2 GlobalFilter y Ordered](./06-19-gateway-global-filter.md) | [Índice](./README.md) | [6.5.1 Token Bucket y RequestRateLimiter →](./06-21-gateway-rate-limiter-config.md)

---

No toda la lógica que vive en el Gateway pertenece a la infraestructura. Hay un conjunto acotado de responsabilidades de negocio que tienen sentido en el Gateway porque aplican de forma transversal antes de que la petición llegue a cualquier microservicio: validación de firmas de petición (HMAC), verificación de API keys, control de acceso basado en planes de suscripción. El criterio para decidir si una lógica pertenece al Gateway o a un microservicio es si requiere acceso al dominio de negocio: si necesita consultar la base de datos de clientes, resolver reglas de negocio complejas o coordinar varios servicios, pertenece a un microservicio. Si puede decidir en función de la petición entrante y un secreto o clave almacenada en configuración, puede vivir en el Gateway.

La validación HMAC es el ejemplo canónico: el cliente firma el body de la petición con una clave compartida, el Gateway verifica la firma antes de reenviar, y los microservicios reciben solo peticiones ya autenticadas sin necesidad de implementar la lógica de verificación en cada uno.

## Diagrama: flujo de validación HMAC

El Gateway intercepta cada petición entrante, extrae el body cacheado y el header de firma, calcula el HMAC localmente y solo reenvía al microservicio si ambas firmas coinciden. Las peticiones con firma ausente o incorrecta son rechazadas en el Gateway sin llegar al servicio destino.

```
Cliente
  │  Petición con:
  │  - Body: {"cantidad": 10, "productoId": "ABC"}
  │  - Header X-Signature: Base64(HMAC-SHA256(body, clave_secreta))
  │
  ▼
Gateway — HmacValidationGatewayFilterFactory
  │
  ├─ Lee body desde cache (CacheRequestBody ya lo almacenó)
  ├─ Calcula HMAC del body con la clave secreta configurada
  ├─ Compara firma calculada con X-Signature usando MessageDigest.isEqual()
  │
  ├─ ¿Firmas coinciden? ─── SÍ ──► Reenvía al microservicio
  │
  └─ NO ──► 401 Unauthorized (no llega al microservicio)
```

## Implementación completa del filtro HMAC

La clase extiende `AbstractGatewayFilterFactory` con una clase `Config` interna que recibe la clave secreta, el nombre del header de firma y el algoritmo. La comparación de firmas usa `MessageDigest.isEqual()` en lugar de `String.equals()` para evitar timing attacks, ya que opera en tiempo constante independientemente de cuántos bytes coincidan.

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Arrays;
import java.util.Base64;
import java.util.List;

@Component
public class HmacValidationGatewayFilterFactory
        extends AbstractGatewayFilterFactory<HmacValidationGatewayFilterFactory.Config> {

    public HmacValidationGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("secretKey", "signatureHeader", "algorithm");
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // El body ya fue cacheado por CacheRequestBody (declarado antes en la ruta)
            String cachedBody = exchange.getAttribute(
                ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR);

            if (cachedBody == null) {
                // Sin body cacheado no podemos validar la firma
                exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
                return exchange.getResponse().setComplete();
            }

            String receivedSignature = exchange.getRequest().getHeaders()
                .getFirst(config.getSignatureHeader());

            if (receivedSignature == null) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            try {
                String calculatedSignature = calculateHmac(
                    cachedBody, config.getSecretKey(), config.getAlgorithm());

                // MessageDigest.isEqual previene timing attacks (compara en tiempo constante)
                boolean valid = MessageDigest.isEqual(
                    receivedSignature.getBytes(StandardCharsets.UTF_8),
                    calculatedSignature.getBytes(StandardCharsets.UTF_8));

                if (!valid) {
                    exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                    return exchange.getResponse().setComplete();
                }

            } catch (Exception e) {
                exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                return exchange.getResponse().setComplete();
            }

            return chain.filter(exchange);
        };
    }

    private String calculateHmac(String data, String secretKey, String algorithm)
            throws Exception {
        Mac mac = Mac.getInstance(algorithm);
        SecretKeySpec keySpec = new SecretKeySpec(
            secretKey.getBytes(StandardCharsets.UTF_8), algorithm);
        mac.init(keySpec);
        byte[] hmacBytes = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
        return Base64.getEncoder().encodeToString(hmacBytes);
    }

    public static class Config {
        private String secretKey;
        private String signatureHeader = "X-Signature";
        private String algorithm = "HmacSHA256";

        public String getSecretKey() { return secretKey; }
        public void setSecretKey(String secretKey) { this.secretKey = secretKey; }

        public String getSignatureHeader() { return signatureHeader; }
        public void setSignatureHeader(String signatureHeader) {
            this.signatureHeader = signatureHeader;
        }

        public String getAlgorithm() { return algorithm; }
        public void setAlgorithm(String algorithm) { this.algorithm = algorithm; }
    }
}
```

> [ADVERTENCIA] El filtro HMAC depende de que `CacheRequestBody` se haya ejecutado antes en la cadena. Si se declara `HmacValidation` sin `CacheRequestBody` antes, `CACHED_REQUEST_BODY_ATTR` es null y el filtro devuelve `400 Bad Request`. El orden de declaración en el YAML es el orden de ejecución.

## Configuración YAML de la ruta con HMAC

La ruta declara tres filtros en orden estricto: `RequestSize` limita el tamaño antes de leer el body, `CacheRequestBody` almacena el body en el atributo de intercambio, y `HmacValidation` lo recupera para calcular la firma. La clave secreta se inyecta desde una propiedad de entorno para evitar exponerla en el repositorio.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-firmados
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
            - Method=POST,PUT
          filters:
            # 1. Limitar tamaño antes de cachear (evita OOM)
            - name: RequestSize
              args:
                maxSize: 1MB
            # 2. Cachear el body para que HmacValidation pueda leerlo
            - name: CacheRequestBody
              args:
                bodyClass: java.lang.String
            # 3. Validar la firma HMAC
            - name: HmacValidation
              args:
                secretKey: ${hmac.secret-key}
                signatureHeader: X-Signature
                algorithm: HmacSHA256
```

> [EXAMEN] El orden en la lista `filters` es el orden de ejecución: `RequestSize` rechaza payloads grandes antes de que `CacheRequestBody` intente cargarlos en memoria, y `CacheRequestBody` almacena el body antes de que `HmacValidation` intente leerlo. Invertir el orden provoca errores silenciosos o un `400` inesperado.

## Gestión segura de la clave secreta

La clave `${hmac.secret-key}` nunca debe estar hardcodeada en el YAML del repositorio. Las opciones en orden de preferencia:

```yaml
# Opción 1: variable de entorno (para contenedores y Kubernetes)
hmac:
  secret-key: ${HMAC_SECRET_KEY}

# Opción 2: Spring Cloud Config Server (centralizado, recargable)
# La clave vive en el repositorio cifrado del Config Server

# Opción 3: Spring Cloud Vault (para entornos con alta exigencia de seguridad)
spring:
  config:
    import: "vault://secret/gateway"
```

> [ADVERTENCIA] Si la clave secreta se loguea accidentalmente (por ejemplo, al imprimir el objeto `Config` en un log de debug), todos los sistemas que usan esa clave quedan comprometidos. Sobreescribir `toString()` en la clase `Config` para enmascarar la clave o no incluirla en la representación string.

## Filtros reactivos con llamadas a servicios externos

Cuando la validación requiere consultar un servicio externo (verificar una API key contra Redis, comprobar permisos en un servicio de autorización), el filtro debe ser completamente reactivo:

```java
@Component
public class ApiKeyValidationGatewayFilterFactory
        extends AbstractGatewayFilterFactory<ApiKeyValidationGatewayFilterFactory.Config> {

    private final ReactiveRedisTemplate<String, String> redisTemplate;

    public ApiKeyValidationGatewayFilterFactory(
            ReactiveRedisTemplate<String, String> redisTemplate) {
        super(Config.class);
        this.redisTemplate = redisTemplate;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String apiKey = exchange.getRequest().getHeaders()
                .getFirst("X-Api-Key");

            if (apiKey == null) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            // Consulta reactiva a Redis: no bloquea el hilo de Netty
            return redisTemplate.hasKey("api-keys:" + apiKey)
                .flatMap(exists -> {
                    if (!exists) {
                        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                        return exchange.getResponse().setComplete();
                    }
                    return chain.filter(exchange);
                })
                .switchIfEmpty(Mono.defer(() -> {
                    exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                    return exchange.getResponse().setComplete();
                }));
        };
    }

    public static class Config { /* sin parámetros */ }
}
```

> [CONCEPTO] El patrón `flatMap` + `switchIfEmpty` es el estándar reactivo para manejar la ausencia de un valor y cortocircuitar la cadena. `switchIfEmpty` se ejecuta cuando el `Mono` upstream completa sin emitir ningún elemento. En este contexto, equivale al `else` de un bloque `if`.

## Testing del filtro HMAC

Los tests arrancan el contexto completo con `@SpringBootTest` y usan `WebTestClient` para enviar peticiones reales al Gateway embebido. Se verifican los tres casos relevantes: firma válida calculada con el mismo algoritmo que el filtro, firma con valor incorrecto, y ausencia total del header de firma.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HmacValidationFilterTest {

    @Autowired
    private WebTestClient webTestClient;

    private static final String SECRET = "test-secret-key-for-tests-only";
    private static final String BODY = "{\"cantidad\": 5}";

    @Test
    void peticion_con_firma_valida_llega_al_servicio() throws Exception {
        String signature = calcularHmac(BODY, SECRET);

        webTestClient.post().uri("/api/pedidos")
            .header("X-Signature", signature)
            .bodyValue(BODY)
            .exchange()
            .expectStatus().isOk();  // el mock del servicio responde 200
    }

    @Test
    void peticion_con_firma_invalida_devuelve_401() {
        webTestClient.post().uri("/api/pedidos")
            .header("X-Signature", "firma-incorrecta")
            .bodyValue(BODY)
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    void peticion_sin_firma_devuelve_401() {
        webTestClient.post().uri("/api/pedidos")
            .bodyValue(BODY)
            .exchange()
            .expectStatus().isUnauthorized();
    }

    private String calcularHmac(String data, String key) throws Exception {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
        return Base64.getEncoder().encodeToString(
            mac.doFinal(data.getBytes(StandardCharsets.UTF_8)));
    }
}
```

## Tabla de consideraciones de seguridad

Cada decisión de implementación en un filtro HMAC tiene implicaciones de seguridad concretas. Los siguientes riesgos son los más frecuentes en implementaciones reales, junto con la mitigación específica que aplica en el contexto de Spring Cloud Gateway.

| Aspecto | Riesgo | Mitigación |
|---|---|---|
| Timing attack | `String.equals()` termina en cuanto encuentra una diferencia, revelando cuántos caracteres coinciden | Usar `MessageDigest.isEqual()` para comparación en tiempo constante |
| Replay attack | La misma firma válida puede reutilizarse en peticiones repetidas | Incluir un timestamp o nonce en el payload firmado y validar su frescura |
| Clave en repositorio | La clave secreta expuesta en Git compromete todos los clientes | Externalizar con Spring Cloud Config cifrado o Vault |
| Log de clave | Un log de debug puede imprimir el secreto | Sobreescribir `Config.toString()` para enmascarar `secretKey` |
| Body en memoria | Payloads grandes agotan la memoria del Gateway | Declarar `RequestSize` antes de `CacheRequestBody` |

## Buenas y malas prácticas

Hacer:
- Combinar siempre `RequestSize` → `CacheRequestBody` → `HmacValidation` en ese orden. El tamaño se verifica antes de cachear para evitar agotar la memoria con payloads maliciosos.
- Registrar los intentos fallidos de validación con nivel `WARN` incluyendo el path, el método y la IP de origen, pero sin incluir la firma recibida ni el body en el log.
- Separar la lógica de cálculo HMAC en un método privado testeable de forma unitaria, independiente del filtro completo.

Evitar:
- Comparar firmas con `String.equals()` o `Arrays.equals()`. Ambos métodos terminan en el primer byte diferente, lo que permite medir el tiempo de respuesta para inferir cuántos bytes de la firma son correctos (timing attack).
- Poner lógica de dominio que requiera acceso a base de datos en el filtro. Un filtro que hace consultas JDBC dentro de la cadena reactiva bloqueará los hilos de Netty y degradará todo el Gateway.
- Loguear en nivel DEBUG el secreto, la firma calculada o el body completo. Los logs de debug suelen activarse en producción durante incidentes, exponiendo secretos en los sistemas de logging centralizados.

---

← [6.4.2 GlobalFilter y Ordered](./06-19-gateway-global-filter.md) | [Índice](./README.md) | [6.5.1 Token Bucket y RequestRateLimiter →](./06-21-gateway-rate-limiter-config.md)
