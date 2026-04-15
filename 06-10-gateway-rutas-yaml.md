# 6.2.3 Configuración de rutas en YAML

← [6.2.2 Catálogo de Predicates](./06-09-gateway-predicates.md) | [Índice](./README.md) | [6.2.4 Rutas con Java DSL →](./06-11-gateway-rutas-dsl.md)

---

YAML es el enfoque declarativo estándar para definir rutas en Spring Cloud Gateway. Frente a la alternativa programática con Java DSL, la configuración en YAML resulta más legible para revisiones de código, es versionable en git sin necesidad de compilar, y se integra de forma nativa con Spring Cloud Config Server para externalizar y recargar rutas en caliente sin reiniciar la aplicación. La mayoría de los proyectos en producción definen sus rutas aquí. El Java DSL sigue siendo necesario cuando la lógica de construcción de una ruta depende de condiciones dinámicas en tiempo de ejecución — por ejemplo, rutas generadas a partir de una base de datos o construidas condicionalmente según perfiles — pero para el resto de casos YAML es suficiente y la opción recomendada.

## Estructura de una ruta en YAML

Cada ruta dentro de `spring.cloud.gateway.routes` es un objeto de la lista con los siguientes campos. Los predicates y filters admiten dos formas de escritura: la forma abreviada (*shortcut*) en una sola línea, y la forma expandida (*expanded*) con nombre y argumentos separados, que mejora la legibilidad cuando los argumentos son numerosos o complejos.

```
spring:
  cloud:
    gateway:
      routes:
        - id: <string>               # obligatorio, único
          uri: <string>              # obligatorio: lb://, http://, https://
          order: <int>               # opcional, default 0
          predicates:                # opcional, lista de condiciones AND
            - <NombrePredicate>=<arg1>,<arg2>      # shortcut
            - name: <NombrePredicate>              # expanded
              args:
                <clave>: <valor>
          filters:                   # opcional, lista de transformaciones
            - <NombreFilter>=<arg1>,<arg2>         # shortcut
            - name: <NombreFilter>                 # expanded
              args:
                <clave>: <valor>
          metadata:                  # opcional, mapa libre
            <clave>: <valor>
```

> [CONCEPTO] Todos los predicates de una misma ruta se evalúan con lógica AND: la petición debe cumplir todos ellos para que la ruta coincida. Si ninguna ruta coincide, el Gateway devuelve 404 por defecto.

## Ejemplo completo con cinco rutas reales

