# 10.12 Spring Cloud Contract — Troubleshooting

← [10.11 Workflow CI/CD](sc-contract-workflow.md) | [Índice](README.md) | [11.1 Spring Cloud Task — Fundamentos](sc-task-fundamentos.md) →

---

## Introducción

El troubleshooting en Spring Cloud Contract se centra en tres categorías principales de errores: stubs no encontrados por Stub Runner, mismatches entre la petición real y el stub WireMock, y contratos obsoletos que no reflejan la implementación actual del productor. Conocer cómo diagnosticar cada uno de estos errores y los pasos de verificación adecuados es contenido práctico y de examen en Spring Cloud Contract.

> [PREREQUISITO] Este nodo requiere comprensión completa del flujo CDC, el plugin del productor y el Stub Runner del consumidor de los nodos anteriores de este módulo.

## Error 1: "No stubs found"

El error `No stubs found` es el más frecuente en el consumidor y ocurre cuando Stub Runner no puede localizar el JAR de stubs según la configuración de `ids` y `stubsMode`.

```
Síntoma típico en el log del consumidor:
─────────────────────────────────────────────────────────────────
org.springframework.cloud.contract.stubrunner.StubRunnerException:
  Stubs were not found on classpath [com.example:order-service:+:stubs]
─────────────────────────────────────────────────────────────────
o
─────────────────────────────────────────────────────────────────
java.lang.IllegalArgumentException:
  No stubs found for [com.example:order-service:1.0.0:stubs] in [LOCAL]
─────────────────────────────────────────────────────────────────
```

```
Lista de verificación para "No stubs found":

PARA stubsMode = LOCAL:
  □ ¿Ha ejecutado el productor mvn install recientemente?
  □ ¿Existe el JAR en ~/.m2/repository/com/example/order-service/1.0.0/
         order-service-1.0.0-stubs.jar ?
  □ ¿El groupId:artifactId:version en ids coincide exactamente con el pom.xml del productor?
  □ ¿El clasificador 'stubs' coincide con el configurado en el plugin del productor?

PARA stubsMode = CLASSPATH:
  □ ¿Está declarado el JAR de stubs como dependencia de test con classifier="stubs"?
  □ ¿Está activado generate-stubs=true si los contratos están en el classpath?
  □ ¿El JAR de stubs contiene el directorio /mappings/ con los JSONs?

PARA stubsMode = REMOTE:
  □ ¿Está configurado repositoryRoot con la URL correcta?
  □ ¿El JAR ha sido publicado en el repositorio remoto con mvn deploy?
  □ ¿Las credenciales de acceso al repositorio son correctas?
  □ ¿Hay conectividad de red al repositorio desde el entorno de CI?
```

> [CONCEPTO] El primer paso siempre es verificar que `groupId:artifactId:version` en el campo `ids` coincide exactamente con las coordenadas del productor. Un error de tipografía en cualquiera de las tres partes genera `No stubs found` aunque el JAR exista.

## Diagnóstico de "No stubs found" paso a paso

El siguiente procedimiento sistemático cubre los casos más frecuentes de `No stubs found`.

```bash
# Paso 1: verificar que el JAR de stubs existe en el repositorio local
ls ~/.m2/repository/com/example/order-service/1.0.0/
# Debe mostrar: order-service-1.0.0.jar, order-service-1.0.0-stubs.jar, order-service-1.0.0.pom

# Paso 2: si el JAR no existe, ejecutar install en el productor
cd /path/to/order-service
mvn install

# Paso 3: verificar el contenido del JAR de stubs
jar tf ~/.m2/repository/com/example/order-service/1.0.0/order-service-1.0.0-stubs.jar
# Debe mostrar: mappings/order/shouldReturnOrder.json, etc.

# Paso 4: verificar la configuración del consumidor
# Comprobar que ids coincide exactamente:
# "com.example:order-service:1.0.0:stubs:8090"
# vs
# pom.xml del productor: <groupId>com.example</groupId><artifactId>order-service</artifactId><version>1.0.0</version>

# Paso 5: habilitar logging verbose de Stub Runner
# En application.yml de test del consumidor:
logging:
  level:
    org.springframework.cloud.contract: DEBUG
    com.github.tomakehurst.wiremock: DEBUG
```

## Error 2: Mismatched request (stub no responde)

Un mismatch ocurre cuando el consumidor realiza una petición que no coincide con ningún stub WireMock. El servidor WireMock devuelve 404 con un cuerpo que describe la diferencia entre la petición real y el stub esperado.

