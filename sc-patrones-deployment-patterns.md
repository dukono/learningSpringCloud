# 13.10 Patrones de despliegue: Strangler Fig, Sidecar, ACL y Database per Service

← [13.9 Patrones de seguridad](sc-patrones-seguridad-patrones.md) | [Índice](README.md) | [13.11 Antipatrones](sc-patrones-antipatrones.md) →

---

## Introducción

Los patrones de despliegue abordan cómo se estructura físicamente un sistema de microservicios y cómo se migra desde un monolito. El patrón Strangler Fig permite migraciones incrementales sin big-bang rewrite. El patrón Sidecar delega responsabilidades cross-cutting a un contenedor co-desplegado. La Anti-Corruption Layer protege los modelos de dominio entre bounded contexts. Database per Service garantiza la autonomía de datos que hace posible el despliegue independiente.

## Strangler Fig: migración incremental del monolito

> [CONCEPTO] **Strangler Fig**: el nombre viene de una higuera tropical que crece alrededor de un árbol huésped y eventualmente lo reemplaza. En software, el patrón consiste en construir nuevos microservicios alrededor del monolito existente, redirigiendo gradualmente funcionalidades del monolito a los nuevos servicios, hasta que el monolito queda vacío y puede eliminarse. Es el único enfoque probado para migrar monolitos sin interrupciones de servicio.

El API Gateway es el componente central del Strangler Fig: actúa como la fachada que enruta el tráfico al monolito o a los nuevos microservicios según el estado de la migración.

```
FASE 1 (inicio):
Cliente ──► Gateway ──► Monolito (100% del tráfico)

FASE 2 (migración):
                      ──► Monolito (funcionalidades no migradas)
Cliente ──► Gateway ─┤
                      ──► Orders Service (nueva implementación)

FASE 3 (completado):
Cliente ──► Gateway ──► Microservicios (monolito eliminado)
```

**Estrategias de extracción durante la migración:**
1. Empezar por las funcionalidades más independientes (menos acopladas al resto del monolito).
2. Usar el Anti-Corruption Layer para aislar el nuevo microservicio del modelo de datos del monolito.
3. Mantener sincronizados los datos del monolito y el nuevo servicio durante la coexistencia (event sourcing del monolito o dual-write con reconciliación).

## Sidecar: responsabilidades cross-cutting como contenedor adicional

> [CONCEPTO] **Sidecar**: en Kubernetes, el patrón Sidecar despliega un contenedor adicional en el mismo Pod que el servicio principal. El sidecar comparte el namespace de red, el filesystem y el ciclo de vida del pod con el servicio principal. Delega responsabilidades cross-cutting (proxy de red, logging, TLS termination, healthchecks, recolección de métricas) al sidecar sin modificar el código del servicio.

El patrón Sidecar es la base de los Service Meshes como Istio (usa Envoy como sidecar) y Linkerd. El sidecar intercepta todo el tráfico de red del pod y aplica las políticas de red (mTLS, retries, circuit breaking) de forma transparente al servicio.

```yaml
# Ejemplo de Pod con sidecar proxy (Envoy / Istio)
# En producción, Istio inyecta el sidecar automáticamente con webhook
apiVersion: v1
kind: Pod
metadata:
  name: orders-service-pod
spec:
  containers:
    - name: orders-service          # contenedor principal
      image: orders-service:1.0.0
      ports:
        - containerPort: 8080

    - name: envoy-proxy             # sidecar: proxy de red
      image: envoyproxy/envoy:v1.28
      ports:
        - containerPort: 9901       # admin interface
      volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
  volumes:
    - name: envoy-config
      configMap:
        name: envoy-config
```

## Anti-Corruption Layer (ACL)

> [CONCEPTO] **Anti-Corruption Layer (ACL)**: la ACL es una capa de traducción que protege el modelo de dominio propio de la "corrupción" que introduciría adoptar directamente el modelo de dominio de un sistema externo o legacy. Se introduce cuando dos Bounded Contexts tienen modelos incompatibles y no se puede modificar el modelo externo (legacy, API de terceros, monolito durante la migración).

La ACL implementa el patrón Adapter o Façade: traduce las entidades del sistema externo al lenguaje ubicuo del bounded context propio. En el módulo de Descomposición (13.1) se muestra un ejemplo de ACL entre Orders y el servicio de Inventario.

## Database per Service: autonomía de datos

> [CONCEPTO] **Database per Service**: cada microservicio posee exclusivamente su base de datos. Ningún otro servicio puede acceder directamente a esa base de datos — la única forma de leer o modificar los datos es a través de la API del servicio propietario. Este principio es lo que hace posible el despliegue independiente y la autonomía real de los microservicios.

Los problemas que este patrón introduce y sus soluciones:

| Problema | Solución |
|---|---|
| JOINs entre servicios ya no son posibles | API Composition / Aggregator (patrón 13.6) |
| Consistencia transaccional entre servicios | Saga (patrón 13.4) |
| Datos duplicados en proyecciones | CQRS con proyecciones (patrón 13.5) |
| Consultas complejas multi-servicio | Read model materializado actualizado por eventos |

