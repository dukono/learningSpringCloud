# 12.2 Spring Cloud Function — FunctionCatalog

← [12.1 Modelo de programación funcional](sc-function-modelo-programacion.md) | [Índice](README.md) | [12.3 Adaptador HTTP →](sc-function-adaptador-http.md)

---

## Introducción

`FunctionCatalog` es el registro central de Spring Cloud Function. Almacena todos los beans funcionales detectados en el contexto y permite obtenerlos por nombre, componerlos programáticamente y resolver sus tipos genéricos en tiempo de ejecución. Es la puerta de entrada para manipular funciones de forma dinámica tanto en código de producción como en tests de integración.

> [CONCEPTO] `FunctionCatalog` es una interfaz de SCF que actúa como directorio de funciones. Su método principal es `lookup(String name)`, que devuelve un `FunctionInvocationWrapper` tipado que envuelve al bean funcional subyacente.

> [PREREQUISITO] Para inyectar `FunctionCatalog` es necesaria la dependencia `spring-cloud-function-context` (incluida en cualquier starter de SCF).

## Diagrama de FunctionCatalog

El siguiente diagrama ilustra el flujo de lookup y composición programática desde el catálogo.

```
ApplicationContext
  @Bean uppercase  ──┐
  @Bean trim       ──┤──► FunctionCatalog
  @Bean logOutput  ──┘         │
                               │  lookup("uppercase")
                               ▼
                    FunctionInvocationWrapper<String,String>
                               │
                    .andThen(catalog.lookup("trim"))
                               ▼
                    Function compuesta: uppercase → trim
```

## Ejemplo central

El siguiente ejemplo muestra cómo inyectar `FunctionCatalog`, obtener funciones por nombre y componerlas programáticamente en un servicio de producción.

```java
package com.example.demo.service;

import org.springframework.cloud.function.context.FunctionCatalog;
import org.springframework.stereotype.Service;

import java.util.function.Function;

@Service
public class PipelineService {

    private final FunctionCatalog catalog;

    public PipelineService(FunctionCatalog catalog) {
        this.catalog = catalog;
    }

    /**
     * Lookup de una función simple por nombre de bean.
     */
    public String applyUppercase(String input) {
        Function<String, String> fn = catalog.lookup("uppercase");
        return fn.apply(input);
    }

    /**
     * Composición programática: uppercase → trim.
     * Equivalente a spring.cloud.function.definition: uppercase|trim
     */
    public String applyPipeline(String input) {
        Function<String, String> uppercase = catalog.lookup("uppercase");
        Function<String, String> trim = catalog.lookup("trim");
        Function<String, String> pipeline = uppercase.andThen(trim);
        return pipeline.apply(input);
    }

    /**
     * Lookup de una composición ya definida por nombre compuesto.
     * SCF registra la composición como "uppercase|trim" en el catálogo.
     */
    public String applyComposed(String input) {
        Function<String, String> composed = catalog.lookup("uppercase|trim");
        return composed.apply(input);
    }
}
```

Definición de los beans funcionales referenciados:

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.function.Function;

@Configuration
public class FunctionConfig {

    @Bean
    public Function<String, String> uppercase() {
        return String::toUpperCase;
    }

    @Bean
    public Function<String, String> trim() {
        return String::trim;
    }
}
```

> [ADVERTENCIA] `catalog.lookup()` devuelve `null` si el nombre de bean no existe en el catálogo. Siempre validar el resultado antes de invocar la función para evitar `NullPointerException`.

## Tabla de elementos clave

La siguiente tabla resume los métodos y tipos relevantes de `FunctionCatalog`.

| Elemento | Firma | Descripción |
|---|---|---|
| `lookup(String name)` | `<T> T lookup(String name)` | Retorna el bean funcional por nombre; null si no existe |
| `lookup(Class<?> type, String name)` | `<T> T lookup(Class<?> type, String name)` | Lookup con tipo esperado explícito |
| `getNames(Class<?> type)` | `Set<String> getNames(Class<?> type)` | Obtiene todos los nombres registrados del tipo dado |
| `FunctionInvocationWrapper` | — | Envoltorio que SCF retorna; implementa `Function`, `Consumer` y `Supplier` |
| `.andThen(fn)` | — | Composición: output del primero → input del segundo |
| `.compose(fn)` | — | Composición inversa: el argumento se ejecuta primero |

## Buenas y malas prácticas

**Buenas prácticas:**
- Inyectar `FunctionCatalog` en servicios que necesiten invocar funciones dinámicamente (por ejemplo, basándose en configuración o headers del mensaje).
- Usar `catalog.getNames(Function.class)` para descubrir en runtime qué funciones están disponibles.
- Preferir `lookup` por nombre compuesto (`"uppercase|trim"`) cuando la composición está definida en `spring.cloud.function.definition` — SCF ya la materializa.

**Malas prácticas:**
- Abusar de `FunctionCatalog` en código de producción cuando una inyección directa del bean es suficiente — añade complejidad innecesaria.
- Asumir que `lookup` siempre devuelve un tipo concreto sin verificar nullabilidad.
- Invocar `lookup` en el constructor sin que el contexto esté completamente inicializado.

## Verificación y práctica

> [EXAMEN] ¿Qué método de `FunctionCatalog` se usa para obtener un bean funcional por nombre y qué devuelve si el nombre no existe?

> [EXAMEN] ¿Cómo se obtiene una composición ya registrada en el catálogo mediante un nombre compuesto, y cuál es el separador usado en ese nombre?

> [EXAMEN] ¿Cuál es la diferencia entre `Function.andThen()` y `Function.compose()` al combinar funciones obtenidas del catálogo?

> [EXAMEN] ¿Qué método de `FunctionCatalog` permite listar todos los nombres de beans funcionales registrados de un tipo determinado?

---

← [12.1 Modelo de programación funcional](sc-function-modelo-programacion.md) | [Índice](README.md) | [12.3 Adaptador HTTP →](sc-function-adaptador-http.md)