```
Síntoma típico en el log del consumidor:
─────────────────────────────────────────────────────────────────
com.github.tomakehurst.wiremock.client.VerificationException:
  Request was not matched:
  {
    "url" : "/orders/1",
    "absoluteUrl" : "http://localhost:8090/orders/1",
    "method" : "GET",
    "headers" : {
      "Accept" : "application/xml"    ← diferente al esperado "application/json"
    }
  }
  Closest stub:
  {
    "request" : {
      "url" : "/orders/1",
      "method" : "GET",
      "headers" : { "Accept" : { "equalTo": "application/json" } }
    }
  }
─────────────────────────────────────────────────────────────────
```

```
Lista de verificación para "Mismatched request":

  □ ¿El método HTTP del cliente coincide con el del contrato? (GET vs POST, etc.)
  □ ¿La URL del cliente coincide exactamente? (case-sensitive, sin trailing slash)
  □ ¿Los headers del request coinciden? (especialmente Content-Type y Accept)
  □ ¿El body del request coincide con el contrato (para POST/PUT)?
  □ ¿Los query parameters están correctamente codificados?
  □ ¿Los matchers del request (bodyMatchers) en el contrato coinciden con el body real?
```

## Habilitar WireMock verbose logging

El logging detallado de WireMock es esencial para diagnosticar mismatches. Muestra exactamente qué petición se recibió y por qué no coincide con ningún stub.

```yaml
# src/test/resources/application.yml del CONSUMIDOR
logging:
  level:
    # Logging de Stub Runner y resolución de stubs
    org.springframework.cloud.contract: DEBUG
    # Logging de WireMock: muestra requests recibidos y stubs evaluados
    com.github.tomakehurst.wiremock: TRACE
    # Logging de WireMock server
    wiremock: DEBUG

# Configuración adicional de WireMock para verbose
spring:
  cloud:
    contract:
      stubrunner:
        # Activa el modo verbose en el servidor WireMock embebido
        # (disponible en versiones recientes de SCC)
        generate-stubs: true
```

```java
// Alternativa: configurar WireMock verbose via @AutoConfigureWireMock
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;

@SpringBootTest
@AutoConfigureWireMock(port = 8090, stubs = "classpath:stubs")
public class VerboseWireMockTest {
    // WireMock en modo verbose muestra en log todos los requests y su evaluación
}
```

## Error 3: Contrato obsoleto

Un contrato obsoleto ocurre cuando el productor ha cambiado su implementación pero el contrato no se ha actualizado, o cuando el consumidor usa una versión de stubs que no refleja la API actual.

```
Síntoma en el build del productor:
─────────────────────────────────────────────────────────────────
[ERROR] Tests run: 3, Failures: 1, Errors: 0, Skipped: 0
[ERROR] ContractVerifierTest.validate_shouldReturnOrder -- Time elapsed: 0.234 s
  Expected: "CONFIRMED"
   Actual:  "confirmed"   ← productor cambió capitalización sin actualizar contrato
─────────────────────────────────────────────────────────────────
```

```
Lista de verificación para contrato obsoleto:

  □ ¿El productor ha cambiado el response recientemente sin actualizar el contrato?
  □ ¿El consumidor usa una versión de stubs más antigua que la implementación actual?
  □ ¿El contrato usa valores fijos donde debería usar matchers (byRegex, byType)?
  □ ¿Los tests del productor pasan localmente pero no en CI? (diferente versión del stub)
  □ ¿La clase base del productor mockea el servicio con valores diferentes a los del contrato?
```

## Tabla de errores comunes y soluciones

La siguiente tabla resume los errores más frecuentes, su causa y la acción correctiva.

| Error | Causa más frecuente | Verificación | Solución |
|---|---|---|---|
| `No stubs found` | `ids` mal configurado o JAR no publicado | `ls ~/.m2/.../stubs.jar` | Ejecutar `mvn install` en el productor o corregir `ids` |
| `404 Not Found` del stub | URL del cliente no coincide con el contrato | WireMock verbose log | Corregir URL en el cliente o en el contrato |
| `AssertionError` en test generado | Productor devuelve valor diferente al contrato | Log del test generado | Actualizar contrato o corregir implementación |
| `NullPointerException` en clase base | `standaloneSetup()` no configurado | Stack trace del test | Añadir `RestAssuredMockMvc.standaloneSetup()` en `@BeforeEach` |
| Tests generados no compilan | Clase base no encontrada | Error de compilación | Verificar `baseClassForTests` o `baseClassMappings` en el plugin |
| Stub devuelve respuesta incorrecta | Stubs desactualizados en el classpath | Contenido del JAR stubs | Re-ejecutar `mvn install` del productor y limpiar caché |