A continuación se muestra un `application.yml` funcional con cinco rutas que cubren los patrones más habituales en producción: versioning con StripPrefix, reescritura de path, restricción por cabecera y red, ventana temporal y ruta catch-all para errores personalizados.

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      default-filters:
        - AddResponseHeader=X-Gateway, spring-cloud-gateway
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin

      routes:
        # 1. Ruta de API versionada con StripPrefix
        - id: productos-v1
          uri: lb://productos-service
          order: 1
          predicates:
            - Path=/api/v1/productos/**
            - Method=GET,POST,PUT,DELETE
          filters:
            - StripPrefix=2                              # elimina /api/v1
            - AddRequestHeader=X-API-Version, 1

        # 2. Ruta con reescritura de path (RewritePath)
        - id: usuarios-route
          uri: lb://usuarios-service
          order: 2
          predicates:
            - Path=/users/**
          filters:
            - RewritePath=/users/(?<segment>.*), /api/usuarios/$\{segment}

        # 3. Ruta con predicate de cabecera y timeout por ruta

        - id: admin-route
          uri: lb://admin-service
          order: 3
          predicates:
            - Path=/admin/**
            - Header=X-Internal-Token, ^[a-f0-9]{40}$
            - RemoteAddr=10.0.0.0/8
          filters:
            - StripPrefix=1
          metadata:
            connect-timeout: 2000
            response-timeout: 5000

        # 4. Ruta temporal con ventana de fechas
        - id: campania-verano
          uri: lb://ofertas-service
          order: 4
          predicates:
            - Path=/api/ofertas/**
            - Between=2025-06-21T00:00:00+02:00[Europe/Madrid], 2025-09-22T23:59:59+02:00[Europe/Madrid]
          filters:
            - StripPrefix=2

        # 5. Ruta catch-all para respuestas 404 personalizadas
        - id: not-found-route
          uri: lb://error-service
          order: 9999
          predicates:
            - Path=/**
          filters:
            - SetStatus=404

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

> [ADVERTENCIA] En el filtro `RewritePath`, la referencia al grupo capturado usa `$\{segment}` con barra invertida antes del `{`, no `${segment}`. Sin la barra invertida, Spring intenta resolver `${segment}` como una propiedad de configuración al cargar el YAML, falla al no encontrarla y el Gateway no arranca. La barra invertida indica al parser de Spring que es una referencia de regex del propio filtro, no un placeholder de configuración.

> [EXAMEN] La ruta `not-found-route` con `order: 9999` actúa como catch-all porque el Gateway evalúa las rutas en orden ascendente de `order` y elige la primera que coincide. Sin un valor de `order` alto, esta ruta interceptaría todas las peticiones antes que las rutas específicas.

## Tabla de propiedades por campo de ruta

Antes de revisar las propiedades individuales conviene recordar que `id` y `uri` son los únicos campos obligatorios; el resto puede omitirse y el Gateway aplicará sus valores por defecto.

| Campo | Tipo | Valor por defecto | Descripción |
|---|---|---|---|
| `id` | `String` | — (obligatorio) | Identificador único de la ruta. Se usa en logs, actuator y para depuración. Si dos rutas comparten el mismo `id`, el Gateway usa la última y descarta la primera silenciosamente. |
| `uri` | `String` | — (obligatorio) | Destino del tráfico. Acepta `lb://nombre-servicio` para balanceo con Eureka/LoadBalancer, `http://host:puerto` y `https://host:puerto` para destinos fijos. |
| `order` | `int` | `0` | Prioridad de evaluación. El Gateway ordena las rutas de menor a mayor y usa la primera que coincide. A igual `order`, el orden de declaración en YAML desempata. |
| `predicates` | `List` | vacío (coincide siempre) | Lista de condiciones que deben cumplirse simultáneamente (AND lógico) para que la ruta se active. |
| `filters` | `List` | vacío | Lista de transformaciones que se aplican a la petición entrante o a la respuesta antes de devolverla al cliente. |
| `metadata.connect-timeout` | `long` | valor global | Tiempo máximo en milisegundos para establecer la conexión TCP con el servicio destino, específico para esta ruta. |
| `metadata.response-timeout` | `long` | valor global | Tiempo máximo en milisegundos para recibir la respuesta completa del servicio destino, específico para esta ruta. |

## Recarga de configuración sin reiniciar

Cuando las rutas se externalizan en Spring Cloud Config Server, es posible actualizar la configuración del Gateway sin reiniciar la aplicación. Este mecanismo resulta especialmente útil para activar o desactivar rutas temporales, cambiar timeouts o añadir filtros de seguridad en caliente.

El proceso consta de tres pasos: primero se modifica el fichero de configuración en el repositorio del Config Server y se hace commit y push; después se llama al endpoint `POST /actuator/gateway/refresh` en el Gateway para que lea la nueva configuración desde el servidor de configuración.

Para que este mecanismo funcione, el Gateway necesita estar configurado como cliente del Config Server y tener expuestos los endpoints de actuator correspondientes:

```yaml
spring:
  config:
    import: "configserver:http://localhost:8888"
management:
  endpoints:
    web:
      exposure:
        include: gateway,refresh
```

> [ADVERTENCIA] El endpoint `/actuator/gateway/refresh` recarga las rutas en memoria pero no persiste los cambios. Si el Gateway se reinicia sin que el Config Server tenga la nueva configuración consolidada, las rutas vuelven al estado del repositorio. Asegúrate de que el commit en el repositorio del Config Server precede siempre a la llamada al endpoint de refresh.

> [CONCEPTO] El endpoint `/actuator/gateway/routes` permite consultar en tiempo real las rutas activas en el Gateway, incluidos sus predicates y filtros resueltos. Es útil para verificar que el refresh se aplicó correctamente.

## Buenas y malas prácticas

Hacer:
- Asignar un `id` descriptivo que refleje el servicio y el patrón de ruta, por ejemplo `productos-v1-get` en lugar de `route1`.
- Usar `order` explícito en todas las rutas para evitar dependencias del orden de declaración en YAML.
- Definir timeouts por ruta en `metadata` cuando los SLAs de los servicios destino son heterogéneos.
- Versionar el fichero YAML junto al código fuente y revisar los cambios en pull requests.
- Usar la forma expandida de predicates y filtros complejos para mejorar la legibilidad.

Evitar:
- Olvidar el guión `-` antes de `id:` — YAML interpreta las rutas como una lista, y sin el guión la configuración es inválida o se combina incorrectamente con la ruta anterior.
- Usar `uri: lb://MiServicio` con mayúsculas cuando Eureka registra el servicio en minúsculas (o viceversa) — el cliente LoadBalancer no encontrará instancias y todas las peticiones fallarán con 503.
- Poner el mismo `id` en dos rutas distintas — el Gateway usa la última y descarta la primera silenciosamente, lo que dificulta depurar comportamientos inesperados.
- Omitir la zona horaria en los predicates `Between`, `After` y `Before` — Spring falla al arrancar con un error de parseo de la fecha ISO-8601 si no se incluye el offset y el identificador de zona, por ejemplo `+02:00[Europe/Madrid]`.
- Acumular demasiados `default-filters` globales sin documentar su impacto — aplican a todas las rutas, incluidas las rutas de health check y actuator.

> [EXAMEN] Una pregunta habitual en certificaciones es: ¿qué ocurre si dos rutas tienen el mismo `id`? La respuesta es que el Gateway registra la última y descarta la primera sin lanzar ningún error en tiempo de arranque.

---

← [6.2.2 Catálogo de Predicates](./06-09-gateway-predicates.md) | [Índice](./README.md) | [6.2.4 Rutas con Java DSL →](./06-11-gateway-rutas-dsl.md)
