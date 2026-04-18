# 10.11 Spring Cloud Contract — Workflow CI/CD

← [10.10 Contratos GraphQL](sc-contract-graphql.md) | [Índice](README.md) | [10.12 Troubleshooting](sc-contract-troubleshooting.md) →

---

## Introducción

El flujo CI/CD de Spring Cloud Contract establece una dependencia de pipeline entre el productor y el consumidor: el productor debe compilar los contratos, verificarlos, generar y publicar los stubs **antes** de que el consumidor pueda ejecutar sus tests. Este orden no es opcional. Comprender las etapas del pipeline, el versionado semántico de los stubs y las estrategias de compatibilidad hacia atrás es fundamental para implementar CDC correctamente en un entorno de integración continua.

> [PREREQUISITO] Este nodo integra conocimiento de todos los nodos anteriores del módulo, especialmente [10.5 Plugin](sc-contract-plugin-config.md) y [10.7 Stub Runner](sc-contract-stub-runner.md).

## Pipeline completo: visión general

El pipeline CDC tiene dos pipelines interdependientes que deben ejecutarse en el orden correcto. El productor siempre va primero.

```
PIPELINE DEL PRODUCTOR (debe completarse primero)
──────────────────────────────────────────────────────────────────────
[1] Checkout código fuente + contratos
    │
    ▼
[2] Compilar código del productor
    │
    ▼
[3] mvn verify (o gradle build)
    │  → Plugin lee contratos de src/test/resources/contracts/
    │  → Genera tests en target/generated-test-sources/contracts/
    │  → Ejecuta tests generados contra el productor real
    │  → Si algún test falla: BUILD FAIL → NO se publican stubs
    │
    ▼
[4] mvn deploy (si los tests pasan)
    │  → Publica order-service-1.2.3.jar en Nexus
    │  → Publica order-service-1.2.3-stubs.jar en Nexus (clasificador stubs)
    │
    ▼
[5] Notifica a downstream pipelines (si CI/CD lo permite)

──────────────────────────────────────────────────────────────────────
PIPELINE DEL CONSUMIDOR (requiere que el productor haya publicado stubs)
──────────────────────────────────────────────────────────────────────
[1] Checkout código fuente del consumidor
    │
    ▼
[2] Compilar código del consumidor
    │
    ▼
[3] mvn test
    │  → @AutoConfigureStubRunner descarga order-service-1.2.3-stubs.jar de Nexus
    │  → Levanta servidor WireMock local en puerto configurado
    │  → Ejecuta tests del consumidor contra el servidor WireMock local
    │
    ▼
[4] Si todos los tests pasan: BUILD SUCCESS
──────────────────────────────────────────────────────────────────────
```

> [CONCEPTO] El orden es crítico: si el consumidor intenta ejecutar sus tests antes de que el productor haya publicado los stubs, Stub Runner lanzará `No stubs found` y el build del consumidor fallará. El pipeline del productor es un prerequisito del pipeline del consumidor.

## Ejemplo de pipeline GitHub Actions

El siguiente ejemplo muestra la configuración de GitHub Actions para el pipeline del productor con publicación de stubs en Nexus.

```yaml
# .github/workflows/producer-pipeline.yml
name: Producer Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-publish-stubs:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build and verify contracts
        run: mvn verify
        # Este paso genera tests y los ejecuta.
        # Si los tests de contratos fallan, el pipeline para aquí.

      - name: Deploy stubs to Nexus
        # Solo se despliegan stubs si todos los tests pasan
        run: mvn deploy -DskipTests
        env:
          NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}

      - name: Notify consumer pipelines
        # Opcional: trigger del pipeline del consumidor
        if: success()
        run: echo "Stubs published successfully for version ${{ env.VERSION }}"
```

```yaml
# .github/workflows/consumer-pipeline.yml
name: Consumer Pipeline

on:
  # Se ejecuta después del productor, o manualmente
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  test-with-stubs:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Run consumer tests with stubs
        run: mvn test
        env:
          # Stub Runner resuelve automáticamente desde el repositorio configurado
          SPRING_CLOUD_CONTRACT_STUBRUNNER_REPOSITORY_ROOT: https://nexus.example.com/repository/maven-releases
          SPRING_CLOUD_CONTRACT_STUBRUNNER_STUBS_MODE: REMOTE
```

## Versionado semántico de stubs

El versionado del JAR de stubs sigue el mismo esquema que el JAR principal del productor. La decisión sobre qué versión de stubs usar en el consumidor tiene implicaciones directas en la detección de incompatibilidades.

```
Estrategias de versionado en el consumidor:

1. Versión fija (mayor control, menor flexibilidad):
   ids = "com.example:order-service:1.2.3:stubs:8090"
   → El consumidor usa siempre la versión 1.2.3
   → Cambios en el productor no afectan al consumidor hasta que se actualice manualmente

2. Versión '+' (última disponible, mayor detección de incompatibilidades):
   ids = "com.example:order-service:+:stubs:8090"
   → Siempre usa la versión más reciente publicada
   → Detecta incompatibilidades inmediatamente cuando el productor publica una nueva versión
   → Riesgo: puede romper el consumidor si el productor hace un cambio incompatible

3. Versión con rango (balance entre control y actualización):
   ids = "com.example:order-service:1.2.+:stubs:8090"
   → Usa la última PATCH de la versión 1.2.x
   → Acepta bugfixes del productor sin aceptar cambios de minor/major
```

