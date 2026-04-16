# 2.8 Alta disponibilidad: Eureka Server en cluster

← [2.7 Dashboard y API REST del Eureka Server](sc-eureka-api-rest.md) | [Índice (README.md)](README.md) | [2.9 Seguridad del Eureka Server →](sc-eureka-seguridad.md)

---

Un único Eureka Server es un single point of failure: si cae, los servicios existentes continúan funcionando gracias a su caché local, pero ningún servicio nuevo puede registrarse y los consumidores no ven los cambios en el registro. Para producción, Eureka ofrece el modo peer-aware, donde múltiples instancias del servidor forman un cluster y replican el registro entre sí. Cada nodo acepta registros de clientes y los propaga al resto de peers; si un nodo cae, los demás continúan sirviendo el registro completo. Esta sección cubre la configuración de un cluster de dos o tres nodos, la gestión de zonas para optimizar la latencia, y el comportamiento del sistema ante la partición de red (split-brain).

> [PREREQUISITO] El lector debe haber configurado el Eureka Server en modo standalone (sección 2.2) y comprendido el modelo de lease y heartbeat (sección 2.5.1) antes de abordar el cluster, ya que la replicación se basa en los mismos mecanismos de registro que opera cada nodo de forma independiente.

## Diagrama: cluster de Eureka con dos nodos y clientes con zonas

El siguiente diagrama muestra la topología de un cluster de dos nodos con clientes distribuidos en dos zonas de disponibilidad, y cómo se propagan los registros entre peers.

```
ZONA eu-west-1a                    ZONA eu-west-1b
┌──────────────────┐               ┌──────────────────┐
│  eureka-server-1 │◄─────────────►│  eureka-server-2 │
│  :8761           │   Replicación │  :8762           │
│                  │   bidireccional│                  │
└────────┬─────────┘               └─────────┬────────┘
         │                                   │
         │ Registro + heartbeat              │ Registro + heartbeat
         │                                   │
┌────────▼─────────┐               ┌─────────▼────────┐
│ servicio-pedidos │               │servicio-catalogo  │
│ inst1 :8081      │               │ inst1 :8082       │
│ (prefer-same-zone│               │(prefer-same-zone  │
│   → eureka-1)    │               │  → eureka-2)      │
└──────────────────┘               └──────────────────┘

Flujo de registro:
1. servicio-pedidos:inst1 → PUT /eureka/apps a eureka-server-1
2. eureka-server-1 → replica el registro a eureka-server-2
3. servicio-catalogo:inst1 → PUT /eureka/apps a eureka-server-2
4. eureka-server-2 → replica a eureka-server-1
5. Ambos servidores tienen el registro completo
```

## Implementación: cluster de dos nodos con perfiles Spring

La forma recomendada de configurar un cluster de Eureka es con perfiles Spring independientes para cada nodo, usando una única aplicación desplegada con distintos perfiles activos.

**pom.xml (igual que standalone, sin cambios adicionales)**

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2025.1.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

**EurekaServerApplication.java (idéntica al modo standalone)**

```java
package com.ejemplo.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**application.yml (configuración base compartida)**

```yaml
spring:
  application:
    name: eureka-server

eureka:
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.75
    eviction-interval-timer-in-ms: 60000
    # Número máximo de hilos de replicación entre peers
    max-threads-for-peer-replication: 20
    # Tiempo de espera para registrar peers iniciales en milisegundos
    peer-eureka-nodes-update-interval-ms: 600000

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

**application-peer1.yml (nodo 1 en eu-west-1a)**

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: eureka-server-1

  client:
    # En modo cluster, el servidor SÍ se registra con sus peers
    register-with-eureka: true
    # En modo cluster, el servidor SÍ descarga el registro de sus peers
    fetch-registry: true
    service-url:
      # Peer2 es el servidor al que apunta peer1 para sincronización.
      # En un cluster de 3 nodos, incluir los 3 separados por coma.
      defaultZone: http://eureka-server-2:8762/eureka/
    # Zona de este nodo (para routing de zona en los clientes)
    availability-zones:
      eu-west-1: "eu-west-1a,eu-west-1b"
    region: eu-west-1
```

**application-peer2.yml (nodo 2 en eu-west-1b)**

```yaml
server:
  port: 8762

eureka:
  instance:
    hostname: eureka-server-2

  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-server-1:8761/eureka/
    availability-zones:
      eu-west-1: "eu-west-1a,eu-west-1b"
    region: eu-west-1
```

**Arranque de cada nodo**

```bash
# Nodo 1 con perfil peer1
java -jar eureka-server.jar --spring.profiles.active=peer1

# Nodo 2 con perfil peer2
java -jar eureka-server.jar --spring.profiles.active=peer2
```

**Configuración del cliente para apuntar al cluster y preferir zona**

```yaml
# application.yml del microservicio cliente
spring:
  application:
    name: servicio-pedidos

eureka:
  client:
    # Múltiples servers separados por coma: el cliente intenta todos en orden
    # para registro y fetch. La primera URL disponible se usa para heartbeat.
    service-url:
      defaultZone: http://eureka-server-1:8761/eureka/,http://eureka-server-2:8762/eureka/

    # Routing de zona: el cliente prefiere llamar a servicios en su misma zona.
    # Reduce latencia de red en topologías multi-zona.
    prefer-same-zone-eureka: true
    region: eu-west-1
    availability-zones:
      eu-west-1: "eu-west-1a,eu-west-1b"

  instance:
    # Declara en qué zona está este cliente
    metadata-map:
      zone: "eu-west-1a"
    prefer-ip-address: true
