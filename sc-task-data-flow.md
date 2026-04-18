# 11.6 Spring Cloud Task — Integración con Spring Cloud Data Flow

← [11.5 Integración con Spring Batch](sc-task-batch.md) | [Índice](README.md) | [11.7 Task Events y TaskExecutionListener](sc-task-events.md) →

---

## Introducción

Spring Cloud Data Flow (SCDF) es la plataforma de orquestación externa que permite lanzar, monitorizar y gestionar Tasks a escala en entornos cloud. Mientras que Spring Cloud Task es la librería embebida en la aplicación que registra las ejecuciones, SCDF es el servidor de control que coordina cuándo y dónde se ejecutan esas aplicaciones. La integración no requiere cambios en el código de la Task: SCDF lee directamente la tabla `TASK_EXECUTION` del datasource compartido para consultar el estado de cada ejecución.

> [CONCEPTO] Spring Cloud Data Flow es un orquestador externo. La aplicación Task no necesita dependencias de SCDF para funcionar: SCDF lanza el proceso y luego lee `TASK_EXECUTION` del datasource compartido para conocer el resultado.

## Arquitectura de la integración

La relación entre SCDF y las Tasks se articula en tres fases: registro de la definición, lanzamiento y monitorización. El siguiente diagrama muestra el flujo completo.

```
SPRING CLOUD DATA FLOW SERVER
        │
        ├─── 1. Registro de Task definition
        │    POST /tasks/definitions
        │    { "name": "etl-task", "definition": "maven://com.example:etl-task:1.0.0" }
        │
        ├─── 2. Lanzamiento de ejecución
        │    POST /tasks/executions
        │    { "name": "etl-task", "arguments": "--date=2025-01-01" }
        │    │
        │    └─── Deployer (Local/K8s/CF) lanza: java -jar etl-task.jar --date=2025-01-01
        │         La Task arranca, ejecuta lógica, escribe en TASK_EXECUTION y termina
        │
        └─── 3. Consulta de estado
             GET /tasks/executions/{id}
             └─── SCDF lee TASK_EXECUTION del datasource compartido
                  → devuelve exitCode, startTime, endTime, etc.
```

## Ejemplo central

El siguiente ejemplo muestra los pasos necesarios para registrar y lanzar una Task desde la API REST de Spring Cloud Data Flow. El código de la Task en sí es el mismo que cualquier aplicación con `@EnableTask`; la integración es del lado del operador/SCDF.

```bash
# 1. Registrar la aplicación Task en SCDF (registro de app)
curl -X POST http://localhost:9393/apps/task/etl-task \
  -d "uri=maven://com.example:etl-task:1.0.0"

# 2. Crear una Task definition con nombre lógico
curl -X POST http://localhost:9393/tasks/definitions \
  -d "name=nightly-etl&definition=etl-task"

# 3. Lanzar la Task con argumentos
curl -X POST "http://localhost:9393/tasks/executions" \
  -d "name=nightly-etl&arguments=--spring.cloud.task.name=nightly-etl,--report.date=2025-01-01"

# 4. Consultar el estado de las ejecuciones
curl http://localhost:9393/tasks/executions?name=nightly-etl

# Respuesta ejemplo:
# {
#   "content": [{
#     "executionId": 1,
#     "taskName": "nightly-etl",
#     "startTime": "2025-01-01T02:00:00",
#     "endTime": "2025-01-01T02:05:30",
#     "exitCode": 0,
#     "exitMessage": null
#   }]
# }
```

La configuración del datasource compartido en SCDF para que pueda leer `TASK_EXECUTION`:

```yaml
# application.yml del servidor SCDF
spring:
  datasource:
    url: jdbc:postgresql://shared-db:5432/tasks_db
    username: scdf_user
    password: secret
    driver-class-name: org.postgresql.Driver
  cloud:
    dataflow:
      task:
        platform:
          local:
            accounts:
              default:
                javaCmd: /usr/bin/java
```

La aplicación Task también debe conectarse al mismo datasource compartido:

```yaml
# application.yml de la Task (la aplicación Spring Boot con @EnableTask)
spring:
  datasource:
    url: jdbc:postgresql://shared-db:5432/tasks_db
    username: task_user
    password: secret
  cloud:
    task:
      initialize-enabled: false  # SCDF ya creó el schema
```

## Tabla de elementos clave

Los componentes del ecosistema SCDF relevantes para la integración con Spring Cloud Task son los siguientes. Los elementos de SCDF propiamente dicho (Skipper, Streams) están fuera del alcance de este módulo.

