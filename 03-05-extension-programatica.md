# 3.5.9 Extensión Programática: Refresco

← [3.5 Refresco](./03-05-config-refresh.md) | [Índice](./README.md) | [3.6 Cifrado y HA →](./03-06-config-avanzado.md)

---

## Puntos de extensión del ciclo de refresco

La capa YAML del refresco configura cuándo se refresca (polling, Bus) y qué se expone (endpoints de Actuator). Pero no permite controlar *cómo* se dispara el refresco desde código: no se puede invocar el refresco como respuesta a un evento interno de la aplicación, no se puede refrescar solo un bean específico sin reiniciar todo el contexto, y no se puede publicar un evento de Bus sin hacer una llamada HTTP. Los tres componentes que expone Spring Cloud Config para resolver esto — `ContextRefresher`, `RefreshScope` y `ApplicationEventPublisher` — son beans ordinarios que se pueden inyectar en cualquier servicio y llamar directamente desde Java.

| Clase / Interface | Para qué |
|---|---|
| `ContextRefresher` | Disparar el refresco del contexto programáticamente |
| `RefreshScope` | Refrescar beans individuales por nombre |
| `ApplicationEventPublisher` | Publicar eventos de refresco al Bus sin llamada HTTP |
| `ApplicationListener<RefreshRemoteApplicationEvent>` | Lógica propia cuando llega un evento de Bus |
| `ScopeRefreshedEvent` | Detectar cuándo un bean @RefreshScope fue destruido y recreado |

---

## 3.5.9.1 Qué ocurre internamente en `ContextRefresher.refresh()`

```
contextRefresher.refresh()
  │
  ├── 1. Crea un ApplicationContext temporal (copia del principal)
  │
  ├── 2. Invoca todos los PropertySourceLocator.locate() de nuevo
  │        ← llama al Config Server HTTP para obtener config actualizada
  │
  ├── 3. Compara las propiedades anteriores con las nuevas
  │        → calcula el Set<String> de claves que cambiaron
  │
  ├── 4. Actualiza el Environment del contexto principal con los nuevos valores
  │
  ├── 5. Publica EnvironmentChangeEvent con las claves que cambiaron
  │        ← aquí es donde @EventListener(EnvironmentChangeEvent) se activa
  │
  ├── 6. Destruye TODOS los beans @RefreshScope del contexto principal
  │        ⚠ Los beans no se recrean aquí — se recrean en la próxima llamada a cada uno
  │
  └── 7. Retorna el Set<String> de claves modificadas al caller

Nota importante:
  ● Los beans SIN @RefreshScope NO se destruyen ni recrean — conservan sus valores
  ● Los beans CON @RefreshScope se recrean lazily en la próxima inyección
  ● Los beans que invocan refresh() no deben tener @RefreshScope (ver antipatrones)
```

---

## 3.5.9.2 `ContextRefresher` — disparar el refresco sin llamada HTTP

`ContextRefresher` es el bean que ejecuta el refresco de forma programática. Es exactamente lo que el endpoint `POST /actuator/refresh` llama internamente. Inyectarlo permite disparar el refresco desde cualquier lugar: un job programado, un listener de eventos de infraestructura, o un endpoint administrativo propio.

```java
@Service
public class ConfigRefreshService {

    private final ContextRefresher contextRefresher;

    public ConfigRefreshService(ContextRefresher contextRefresher) {
        this.contextRefresher = contextRefresher;
    }

    /**
     * Refresca la configuración y devuelve las claves que cambiaron.
     * Equivale exactamente a POST /actuator/refresh.
     */
    public Set<String> refrescarConfiguracion() {
        Set<String> keysModificadas = contextRefresher.refresh();
        log.info("Configuración refrescada. Propiedades modificadas: {}", keysModificadas);
        return keysModificadas;
    }

    /**
     * Refresco programado cada 5 minutos — alternativa al watch polling
     * cuando se quiere controlar exactamente cuándo se aplica el refresco.
     */
    @Scheduled(fixedDelay = 5 * 60 * 1000)
    public void refrescoAutomatico() {
        Set<String> cambios = contextRefresher.refresh();
        if (!cambios.isEmpty()) {
            log.info("Refresco automático — {} propiedades cambiadas: {}", cambios.size(), cambios);
        }
    }
}
```