```

**Cluster de tres nodos (fragmento de application-peer1.yml para 3 nodos)**

```yaml
# Con tres nodos, cada nodo debe apuntar a los otros dos
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server-2:8762/eureka/,http://eureka-server-3:8763/eureka/
```

> [ADVERTENCIA] En un cluster de N nodos, cada nodo debe referenciar a los otros N-1 nodos en `defaultZone`. Si un nodo solo conoce a un peer y ese peer cae, el nodo pierde la sincronización con el resto del cluster aunque los demás nodos sigan activos.

## Tabla de propiedades del cluster

La siguiente tabla recoge las propiedades específicas del modo peer-aware que un senior debe conocer.

| Parámetro | Tipo | Default | Descripción |
|---|---|---|---|
| `eureka.client.register-with-eureka` | boolean | `true` | En el servidor de cluster, debe ser `true` para registrarse con peers |
| `eureka.client.fetch-registry` | boolean | `true` | En el servidor de cluster, debe ser `true` para descargar el registro de peers |
| `eureka.client.service-url.defaultZone` | String | `http://localhost:8761/eureka/` | URLs de los peers separadas por coma |
| `eureka.client.prefer-same-zone-eureka` | boolean | `false` | Preferir instancias en la misma zona para reducir latencia |
| `eureka.client.region` | String | `us-east-1` | Región en la que opera el cliente |
| `eureka.client.availability-zones.{region}` | String | — | Zonas disponibles de la región, separadas por coma |
| `eureka.server.peer-eureka-nodes-update-interval-ms` | long | `600000` | Frecuencia de refresco de la lista de peers conocidos |
| `eureka.server.max-threads-for-peer-replication` | int | `20` | Máximo de hilos para replicar registros a peers |

## Buenas y malas prácticas

Hacer:
- Desplegar siempre un número impar de nodos Eureka en cluster (3 o 5): con 2 nodos y un fallo de red entre ellos, ambos nodos se consideran aislados y activan self-preservation simultáneamente, que es el escenario de split-brain. Con 3 nodos, al menos 2 pueden mantener el quórum.
- Usar `prefer-same-zone-eureka: true` en topologías multi-zona: el cliente enviará sus heartbeats al servidor Eureka más cercano (misma zona), reduciendo la latencia del heartbeat y la probabilidad de falsos expirations por latencia de red inter-zona.
- Incluir las URLs de todos los peers en la configuración del cliente (`defaultZone`): si el cliente solo conoce un peer y ese peer cae, el cliente no puede hacer heartbeat aunque los otros peers del cluster estén activos, lo que resulta en un desregistro innecesario.

Evitar:
- Usar DNS round-robin como única forma de balancear entre peers de Eureka en el cliente: si el DNS resuelve a un peer caído y el cliente no tiene otras URLs configuradas, el heartbeat falla durante el TTL de DNS (que puede ser minutos). Es más seguro listar todas las URLs explícitamente.
- Asumir que el registro es consistente inmediatamente después de un cambio: la replicación entre peers es asíncrona. Un registro en peer1 puede tardar varios segundos en aparecer en peer2, durante los cuales los clientes que consultan peer2 no ven la nueva instancia.
- Desplegar los nodos Eureka en el mismo rack físico o zona de disponibilidad: si toda la zona cae, se pierde el cluster completo. Los nodos deben estar en zonas o racks independientes para que la topología sea realmente de alta disponibilidad.

## Comparación: comportamiento ante fallo según número de nodos

El comportamiento del cluster ante un fallo de nodo depende del número de nodos y la distribución de los clientes.

| Escenario | Cluster 1 nodo (standalone) | Cluster 2 nodos | Cluster 3 nodos |
|---|---|---|---|
| Fallo de 1 nodo | Registro inaccesible | 1 nodo disponible (sin quórum clásico) | 2 nodos disponibles |
| Nuevos registros durante el fallo | Imposibles | Posibles en el nodo superviviente | Posibles en ambos supervivientes |
| Self-preservation durante split-brain | N/A | Ambos nodos activan self-preservation | Solo los nodos particionados |
| Reconvergencia tras fallo | N/A | Automática al restaurar el peer | Automática |

> [EXAMEN] En entrevistas se pregunta con frecuencia: "¿Qué ocurre en Eureka cuando hay un split-brain entre los servidores del cluster?". La respuesta correcta es: cada partición activa self-preservation de forma independiente (porque la mitad de los peers dejan de responder), lo que significa que ninguna de las dos particiones elimina instancias registradas con ellas. Cuando la partición se repara, los registros se re-sincronizan entre peers. El modelo AP de Eureka garantiza disponibilidad sobre consistencia: puede haber divergencia temporal entre peers, pero siempre se puede registrar y descubrir.

---

← [2.7 Dashboard y API REST del Eureka Server](sc-eureka-api-rest.md) | [Índice (README.md)](README.md) | [2.9 Seguridad del Eureka Server →](sc-eureka-seguridad.md)