## Ejemplo central: configuración Strangler Fig con Spring Cloud Gateway

El siguiente ejemplo configura el API Gateway para implementar el patrón Strangler Fig, enrutando gradualmente el tráfico del monolito a los nuevos microservicios según las rutas migradas. La configuración permite migración incremental sin cambios en los clientes.

```yaml
# application.yml — Spring Cloud Gateway como Strangler Fig facade

spring:
  cloud:
    gateway:
      routes:
        # Rutas ya migradas a microservicios — enrutan al nuevo servicio
        - id: orders-migrated
          uri: lb://orders-service         # descubierto via Eureka
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
          order: 1                          # mayor prioridad que el catch-all del monolito

        - id: inventory-migrated
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - StripPrefix=1
          order: 1

        # Catch-all: todo lo que no ha migrado va al monolito
        - id: legacy-monolith
          uri: http://legacy-monolith:8080  # URL directa del monolito
          predicates:
            - Path=/**
          order: 999                        # menor prioridad — solo si no matchea antes
```

```java
// ACL para proteger el modelo de Orders del modelo legacy del monolito
package com.example.orders.infrastructure.acl;

import com.example.orders.domain.Customer;
import com.example.legacy.client.LegacyUserClient;
import com.example.legacy.client.dto.LegacyUserDTO;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component
public class LegacyCustomerACL {

    private final LegacyUserClient legacyUserClient;

    public LegacyCustomerACL(LegacyUserClient legacyUserClient) {
        this.legacyUserClient = legacyUserClient;
    }

    // Traduce el modelo legacy (LegacyUserDTO) al modelo de dominio propio (Customer)
    // Cacheable para evitar llamadas repetidas al monolito durante la coexistencia
    @Cacheable(value = "customers", key = "#userId")
    public Customer findCustomer(String userId) {
        LegacyUserDTO legacyUser = legacyUserClient.getUser(userId);

        // La ACL aísla el modelo de dominio de Orders del esquema del monolito
        // Si el monolito cambia LegacyUserDTO, solo cambia esta traducción
        return new Customer(
            legacyUser.getUserId(),              // monolito: userId → dominio: customerId
            legacyUser.getFullName(),            // monolito: fullName → dominio: name
            legacyUser.getPrimaryEmail(),        // monolito: primaryEmail → dominio: email
            legacyUser.getShippingAddressList()  // monolito: lista → dominio: dirección por defecto
                .stream().findFirst()
                .map(addr -> new Address(addr.getStreet(), addr.getCity(), addr.getZipCode()))
                .orElseThrow(() -> new CustomerHasNoAddressException(userId))
        );
    }
}
```

## Buenas y malas prácticas

**Buenas prácticas:**
- En Strangler Fig, empezar por las funcionalidades con menos dependencias en el monolito.
- Usar feature flags para controlar el porcentaje de tráfico enrutado al nuevo microservicio durante la migración.
- En Database per Service, las migraciones de esquema son responsabilidad exclusiva del servicio propietario (sin scripts compartidos).
- La ACL debe estar en el lado del contexto que quiere protegerse del modelo externo, no en el sistema externo.

**Malas prácticas:**
- Hacer un big-bang rewrite del monolito en microservicios sin un plan de migración incremental.
- Acceder directamente a la base de datos de otro microservicio (violación de Database per Service) — aunque sea más fácil, acopla los despliegues.
- Crear un Sidecar con lógica de negocio — el Sidecar es solo para infraestructura cross-cutting.

> [ADVERTENCIA] Durante la migración con Strangler Fig, el estado puede existir tanto en el monolito como en el nuevo microservicio. Es necesario un mecanismo de sincronización de datos (event-driven sync, dual-write, o migración de datos one-time) para mantener consistencia. La fase de coexistencia es la más compleja y propensa a inconsistencias.

## Verificación y práctica

> [EXAMEN] 1. ¿Cómo aplicas el Strangler Fig para migrar un monolito a microservicios de forma incremental, y qué rol juega el API Gateway en ese proceso?

> [EXAMEN] 2. ¿Qué responsabilidades cross-cutting delega el patrón Sidecar en un contenedor co-desplegado, y cómo se relaciona con el Service Mesh?

> [EXAMEN] 3. ¿Cuándo introduces una Anti-Corruption Layer entre dos Bounded Contexts, y qué traduce exactamente esa capa?

> [EXAMEN] 4. ¿Por qué Database per Service es una regla fundamental en microservicios, y qué problemas genera en consultas que antes eran JOINs en el monolito?

> [EXAMEN] 5. ¿Qué estrategias existen para sincronizar datos entre el monolito y un nuevo microservicio durante la fase de coexistencia en el Strangler Fig?

---

← [13.9 Patrones de seguridad](sc-patrones-seguridad-patrones.md) | [Índice](README.md) | [13.11 Antipatrones](sc-patrones-antipatrones.md) →
