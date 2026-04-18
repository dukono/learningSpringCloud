# 8.8 mTLS y seguridad avanzada en Spring Cloud Gateway

← [sc-security-propagacion-servicios.md](sc-security-propagacion-servicios.md) | [Índice](README.md) | [sc-security-testing.md](sc-security-testing.md) →

---

## Introducción

Mutual TLS (mTLS) eleva la seguridad de la comunicación entre servicios al requerir que tanto el servidor como el cliente presenten certificados X.509 válidos durante el handshake TLS. En una arquitectura de microservicios, esto garantiza que solo los servicios con certificados emitidos por la CA corporativa pueden comunicarse entre sí, independientemente de que tengan tokens OAuth2. mTLS protege contra ataques de hombre en el medio y suplantación de servicios, añadiendo una capa de identidad a nivel de transporte que complementa la identidad a nivel de aplicación (JWT).

> [PREREQUISITO] Comprender el flujo de autenticación OAuth2 (sc-security-token-relay-gateway.md). mTLS y OAuth2 son capas de seguridad complementarias: mTLS autentica el canal de transporte; OAuth2 autentica la aplicación o el usuario.

## Conceptos fundamentales de mTLS

En TLS estándar (unilateral), solo el cliente verifica el certificado del servidor. En mTLS (mutuo), el servidor también exige que el cliente presente un certificado válido firmado por una CA de confianza. El proceso de handshake mTLS incluye: (1) el servidor presenta su certificado; (2) el cliente verifica el certificado del servidor; (3) el servidor solicita el certificado del cliente; (4) el cliente presenta su certificado; (5) el servidor lo verifica contra su TrustStore.

```
Cliente                    Servidor (Spring Gateway)
  │                               │
  │──── ClientHello ────────────→ │
  │←─── ServerHello + Cert ─────── │  (certificado del servidor)
  │──── Verify ServerCert ──────→ │
  │──── ClientCert ─────────────→ │  (certificado del cliente)
  │←─── Verify ClientCert ──────── │
  │──── Finished ───────────────→ │
  │←─── Finished ───────────────── │
  │          TLS Handshake OK      │
  │──── HTTP Request ───────────→ │  canal cifrado + autenticado
```

## Certificate-bound tokens (RFC 8705)

RFC 8705 (OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens) define cómo vincular un Access Token al certificado cliente TLS. El token contiene el thumbprint del certificado en el claim `cnf.x5t#S256`. El Resource Server verifica que el certificado presentado en el TLS coincide con el thumbprint del claim del token. Esto impide el robo de tokens: incluso si alguien intercepta el token, no puede usarlo sin el certificado privado correspondiente.

> [CONCEPTO] Certificate-Bound Tokens vinculan criptográficamente el Access Token al certificado TLS del cliente. El claim `cnf` (Confirmation) del JWT contiene `x5t#S256` (thumbprint SHA-256 del certificado). Es la base del "Proof-of-Possession" de tokens.

## Ejemplo central: configuración mTLS en Spring Cloud Gateway

La configuración de mTLS en el Gateway involucra dos partes: (1) el servidor TLS que requiere certificado del cliente (para peticiones entrantes al Gateway), y (2) el cliente HTTP del Gateway que presenta certificado al llamar a microservicios internos (peticiones salientes).

```yaml
# application.yml del Gateway

server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:gateway-keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: gateway
    # mTLS: requerir certificado del cliente
    client-auth: need        # "need" = obligatorio | "want" = opcional
    trust-store: classpath:ca-truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12

spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          # Certificado que el Gateway presenta al llamar a microservicios internos
          key-store: classpath:gateway-keystore.p12
          key-store-password: ${KEYSTORE_PASSWORD}
          key-store-type: PKCS12
          # CA de confianza para verificar certificados de los microservicios
          trusted-x509-certificates:
            - classpath:ca-cert.pem
          # Deshabilitar verificación de hostname en red interna (solo dev)
          use-insecure-trust-manager: false
```

> [ADVERTENCIA] `use-insecure-trust-manager: true` deshabilita toda verificación de certificados. Solo usar en desarrollo local; nunca en producción. En producción, configurar siempre un `TrustStore` con la CA corporativa.

## Configuración programática de mTLS para el HttpClient del Gateway

Para mayor control sobre la configuración SSL del cliente HTTP del Gateway (Netty), se puede configurar programáticamente mediante `HttpClient`. Esto permite ajustar el protocolo TLS, los cipher suites, y el comportamiento de verificación de hostname.

