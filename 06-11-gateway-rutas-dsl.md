# 6.2.4 Rutas con Java DSL

← [6.2.3 Configuración en YAML](./06-10-gateway-rutas-yaml.md) | [Índice](./README.md) | [6.2.5 Canary, Weight y Forward →](./06-12-gateway-canary-forward.md)

---

El DSL Java de Spring Cloud Gateway no es un sustituto del YAML, sino un complemento para los casos en que la configuración declarativa no es suficiente. La mayoría de las rutas en producción se definen perfectamente en YAML: la configuración es visible, versionable y recargable sin reiniciar la aplicación. El DSL Java es necesario cuando la condición de enrutamiento requiere lógica que no tiene predicado YAML equivalente —como una operación OR entre dos condiciones independientes—, cuando los predicates o filtros deben construirse dinámicamente en tiempo de arranque según el entorno, cuando se necesita un predicate completamente personalizado expresado como `exchange -> boolean`, o cuando se quiere reutilizar lógica de filtro entre varias rutas mediante métodos Java ordinarios.

## Flujo del builder DSL

El siguiente diagrama muestra cómo se encadenan los elementos del `RouteLocatorBuilder` para producir un objeto `RouteLocator` que el Gateway registra junto con las rutas YAML:

```
RouteLocatorBuilder
  .routes()
    .route(id, routeSpec -> routeSpec
        .predicate(PredicateSpec)    ← condición de entrada
            .and() / .or() / .negate()
        .filters(FilterSpec)         ← transformaciones
        .uri(String)                 ← destino
    )
    .build()
                │
                ▼
        RouteLocator (Bean)
                │
                ▼
   Gateway combina con rutas YAML
   y ordena por campo `order`
```

Cada llamada a `.route(id, spec)` añade una ruta al conjunto. El bean `RouteLocator` resultante convive sin conflicto con las rutas declaradas en `application.yml` o `application.yaml`.

## Ejemplo completo con RouteLocatorBuilder

La forma canónica de registrar rutas con el DSL es declarar un bean `RouteLocator` en una clase `@Configuration`. El ejemplo siguiente define tres rutas que ilustran los principales casos de uso del DSL.

```java
@Configuration
public class GatewayRoutesConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()

            // Ruta 1: equivalente a Path=/api/productos/** + StripPrefix=1
            .route("productos-dsl", r -> r
                .path("/api/productos/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addResponseHeader("X-Source", "gateway-dsl")
                )
                .uri("lb://productos-service")
            )

            // Ruta 2: OR lógico — red interna O token de servicio
            .route("admin-or-token", r -> r
                .path("/admin/**")
                .and()
                .predicate(exchange -> {
                    String remoteAddr = exchange.getRequest()
                        .getRemoteAddress().getAddress().getHostAddress();
                    boolean esRedInterna = remoteAddr.startsWith("10.");
                    boolean tieneToken = exchange.getRequest().getHeaders()
                        .containsKey("X-Service-Token");
                    return esRedInterna || tieneToken;  // OR — imposible en YAML
                })
                .filters(f -> f.stripPrefix(1))
                .uri("lb://admin-service")
            )

            // Ruta 3: predicate dinámico según propiedad de entorno
            .route("debug-route", r -> r
                .path("/debug/**")
                .and()
                .host("*.dev.miempresa.com")  // solo en entornos dev
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Debug", "true")
                )
                .uri("lb://debug-service")
            )

            .build();
    }
}
```

> [CONCEPTO] Los métodos `.and()`, `.or()` y `.negate()` del DSL operan sobre objetos `Predicate<ServerWebExchange>` de Java funcional. Esto permite componer condiciones de cualquier complejidad sin necesidad de implementar una clase específica de predicate.

> [EXAMEN] En los exámenes de certificación Spring Professional, la pregunta habitual es: "¿En qué situación es obligatorio usar el DSL Java en lugar de YAML?". La respuesta esperada es cuando se necesita lógica OR entre predicates o cuando el predicate debe evaluarse dinámicamente con acceso al `ServerWebExchange` completo.

## Coexistencia de rutas YAML y rutas DSL

Las rutas definidas en YAML y las definidas con el DSL Java coexisten sin conflicto. El Gateway las combina en un único conjunto y las ordena por el campo `order`. Una ruta con `order` menor tiene mayor prioridad. Para asignar orden a una ruta DSL se encadena `.order(n)` antes del predicate:

```java
.route("mi-ruta-prioritaria", r -> r
    .order(10)
    .path("/api/**")
    .uri("lb://servicio")
)
```

Las rutas YAML sin `order` explícito reciben el valor `0` por defecto. Si una ruta DSL no especifica `order`, también recibe `0`. Cuando dos rutas tienen el mismo `order`, el orden de evaluación no está garantizado, por lo que se recomienda asignar valores explícitos cuando la prioridad relativa importa.