```java
// También se puede usar refreshEnvironment() para refrescar solo el Environment
// sin destruir y recrear los beans @RefreshScope:
Set<String> cambios = contextRefresher.refreshEnvironment();
// refreshEnvironment() actualiza las propiedades pero no recrea los beans
// refresh() actualiza las propiedades Y destruye/recrea los beans @RefreshScope
```

> **Diferencia entre `refresh()` y `refreshEnvironment()`:**
> - `refreshEnvironment()`: actualiza el `Environment` con los valores del Config Server. Los beans `@RefreshScope` siguen usando los valores anteriores hasta que se acceda a ellos por primera vez.
> - `refresh()`: hace lo mismo y además destruye todos los beans `@RefreshScope`, forzando su recreación con los nuevos valores en el próximo acceso.

---

## 3.5.9.3 `RefreshScope` — refrescar beans individuales

Para casos donde solo un bean específico necesita refrescarse (no toda la configuración), `RefreshScope` permite hacerlo por nombre sin tocar el resto:

```java
@Service
public class SelectiveRefreshService {

    private final RefreshScope refreshScope;

    public SelectiveRefreshService(RefreshScope refreshScope) {
        this.refreshScope = refreshScope;
    }

    /**
     * Refresca solo el bean de features sin tocar el resto de @RefreshScope beans.
     * Útil cuando se sabe exactamente qué bean necesita actualizarse.
     */
    public void refrescarFeatureController() {
        refreshScope.refresh("featureController");   // nombre del bean en el contexto
    }

    /**
     * Refresca todos los beans @RefreshScope sin actualizar el Environment.
     * Solo tiene sentido si el Environment ya fue actualizado antes (p.ej. con refreshEnvironment()).
     */
    public void refrescarTodosLosBeans() {
        refreshScope.refreshAll();
    }
}
```

```java
// Determinar el nombre del bean en el contexto:
// Por defecto es el nombre de la clase en camelCase: "featureController", "pedidosProperties", etc.
// Si tiene @Component("miNombre"), usar ese nombre.

// Ejemplo de bean @RefreshScope cuyo nombre es "pedidosProperties":
@Component("pedidosProperties")
@RefreshScope
@ConfigurationProperties(prefix = "pedidos")
public class PedidosProperties { ... }
```

---

## 3.5.9.4 `ApplicationEventPublisher` — publicar eventos de Bus sin HTTP

Publicar un `RefreshRemoteApplicationEvent` directamente, sin necesidad de hacer una petición HTTP a `/actuator/busrefresh`. Útil cuando el refresco debe dispararse desde lógica de negocio (p.ej. cuando se guarda una nueva configuración en una base de datos):

```java
@Service
public class ConfigChangeBroadcaster {

    private final ApplicationEventPublisher eventPublisher;

    @Value("${spring.cloud.bus.id:unknown}")
    private String busId;

    public ConfigChangeBroadcaster(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    /**
     * Notifica a todos los servicios del Bus que deben refrescarse.
     * Equivale a POST /actuator/busrefresh pero sin HTTP.
     */
    public void notificarCambioGlobal() {
        // "**" = todos los servicios del Bus
        RefreshRemoteApplicationEvent evento = new RefreshRemoteApplicationEvent(
            this,      // source
            busId,     // originService — quién envió el evento
            "**"       // destinationService — a quién va dirigido ("**" = todos)
        );
        eventPublisher.publishEvent(evento);
        log.info("Evento de refresco publicado al Bus desde: {}", busId);
    }

    /**
     * Notifica solo a instancias de un servicio específico.
     * Equivale a POST /actuator/busrefresh/pedidos-service:**
     */
    public void notificarServicio(String serviceName) {
        RefreshRemoteApplicationEvent evento = new RefreshRemoteApplicationEvent(
            this,
            busId,
            serviceName + ":**"   // todas las instancias del servicio
        );
        eventPublisher.publishEvent(evento);
    }
}
```

---

## 3.5.9.5 Listeners de eventos de refresco

### `ApplicationListener<RefreshRemoteApplicationEvent>` — cuando llega un evento del Bus

```java
@Component
public class BusRefreshListener implements ApplicationListener<RefreshRemoteApplicationEvent> {

    @Override
    public void onApplicationEvent(RefreshRemoteApplicationEvent event) {
        // Se dispara cuando ESTE servicio recibe un RefreshRemoteApplicationEvent del Bus
        // (tanto si va dirigido a él como si es para "**")

        log.info("Evento de refresco recibido. Origen: {}, Destino: {}",
            event.getOriginService(),
            event.getDestinationService());

        // Lógica propia: invalidar caché, notificar a otros componentes, etc.
        // OJO: en este punto el contexto aún NO ha sido refrescado.
        // Para ejecutar lógica DESPUÉS del refresco, escuchar EnvironmentChangeEvent.
    }
}
```