```java
package com.example.gateway.config;

import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;
import org.springframework.cloud.gateway.config.HttpClientCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.netty.http.client.HttpClient;

import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.TrustManagerFactory;
import java.io.InputStream;
import java.security.KeyStore;

@Configuration
public class GatewayMtlsConfig {

    @Bean
    public HttpClientCustomizer mtlsHttpClientCustomizer() {
        return httpClient -> {
            try {
                // Cargar KeyStore con el certificado del Gateway (cliente mTLS)
                KeyStore keyStore = KeyStore.getInstance("PKCS12");
                try (InputStream ks = getClass().getResourceAsStream("/gateway-keystore.p12")) {
                    keyStore.load(ks, "keystorepass".toCharArray());
                }
                KeyManagerFactory kmf = KeyManagerFactory
                    .getInstance(KeyManagerFactory.getDefaultAlgorithm());
                kmf.init(keyStore, "keystorepass".toCharArray());

                // Cargar TrustStore con la CA de confianza
                KeyStore trustStore = KeyStore.getInstance("PKCS12");
                try (InputStream ts = getClass().getResourceAsStream("/ca-truststore.p12")) {
                    trustStore.load(ts, "truststorepass".toCharArray());
                }
                TrustManagerFactory tmf = TrustManagerFactory
                    .getInstance(TrustManagerFactory.getDefaultAlgorithm());
                tmf.init(trustStore);

                SslContext sslContext = SslContextBuilder.forClient()
                    .keyManager(kmf)
                    .trustManager(tmf)
                    .build();

                return httpClient.secure(ssl -> ssl.sslContext(sslContext));
            } catch (Exception e) {
                throw new RuntimeException("Error configurando mTLS en Gateway", e);
            }
        };
    }
}
```

## Tabla de propiedades spring.cloud.gateway.httpclient.ssl.*

Las propiedades del cliente HTTP del Gateway controlan cómo se conecta a los microservicios downstream.

| Propiedad | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `httpclient.ssl.key-store` | Resource | — | KeyStore con el certificado del Gateway |
| `httpclient.ssl.key-store-password` | String | — | Password del KeyStore |
| `httpclient.ssl.key-store-type` | String | `JKS` | Tipo: `JKS`, `PKCS12` |
| `httpclient.ssl.trusted-x509-certificates` | List | — | Certificados CA de confianza para servicios downstream |
| `httpclient.ssl.use-insecure-trust-manager` | boolean | `false` | Deshabilita verificación (solo dev) |
| `server.ssl.client-auth` | Enum | `none` | `need` = mTLS obligatorio, `want` = mTLS opcional |

## mTLS en comunicación interna de microservicios

En arquitecturas de producción, mTLS se implementa a nivel de service mesh (Istio, Linkerd) que gestiona automáticamente los certificados rotación y el handshake mTLS entre servicios, sin que el código de la aplicación necesite configurarlo. Spring Cloud Gateway + mTLS es la alternativa cuando no se usa service mesh.

> [EXAMEN] `server.ssl.client-auth=need` significa que el servidor rechaza conexiones TLS de clientes que no presenten certificado válido (HTTP 400 Bad Request durante el handshake). `client-auth=want` solicita el certificado pero no lo exige — la conexión continúa sin él.

## Buenas y malas prácticas

Hacer:
- Usar `server.ssl.client-auth=need` para comunicación interna entre servicios donde todos los clientes son conocidos.
- Almacenar passwords de KeyStore/TrustStore en variables de entorno o Config Server cifrado.
- Rotar certificados periódicamente y automatizar la rotación (Vault PKI, cert-manager en Kubernetes).
- Usar certificados diferentes para el servidor TLS del Gateway y para el cliente HTTP del Gateway.

Evitar:
- Usar `use-insecure-trust-manager: true` en entornos no locales — deshabilita toda verificación de certificados.
- Usar certificados autofirmados en producción — usar una CA corporativa o Vault PKI.
- Combinar `client-auth=need` con `client-auth=want` en el mismo servidor sin documentar explícitamente la política.

## Verificación y práctica

```bash
# Verificar mTLS con openssl desde el cliente
openssl s_client \
  -connect localhost:8443 \
  -cert client-cert.pem \
  -key client-key.pem \
  -CAfile ca-cert.pem \
  -verify_return_error

# Sin certificado de cliente (debe fallar con client-auth=need)
openssl s_client -connect localhost:8443 -CAfile ca-cert.pem
# Error: HANDSHAKE FAILURE (el servidor requiere certificado)

# Con curl y mTLS
curl --cert client-cert.pem \
     --key client-key.pem \
     --cacert ca-cert.pem \
     https://localhost:8443/api/health
```

**Preguntas estilo examen VMware Spring Professional:**

1. ¿Cuál es la diferencia entre `server.ssl.client-auth=need` y `server.ssl.client-auth=want`? ¿Qué ocurre en cada caso cuando el cliente no presenta certificado?

2. ¿Cómo se vincula un Access Token OAuth2 al certificado TLS del cliente según el RFC 8705? ¿Qué claim del JWT contiene esta vinculación?

3. ¿Qué propiedad del Gateway controla si el Gateway presenta un certificado propio al conectarse a microservicios internos (mTLS en peticiones salientes)?

---

← [sc-security-propagacion-servicios.md](sc-security-propagacion-servicios.md) | [Índice](README.md) | [sc-security-testing.md](sc-security-testing.md) →