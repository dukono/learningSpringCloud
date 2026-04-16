# 3.4.7 Filtros de seguridad y sesión (SecureHeaders, SaveSession, TokenRelay)

← [3.4.6 Retry: política de reintentos automáticos](sc-gateway-retry.md) | [Índice](README.md) | [3.5 GlobalFilters: filtros transversales y personalización](sc-gateway-globalfilters.md) →

---

## Introducción

Un gateway que enruta peticiones sin añadir cabeceras de seguridad HTTP expone a los clientes a ataques de clickjacking, XSS y sniffing de contenido. Además, en arquitecturas con sesión distribuida o SSO OAuth2, el gateway necesita participar activamente en la propagación del contexto de seguridad hacia los servicios backend. Los tres filtros de este fichero cubren ese espacio: `SecureHeadersGatewayFilterFactory` añade cabeceras defensivas estándar, `SaveSessionGatewayFilterFactory` persiste la sesión antes del proxy, y `TokenRelayGatewayFilterFactory` propaga el access token OAuth2 downstream.

> [PREREQUISITO] `TokenRelayGatewayFilterFactory` requiere que Spring Security OAuth2 Client esté configurado con un `ReactiveOAuth2AuthorizedClientManager`. Ver [8.6 Token Relay en Spring Cloud Gateway](sc-security-token-relay.md) para la configuración completa de OAuth2.

## Diagrama: orden de aplicación de los filtros de seguridad

El siguiente diagrama muestra cómo se encadenan los tres filtros en una ruta que los utiliza todos.

```
Petición entrante del cliente
          │
          ▼
  SecureHeadersFilter (PRE)
  → añade cabeceras de seguridad a la response (pendientes hasta POST)
          │
          ▼
  SaveSessionFilter (PRE)
  → persiste la WebSession antes de enviar al backend
  → garantiza que el backend recibe la sesión en estado consistente
          │
          ▼
  TokenRelayFilter (PRE)
  → lee el access token del ReactiveOAuth2AuthorizedClientManager
  → añade Authorization: Bearer <token> al request hacia el backend
          │
          ▼
       Backend
          │
          ▼
  SecureHeadersFilter (POST)
  → añade las cabeceras a la response que se devuelve al cliente
          │
          ▼
  Cliente recibe response con cabeceras de seguridad HTTP
```

## Ejemplo central

El siguiente ejemplo muestra la configuración de los tres filtros, con especial detalle en `TokenRelay` por ser el más complejo de configurar correctamente.

### Dependencias Maven

```xml
<!-- Para TokenRelayGatewayFilterFactory -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### Configuración YAML

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # Ruta con SecureHeaders: añade cabeceras defensivas a todas las responses
        - id: public-api-route
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - SecureHeaders   # sin parámetros; aplica el set estándar de cabeceras

        # Ruta con SaveSession: persistencia de sesión antes del proxy
        - id: session-route
          uri: lb://portal-service
          predicates:
            - Path=/portal/**
          filters:
            - SaveSession     # persiste WebSession antes de enviar al backend

        # Ruta con TokenRelay: propaga access token OAuth2 al backend
        - id: protected-api-route
          uri: lb://protected-service
          predicates:
            - Path=/protected/**
          filters:
            - TokenRelay=     # nombre del registration (vacío = usa el principal actual)

  # Configuración OAuth2 Client requerida por TokenRelay
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: gateway-client
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: authorization_code
            scope: openid, profile, email
        provider:
          keycloak:
            issuer-uri: https://keycloak.example.com/realms/myrealm
```

### Bean ReactiveOAuth2AuthorizedClientManager (requerido por TokenRelay)

```java
package com.example.gateway;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientProvider;
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientProviderBuilder;
import org.springframework.security.oauth2.client.registration.ReactiveClientRegistrationRepository;
import org.springframework.security.oauth2.client.web.DefaultReactiveOAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.web.server.ServerOAuth2AuthorizedClientRepository;

@Configuration
public class OAuth2Config {

    /**
     * Bean requerido por TokenRelayGatewayFilterFactory.
     * Sin este bean, TokenRelay lanza NoSuchBeanDefinitionException en el arranque.
     */
    @Bean
    public ReactiveOAuth2AuthorizedClientManager authorizedClientManager(
            ReactiveClientRegistrationRepository clientRegistrationRepository,
            ServerOAuth2AuthorizedClientRepository authorizedClientRepository) {

        ReactiveOAuth2AuthorizedClientProvider authorizedClientProvider =
            ReactiveOAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()
                .refreshToken()
                .clientCredentials()
                .build();

        DefaultReactiveOAuth2AuthorizedClientManager manager =
            new DefaultReactiveOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        manager.setAuthorizedClientProvider(authorizedClientProvider);
        return manager;
    }
}
```