| Componente | Origen | Descripción |
|---|---|---|
| SCDF Server | Spring Cloud Data Flow | Servidor de orquestación; expone REST API en puerto 9393 |
| Task Definition | SCDF | Nombre lógico asociado a un URI de artefacto |
| Task Execution (SCDF) | SCDF | Vista sobre `TASK_EXECUTION` del datasource compartido |
| `POST /tasks/executions` | SCDF REST API | Endpoint para lanzar una Task con parámetros |
| `GET /tasks/executions` | SCDF REST API | Endpoint para consultar ejecuciones por nombre |
| Deployer Local | spring-cloud-deployer-local | Lanza la Task como proceso Java en la misma máquina |
| Deployer Kubernetes | spring-cloud-deployer-kubernetes | Lanza la Task como Pod en Kubernetes |
| Deployer Cloud Foundry | spring-cloud-deployer-cloudfoundry | Lanza la Task como Task en Cloud Foundry |

## Deployers disponibles

SCDF soporta múltiples plataformas de despliegue mediante la abstracción de deployers. El deployer recibe la especificación de la Task (imagen Docker o URI de artefacto Maven) y crea el proceso/contenedor en la plataforma objetivo.

El deployer Local es adecuado para desarrollo y pruebas: lanza un proceso Java en el mismo servidor donde corre SCDF. El deployer Kubernetes crea un `Pod` (o `Job`) con la imagen Docker de la Task y, una vez finalizado, SCDF lee el estado desde `TASK_EXECUTION`. El deployer Cloud Foundry usa la API de CF para lanzar una Task de CF apuntando al artefacto.

> [ADVERTENCIA] SCDF y la Task deben compartir el mismo datasource y el mismo schema de `TASK_EXECUTION` para que SCDF pueda leer el estado de ejecución. Si usan bases de datos distintas, SCDF no podrá consultar el resultado de la Task.

## Relación Task-Stream en SCDF

SCDF permite combinar Tasks con Streams en pipelines más complejos. Un Stream puede publicar eventos que disparan Tasks, o una Task puede iniciar un Stream. Este tipo de pipeline se configura en el DSL de SCDF y es gestionado por Skipper (el gestor de ciclo de vida de Streams). Para el módulo de Spring Cloud Task, el punto de contacto relevante es que las Tasks pueden recibir parámetros desde un Stream vía mensajes.

## Buenas y malas prácticas

**Buenas prácticas:**
- Asegurarse de que SCDF y la Task usen el mismo datasource compartido con las mismas tablas `TASK_EXECUTION` para que la monitorización funcione correctamente.
- Configurar `spring.cloud.task.initialize-enabled=false` en la Task cuando SCDF ya ha creado el schema: evita intentos de crear tablas existentes.
- Versionar las Task definitions en SCDF alineadas con las versiones del artefacto Maven o imagen Docker.

**Malas prácticas:**
- Mezclar la configuración del servidor SCDF en la aplicación Task: son componentes completamente independientes con sus propios `application.yml`.
- Asumir que SCDF monitoriza la Task en tiempo real: SCDF lee `TASK_EXECUTION` en el momento de la consulta, no hay polling ni WebSocket activo por defecto.
- Ignorar la diferencia entre "Task registrada en SCDF" y "Task ejecutada": una Task puede estar registrada pero no haberse ejecutado aún.

> [PREREQUISITO] Para que SCDF pueda lanzar una Task, el artefacto (JAR o imagen Docker) debe estar accesible para el deployer: repositorio Maven accesible, registro Docker accesible, o ruta local disponible. Si el deployer no puede resolver el artefacto, el lanzamiento falla antes de que la Task arranque.

## Verificación y práctica

> [EXAMEN] **Pregunta 1:** ¿Cómo consulta Spring Cloud Data Flow el estado de una ejecución de Task y de dónde obtiene esa información?

> [EXAMEN] **Pregunta 2:** ¿Cuáles son los tres tipos de deployers disponibles en SCDF para lanzar Tasks?

> [EXAMEN] **Pregunta 3:** ¿Qué endpoint REST de SCDF se usa para lanzar una Task con parámetros?

> [EXAMEN] **Pregunta 4:** ¿Por qué es necesario que SCDF y la aplicación Task compartan el mismo datasource?

> [EXAMEN] **Pregunta 5:** ¿Qué diferencia hay entre "Task definition" y "Task execution" en el modelo de SCDF?

---

← [11.5 Integración con Spring Batch](sc-task-batch.md) | [Índice](README.md) | [11.7 Task Events y TaskExecutionListener](sc-task-events.md) →
