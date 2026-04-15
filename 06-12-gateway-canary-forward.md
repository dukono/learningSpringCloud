# 6.2.5 Canary deployment con Weight y filtro Forward

← [6.2.4 Rutas con Java DSL](./06-11-gateway-rutas-dsl.md) | [Índice](./README.md) | [6.3.1 Filtros de transformación de petición →](./06-13-gateway-filtros-path.md)

---

Desplegar una nueva versión de un microservicio de forma segura implica enviar solo una fracción del tráfico real a esa versión antes de promoverla al 100%. El predicado `Weight` y el filtro `Forward` de Spring Cloud Gateway permiten implementar ese patrón —conocido como canary deployment— directamente en el fichero de configuración, sin necesidad de un balanceador externo adicional, sin código Java y sin tocar la infraestructura de red.

## Distribución ponderada de tráfico con Weight

El predicado `Weight` distribuye el tráfico entre dos o más rutas que pertenecen al mismo grupo lógico. Cada ruta del grupo declara un peso entero, y el Gateway normaliza automáticamente esos pesos en porcentajes: no es necesario que sumen 100. Si una ruta tiene peso 90 y otra peso 10, el Gateway enviará aproximadamente el 90% de las peticiones a la primera y el 10% a la segunda.

El siguiente diagrama muestra el flujo completo de una petición canary:

```
Cliente
  │
  ▼
┌─────────────────────────────────────────────────┐
│             Spring Cloud Gateway                │
│                                                 │
│  WeightCalculatorWebFilter (antes de predicates)│
│    ┌──────────────────────────────────────┐     │
│    │  productos-group: peso total = 100   │     │
│    │  sorteo ponderado → resultado: 73    │     │
│    └──────────────────────────────────────┘     │
│                     │                           │
│         ┌───────────┴────────────┐              │
│         │ 73 < 90 → ruta estable │              │
│         └───────────┬────────────┘              │
│                     │                           │
│    ┌────────────────▼───────────────────────┐   │
│    │  Predicates: Path=/api/productos/**    │   │
│    │             Weight=productos-group, 90 │   │
│    └────────────────┬───────────────────────┘   │
└─────────────────────┼───────────────────────────┘
                      │
          ┌───────────┴────────────┐
          │                        │
          ▼                        ▼
 productos-service-v1    productos-service-v2
  (ruta estable: 90%)      (ruta canary: 10%)
```

La asignación ocurre en el `WeightCalculatorWebFilter`, que se ejecuta antes de que los predicados sean evaluados. Eso significa que Weight trabaja en paralelo con el resto de predicados, no en lugar de ellos.

## Ejemplo completo de canary con Weight

