# 6.1.5 SSL/TLS: terminación y passthrough

← [6.1.4 Propiedades globales](./06-04-gateway-propiedades-globales.md) | [Índice](./README.md) | [6.1.6 WebSocket y SSE →](./06-06-gateway-websocket.md)

---

El Gateway es el único componente de la arquitectura expuesto directamente a internet, y por tanto el lugar natural para gestionar HTTPS. Existen dos estrategias distintas: **terminación SSL**, donde el Gateway descifra el tráfico y reenvía HTTP plano a los backends, y **SSL passthrough**, donde el Gateway reenvía el tráfico cifrado sin abrirlo y el backend es quien termina TLS. La elección depende de si la red interna entre Gateway y backends es de confianza y de si las políticas de seguridad exigen que ningún componente intermediario pueda leer el contenido.

---

## Terminación vs passthrough: flujo comparado

En la terminación SSL el Gateway descifra la conexión TLS del cliente y establece una conexión HTTP plana hacia el backend, de modo que el certificado reside únicamente en el Gateway. En el passthrough el Gateway reenvía el tráfico cifrado sin abrirlo, transfiriendo la responsabilidad del certificado al propio backend y renunciando a la capacidad de inspeccionar o modificar la petición.

```
TERMINACIÓN SSL (más habitual)

  Cliente                 Gateway                 Backend
     │                      │                       │
     │──HTTPS (TLS)─────────►│                       │
     │  cifrado               │ descifra aquí         │
     │                        │──HTTP plano──────────►│
     │                        │◄─HTTP plano───────────│
     │◄─HTTPS (TLS)───────────│                       │
     │  cifrado de vuelta      │                       │

  Gateway gestiona el certificado.
  Backend no necesita TLS. Red interna debe ser de confianza.


PASSTHROUGH

  Cliente                 Gateway                 Backend
     │                      │                       │
     │──HTTPS (TLS)─────────►│                       │
     │  cifrado               │ no descifra           │
     │                        │──HTTPS (TLS)─────────►│
     │                        │  cifrado               │ termina TLS aquí
     │                        │◄─HTTPS (TLS)───────────│
     │◄─HTTPS (TLS)───────────│                        │

  Backend gestiona el certificado.
  Gateway no puede leer ni modificar el contenido.
  Los filtros HTTP no aplican en este modo.
```

---

## Passthrough: configuración

En passthrough el Gateway no termina TLS: actúa como proxy TCP y reenvía la conexión cifrada directamente al backend. Para conseguirlo, el Gateway se configura **sin** `server.ssl` (no termina HTTPS en su propio puerto) y la `uri` de la ruta apunta al endpoint HTTPS del backend. El Gateway establece una conexión TLS con el backend igual que haría con cualquier cliente HTTPS.

```yaml
# application.yml — Gateway en modo passthrough (sin server.ssl)
server:
  port: 443    # recibe la conexión, pero NO la descifra

spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          # El backend usa un certificado de CA interna
          trusted-x509-certificates:
            - classpath:internal-ca.crt
      routes:
        - id: backend-passthrough
          uri: https://backend-service:8443    # HTTPS directo, no lb:// en passthrough puro
          predicates:
            - Path=/api/seguro/**
```

> [ADVERTENCIA] En modo passthrough el Gateway no puede aplicar ningún filtro que lea o modifique el contenido HTTP (headers, body, path rewriting basado en contenido) porque el tráfico llega cifrado. Los filtros `AddRequestHeader`, `StripPrefix` y equivalentes se aplican al handshake TCP inicial, no a la petición HTTP que viaja dentro del túnel TLS.

---

## Terminación SSL: configuración completa

Para habilitar HTTPS en el Gateway se configuran las propiedades `server.ssl.*` estándar de Spring Boot. El Gateway actúa como servidor HTTPS para los clientes y como cliente HTTP hacia los backends.

```yaml
# application.yml
server:
  port: 443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    key-alias: gateway-cert

spring:
  cloud:
    gateway:
      routes:
        - id: productos-route
          uri: lb://productos-service   # HTTP plano hacia el backend
          predicates:
            - Path=/api/productos/**
          filters:
            - StripPrefix=1
```

Para generar un keystore de desarrollo autofirmado:

```bash
keytool -genkeypair \
  -alias gateway-cert \
  -keyalg RSA \
  -keysize 2048 \
  -storetype PKCS12 \
  -keystore src/main/resources/keystore.p12 \
  -storepass changeit \
  -validity 365 \
  -dname "CN=localhost, OU=Dev, O=MiEmpresa, L=Madrid, C=ES"
```

### Redirect HTTP → HTTPS

Para forzar que el tráfico HTTP del puerto 80 se redirija automáticamente al puerto 443, se registra un segundo conector HTTP en la misma aplicación. Este enfoque requiere `spring-boot-starter-security` en el classpath, ya que `SecurityWebFilterChain` y `ServerHttpSecurity` pertenecen a Spring Security WebFlux:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```java
@Configuration
public class HttpsRedirectConfig {

    @Bean
    public WebServerFactoryCustomizer<NettyReactiveWebServerFactory> httpToHttpsRedirect() {
        return factory -> factory.addServerCustomizers(httpServer ->
            httpServer.port(8080)   // puerto HTTP adicional
        );
    }

    @Bean
    public SecurityWebFilterChain redirectFilter(ServerHttpSecurity http) {
        return http
            .redirectToHttps(redirect -> redirect
                .httpsRedirectWhen(exchange ->
                    exchange.getRequest().getURI().getScheme().equals("http")))
            .build();
    }
}
```