### `EnvironmentChangeEvent` — después de que el refresco se aplicó

```java
@Component
public class PostRefreshHandler {

    @EventListener
    public void onEnvironmentChange(EnvironmentChangeEvent event) {
        // Se dispara DESPUÉS de que el contexto fue refrescado
        // event.getKeys() contiene las propiedades que cambiaron

        Set<String> keysModificadas = event.getKeys();
        log.info("Propiedades modificadas tras el refresco: {}", keysModificadas);

        // Reaccionar a cambios específicos:
        if (keysModificadas.contains("feature.nueva-ui")) {
            log.info("Feature flag nueva-ui cambió — invalidando caché de UI");
        }

        if (keysModificadas.stream().anyMatch(k -> k.startsWith("database."))) {
            log.warn("Parámetros de BD cambiados — considerar reinicio del pool de conexiones");
        }
    }
}
```

### `ScopeRefreshedEvent` — cuando un bean @RefreshScope fue destruido y recreado

```java
@Component
public class BeanRefreshObserver {

    @EventListener
    public void onScopeRefreshed(ScopeRefreshedEvent event) {
        // Se dispara cada vez que el RefreshScope destruye y recrea sus beans
        // Útil para métricas (cuántos refrescos ocurrieron, con qué frecuencia)
        log.debug("RefreshScope refrescado. Beans destruidos y recreados.");

        // Ejemplo: publicar métrica de refresco
        meterRegistry.counter("config.refresh.count").increment();
    }
}
```

---

## 3.5.9.6 Antipatrones

> **[ADVERTENCIA]** Publicar `RefreshRemoteApplicationEvent` con un `destinationService` que no coincide con `spring.cloud.bus.id` de los servicios destino. El evento se publica en el broker sin errores, pero ningún servicio lo recibe — el Bus lo descarta silenciosamente porque el patrón de destino no coincide. Siempre verificar que el `spring.cloud.bus.id` está configurado con el patrón `{appName}:{port}:{uuid}` y que el `destinationService` del evento usa el mismo formato.

> **[ADVERTENCIA]** Llamar a `contextRefresher.refresh()` desde un bean marcado con `@RefreshScope`. Al invocar el refresco, Spring destruye todos los beans `@RefreshScope` incluyendo el que está ejecutando el método — el bean pierde la referencia al `ContextRefresher` a mitad de ejecución, causando `NullPointerException` o comportamiento indeterminado. El bean que dispara el refresco debe ser un `@Component` sin `@RefreshScope`.

> **[ADVERTENCIA]** Llamar a `contextRefresher.refresh()` frecuentemente en un `@Scheduled` sin controlar si hay cambios reales. Cada llamada destruye y recrea todos los beans `@RefreshScope`, lo que bajo carga puede causar picos de latencia visibles. Si se necesita polling, usar `spring.cloud.config.watch.enabled=true` — implementa el polling con un hilo dedicado y solo destruye beans cuando hay cambios reales.

---

### Tabla resumen: cuándo YAML es suficiente vs cuándo se necesita Java para el refresco

| Caso | YAML suficiente | Requiere Java |
|---|---|---|
| Refresco manual vía HTTP | Sí (`/actuator/refresh`) | No |
| Refresco masivo con Spring Cloud Bus | Sí (`/actuator/busrefresh`) | No |
| Refresco programado (job) | Parcialmente (`spring.cloud.config.watch`) | Sí para lógica custom — `ContextRefresher` inyectado en `@Scheduled` |
| Refresco disparado por evento de negocio (no HTTP) | No | Sí — `ApplicationEventPublisher.publishEvent(RefreshRemoteApplicationEvent)` |
| Refrescar solo un bean específico | No | Sí — `RefreshScope.refresh("beanName")` |
| Lógica propia tras el refresco | No | Sí — `@EventListener(EnvironmentChangeEvent.class)` |
| Detectar qué beans @RefreshScope se recrearon | No | Sí — `@EventListener(ScopeRefreshedEvent.class)` |

---

← [3.5 Refresco](./03-05-config-refresh.md) | [Índice](./README.md) | [3.6 Cifrado y HA →](./03-06-config-avanzado.md)
