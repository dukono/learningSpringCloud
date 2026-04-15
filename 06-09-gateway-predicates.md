# 6.2.2 Catálogo de Predicates: Path, Method, Header, Query, Host, After/Before/Between, Cookie, RemoteAddr, XForwardedRemoteAddr

← [6.2.1 Modelo de Route](./06-08-gateway-route-modelo.md) | [Índice](./README.md) | [6.2.3 Configuración en YAML →](./06-10-gateway-rutas-yaml.md)

---

Sin Predicates, el Gateway no sabría distinguir qué peticiones pertenecen a qué ruta. Los **Predicates** son las condiciones que una petición entrante debe cumplir para que el Gateway la considere candidata de una ruta concreta. Cada Predicate inspecciona un atributo distinto de la petición HTTP: su path, su método, sus headers, sus query params, el hostname, la IP del cliente o la ventana temporal en que llega. Cuando todos los Predicates de una ruta se evalúan como verdadero, el Gateway aplica sus filtros y enruta la petición al servicio destino.

> [CONCEPTO] En YAML, varios Predicates declarados en la misma ruta se combinan siempre con AND lógico. No existe sintaxis YAML para OR. Para lógica OR se declaran rutas separadas apuntando al mismo `uri`, o se usa el Java DSL (ver sección de comparación al final).

> [CONCEPTO] Este catálogo cubre los predicates de condición de petición. El predicate `Weight`, usado para distribución de tráfico canary y A/B testing, se cubre en [6.2.5](./06-12-gateway-canary-forward.md) junto al modelo completo de canary deployment. Cuando ningún predicate de ninguna ruta coincide, el Gateway devuelve **404** directamente sin llegar a ningún servicio.

---

## Categorías de Predicates

Los Predicates disponibles se agrupan según el atributo de la petición HTTP que inspeccionan.

```
Petición HTTP entrante
       │
       ├── Path URL ─────────────── Path=/api/**
       │
       ├── Método HTTP ──────────── Method=GET,POST
       │
       ├── Headers ─────────────── Header=Authorization, Bearer .+
       │
       ├── Cookies ─────────────── Cookie=session, [a-f0-9]+
       │
       ├── Query string ─────────── Query=format, json|xml
       │
       ├── Host (header Host) ───── Host=**.miempresa.com
       │
       ├── Ventana temporal ─────── After=2025-01-01T00:00:00+01:00[Europe/Madrid]
       │                            Before=2025-12-31T23:59:59+01:00[Europe/Madrid]
       │                            Between=2025-01-01T..., 2025-06-30T...
       │
       └── IP del cliente ───────── RemoteAddr=10.0.0.0/8
                                    XForwardedRemoteAddr=203.0.113.0/24
```

---

## Predicates en detalle

### Path

Evalúa el path de la URL usando patrones Ant. Es el predicate más habitual: prácticamente toda ruta lo incluye como primer filtro de selección.

```yaml
predicates:
  - Path=/api/productos/**         # /api/productos/123, /api/productos/abc/detalle
  - Path=/static/?                 # /static/a (exactamente un segmento extra)
  - Path=/api/{version}/items      # captura {version} como variable de path
  - Path=/api/v1/**,/api/v2/**     # varios patrones en el mismo predicate (OR entre patrones)
```

La distinción entre `*` y `**` es importante: `*` coincide con exactamente un segmento de path (`/api/v1` pero no `/api/v1/items`), mientras que `**` coincide con uno o varios segmentos (`/api/v1`, `/api/v1/items`, `/api/v1/items/42`). En la práctica casi siempre se usa `**` para rutas de servicio.

Las variables capturadas (`{version}`) quedan disponibles en los filtros de la misma ruta bajo la clave `ServerWebExchange.getAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE)`.

### Method

Filtra por método HTTP. Acepta uno o varios métodos separados por coma.

```yaml
predicates:
  - Method=GET,HEAD          # solo lectura
  - Method=POST,PUT,PATCH    # escritura
  - Method=DELETE            # eliminación
```

> [ADVERTENCIA] `Method` distingue mayúsculas y minúsculas. `Method=get` no funciona; el valor debe estar en mayúsculas (`GET`, `POST`, etc.).

### Header

Comprueba que la petición incluye un header cuyo valor cumple una expresión regular Java. Si se omite la regex, solo verifica que el header existe con cualquier valor.

```yaml
predicates:
  - Header=X-Request-Id, ^\d+$         # header numérico obligatorio (regex anclada: valor completo)
  - Header=Authorization, Bearer .+    # token Bearer
  - Header=X-Internal-Token            # solo comprueba presencia, sin validar valor
```

### Cookie

Comprueba que la petición incluye una cookie cuyo valor cumple una expresión regular.