### Personalización de SecureHeaders (override de valores por defecto)

```yaml
# application.yml — override de cabeceras SecureHeaders
spring:
  cloud:
    gateway:
      filter:
        secure-headers:
          # Sobreescribir valores por defecto
          content-security-policy: "default-src 'self'; script-src 'self' 'unsafe-inline'"
          x-frame-options: SAMEORIGIN    # default: DENY
          # Deshabilitar cabeceras individuales
          disable:
            - x-xss-protection           # deshabilitada porque CSP es suficiente
```

## Tabla de elementos clave

La siguiente tabla cubre los tres filtros con sus parámetros y las cabeceras que aplica `SecureHeaders`.

| Filtro | Parámetros | Efecto |
|--------|-----------|--------|
| `SecureHeaders` | ninguno | Añade cabeceras de seguridad HTTP estándar a la response |
| `SaveSession` | ninguno | Persiste `WebSession` antes del proxy hacia el backend |
| `TokenRelay` | clientRegistrationId (opcional) | Añade `Authorization: Bearer <token>` al request hacia el backend |

Cabeceras añadidas por `SecureHeaders` (valores por defecto):

| Cabecera | Valor por defecto | Protección |
|----------|-------------------|------------|
| `X-Xss-Protection` | `1 ; mode=block` | Activa filtro XSS del navegador |
| `Strict-Transport-Security` | `max-age=631138519` | Fuerza HTTPS por ~20 años |
| `X-Frame-Options` | `DENY` | Previene clickjacking |
| `X-Content-Type-Options` | `nosniff` | Previene MIME sniffing |
| `Referrer-Policy` | `no-referrer` | Controla el header Referer |
| `Content-Security-Policy` | política restrictiva | Controla fuentes de scripts y recursos |
| `X-Download-Options` | `noopen` | IE: evita abrir descargas directamente |
| `X-Permitted-Cross-Domain-Policies` | `none` | Bloquea políticas cross-domain de Flash |

> [EXAMEN] `TokenRelayGatewayFilterFactory` requiere que el usuario esté autenticado a nivel de Spring Security antes de que el filtro actúe. Si el usuario no está autenticado, no hay token que propagar y el filtro no añade el header `Authorization`. La autenticación del usuario (flujo OAuth2 authorization code) es responsabilidad de Spring Security, no del filtro.

## Buenas y malas prácticas

**Hacer:**
- Aplicar `SecureHeaders` como `default-filter` a nivel global en lugar de por ruta individual: las cabeceras de seguridad HTTP deben proteger todas las responses, no solo algunas.
- Verificar la compatibilidad del `Content-Security-Policy` por defecto de `SecureHeaders` con la aplicación frontend: la política restrictiva puede bloquear scripts inline o recursos de CDN, produciendo páginas en blanco sin error explícito.
- Usar `SaveSession` en rutas que necesitan acceso a la sesión Spring en el controlador backend: sin él, los cambios de sesión en el gateway (ej: atributos añadidos por filtros) no se persisten antes del proxy.

**Evitar:**
- Usar `TokenRelay` sin configurar `ReactiveOAuth2AuthorizedClientManager`: el filtro lanzará error en el arranque o en la primera petición si no existe el bean requerido.
- Deshabilitar `Strict-Transport-Security` en producción: permite ataques de downgrade de HTTPS a HTTP en la misma sesión de usuario.
- Configurar `TokenRelay` en rutas públicas (sin autenticación): si no hay token disponible, el filtro puede lanzar un error 500 en lugar de simplemente no añadir el header; usar siempre con rutas protegidas por Spring Security.

---

← [3.4.6 Retry: política de reintentos automáticos](sc-gateway-retry.md) | [Índice](README.md) | [3.5 GlobalFilters: filtros transversales y personalización](sc-gateway-globalfilters.md) →
