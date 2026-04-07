# Parte 14 — Spring Cloud Contract (Consumer-Driven Contract Testing)

← [Parte 12 — Proyecto Práctico](./12-proyecto-practico.md) | [Volver al índice](./README.md)

---

## 14.1 El problema: ¿cómo saber que los servicios son compatibles?

En microservicios, un servicio `pedidos-service` llama a `productos-service`. El equipo de productos cambia su API sin avisar. Los tests unitarios de cada servicio pasan, pero en integración todo se rompe.

```
pedidos-service          productos-service
     │                         │
     │  GET /productos/{id}     │
     │ ─────────────────────→  │
     │                         │  ← equipo cambia respuesta
     │  ← { "productId": 1 }   │     de "id" a "productId"
     │                         │
     ↓                         ↓
  Tests pasan             Tests pasan
                  ↓
          Integración → ROMPE
```

**Consumer-Driven Contract Testing** resuelve esto: el consumidor define qué espera de la API del productor, y el productor verifica que su implementación cumple esas expectativas.

---

## 14.2 Conceptos clave

| Término | Definición |
|---------|-----------|
| **Contrato** | Acuerdo formal entre consumidor y productor (request + response esperados) |
| **Productor (Provider)** | Servicio que expone la API |
| **Consumidor (Consumer)** | Servicio que llama la API |
| **Stub** | Servidor HTTP falso generado desde el contrato, usado por el consumidor en tests |
| **Verifier** | Plugin que genera tests en el productor para verificar que cumple los contratos |

---

## 14.3 Arquitectura de Spring Cloud Contract

```
CONSUMIDOR                          PRODUCTOR
    │                                   │
    │  1. Define contrato               │
    │     (Groovy/YAML DSL)             │
    │                                   │
    │  ──── contrato ────────────────→  │
    │                                   │  2. Genera tests automáticos
    │                                   │     desde el contrato
    │                                   │  3. Ejecuta tests contra
    │                                   │     implementación real
    │                                   │  4. Publica stubs en repositorio
    │                                   │
    │  ←──── stubs (JAR) ────────────   │
    │                                   │
    │  5. Descarga stubs                │
    │  6. Levanta WireMock con stubs    │
    │  7. Ejecuta sus propios tests     │
    │     contra WireMock               │
```

---

## 14.4 Dependencias Maven

### En el Productor

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>

<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>${spring-cloud-contract.version}</version>
    <extensions>true</extensions>
    <configuration>
        <!-- Clase base para los tests generados -->
        <baseClassForTests>
            com.miempresa.productos.ContractBaseTest
        </baseClassForTests>
    </configuration>
</plugin>
```

### En el Consumidor

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 14.5 Escribir contratos (Groovy DSL)

Los contratos se colocan en `src/test/resources/contracts/` del productor.

```groovy
// src/test/resources/contracts/obtener_producto_existente.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Debe devolver un producto cuando existe"

    request {
        method GET()
        url "/productos/1"
        headers {
            contentType(applicationJson())
        }
    }

    response {
        status OK()   // 200
        body([
            id      : 1,
            nombre  : "Laptop",
            precio  : 999.99,
            stock   : $(anyPositiveInt())   // valor dinámico
        ])
        headers {
            contentType(applicationJson())
        }
    }
}
```

```groovy
// src/test/resources/contracts/producto_no_encontrado.groovy
Contract.make {
    description "Debe devolver 404 cuando el producto no existe"

    request {
        method GET()
        url "/productos/999"
    }

    response {
        status NOT_FOUND()   // 404
        body([
            error: "Producto no encontrado"
        ])
    }
}
```

---

## 14.6 Contratos en YAML (alternativa a Groovy)

```yaml
# src/test/resources/contracts/crear_producto.yml
description: "Debe crear un producto y devolver 201"

request:
  method: POST
  url: /productos
  headers:
    Content-Type: application/json
  body:
    nombre: "Monitor"
    precio: 299.99

response:
  status: 201
  headers:
    Content-Type: application/json
  body:
    id: 1
    nombre: "Monitor"
    precio: 299.99
```

---

## 14.7 Lado del Productor: clase base y tests generados

### Clase base (obligatoria)

```java
// El plugin genera tests que extienden esta clase
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
public abstract class ContractBaseTest {

    @Autowired
    private MockMvc mockMvc;

    @BeforeEach
    void setup() {
        // RestAssuredMockMvc se usa internamente por los tests generados
        RestAssuredMockMvc.mockMvc(mockMvc);
    }
}
```

### Tests generados automáticamente (no editar)

```java
// target/generated-test-sources/.../ContractVerifierTest.java
// Generado por el plugin — NO modificar manualmente
public class ContractVerifierTest extends ContractBaseTest {

    @Test
    public void validate_obtener_producto_existente() throws Exception {
        // given:
        MockMvcRequestSpecification request = given()
            .header("Content-Type", "application/json");

        // when:
        ResponseOptions response = given().spec(request)
            .get("/productos/1");

        // then:
        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.header("Content-Type"))
            .matches("application/json.*");