## Error 4: Tests generados no compilan

Cuando el plugin no puede encontrar la clase base, los tests generados no compilarán. Este error es fácil de diagnosticar pero frecuente en proyectos nuevos.

```
Error de compilación típico:
─────────────────────────────────────────────────────────────────
[ERROR] .../target/generated-test-sources/contracts/ContractVerifierTest.java:5:
  error: cannot find symbol
  public class ContractVerifierTest extends BaseContractTest {
                                            ^
  symbol: class BaseContractTest
─────────────────────────────────────────────────────────────────
```

```
Solución: verificar que baseClassForTests apunta a la clase correcta
y que esa clase está en el src/test/java del productor.

En pom.xml:
<baseClassForTests>com.example.BaseContractTest</baseClassForTests>

Verificar que existe:
src/test/java/com/example/BaseContractTest.java
```

## Checklist de diagnóstico rápido

Ante cualquier fallo en Spring Cloud Contract, seguir este checklist en orden ahorra tiempo de diagnóstico.

```
CHECKLIST DE DIAGNÓSTICO SPRING CLOUD CONTRACT
═══════════════════════════════════════════════

LADO PRODUCTOR:
  □ 1. ¿Existe src/test/resources/contracts/ con al menos un contrato?
  □ 2. ¿El plugin está configurado en pom.xml con <extensions>true</extensions>?
  □ 3. ¿baseClassForTests o baseClassMappings apuntan a una clase existente?
  □ 4. ¿La clase base tiene @BeforeEach con RestAssuredMockMvc.standaloneSetup()?
  □ 5. ¿mvn verify pasa localmente (tests generados + tests normales)?
  □ 6. ¿mvn install o mvn deploy genera el JAR -stubs en target/stubs/?

LADO CONSUMIDOR:
  □ 7. ¿@AutoConfigureStubRunner está en la clase de test?
  □ 8. ¿El campo ids tiene el formato correcto groupId:artifactId:version:stubs:port?
  □ 9. ¿El stubsMode coincide con dónde están los stubs (LOCAL/CLASSPATH/REMOTE)?
  □ 10. ¿Si LOCAL, el productor ha ejecutado mvn install?
  □ 11. ¿Si REMOTE, repositoryRoot apunta al repositorio correcto?
  □ 12. ¿El puerto del stub coincide con la URL configurada en el cliente HTTP?
  □ 13. ¿WireMock verbose logging está habilitado para ver el mismatch?
```

## Buenas y malas prácticas

**Buenas prácticas**:
- Habilitar `logging.level.com.github.tomakehurst.wiremock=TRACE` en tests para ver exactamente qué stub se está evaluando.
- Usar matchers (`byRegex`, `byType`) en lugar de valores fijos para campos dinámicos — evita contratos obsoletos por cambios de valor.
- Ejecutar `mvn verify` antes de crear un PR para detectar contratos rotos en el productor antes de que afecten al consumidor.
- Verificar el contenido del JAR de stubs con `jar tf` cuando Stub Runner no encuentra stubs.

**Malas prácticas**:
- Ignorar los tests del productor que fallan y avanzar con `-DskipTests` — los stubs publicados no reflejarán el contrato real.
- Modificar los tests generados para que pasen — se regeneran en el siguiente build y el fix se pierde.
- Confundir el error 404 de WireMock (stub no coincide) con un 404 real del productor — son contextos distintos.
- No limpiar el repositorio Maven local (`mvn dependency:purge-local-repository`) cuando los stubs están en un estado inconsistente.

## Verificación y práctica

> [EXAMEN] 1. ¿Qué se debe verificar primero cuando Stub Runner lanza `No stubs found` al ejecutar los tests del consumidor?

> [EXAMEN] 2. ¿Cómo se habilita el logging verbose de WireMock para diagnosticar un mismatch en el stub?

> [EXAMEN] 3. ¿Qué indica un error de compilación `cannot find symbol` en los tests generados por Spring Cloud Contract?

> [EXAMEN] 4. ¿En qué fase del ciclo Maven se puede producir `No stubs found` y en cuál se produce un `AssertionError` por contrato obsoleto?

> [EXAMEN] 5. ¿Qué herramienta de línea de comandos permite verificar el contenido del JAR de stubs generado por el plugin?

---

← [10.11 Workflow CI/CD](sc-contract-workflow.md) | [Índice](README.md) | [11.1 Spring Cloud Task — Fundamentos](sc-task-fundamentos.md) →
