# 13.11 Antipatrones: Distributed Monolith, Chatty Services, Shared Database, Mega-Service y Two-Phase Commit

← [13.10 Patrones de despliegue](sc-patrones-deployment-patterns.md) | [Índice](README.md) | [13.12 Testing de microservicios](sc-patrones-testing.md) →

---

## Introducción

Los antipatrones en microservicios son tan peligrosos como los del monolito — y frecuentemente más difíciles de diagnosticar porque el sistema aparenta tener la forma correcta (múltiples servicios, múltiples repositorios) pero ha replicado los problemas del monolito en un entorno distribuido. Reconocer estos antipatrones es el primer paso para refactorizarlos; cada uno tiene síntomas identificables y un patrón correctivo conocido.

## Distributed Monolith

> [CONCEPTO] **Distributed Monolith**: un sistema de "microservicios" donde los servicios están tan acoplados que deben desplegarse juntos. Los síntomas son: cambios en un servicio requieren cambios coordinados en otros servicios, los despliegues siempre afectan a múltiples servicios simultáneamente, y los equipos coordinan constantemente sus cambios porque no pueden trabajar de forma autónoma. Tiene todos los costes de un sistema distribuido y ninguno de los beneficios.

Los dos síntomas más reveladores del Distributed Monolith:
1. **Acoplamiento de contratos**: los contratos de API entre servicios son tan rígidos que un cambio de esquema en un servicio fuerza cambios en todos sus consumidores simultáneamente.
2. **Transacciones distribuidas masivas**: los flujos de negocio básicos requieren coordinación síncrona entre 5+ servicios en cada petición.

**Diagnóstico**: si el equipo no puede desplegar un servicio de forma independiente sin coordinar con otros equipos, es un Distributed Monolith.

**Corrección**: revisar los límites de los servicios aplicando DDD (Bounded Contexts), consolidar servicios con alto acoplamiento en un único servicio mejor delimitado, y rediseñar la comunicación hacia asíncrona con contratos evolutivos.

## Chatty Services

> [CONCEPTO] **Chatty Services**: un servicio realiza un número excesivo de llamadas a otros servicios para completar una operación. En lugar de obtener los datos que necesita en una o pocas llamadas, hace decenas de llamadas de grano fino. Esto impacta la latencia (cada llamada añade overhead de red), la disponibilidad (más llamadas = más superficie de fallo) y la carga en los servicios llamados.

La causa raíz suele ser una descomposición demasiado granular (nano-services) o la exposición de entidades en lugar de casos de uso.

**Refactorización**:
- **API Composition**: consolidar las múltiples llamadas en un aggregator que hace las llamadas en paralelo y devuelve el resultado compuesto.
- **Rediseñar contratos**: exponer endpoints orientados a casos de uso en lugar de entidades CRUD ("dame los datos del checkout" en lugar de "dame el usuario", "dame el carrito", "dame los precios", "dame el stock"...).
- **Eventos asíncronos**: si el patrón Chatty aparece en flujos asíncronos, usar mensajería para reducir las llamadas síncronas.

## Shared Database (antipatrón)

> [CONCEPTO] **Shared Database (antipatrón)**: múltiples microservicios leen y escriben en la misma base de datos. Viola el principio de autonomía de datos: los servicios quedan acoplados a nivel de esquema (un cambio de columna en la tabla compartida afecta a todos los servicios que la usan), no pueden escalar de forma independiente, y no pueden usar la tecnología de persistencia más adecuada para cada caso de uso.

Es el antipatrón más común en migraciones "lift and shift" de monolitos: se divide el código en servicios pero se mantiene la base de datos única.

**Corrección progresiva** (no big-bang):
1. Identificar qué tablas son "propiedad" de qué servicio (qué servicio es el que escribe).
2. Hacer que los demás servicios accedan a esos datos exclusivamente a través de la API del servicio propietario.
3. Eventualmente, separar físicamente los esquemas y bases de datos.
4. Reemplazar los JOINs cruzados con API Composition o proyecciones de lectura alimentadas por eventos.

## Mega-Service

> [CONCEPTO] **Mega-Service**: un servicio que ha acumulado demasiadas responsabilidades no cohesionadas. Síntomas: el servicio tiene más de un equipo trabajando en él, el dominio del servicio abarca múltiples capacidades de negocio no relacionadas, el contrato de API tiene docenas de endpoints de dominios diferentes, y el deploy del servicio requiere regresión completa porque cualquier cambio puede afectar a cualquier parte.