> [ADVERTENCIA] Si se define la misma ruta tanto en YAML como en el DSL Java con el mismo identificador de ruta, el Gateway no arranca con error pero sí puede producir comportamiento inesperado dependiendo del orden de carga de los beans. Conviene usar identificadores distintos y bien nombrados para cada origen.

## Predicate programático personalizado

Para los casos donde ningún predicado YAML es suficiente, el DSL permite pasar directamente una lambda que recibe el `ServerWebExchange` completo y devuelve un `boolean`. La lambda tiene acceso a cabeceras, parámetros de consulta, cuerpo (con precaución en reactivo), dirección remota y cualquier atributo del contexto de la petición.

```java
.route("ruta-custom", r -> r
    .predicate(exchange -> {
        // Acceso completo al ServerWebExchange
        String userAgent = exchange.getRequest().getHeaders()
            .getFirst("User-Agent");
        return userAgent != null && userAgent.contains("MiAppMovil/3.");
    })
    .filters(f -> f.addRequestHeader("X-Client-Type", "mobile-v3"))
    .uri("lb://mobile-backend")
)
```

> [ADVERTENCIA] Las lambdas en predicates programáticos se ejecutan en el hilo reactivo del Gateway para cada petición. Cualquier operación bloqueante —consultas a base de datos, llamadas HTTP síncronas— dentro de la lambda degradará el rendimiento del gateway completo. Si la evaluación del predicate requiere E/S, se debe implementar la interfaz `AsyncPredicate<ServerWebExchange>` en lugar de usar una lambda síncrona.

## Buenas y malas prácticas

**Buenas prácticas**

- Reservar el DSL Java exclusivamente para las rutas que realmente necesitan lógica que YAML no puede expresar. Mezclar estilos sin criterio dificulta la revisión de la configuración.
- Extraer las lambdas de predicates complejos a métodos privados con nombre descriptivo dentro de la clase `@Configuration`. Esto facilita las pruebas unitarias de cada condición de forma independiente.
- Documentar en un comentario inline la razón por la que esa ruta usa DSL en lugar de YAML. En revisiones de seguridad o auditorías, la justificación inline acelera la comprensión.
- Asignar siempre un identificador de ruta (`id`) descriptivo y único. El identificador aparece en los logs del actuator (`/actuator/gateway/routes`) y es clave para diagnóstico.
- Testear las rutas DSL con `@SpringBootTest` y `WebTestClient` igual que las rutas YAML. El DSL no ofrece ninguna garantía adicional de corrección.

**Malas prácticas**

- Migrar todas las rutas YAML existentes al DSL Java sin una razón técnica. La configuración YAML es más legible para personas que no conocen el código Java del proyecto.
- Hacer llamadas bloqueantes (`.block()`, acceso a JDBC) dentro de una lambda de predicate. El scheduler reactivo no está diseñado para bloquear y el impacto en latencia es severo.
- Usar el DSL para "esconder" lógica de negocio dentro del gateway. El gateway debe enrutar y transformar cabeceras/paths; la lógica de negocio pertenece a los servicios.
- Definir predicates con efectos secundarios (escritura a logs externos, incremento de métricas manuales) que puedan fallar y convertir una petición válida en un error de routing.

## Comparación: YAML vs Java DSL

Elegir entre YAML y el DSL Java no es una decisión de preferencia personal, sino técnica. La tabla siguiente resume los criterios decisivos.

| Aspecto | YAML | Java DSL |
|---|---|---|
| Lógica de predicates | Solo AND implícito | AND, OR, NOT, lambda arbitraria |
| Recarga sin reiniciar | Sí (Config Server + `/refresh`) | No (requiere recompilación y reinicio) |
| Legibilidad para revisiones de seguridad | Alta | Media |
| Predicates condicionales por entorno | Limitado (perfiles Spring) | Total (código Java con condiciones) |
| Reutilización de lógica de filtros | No | Sí (métodos Java ordinarios) |
| Visibilidad en `/actuator/gateway/routes` | Completa | Completa (mismo formato) |
| Testabilidad unitaria del predicate | No aplicable | Sí, la lambda es una función Java pura |
| Caso de uso recomendado | La mayoría de rutas en producción | Rutas con lógica que YAML no puede expresar |

La recomendación general es mantener las rutas en YAML y añadir DSL únicamente donde el YAML es insuficiente. Un proyecto con decenas de rutas en YAML y dos o tres rutas DSL para casos excepcionales es el patrón más habitual y mantenible.

> [EXAMEN] Una pregunta frecuente en exámenes es si las rutas definidas con el DSL Java pueden recargarse en caliente con Spring Cloud Config Bus. La respuesta es no: las rutas DSL son beans Spring que se construyen en el arranque de la aplicación y no participan en el mecanismo de refresco de `@RefreshScope` de forma nativa.

---

← [6.2.3 Configuración en YAML](./06-10-gateway-rutas-yaml.md) | [Índice](./README.md) | [6.2.5 Canary, Weight y Forward →](./06-12-gateway-canary-forward.md)