El siguiente ejemplo configura dos rutas para el mismo path. La versión estable absorbe el 90% del tráfico y la versión canary el 10%. Ambas pertenecen al grupo `productos-group`.

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Versión estable: recibe el 90% del tráfico del grupo "productos-group"
        - id: productos-estable
          uri: lb://productos-service-v1
          predicates:
            - Path=/api/productos/**
            - Weight=productos-group, 90

        # Versión canary: recibe el 10% del tráfico del grupo "productos-group"
        - id: productos-canary
          uri: lb://productos-service-v2
          predicates:
            - Path=/api/productos/**
            - Weight=productos-group, 10
          filters:
            - AddResponseHeader=X-Canary, true
```

La cabecera `X-Canary: true` permite identificar en los logs y sistemas de observabilidad qué respuestas proceden de la versión nueva, sin modificar el comportamiento funcional de la ruta.

> [CONCEPTO] El peso no necesita sumar 100. `Weight=g, 3` y `Weight=g, 1` son equivalentes a una distribución 75%/25%. El Gateway calcula el porcentaje de cada ruta como `peso_ruta / suma_total_del_grupo`.

> [EXAMEN] El nombre del grupo en `Weight` debe ser idéntico en todas las rutas del mismo canary, incluyendo mayúsculas y minúsculas. Si los nombres difieren, el Gateway trata cada ruta como un grupo independiente de un solo elemento, y ambas rutas recibirán el 100% del tráfico que coincida con sus predicados, sin ninguna distribución ponderada.

## Rollout gradual sin reiniciar el Gateway

En un despliegue progresivo real se suele empezar con un 5% o 10% en la versión canary, observar métricas durante un período de tiempo y aumentar el porcentaje hasta el 100% si no se detectan regresiones. Esto exige actualizar los pesos sin reiniciar el Gateway.

La aproximación recomendada combina Spring Cloud Config Server con el endpoint `/actuator/gateway/refresh`. Los pesos se almacenan en el repositorio de configuración centralizado y se actualizan con una llamada HTTP, sin ningún downtime.

El siguiente ejemplo muestra cómo actualizar los pesos mediante la API de Actuator. Primero se elimina la ruta canary existente:

```bash
curl -X DELETE http://localhost:8080/actuator/gateway/routes/productos-canary
```

A continuación se crea de nuevo con el nuevo peso (por ejemplo, del 10% al 25%):

```bash
curl -X POST http://localhost:8080/actuator/gateway/routes/productos-canary \
  -H "Content-Type: application/json" \
  -d '{
    "id": "productos-canary",
    "uri": "lb://productos-service-v2",
    "predicates": [
      { "name": "Path", "args": { "pattern": "/api/productos/**" } },
      { "name": "Weight", "args": { "group": "productos-group", "weight": "25" } }
    ],
    "filters": [
      { "name": "AddResponseHeader", "args": { "name": "X-Canary", "value": "true" } }
    ]
  }'
```

Después se fuerza el refresco para que el Gateway aplique el cambio:

```bash
curl -X POST http://localhost:8080/actuator/gateway/refresh
```

Con el estable en 90 y el canary en 25, el total del grupo es 115. El Gateway normaliza automáticamente: `90/115 ≈ 78%` para el estable y `25/115 ≈ 22%` para el canary. Si se necesitan porcentajes exactos (75%/25%), hay que actualizar también la ruta estable a peso 75 con el mismo patrón de DELETE + POST antes del refresh.

Para que estos endpoints estén disponibles, el fichero `application.yml` debe exponer el endpoint `gateway` de Actuator:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: gateway, refresh
```

## Filtro Forward — redirección interna al Gateway

El filtro `Forward` no reenvía la petición a un servicio externo: la redirige a otro path dentro del propio Gateway, que vuelve a procesar la petición desde el inicio del pipeline con el nuevo path. No produce ninguna respuesta HTTP de redirección visible para el cliente; el reenvío es completamente transparente.

Este mecanismo es útil para implementar mantenimiento planificado centralizado, para redirigir peticiones a una ruta de fallback interna o para construir endpoints de health ficticios que delegan en rutas existentes.

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Durante el mantenimiento: cualquier petición a /api/** se redirige internamente a /maintenance
        - id: maintenance-redirect
          uri: forward:///maintenance
          predicates:
            - Path=/api/**

        # Ruta que sirve la respuesta de mantenimiento desde un servicio estático interno
        - id: maintenance-page
          uri: lb://static-service
          predicates:
            - Path=/maintenance
```

La URI `forward:///maintenance` instruye al Gateway para que reenvíe la petición al path `/maintenance` dentro de sí mismo. La segunda ruta, `maintenance-page`, recoge esa petición y la envía al servicio correspondiente. El cliente recibe la respuesta del servicio estático sin haber sido redirigido con un `302`.

> [ADVERTENCIA] Un `Forward` mal configurado puede crear un bucle infinito. Si la ruta de destino del forward contiene a su vez otro forward al mismo path, el Gateway procesará la petición en bucle hasta agotar la pila de llamadas. Para evitarlo, hay que asegurarse de que la ruta destino del forward no contenga otro forward hacia el mismo path ni hacia ningún path que forme un ciclo.

## Tabla de propiedades

La tabla siguiente resume los parámetros configurables del predicado `Weight` y del filtro `Forward`.

| Componente | Parámetro | Tipo | Descripción |
|---|---|---|---|
| `Weight` predicate | `group` | String | Nombre del grupo de canary. Debe ser idéntico en todas las rutas del mismo grupo, incluyendo mayúsculas y minúsculas. |
| `Weight` predicate | `weight` | int | Peso relativo de esta ruta dentro del grupo. No necesita sumar 100 con el resto de pesos del grupo. |
| `forward:///path` URI | path | String | Path interno al que se reenvía la petición. Debe comenzar con `/` y apuntar a un path que pueda ser procesado por otra ruta del Gateway. |

## Buenas y malas prácticas

Usar `AddResponseHeader=X-Canary, true` en la ruta canary facilita enormemente la correlación de logs y el análisis de métricas diferenciado por versión. Sin esta cabecera es imposible distinguir en los sistemas de observabilidad qué peticiones fueron atendidas por la versión nueva.

Gestionar los pesos en un repositorio de configuración centralizado (Spring Cloud Config) y actualizarlos con `/actuator/gateway/refresh` permite un rollout gradual seguro sin downtime y con posibilidad de rollback inmediato.

No usar el mismo `id` de ruta para la versión estable y la canary. El `id` es el identificador único de cada ruta en el Gateway; si coinciden, la segunda definición sobreescribe a la primera y el grupo de canary queda roto.

No combinar `Weight` con un predicado `Query` que solo existe en una de las dos rutas del grupo, a menos que sea intencionado. Si el predicado adicional excluye peticiones en una ruta pero no en la otra, la distribución de tráfico real no coincidirá con los pesos declarados.

No usar `Forward` para redirigir a rutas que a su vez realizan llamadas a servicios externos con alta latencia sin haber configurado timeouts apropiados. El forward es síncrono dentro del pipeline del Gateway, y un timeout en la ruta de destino bloquea la respuesta al cliente igual que lo haría en una ruta directa.

## Comparación: Weight vs Load Balancer custom para canary

Ambos enfoques permiten implementar canary deployments, pero difieren en el nivel de control y en el esfuerzo de implementación.

| Aspecto | Weight predicate | Custom LoadBalancer |
|---|---|---|
| Configuración | Solo YAML o Java DSL | Código Java + `@Bean` de `ReactorLoadBalancer` |
| Granularidad | Por ruta; se puede combinar con otros predicados para afinar el subconjunto de tráfico | Por instancia dentro del pool de un mismo servicio |
| Sticky session | No soportado de forma nativa | Se puede implementar basándose en cookie o cabecera |
| Actualización sin reiniciar | Sí, mediante Config Server y `/actuator/gateway/refresh` | No; requiere recompilación y redespliegue |
| Caso de uso principal | Canary entre dos versiones de un servicio registradas con nombres distintos en el registro de servicios | Canary entre instancias de la misma versión dentro de un pool compartido |
| Observabilidad | Fácil: cabecera de respuesta en la ruta canary | Requiere instrumentación adicional en el balanceador |

El predicado `Weight` es la elección habitual cuando las dos versiones del servicio están registradas con nombres distintos en el registro de servicios (por ejemplo, `productos-service-v1` y `productos-service-v2`). El Load Balancer custom tiene sentido cuando ambas versiones comparten el mismo nombre de servicio y se distinguen únicamente por metadatos de instancia.

---

← [6.2.4 Rutas con Java DSL](./06-11-gateway-rutas-dsl.md) | [Índice](./README.md) | [6.3.1 Filtros de transformación de petición →](./06-13-gateway-filtros-path.md)