```yaml
predicates:
  - Cookie=session, [a-f0-9]{32}    # cookie de sesión hexadecimal de 32 caracteres
  - Cookie=consent, true            # cookie de consentimiento aceptado
  - Cookie=ab-group, (A|B)          # usuario asignado al grupo A o B de un test A/B
```

### Query

Evalúa la presencia y el valor de un query parameter. La regex es opcional.

```yaml
predicates:
  - Query=debug                  # el parámetro existe con cualquier valor
  - Query=version, v[12]         # version=v1 o version=v2
  - Query=format, json|xml       # format=json o format=xml
```

### Host

Compara el header `Host` de la petición contra un patrón glob. El doble asterisco `**` coincide con cualquier número de subdominios.

```yaml
predicates:
  - Host=**.miempresa.com                    # api.miempresa.com, v2.api.miempresa.com
  - Host=**.miempresa.com,**.miempresa.es    # múltiples dominios en el mismo predicate
  - Host=localhost                           # desarrollo local sin dominio
```

### After, Before, Between

Condicionan la ruta a una ventana temporal. Las fechas usan formato ISO-8601 con zona horaria explícita (`ZonedDateTime`).

```yaml
predicates:
  # La ruta solo aplica A PARTIR de esta fecha
  - After=2025-06-01T00:00:00+02:00[Europe/Madrid]

  # La ruta solo aplica HASTA esta fecha
  - Before=2025-06-30T23:59:59+02:00[Europe/Madrid]

  # La ruta aplica dentro de la ventana temporal
  - Between=2025-06-01T00:00:00+02:00[Europe/Madrid], 2025-06-30T23:59:59+02:00[Europe/Madrid]
```

> [EXAMEN] La zona horaria es obligatoria. Omitirla hace que Spring falle al arrancar con un error de parseo de fecha. Usar `+02:00[Europe/Madrid]` en lugar de `Z` (UTC) evita sorpresas en cambios de horario de verano/invierno.

### RemoteAddr

Evalúa la IP del cliente directamente desde la conexión TCP. Acepta IPs individuales o rangos CIDR.

```yaml
predicates:
  - RemoteAddr=192.168.1.0/24    # red corporativa
  - RemoteAddr=10.0.0.5          # IP única
  - RemoteAddr=10.0.0.0/8,172.16.0.0/12    # varios rangos
```

### XForwardedRemoteAddr

Igual que `RemoteAddr`, pero lee la IP del header `X-Forwarded-For` en lugar de la conexión TCP. Necesario cuando el Gateway está detrás de un proxy o load balancer externo que añade ese header.

```yaml
predicates:
  - XForwardedRemoteAddr=203.0.113.0/24    # rango de IPs del cliente final según el proxy
```

> [ADVERTENCIA] `XForwardedRemoteAddr` solo es seguro si el proxy externo es de confianza y sobreescribe el header `X-Forwarded-For`. Si un cliente puede inyectar ese header directamente, puede suplantar cualquier IP y eludir el filtro.

**Regla de elección entre RemoteAddr y XForwardedRemoteAddr:**

| Situación | Predicate a usar |
|---|---|
| El Gateway está expuesto directamente a internet (primer hop) | `RemoteAddr` — la IP TCP es la del cliente real |
| Hay un proxy, CDN o load balancer externo delante del Gateway | `XForwardedRemoteAddr` — la IP TCP es la del proxy, no del cliente |
| El proxy externo no es de confianza o el header puede ser inyectado | Ninguno — usar autenticación en su lugar |

---

## Parámetros por Predicate

La tabla siguiente resume los parámetros que acepta cada Predicate, su tipo y las notas más relevantes para su uso correcto, incluyendo los casos donde la zona horaria o el formato de la expresión regular son críticos.

| Predicate | Parámetros | Notas |
|---|---|---|
| `Path` | patterns (1+, Ant glob) | Varios patrones = OR entre ellos |
| `Method` | methods (1+, mayúsculas) | `GET`, `POST`, `PUT`, `DELETE`... |
| `Header` | name, regexp (opcional) | Regex Java; sin regexp solo verifica presencia |
| `Cookie` | name, regexp | Regex Java obligatoria |
| `Query` | param, regexp (opcional) | Sin regexp solo verifica presencia del param |
| `Host` | patterns (1+, glob) | `**` coincide con múltiples segmentos de dominio |
| `After` | datetime (ZonedDateTime) | ISO-8601 con zona horaria obligatoria |
| `Before` | datetime (ZonedDateTime) | ISO-8601 con zona horaria obligatoria |
| `Between` | datetime1, datetime2 | Ambas con zona horaria obligatoria |
| `RemoteAddr` | sources (1+, IP o CIDR) | Lee IP de la conexión TCP |
| `XForwardedRemoteAddr` | sources (1+, IP o CIDR) | Lee IP del header `X-Forwarded-For` |