---

## Confianza en backends con certificados propios

Cuando los backends internos usan HTTPS con certificados autofirmados o de una CA interna, el Gateway (como cliente HTTP) debe configurarse para confiar en ellos.

Para **desarrollo** (acepta cualquier certificado — nunca en producción):

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          use-insecure-trust-manager: true
```

Para **producción** con CA interna:

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          trusted-x509-certificates:
            - classpath:internal-ca.crt   # certificado de la CA que firmó los backends
```

> [ADVERTENCIA] `use-insecure-trust-manager: true` desactiva toda verificación de certificado en las conexiones salientes del Gateway. Un atacante con acceso a la red interna puede presentar cualquier certificado y el Gateway lo aceptará. Esta propiedad solo es aceptable en entornos de desarrollo aislados.

---

## mTLS: autenticación mutua

En mTLS (mutual TLS), el cliente también presenta un certificado que el servidor verifica. Es el mecanismo habitual para autenticar máquina a máquina (otro servicio que llama al Gateway) sin necesidad de tokens en cabeceras.

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    trust-store: classpath:truststore.p12       # CA que firmó los certificados de clientes
    trust-store-password: changeit
    trust-store-type: PKCS12
    client-auth: NEED    # NEED = certificado de cliente obligatorio
                         # WANT = opcional (la petición pasa aunque no haya certificado)
```

Con `client-auth: NEED`, cualquier petición sin certificado de cliente válido es rechazada en la capa TLS, antes de llegar a los predicates o filtros del Gateway.

---

## Parámetros

Las propiedades `server.ssl.*` controlan cómo el Gateway expone su propio endpoint HTTPS a los clientes, mientras que las propiedades `spring.cloud.gateway.httpclient.ssl.*` controlan cómo el Gateway se comporta como cliente HTTP cuando se conecta a los backends.

| Propiedad | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `server.ssl.enabled` | boolean | `false` | Activa HTTPS en el servidor |
| `server.ssl.key-store` | String | — | Ruta al keystore (classpath: o file:) |
| `server.ssl.key-store-password` | String | — | Contraseña del keystore |
| `server.ssl.key-store-type` | String | `JKS` | Tipo de keystore: `PKCS12` o `JKS` |
| `server.ssl.key-alias` | String | — | Alias del certificado dentro del keystore |
| `server.ssl.trust-store` | String | — | Ruta al truststore para mTLS |
| `server.ssl.trust-store-password` | String | — | Contraseña del truststore |
| `server.ssl.client-auth` | enum | `NONE` | `NONE`, `WANT`, `NEED` |
| `spring.cloud.gateway.httpclient.ssl.use-insecure-trust-manager` | boolean | `false` | Desactiva verificación de certificado en backends (solo desarrollo) |
| `spring.cloud.gateway.httpclient.ssl.trusted-x509-certificates` | list | — | Certificados de CA interna para confiar en backends |

---

## Buenas y malas prácticas

Hacer:
- Usar `PKCS12` en lugar de `JKS` para keystores nuevos. JKS es el formato legacy de Java; PKCS12 es el estándar actual, compatible con OpenSSL y con herramientas de gestión de certificados corporativas.
- Externalizar las contraseñas del keystore con Spring Cloud Config o variables de entorno. Las contraseñas en `application.yml` commiteado en el repositorio son una vulnerabilidad de seguridad común.
- Configurar `server.http2.enabled: true` junto con SSL para aprovechar HTTP/2 con TLS, que mejora el rendimiento mediante multiplexación de peticiones sobre la misma conexión TCP.

Evitar:
- Dejar `use-insecure-trust-manager: true` en configuraciones que puedan llegar a staging o producción. Un perfil `dev` que accidentalmente se activa en producción desactiva toda la verificación de certificados en las conexiones salientes del Gateway.
- Usar certificados autofirmados directamente en producción. Un certificado de una CA de confianza (Let's Encrypt, o la CA corporativa interna) permite la verificación automática por parte de los clientes sin distribuir manualmente el certificado.

---

## Comparación: terminación SSL vs passthrough vs mTLS

Cada estrategia ofrece un compromiso diferente entre dónde reside el certificado, qué visibilidad tiene el Gateway sobre el contenido y qué tipo de autenticación del cliente es posible. La elección determina qué filtros HTTP pueden aplicarse y si la red interna necesita cifrado propio.

| Aspecto | Terminación SSL | Passthrough | mTLS |
|---|---|---|---|
| Quién termina TLS | Gateway | Backend | Gateway (con verificación del cliente) |
| Filtros HTTP aplicables | Sí | No — Gateway no lee el contenido | Sí |
| Backends necesitan certificado | No | Sí | No |
| Autenticación del cliente | No (solo del servidor) | Depende del backend | Sí (certificado de cliente) |
| Visibilidad del tráfico en Gateway | Sí | No | Sí |
| Caso de uso típico | Producción general | Cumplimiento estricto (PCI-DSS, datos médicos) | Comunicación servicio a servicio autenticada |

> [EXAMEN] La diferencia clave entre terminación SSL y passthrough no es de seguridad absoluta sino de dónde reside la responsabilidad del certificado y si el Gateway puede inspeccionar el tráfico. En passthrough el Gateway no puede aplicar ningún filtro HTTP (AddRequestHeader, JWT validation, rate limiting basado en body) porque no tiene acceso al contenido.

---

← [6.1.4 Propiedades globales](./06-04-gateway-propiedades-globales.md) | [Índice](./README.md) | [6.1.6 WebSocket y SSE →](./06-06-gateway-websocket.md)
