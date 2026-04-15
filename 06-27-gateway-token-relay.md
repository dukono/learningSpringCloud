# 6.6.4 TokenRelay: propagación del Bearer token al microservicio

← [6.6.3 Spring Security OAuth2](./06-26-gateway-oauth2.md) | [Índice](./README.md) | [6.7.1 Estrategia de testing →](./06-28-gateway-testing-estrategia.md)

---

El Gateway valida el JWT y protege los microservicios, pero hay un escenario donde el microservicio también necesita el token: cuando ese microservicio a su vez llama a otro microservicio o a una API externa en nombre del usuario. Sin el token original, el microservicio no puede propagar la identidad del usuario a la cadena de llamadas. El filtro `TokenRelay` resuelve este problema reenviando automáticamente el Bearer token del cliente al microservicio de destino como parte de la petición.

## Diagrama: flujo con TokenRelay

El diagrama ilustra el escenario donde el Microservicio A recibe el Bearer token reenviado por el Gateway y lo utiliza a su vez para llamar al Microservicio B, formando una cadena de propagación de identidad. Sin `TokenRelay`, la petición llega al Microservicio A sin ningún token y la llamada downstream al Microservicio B no puede acreditar la identidad del usuario original.

```
Cliente
  │  Authorization: Bearer eyJhbGc... (token del usuario)
  ▼
Gateway
  │  Valida el token (6.6.2 ó 6.6.3)
  │  TokenRelay: reenvía el mismo token
  ▼
Microservicio A
  │  Recibe: Authorization: Bearer eyJhbGc...
  │  Necesita llamar a Microservicio B
  │  Usa WebClient con el token recibido
  ▼
Microservicio B
  │  Recibe: Authorization: Bearer eyJhbGc...
  │  Verifica el token (o confía en la red interna)
  ▼
  Respuesta
```

Sin `TokenRelay`, el Microservicio A recibe la petición sin el token y no puede propagarlo a llamadas downstream.

## Dependencia requerida

`TokenRelay` requiere el starter OAuth2 Client, que gestiona el ciclo de vida de los tokens (incluyendo refresh si el token expira):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

> [ADVERTENCIA] `TokenRelay` solo funciona cuando el Gateway está configurado como OAuth2 Resource Server (6.6.3). No funciona con el filtro `JwtAuthFilter` personalizado (6.6.2) porque requiere el `SecurityContext` de Spring Security para obtener el token autenticado. Si se usa el filtro JJWT manual, hay que propagar el token manualmente como se describe al final de este documento.

## Configuración como default-filter (aplica a todas las rutas)