A diferencia del Chatty Services (demasiado fragmentado), el Mega-Service es el problema opuesto (demasiado grande). Los dos antipatrones definen los extremos del espectro de granularidad.

**Métricas para detectar un Mega-Service**:
- ¿El equipo que lo mantiene supera las 8-10 personas? (violation of Two-Pizza Team)
- ¿El servicio tiene responsabilidades que pertenecen a más de un Bounded Context DDD?
- ¿La frecuencia de deploy es baja porque los cambios siempre afectan a partes no relacionadas?

**Corrección**: descomponer aplicando DDD — identificar los Bounded Contexts dentro del Mega-Service y extraerlos como servicios independientes.

## Two-Phase Commit: antipatrón en microservicios

> [CONCEPTO] **Two-Phase Commit (2PC) como antipatrón**: el protocolo 2PC intenta garantizar atomicidad en transacciones distribuidas mediante un coordinador que primero pide a todos los participantes que preparen la transacción (fase 1: Prepare) y luego les ordena que la comenten (fase 2: Commit). El problema: durante la fase 1, todos los participantes mantienen bloqueos sobre los recursos involucrados. Si el coordinador falla entre las dos fases, los participantes quedan bloqueados indefinidamente esperando la resolución — el sistema se degrada en disponibilidad exactamente cuando más lo necesita.

El 2PC viola el teorema CAP: prioriza Consistencia sobre Disponibilidad. En microservicios con alta disponibilidad como requisito fundamental, esto lo convierte en un antipatrón.

**Alternativas que reemplazan el 2PC**:

| Necesidad | Alternativa sin 2PC |
|---|---|
| Transacción distribuida de larga duración | Saga (Patrón 13.4) |
| Publicación atómica de cambio + evento | Outbox Pattern (Patrón 13.3) |
| Consultas multi-servicio | API Composition (Patrón 13.6) o CQRS read model |

```mermaid
quadrantChart
    title Antipatrones: granularidad vs acoplamiento de despliegue
    x-axis "Demasiado fragmentado" --> "Demasiado grande"
    y-axis "Bajo acoplamiento" --> "Alto acoplamiento"
    quadrant-1 Mega-Service
    quadrant-2 Distributed Monolith
    quadrant-3 Correcto (bien delimitado)
    quadrant-4 Chatty Services
    "Chatty Services": [0.1, 0.2]
    "Nano-service": [0.05, 0.1]
    "Bien delimitado": [0.5, 0.25]
    "Mega-Service": [0.9, 0.6]
    "Distributed Monolith": [0.6, 0.9]
    "Shared Database": [0.5, 0.85]
```
*Espectro de granularidad: Chatty Services en el extremo fragmentado; Mega-Service en el extremo grande; Distributed Monolith con alto acoplamiento independientemente del tamaño.*

## Tabla de diagnóstico y corrección de antipatrones

La siguiente tabla resume los síntomas y correcciones de los cinco antipatrones para facilitar el diagnóstico rápido:

| Antipatrón | Síntoma principal | Corrección |
|---|---|---|
| Distributed Monolith | No se puede desplegar un servicio sin coordinar con otros | Rediseñar límites con DDD, comunicación asíncrona |
| Chatty Services | Decenas de llamadas para una operación | API Composition, endpoints por caso de uso |
| Shared Database | Cambio de esquema afecta a múltiples servicios | Database per Service + ACL |
| Mega-Service | Múltiples equipos en el mismo servicio | Descomponer por Bounded Context |
| Two-Phase Commit | Bloqueos distribuidos, baja disponibilidad | Saga + Outbox Pattern |

## Verificación y práctica

> [EXAMEN] 1. ¿Cómo identificas que un sistema de "microservicios" es en realidad un Distributed Monolith? Nombra dos síntomas concretos.

> [EXAMEN] 2. ¿Qué causa el antipatrón Chatty Services y cómo lo refactorizas mediante API Composition o rediseño de contratos?

> [EXAMEN] 3. ¿Por qué compartir una base de datos entre microservicios viola el principio de autonomía y cómo se corrige progresivamente?

> [EXAMEN] 4. ¿Por qué el Two-Phase Commit es un antipatrón en microservicios, y qué alternativas lo reemplazan sin bloqueo distribuido?

> [EXAMEN] 5. ¿Cuál es la diferencia entre un Chatty Services y un Mega-Service? ¿En qué extremo del espectro de granularidad se sitúa cada uno?

---

← [13.10 Patrones de despliegue](sc-patrones-deployment-patterns.md) | [Índice](README.md) | [13.12 Testing de microservicios](sc-patrones-testing.md) →