> [CONCEPTO] El **versionado semántico** (MAJOR.MINOR.PATCH) en los stubs refleja la compatibilidad de la API: MAJOR para cambios incompatibles (breaking changes), MINOR para nuevas funcionalidades compatibles, PATCH para correcciones. El consumidor que fija la versión MAJOR garantiza compatibilidad hacia atrás.

## Compatibilidad hacia atrás: estrategia de contratos

Un cambio en el productor que rompe un contrato existente romperá también los tests del consumidor. La estrategia de compatibilidad define cómo gestionar estos cambios.

```
CAMBIO COMPATIBLE (no rompe consumidores):
───────────────────────────────────────────────
Productor añade campo 'description' al response de /orders/{id}:
{
  "id": 1,
  "status": "CONFIRMED",
  "description": "New field"    ← nuevo campo opcional
}
→ Los contratos existentes NO validan 'description'
→ Los stubs existentes NO incluyen 'description'
→ Los consumidores existentes NO se rompen

CAMBIO INCOMPATIBLE (rompe consumidores):
───────────────────────────────────────────────
Productor renombra 'status' a 'orderStatus':
{
  "id": 1,
  "orderStatus": "CONFIRMED"   ← campo renombrado
}
→ Los contratos existentes validan '$.status'
→ Los tests del productor FALLAN (contrato roto)
→ Los stubs devuelven 'status' pero el productor devuelve 'orderStatus'
→ Los consumidores que leen 'status' obtendrán null
```

> [ADVERTENCIA] Cuando el productor necesita hacer un cambio incompatible, el flujo recomendado es: (1) mantener la versión antigua del endpoint/campo temporalmente, (2) negociar con los consumidores para que actualicen sus contratos, (3) eliminar la versión antigua solo cuando todos los consumidores hayan migrado. Esto es el patrón **Tolerant Reader** aplicado a CDC.

## Workflow con repositorio Git de contratos

Una variante del flujo CDC es mantener los contratos en un repositorio Git compartido entre productor y consumidor, en lugar de en el repositorio del productor.

```
Repositorio compartido de contratos (git://contracts-repo):
──────────────────────────────────────────────────────────────
contracts/
  order-service/
    v1/
      shouldReturnOrder.groovy
      shouldCreateOrder.groovy
    v2/
      shouldReturnOrder.groovy  ← versión actualizada

Pipeline del productor:
  → Lee contratos de contracts/order-service/v1/
  → Genera tests y verifica
  → Publica stubs en Nexus

Pipeline del consumidor:
  → @AutoConfigureStubRunner apunta a stubs en Nexus
  → Descarga y ejecuta stubs de v1
```

## Tabla de decisiones del workflow

| Situación | Acción recomendada | Riesgo si no se hace |
|---|---|---|
| El productor cambia un campo del response | Actualizar el contrato afectado y negociar con consumidor | Tests del consumidor fallan silenciosamente si no usan matchers |
| El consumidor necesita un campo nuevo | El consumidor actualiza el contrato y notifica al productor | El productor no sabe que debe añadir el campo |
| Nueva versión MAJOR del productor | Publicar nuevos stubs con versión MAJOR+1 y mantener la anterior | Los consumidores de la versión antigua dejan de funcionar |
| Contrato obsoleto (nadie lo usa) | Eliminar el contrato y el test generado | Tests innecesarios que ralentizan el build |

## Buenas y malas prácticas

**Buenas prácticas**:
- Ejecutar `mvn verify` (no solo `mvn test`) en el pipeline del productor — genera y ejecuta los tests de contratos.
- Usar `mvn deploy` (no `mvn install`) en CI/CD para publicar stubs en el repositorio compartido accesible por todos los consumidores.
- Versionar los stubs con el mismo esquema semántico que el JAR principal para facilitar la gestión de compatibilidad.
- Añadir gates en el pipeline: si los tests de contratos fallan, no desplegar a ningún entorno.

**Malas prácticas**:
- Ejecutar el pipeline del consumidor antes de que el productor haya publicado los stubs — Stub Runner fallará.
- Usar `+` como versión en producción sin monitorizar — un cambio incompatible del productor romperá todos los consumidores.
- Omitir el paso `generateStubs` en el pipeline del productor — los consumidores descargarán stubs desactualizados.
- Mantener contratos sin versionar en ramas de feature — dificulta saber qué contratos están activos en producción.

## Verificación y práctica

> [EXAMEN] 1. ¿En qué orden deben ejecutarse los pipelines del productor y del consumidor para garantizar que los stubs estén disponibles?

> [EXAMEN] 2. ¿Qué ocurre en el pipeline del productor si los tests generados a partir de los contratos fallan?

> [EXAMEN] 3. ¿Qué comando Maven publica el JAR principal y el JAR de stubs en un repositorio remoto compartido?

> [EXAMEN] 4. ¿Cuál es la diferencia entre usar `+` y una versión fija en el campo `ids` de `@AutoConfigureStubRunner` en un pipeline CI/CD?

> [EXAMEN] 5. ¿Qué tipo de cambio en el productor se considera un breaking change desde la perspectiva de CDC?

---

← [10.10 Contratos GraphQL](sc-contract-graphql.md) | [Índice](README.md) | [10.12 Troubleshooting](sc-contract-troubleshooting.md) →