---

## Ejemplo completo: ruta de administración con múltiples Predicates

Una ruta de producción combina varios Predicates para reducir la superficie de ataque. Este ejemplo restringe el acceso al panel de administración por path, método, header de autenticación interna y red corporativa.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: admin-route
          uri: lb://admin-service
          predicates:
            - Path=/admin/**                           # solo rutas de administración
            - Method=GET,POST,DELETE                   # operaciones de lectura y gestión
            - Header=X-Internal-Token, [a-f0-9]{40}   # token interno de 40 caracteres hex
            - RemoteAddr=10.0.0.0/8                   # solo desde red corporativa
          filters:
            - StripPrefix=1

        - id: campaña-verano
          uri: lb://productos-service
          predicates:
            - Path=/api/ofertas/**
            - Between=2025-06-21T00:00:00+02:00[Europe/Madrid], 2025-09-22T23:59:59+02:00[Europe/Madrid]
          filters:
            - StripPrefix=2
```

En la ruta `admin-route`, los cuatro Predicates deben cumplirse simultáneamente. Una petición `DELETE /admin/users/42` desde `10.0.5.3` con el header `X-Internal-Token: a3f9c1...` pasa todos los controles. La misma petición desde una IP externa falla en `RemoteAddr` y el Gateway devuelve 404, sin revelar que la ruta existe.

La ruta `campaña-verano` se activa y desactiva automáticamente según la fecha del servidor, sin necesidad de redesplegar ni modificar configuración.

---

## Buenas y malas prácticas

Hacer:
- Colocar `Path` como primer Predicate en cada ruta. El Gateway evalúa los Predicates en orden; `Path` descarta rápidamente las peticiones que no encajan antes de evaluar condiciones más costosas como `Header` o `RemoteAddr`.
- Usar `RemoteAddr` junto a `Header` en rutas internas sensibles: dos barreras independientes reducen el riesgo si una de ellas falla por configuración incorrecta.
- Incluir siempre la zona horaria en `After`, `Before` y `Between`. Las zonas horarias implícitas o en UTC producen activaciones inesperadas en cambios de horario de verano.

Evitar:
- Confiar únicamente en `RemoteAddr` como mecanismo de autenticación. En redes no controladas las IPs pueden ser suplantadas; combínalo con un header de autenticación.
- Usar `XForwardedRemoteAddr` sin garantizar que el proxy externo establece ese header de forma segura. Un proxy mal configurado permite que cualquier cliente inyecte el header y eluda el control de IP.
- Escribir expresiones regulares sin anclar en `Header` o `Cookie`. La regex `\d+` coincide con el fragmento numérico dentro de `abc123xyz`, lo que puede permitir valores no previstos. Usar `^\d+$` para validar el valor completo.
- Depurar un 404 inesperado mirando los logs del servicio. Si ningún predicate de ninguna ruta coincide, el Gateway devuelve 404 directamente, sin llamar a ningún servicio. Para verificar qué rutas están activas y qué predicates tienen, consultar `GET /actuator/gateway/routes` (requiere `management.endpoints.web.exposure.include: gateway` en la configuración).

---

## AND lógico vs OR lógico

En YAML todos los Predicates de una ruta son AND. Para OR hay dos opciones:

**Opción A — Dos rutas separadas (YAML, recomendada para casos simples)**

```yaml
routes:
  # Aplica si viene de red interna O si lleva token de servicio
  - id: admin-por-red
    uri: lb://admin-service
    predicates:
      - Path=/admin/**
      - RemoteAddr=10.0.0.0/8

  - id: admin-por-token
    uri: lb://admin-service
    predicates:
      - Path=/admin/**
      - Header=X-Service-Token, [a-f0-9]+
```

**Opción B — Java DSL con `.or()`**

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("admin-route", r -> r
            .path("/admin/**")
            .and()
            .predicate(
                exchange -> {
                    String ip = exchange.getRequest().getRemoteAddress()
                        .getAddress().getHostAddress();
                    boolean esRedInterna = ip.startsWith("10.");
                    boolean tieneToken = exchange.getRequest().getHeaders()
                        .containsKey("X-Service-Token");
                    return esRedInterna || tieneToken;
                })
            .uri("lb://admin-service"))
        .build();
}
```

> [EXAMEN] La opción A con rutas YAML separadas es preferible para la mayoría de casos: más legible, sin código Java y fácil de auditar. La opción B con DSL tiene sentido cuando la condición OR es dinámica o cuando no hay equivalente declarativo disponible.

---

← [6.2.1 Modelo de Route](./06-08-gateway-route-modelo.md) | [Índice](./README.md) | [6.2.3 Configuración en YAML →](./06-10-gateway-rutas-yaml.md)
