# 7.2 Spring Cloud Bus — Setup y auto-configuración

← [7.1 Spring Cloud Bus — Arquitectura y propósito](sc-bus-arquitectura.md) | [Índice](README.md) | [7.3 Spring Cloud Bus — Refresh distribuido de configuración](sc-bus-refresh-distribuido.md) →

---

## Introducción

Configurar Spring Cloud Bus es un proceso de muy bajo esfuerzo gracias a la auto-configuración de Spring Boot. Añadir el starter adecuado junto con las propiedades mínimas del broker es suficiente para que `BusAutoConfiguration` configure los canales de Spring Cloud Stream, registre los listeners de eventos y exponga los endpoints de Actuator.

> [PREREQUISITO] Antes de añadir Spring Cloud Bus es necesario tener Spring Boot Actuator en el classpath, ya que los endpoints `/actuator/bus-refresh` y `/actuator/bus-env` se construyen sobre la infraestructura de Actuator.

## Dependencias — Maven y Gradle

La elección del starter determina qué broker se usa. Ambos starters incluyen transitivamente `spring-cloud-bus`, el binder correspondiente de Spring Cloud Stream, y el cliente del broker.

Las dependencias Maven para cada broker son las siguientes:

```xml
<!-- pom.xml — opción A: RabbitMQ (AMQP) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>

<!-- pom.xml — opción B: Apache Kafka -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

Las dependencias equivalentes en Gradle son:

```groovy
// build.gradle — opción A: RabbitMQ
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}

// build.gradle — opción B: Kafka
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

> [CONCEPTO] No se puede usar ambos starters simultáneamente en el mismo proyecto sin configuración adicional. Si se necesita soporte para múltiples brokers, hay que configurar múltiples binders de Spring Cloud Stream manualmente.

## BusAutoConfiguration y activación automática

`BusAutoConfiguration` es la clase de auto-configuración principal del Bus. Se activa cuando detecta `spring-cloud-bus` en el classpath y la propiedad `spring.cloud.bus.enabled` es `true` (valor por defecto).

Las responsabilidades de `BusAutoConfiguration` son:

| Responsabilidad | Descripción |
|----------------|-------------|
| Crear `BusProperties` | Lee y valida todas las propiedades `spring.cloud.bus.*` |
| Configurar `ServiceMatcher` | Construye el comparador de destinos a partir de `spring.cloud.bus.id` |
| Registrar `BusRefreshEndpoint` | Expone `/actuator/bus-refresh` si está habilitado |
| Registrar `BusEnvironmentEndpoint` | Expone `/actuator/bus-env` si está habilitado |
| Crear el publisher de eventos | Configura el canal de salida de Stream para publicar eventos |

## Ejemplo central — Configuración completa con RabbitMQ y Kafka

El siguiente ejemplo muestra la configuración completa de dos variantes de `application.yml`, una para RabbitMQ y otra para Kafka, con todos los parámetros relevantes explicados.

```yaml
# application.yml — configuración con RabbitMQ
spring:
  application:
    name: config-client
  profiles:
    active: dev

  # Conexión al broker RabbitMQ
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /

  cloud:
    bus:
      enabled: true
      # Nombre del exchange fanout en RabbitMQ
      destination: springCloudBus
      # Identificador único de esta instancia: appName:profile:port
      id: ${spring.application.name}:${spring.profiles.active:default}:${server.port:8080}
      refresh:
        enabled: true    # habilita /actuator/bus-refresh
      env:
        enabled: true    # habilita /actuator/bus-env

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env, health, info, refresh
  endpoint:
    bus-refresh:
      enabled: true
    bus-env:
      enabled: true
```

```yaml
# application.yml — configuración con Apache Kafka
spring:
  application:
    name: config-client
  profiles:
    active: dev

  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        # Consumer group único por instancia para evitar mensajes duplicados
        springCloudBusInput:
          group: ${spring.application.name}-${random.uuid}
    bus:
      enabled: true
      destination: springCloudBus    # nombre del topic Kafka
      id: ${spring.application.name}:${spring.profiles.active:default}:${server.port:8080}
      refresh:
        enabled: true
      env:
        enabled: true

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh, bus-env, health, info
```