Cuando todos los microservicios del sistema necesitan recibir el token del usuario —porque cada uno propaga la identidad a sus llamadas downstream— se declara `TokenRelay` en `default-filters`. Con esta configuración, todas las rutas del Gateway reenvían automáticamente el Bearer token sin necesidad de declararlo en cada ruta individualmente.

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=
      routes:
        - id: productos-route
          uri: lb://productos-service
          predicates:
            - Path=/api/productos/**
```

## Configuración por ruta individual

Cuando solo algunos microservicios necesitan el token —por ejemplo, `pedidos-service` necesita la identidad del usuario para auditoría, pero `catalogo-service` es de solo lectura y no la necesita— se declara `TokenRelay` únicamente en la ruta correspondiente para evitar el overhead de reenviar el token donde no aporta valor.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: pedidos-con-relay
          uri: lb://pedidos-service
          predicates:
            - Path=/api/pedidos/**
          filters:
            - TokenRelay=   # reenvía el Bearer token a pedidos-service

        - id: catalogo-sin-relay
          uri: lb://catalogo-service
          predicates:
            - Path=/api/catalogo/**
          # sin TokenRelay: catalogo-service no recibirá el token
```

## Configuración del cliente OAuth2

`TokenRelay` necesita que el Gateway esté registrado como cliente OAuth2 para gestionar los tokens de acceso (incluyendo su renovación):

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          gateway-client:
            client-id: ${OAUTH2_CLIENT_ID}
            client-secret: ${OAUTH2_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          gateway-client:
            issuer-uri: https://auth.miempresa.com
      resourceserver:
        jwt:
          issuer-uri: https://auth.miempresa.com
```

> [CONCEPTO] `TokenRelay` propaga el token de acceso del usuario actualmente autenticado, no el token del Gateway. El Gateway actúa como relay (intermediario): recoge el token que el cliente envió, lo valida como Resource Server, y lo reenvía al microservicio como Bearer token. El microservicio recibe el token original del usuario.

## Renovación automática de tokens

Cuando el token de acceso ha expirado y el cliente tiene un refresh token, `TokenRelay` puede renovar el token automáticamente antes de reenviarlo, si se configura el `ReactiveOAuth2AuthorizedClientManager`:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientProviderBuilder;
import org.springframework.security.oauth2.client.registration.ReactiveClientRegistrationRepository;
import org.springframework.security.oauth2.client.web.DefaultReactiveOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.web.server.ServerOAuth2AuthorizedClientRepository;

@Configuration
public class TokenRelayConfig {

    @Bean
    public ReactiveOAuth2AuthorizedClientManager authorizedClientManager(
            ReactiveClientRegistrationRepository clientRegistrationRepository,
            ServerOAuth2AuthorizedClientRepository authorizedClientRepository) {

        var provider = ReactiveOAuth2AuthorizedClientProviderBuilder.builder()
            .authorizationCode()
            .refreshToken()  // habilita la renovación automática con refresh token
            .clientCredentials()
            .build();

        var manager = new DefaultReactiveOAuth2AuthorizedClientManager(
            clientRegistrationRepository, authorizedClientRepository);
        manager.setAuthorizedClientProvider(provider);
        return manager;
    }
}
```

## Propagación manual del token (sin Spring Security)

Cuando se usa el filtro `JwtAuthFilter` personalizado (sin Spring Security), el token debe propagarse manualmente desde el `GlobalFilter`:

```java
// En el JwtAuthFilter personalizado, después de validar el token,
// añadir el token original a la petición para que los microservicios lo reciban:
ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
    .headers(headers -> {
        // Mantener el Authorization header original para que el microservicio lo reciba
        // (ya está en la petición, no es necesario añadirlo manualmente)
        headers.add("X-User-Id", userId);
        headers.add("X-User-Roles", roles);
    })
    .build();
```

En este caso, el header `Authorization: Bearer <token>` original ya está en la petición y se reenvía automáticamente al microservicio sin ninguna acción adicional. No hay renovación automática: si el token expira entre la petición del cliente y la llamada del microservicio downstream, el microservicio recibirá un token expirado.

## Client Credentials: token del Gateway, no del usuario

En flujos machine-to-machine donde el Gateway llama a servicios internos en su propio nombre (no en nombre de un usuario), se usa el flujo Client Credentials:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          gateway-m2m:
            client-id: ${M2M_CLIENT_ID}
            client-secret: ${M2M_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: internal.read, internal.write
```

```java
// En un GlobalFilter que añade el token de Gateway (no del usuario):
return authorizedClientManager.authorize(OAuth2AuthorizeRequest
    .withClientRegistrationId("gateway-m2m")
    .principal("gateway")
    .build())
    .flatMap(client -> {
        String token = client.getAccessToken().getTokenValue();
        var mutated = exchange.getRequest().mutate()
            .header("Authorization", "Bearer " + token)
            .build();
        return chain.filter(exchange.mutate().request(mutated).build());
    });
```

## Tabla comparativa: TokenRelay vs propagación manual vs Client Credentials

Los tres mecanismos difieren en qué token recibe el microservicio de destino y en si ese token puede renovarse automáticamente. `TokenRelay` reenvía el token del usuario con soporte de renovación vía refresh token; la propagación manual también reenvía el token del usuario pero sin renovación; y Client Credentials emite un token propio del Gateway para flujos machine-to-machine donde no existe un usuario autenticado.

| Aspecto | TokenRelay | Propagación manual del header | Client Credentials |
|---|---|---|---|
| Token propagado | Token del usuario original | Token del usuario original | Token del Gateway (servicio) |
| Renovación automática | Sí (con ReactiveOAuth2AuthorizedClientManager) | No | Sí |
| Requiere Spring Security | Sí (OAuth2 Resource Server) | No | Sí |
| Caso de uso | API pública donde el microservicio necesita la identidad del usuario | Gateway con filtro JWT manual | Llamadas internas sin usuario (cron jobs, pipelines) |
| Configuración | YAML (`TokenRelay=`) | No necesaria (header ya presente) | Bean `ReactiveOAuth2AuthorizedClientManager` |

## Buenas y malas prácticas

Hacer:
- Usar `TokenRelay` como `default-filter` solo si todos los microservicios necesitan el token. Para microservicios de solo lectura que no propagan identidad, el overhead de reenviar el token es innecesario.
- Configurar TTL cortos en los tokens de acceso (15-30 minutos) y usar refresh tokens para cadenas de llamadas largas. Si el token expira durante una operación de larga duración, `TokenRelay` con renovación automática garantiza que las llamadas downstream siempre tienen un token válido.

Evitar:
- Loguear el header `Authorization` en los logs de acceso del Gateway o de los microservicios. El token Bearer es una credencial equivalente a una contraseña.
- Configurar `TokenRelay` en rutas que apuntan a servicios externos no confiables. El token del usuario se enviaría a un tercero, que podría usarlo para llamadas no autorizadas en nombre del usuario.

---

← [6.6.3 Spring Security OAuth2](./06-26-gateway-oauth2.md) | [Índice](./README.md) | [6.7.1 Estrategia de testing →](./06-28-gateway-testing-estrategia.md)