        DocumentContext parsedJson = JsonPath.parse(response.getBody().asString());
        assertThatJson(parsedJson).field("['id']").isEqualTo(1);
        assertThatJson(parsedJson).field("['nombre']").isEqualTo("Laptop");
    }
}
```

> `[ADVERTENCIA]` Nunca edites los tests generados. Si el test falla, modifica el contrato o la implementación.

---

## 14.8 Lado del Consumidor: Stub Runner

```java
@SpringBootTest
@AutoConfigureStubRunner(
    // Descarga stubs del repositorio Maven local (~/.m2)
    stubsMode = StubRunnerProperties.StubsMode.LOCAL,
    // groupId:artifactId:version:puerto
    ids = "com.miempresa:productos-service:+:8081"
)
class PedidosServiceTest {

    @Autowired
    private PedidosService pedidosService;

    @Test
    void debeObtenerProductoCuandoExiste() {
        // WireMock ya está levantado en puerto 8081 con los stubs
        // La llamada Feign de pedidosService irá contra WireMock
        Producto producto = pedidosService.obtenerProducto(1L);

        assertThat(producto).isNotNull();
        assertThat(producto.getNombre()).isEqualTo("Laptop");
    }

    @Test
    void debeManejarProductoNoEncontrado() {
        assertThatThrownBy(() -> pedidosService.obtenerProducto(999L))
            .isInstanceOf(ProductoNoEncontradoException.class);
    }
}
```

---

## 14.9 Stub Runner con repositorio remoto

```java
@AutoConfigureStubRunner(
    // Descarga stubs desde Artifactory/Nexus
    stubsMode = StubRunnerProperties.StubsMode.REMOTE,
    repositoryRoot = "https://artifactory.miempresa.com/libs-snapshot",
    ids = {
        "com.miempresa:productos-service:1.0.0:8081",
        "com.miempresa:inventario-service:1.0.0:8082"
    }
)
```

```yaml
# application-test.yml — alternativa a la anotación
stubrunner:
  stubs-mode: LOCAL
  ids:
    - com.miempresa:productos-service:+:8081
  # + significa "última versión disponible"
```

---

## 14.10 Flujo completo de trabajo en equipo

```
1. Equipo consumidor escribe el contrato
   └── src/test/resources/contracts/obtener_producto.groovy

2. PR al repositorio del PRODUCTOR con el contrato

3. CI del productor:
   └── mvn verify
       ├── Plugin genera tests desde contratos
       ├── Ejecuta tests generados contra implementación
       ├── Si pasan → genera JAR de stubs
       └── mvn install → publica stubs en Maven local/Nexus

4. CI del consumidor:
   └── mvn test
       ├── @AutoConfigureStubRunner descarga stubs
       ├── Levanta WireMock con los stubs
       └── Ejecuta tests del consumidor contra WireMock
```

> `[EXAMEN]` El contrato lo escribe el **consumidor**, pero se verifica en el **productor**. Esta es la diferencia clave con los tests de integración clásicos donde el productor define su API unilateralmente.

---

## 14.11 Valores dinámicos en contratos

```groovy
Contract.make {
    request {
        method POST()
        url "/pedidos"
        body([
            productoId : $(consumer(regex('[0-9]+')), producer(1)),
            cantidad   : $(consumer(anyPositiveInt()), producer(3))
        ])
    }

    response {
        status CREATED()
        body([
            pedidoId  : $(producer(regex('[0-9a-f-]+')), consumer("abc-123")),
            estado    : "PENDIENTE",
            fechaHora : $(producer(execute('now()')), consumer("2024-01-01T10:00:00"))
        ])
    }
}
```

| Matcher | Descripción |
|---------|-------------|
| `regex(...)` | Valor que cumple la expresión regular |
| `anyPositiveInt()` | Cualquier entero positivo |
| `anyNonEmptyString()` | Cualquier String no vacío |
| `anyUuid()` | Cualquier UUID válido |
| `anyDate()` | Cualquier fecha en formato ISO |
| `consumer(...)` | Valor usado en el stub (lado consumidor) |
| `producer(...)` | Valor usado en el test generado (lado productor) |

---

## 14.12 Errores comunes

| Error | Causa | Solución |
|-------|-------|----------|
| `No contracts found` | Contratos en directorio incorrecto | Deben estar en `src/test/resources/contracts/` |
| `Base class not found` | `baseClassForTests` mal configurado | Verificar paquete y nombre de la clase base |
| `WireMock no responde` | Puerto en conflicto | Cambiar el puerto en `ids` del Stub Runner |
| `Stub no coincide` | Request del test no coincide con el contrato | Revisar headers, URL y body exactos |
| Tests generados fallan | Implementación no cumple el contrato | Actualizar implementación o el contrato |

---

← [Parte 12 — Proyecto Práctico](./12-proyecto-practico.md) | [Volver al índice](./README.md)