> [ADVERTENCIA] Con Kafka es imprescindible configurar un consumer group único por instancia en `spring.cloud.stream.bindings.springCloudBusInput.group`. Si todas las instancias del mismo servicio comparten el mismo consumer group, Kafka solo entregará cada mensaje a una instancia del grupo, impidiendo el broadcast.

## Tabla de propiedades spring.cloud.bus.*

Las propiedades del namespace `spring.cloud.bus` controlan todos los aspectos del comportamiento del Bus.

| Propiedad | Valor por defecto | Descripción |
|-----------|-------------------|-------------|
| `spring.cloud.bus.enabled` | `true` | Activa o desactiva el Bus completamente |
| `spring.cloud.bus.destination` | `springCloudBus` | Nombre del topic Kafka o exchange RabbitMQ |
| `spring.cloud.bus.id` | Auto-generado | Identificador único de la instancia (`appName:profiles:index`) |
| `spring.cloud.bus.refresh.enabled` | `true` | Habilita el endpoint `/actuator/bus-refresh` |
| `spring.cloud.bus.env.enabled` | `true` | Habilita el endpoint `/actuator/bus-env` |

## Verificar la activación del Bus

Una vez configurado el Bus, es posible verificar su activación de varias formas. La primera es observar los logs de arranque, donde `BusAutoConfiguration` registra los bindings de Stream.

La segunda es consultar el endpoint de Actuator para ver los endpoints disponibles:

```bash
# Listar endpoints Actuator disponibles
curl http://localhost:8080/actuator | jq '.["_links"] | keys[]' | grep bus

# Resultado esperado:
# "bus-env"
# "bus-refresh"
```

La tercera es verificar los bindings activos de Spring Cloud Stream:

```bash
# Ver bindings activos del Bus en Actuator
curl http://localhost:8080/actuator/bindings | jq .

# Se deben ver springCloudBusInput y springCloudBusOutput activos
```

## Buenas y malas prácticas

**Buenas prácticas:**

- Configurar `spring.cloud.bus.id` explícitamente en entornos de contenedores (Docker, Kubernetes) donde el nombre del host puede no ser único o estable.
- Exponer solo los endpoints de Bus necesarios. Usar `management.endpoints.web.exposure.include` con lista explícita en lugar de `*`.
- En entornos con múltiples perfiles, verificar que el `bus.id` incluye el perfil activo para identificar correctamente la instancia.

**Malas prácticas:**

- [LEGACY] Añadir `@EnableBus` — esta anotación no existe en Spring Cloud 3.x. La auto-configuración es suficiente.
- Usar `management.endpoints.web.exposure.include=*` en producción. Esto expone todos los endpoints de Actuator, incluyendo los sensibles.
- Omitir la configuración del consumer group en Kafka. Sin un group único, el comportamiento del Bus es impredecible.

## Verificación y práctica

> [EXAMEN] **1.** ¿Qué starter debe añadirse para usar Spring Cloud Bus con RabbitMQ, y qué dependencias incluye transitivamente?

> [EXAMEN] **2.** ¿Qué propiedad permite deshabilitar completamente Spring Cloud Bus sin eliminar la dependencia?

> [EXAMEN] **3.** ¿Por qué es necesario configurar un consumer group único en `springCloudBusInput` cuando se usa Kafka, y qué ocurre si no se hace?

> [EXAMEN] **4.** ¿Qué propiedad debe exponerse en `management.endpoints.web.exposure.include` para que el endpoint `/actuator/bus-refresh` sea accesible por HTTP?

> [EXAMEN] **5.** ¿Cuál es el formato por defecto del `spring.cloud.bus.id` y para qué sirve este identificador?

---

← [7.1 Spring Cloud Bus — Arquitectura y propósito](sc-bus-arquitectura.md) | [Índice](README.md) | [7.3 Spring Cloud Bus — Refresh distribuido de configuración](sc-bus-refresh-distribuido.md) →
