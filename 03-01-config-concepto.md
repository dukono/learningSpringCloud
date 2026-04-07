# Parte 3.1 — Spring Cloud Config: Concepto y Arquitectura

← [Parte 2 — Arquitectura](./02-arquitectura.md) | [Volver al índice](./README.md) | Siguiente: [Backends →](./03-02-config-backends.md)

---

## 3.1 Qué es y para qué sirve

En una arquitectura de microservicios con 20, 50 o 100 servicios, cada uno tiene su propio `application.yml`. Gestionar esa configuración de forma individual es inmanejable:

- Cambiar la URL de una base de datos implica modificar y redesplegar N servicios
- Los secretos (contraseñas, API keys) están dispersos en múltiples repos
- No hay historial ni auditoría de cambios de configuración
- Distintos entornos (dev, staging, prod) requieren configuraciones distintas sin mecanismo unificado

**Spring Cloud Config** resuelve esto con un **servidor centralizado de configuración** que actúa como fuente única de verdad para todos los microservicios.

**Características clave:**
- Almacena la configuración en un repositorio Git (u otros backends)
- Los servicios la obtienen en tiempo de arranque mediante HTTP
- Soporta múltiples perfiles (`dev`, `staging`, `prod`)
- Permite refresco de configuración **sin reiniciar** los servicios
- Puede cifrar/descifrar valores sensibles

---

## 3.2 Arquitectura: Config Server y Config Client

```
┌─────────────────────────────────────┐
│         Repositorio Git             │
│  ├── application.yml                │  ← config global para todos
│  ├── pedidos-service.yml            │  ← config específica del servicio
│  ├── pedidos-service-prod.yml       │  ← config de producción
│  └── productos-service.yml          │
└─────────────────┬───────────────────┘
                  │ git pull / fetch
┌─────────────────▼───────────────────┐
│          CONFIG SERVER              │
│   (Spring Boot App con             │
│    @EnableConfigServer)             │
│                                     │
│  GET /pedidos-service/prod          │
│  → devuelve la config fusionada     │
└─────────────────┬───────────────────┘
                  │ HTTP al arrancar
        ┌─────────┼─────────┐
        ▼         ▼         ▼
  [pedidos]  [productos]  [gateway]
  (Config    (Config      (Config
   Client)    Client)      Client)
```

### El concepto de "Config Client"

Para un programador Spring Boot acostumbrado a un `application.yml` local, este es el cambio conceptual más importante:

> **[CONCEPTO]** **Config Client** es cualquier microservicio que obtiene su configuración del Config Server en lugar de leerla de su `application.yml` local. Solo necesita saber la URL del Config Server; el resto de propiedades llegan por HTTP al arrancar.

> **A partir de usar Spring Cloud Config, el `application.yml` local del microservicio ya no contiene las propiedades del negocio.** Solo contiene lo mínimo para saber dónde está el Config Server. El resto llega por HTTP al arrancar.

```
SIN Config Server:
  microservicio → lee application.yml local → arranca

CON Config Server:
  microservicio → lee application.yml local (solo URL del Config Server)
               → llama al Config Server por HTTP
               → recibe la configuración completa
               → arranca
```

**Implicación directa:** el Config Server **debe estar levantado antes** que cualquier microservicio que lo use. En Docker Compose o Kubernetes, el orden de arranque importa.

#### Forzar el orden de arranque en Docker Compose

```yaml
# docker-compose.yml
services:
  config-server:
    image: config-server:latest
    ports:
      - "8888:8888"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s   # espera inicial antes de empezar a comprobar

  pedidos-service:
    image: pedidos-service:latest
    depends_on:
      config-server:
        condition: service_healthy   # NO arranca hasta que el healthcheck pase
    environment:
      SPRING_CONFIG_IMPORT: "configserver:http://config-server:8888"
      SPRING_PROFILES_ACTIVE: dev
```

> `depends_on` sin `condition: service_healthy` solo espera a que el contenedor arranque, no a que la aplicación esté lista. Usar siempre `condition: service_healthy` con un healthcheck real.

#### Forzar el orden de arranque en Kubernetes

En K8s no existe `depends_on`. La solución es un `initContainer` que espera a que el Config Server responda:

```yaml
# deployment.yaml del microservicio
spec:
  initContainers:
    - name: wait-for-config-server
      image: curlimages/curl:8.1.0
      command:
        - sh
        - -c
        - |
          until curl -sf http://config-server:8888/actuator/health | grep '"status":"UP"'; do
            echo "Esperando Config Server..."; sleep 3
          done
          echo "Config Server listo"
  containers:
    - name: pedidos-service
      image: pedidos-service:latest
      env:
        - name: SPRING_CONFIG_IMPORT
          value: "configserver:http://config-server:8888"
```

> Alternativa más robusta para K8s: configurar `spring.cloud.config.fail-fast=true` + `retry` en el cliente (ver `03-04-config-client.md`). El microservicio arranca, reintenta con backoff exponencial y solo falla si el Config Server sigue sin responder tras todos los intentos.

### Regla de fusión de configuración

> **[CONCEPTO]** La **regla de fusión** determina qué fichero "gana" cuando la misma propiedad está definida en múltiples ficheros. El fichero más específico (el que combina servicio + perfil) siempre tiene prioridad sobre el más genérico (el `application.yml` global).

El Config Server combina propiedades en este orden de prioridad (mayor prioridad primero):

```
{service-name}-{profile}.yml   (ej: pedidos-service-prod.yml)
{service-name}.yml             (ej: pedidos-service.yml)
application-{profile}.yml      (ej: application-prod.yml)
application.yml                (global)
```

Las propiedades del nivel más específico **sobreescriben** las del nivel más genérico.

**Ejemplo concreto de fusión:**

```yaml
# application.yml (global)
timeout: 1000
logging.level.root: INFO

# pedidos-service.yml
timeout: 3000          # sobreescribe el global

# pedidos-service-prod.yml
timeout: 2000          # sobreescribe el específico del servicio
logging.level.root: WARN
```

Resultado para `pedidos-service` en perfil `prod`:
```yaml
timeout: 2000          # del fichero más específico
logging.level.root: WARN
```

---

← [Parte 2 — Arquitectura](./02-arquitectura.md) | [Volver al índice](./README.md) | Siguiente: [Backends →](./03-02-config-backends.md)
