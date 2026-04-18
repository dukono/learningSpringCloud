# 13.1 Descomposición por capacidades de negocio y subdominios DDD

← [12.10 Spring Cloud Function — Testing y packaging](sc-function-testing.md) | [Índice](README.md) | [13.2 Comunicación síncrona](sc-patrones-comunicacion-sincrona.md) →

---

## Introducción

La descomposición es la decisión más crítica al diseñar una arquitectura de microservicios: determina qué código vive junto, qué puede desplegarse de forma independiente y qué equipos son autónomos. Existen dos estrategias complementarias — descomposición por capacidades de negocio y por subdominios DDD — y la elección entre ellas, o su combinación, define la cohesión y el acoplamiento del sistema resultante.

## Estrategias de descomposición

> [CONCEPTO] **Decompose by Business Capability**: una capacidad de negocio es algo que la organización hace para generar valor, independientemente de cómo lo implementa técnicamente. Ejemplos: "Gestión de pedidos", "Facturación", "Notificaciones al cliente". Cada capacidad mapea a un servicio.

> [CONCEPTO] **Decompose by Subdomain (DDD)**: en Domain-Driven Design, el dominio se divide en subdominios según el conocimiento del negocio. Cada subdominio tiene un **Bounded Context** propio con su propio modelo de dominio, lenguaje ubicuo y reglas de negocio. El límite del Bounded Context es el límite natural del microservicio.

Los tres tipos de subdominios DDD determinan la prioridad de inversión:

| Tipo | Descripción | Estrategia |
|------|-------------|-----------|
| Core Domain | Diferenciador competitivo, lógica propietaria | Desarrollar internamente con máxima calidad |
| Supporting Subdomain | Necesario pero no diferenciador | Desarrollar internamente o subcontratar |
| Generic Subdomain | Commodity (autenticación, email) | Usar solución off-the-shelf o SaaS |

## Criterios de granularidad

Un microservicio bien delimitado cumple el **principio de responsabilidad única** a nivel de servicio: cambia por una única razón de negocio. Los criterios de granularidad evitan dos antipatrones opuestos.

El **nano-service** (demasiado fragmentado) genera acoplamiento temporal entre despliegues, explosión de llamadas inter-servicio (chatty services) y overhead operacional desproporcionado al valor aportado.

El **mega-service** (demasiado grande) acumula múltiples responsabilidades no relacionadas, crece hasta tener un equipo demasiado grande para coordinarse eficientemente y pierde la capacidad de desplegarse de forma independiente en la práctica.

> [EXAMEN] La regla heurística de **"Two-Pizza Team"** de Amazon establece que el equipo que mantiene un microservicio debe poder alimentarse con dos pizzas (~5-8 personas). Esto refleja que la granularidad del servicio debe ser coherente con la capacidad cognitiva y de coordinación de un equipo pequeño.

## Ejemplo central: mapeo Bounded Context a microservicio

El siguiente ejemplo muestra cómo definir los límites de un microservicio de Pedidos usando contextos DDD en Spring Boot, incluyendo la separación de la entidad `Order` del Bounded Context de Inventario.

```java
// Bounded Context: Orders
// Paquete: com.example.orders

package com.example.orders.domain;

// La entidad Order es el agregado raíz del Bounded Context de pedidos.
// NO referencia entidades del Bounded Context de Inventario directamente.
public class Order {

    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> lines; // contiene productId (referencia por ID, no por objeto)
    private Money totalAmount;
    private OrderStatus status;

    public static Order place(CustomerId customerId, List<OrderLine> lines) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.customerId = customerId;
        order.lines = List.copyOf(lines);
        order.totalAmount = lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
        order.status = OrderStatus.PENDING;
        return order;
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Only PENDING orders can be confirmed");
        }
        this.status = OrderStatus.CONFIRMED;
    }

    // getters omitidos por brevedad pero presentes en producción
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
}

// Anti-Corruption Layer: traducción entre el modelo de Inventario y el de Pedidos
// Protege el modelo de dominio propio de cambios en el modelo externo.
package com.example.orders.infrastructure.acl;

import com.example.orders.domain.ProductAvailability;
import com.example.inventory.client.InventoryClient;
import com.example.inventory.client.dto.StockResponseDTO;

@Component
public class InventoryACL {

    private final InventoryClient inventoryClient;

    public InventoryACL(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    public ProductAvailability checkAvailability(String productId, int quantity) {
        StockResponseDTO dto = inventoryClient.getStock(productId);
        // Traducción: el modelo externo usa 'availableUnits', el nuestro usa 'available'
        return new ProductAvailability(
            productId,
            dto.getAvailableUnits() >= quantity
        );
    }
}
```

## Tabla de elementos clave

La siguiente tabla resume los conceptos de DDD aplicados a la descomposición de microservicios para facilitar su memorización en el contexto de examen.

| Concepto DDD | Significado | Impacto en microservicios |
|---|---|---|
| Bounded Context | Límite explícito del modelo de dominio | Define el límite del servicio |
| Ubiquitous Language | Vocabulario compartido negocio-técnico | Evita traducciones innecesarias |
| Aggregate Root | Entidad raíz que garantiza consistencia | Unidad mínima de transacción |
| Domain Event | Hecho ocurrido en el dominio | Mecanismo de integración entre contextos |
| Context Map | Relaciones entre Bounded Contexts | Identifica ACLs y dependencias |

## Buenas y malas prácticas

**Buenas prácticas:**
- Comenzar con un **monolito modular** y extraer servicios cuando los límites están bien entendidos (no diseñar microservicios desde el día cero).
- Usar el **Context Map** para documentar explícitamente las relaciones entre Bounded Contexts (Shared Kernel, Customer-Supplier, Conformist, ACL).
- Alinear los límites de servicio con los límites de equipo (Conway's Law: la arquitectura refleja la estructura de comunicación de la organización).
- Referenciar entidades de otros contextos únicamente por ID, nunca por objeto.

**Malas prácticas:**
- Descomponer por capas técnicas (servicio de persistencia, servicio de negocio) en lugar de por capacidades de negocio.
- Compartir entidades de dominio entre servicios — viola la autonomía del modelo.
- Crear un servicio por entidad CRUD sin considerar la cohesión funcional.
- Ignorar el lenguaje del negocio y usar términos técnicos en los límites del servicio.

## Verificación y práctica

> [EXAMEN] 1. ¿Cuál es la diferencia fundamental entre descomponer por Business Capability y por Subdomain DDD? ¿Cuándo coinciden y cuándo divergen?

> [EXAMEN] 2. Un sistema tiene un servicio `UserService` que gestiona autenticación, perfil de usuario y preferencias de notificación. ¿Es un nano-service, un servicio bien granulado o un mega-service? Justifica aplicando los criterios DDD.

> [EXAMEN] 3. ¿Qué es un Bounded Context y cómo se mapea al límite de un microservicio? ¿Puede un Bounded Context contener más de un microservicio?

> [EXAMEN] 4. ¿Para qué sirve una Anti-Corruption Layer (ACL) y en qué relación del Context Map la introduces obligatoriamente?

> [EXAMEN] 5. Explica la Conway's Law y su implicación práctica al diseñar la descomposición de microservicios en una organización con equipos funcionales (frontend, backend, DBA) versus equipos cross-funcionales.

---

← [12.10 Spring Cloud Function — Testing y packaging](sc-function-testing.md) | [Índice](README.md) | [13.2 Comunicación síncrona](sc-patrones-comunicacion-sincrona.md) →
