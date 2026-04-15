PROMPT 1 — GENERADOR DE ÍNDICE DESGLOSADO

Actúa como arquitecto de documentación técnica. Tu única tarea es generar el índice
jerárquico completo para una guía de estudio sobre el tema indicado.

El prompt es domain-agnostic: funciona para cualquier tecnología (lenguaje, framework,
plataforma cloud, certificación, herramienta DevOps, orquestador, middleware, etc.).
No asumir ninguna estructura previa: se infiere y justifica a partir del TEMA.

---

ENTRADA

TEMA = "[INSERTA EL TEMA]"

---

FASE 1 — ANÁLISIS DEL DOMINIO

Antes de generar ningún tema ni fichero, razona explícitamente sobre el TEMA.
Escribe esta sección con el encabezado "## Análisis del dominio" al inicio de la salida.
Sus respuestas determinan todas las decisiones de Fase 2 y Fase 3.


1. Clasificación del tipo de tecnología

Identifica a qué categoría pertenece el TEMA:

  Tipo                         Ejemplos
  ───────────────────────────  ───────────────────────────────────────────────
  Lenguaje de programación     Python, Java, Go, Kotlin, Rust
  Framework / Librería         Spring AI, FastAPI, React, Spring Boot, Quarkus
  Plataforma cloud             Azure, AWS, GCP, Oracle Cloud
  Certificación técnica        CKA, CKAD, AZ-900, AZ-104, AWS-SAA, RHCSA
  Herramienta / CLI            Docker, kubectl, Terraform, Ansible, Helm, Git
  Orquestador / Middleware     Kubernetes, Kafka, Elasticsearch, Redis, RabbitMQ

Si el TEMA es híbrido (ej: "Kubernetes CKA" = certificación + orquestador), declara
el tipo primario y el secundario y describe qué implica la combinación para el índice:
el tipo primario domina la estructura de primer nivel; el secundario informa la
profundidad y los sub-temas internos.


2. Fuente autoritativa

Nombra la fuente exacta que usarás para definir los temas y su cobertura:

  - Certificación  → temario oficial del proveedor (dominios + porcentajes de peso)
  - Lenguaje       → documentación oficial + track de aprendizaje canónico
                     (ej: "docs.python.org + PSF tutorial path")
  - Framework      → reference guide oficial del proyecto (ej: "docs.spring.io/spring-ai")
  - Plataforma     → documentación oficial + Well-Architected Framework del proveedor
  - Herramienta    → docs oficiales + top casos de uso de la industria
  - Orquestador    → docs oficiales + guías de operación estándar del ecosistema


3. Alcance y audiencia

Define con una frase el perfil objetivo y los límites del índice:
  - ¿A quién va dirigido? (principiante / profesional en transición / preparación de examen)
  - ¿Qué queda DENTRO? (los sub-dominios que el índice sí cubre)
  - ¿Qué queda FUERA?   (sub-dominios que se excluyen explícitamente y por qué)

Para dominios muy amplios, el índice DEBE acotarse. Si el TEMA no especifica alcance,
infiere el más habitual:

  Azure       → alcance AZ-104 Administrator Associate (infraestructura y servicios core)
  AWS         → alcance SAA-C03 Solutions Architect Associate
  GCP         → alcance ACE Associate Cloud Engineer
  Kubernetes  → operador + desarrollador de aplicaciones, nivel CKA/CKAD
  Java        → profesional con Spring Boot (excluye Jakarta EE / legacy EE)
  Python      → profesional generalista (excluye CPython internals, ciencia de datos)
  Spring AI   → versión 1.x estable; excluye integraciones experimentales

Si el índice superaría 150 nodos hoja, el alcance está mal definido: reduce el scope
antes de continuar.


4. Conocimientos previos implícitos

Lista lo que el lector ya debe saber antes de empezar. El índice no generará ficheros
para estos temas; prompt2 los mencionará como [PREREQUISITO] en los primeros subtemas.


5. Versión de referencia (solo para lenguajes y frameworks)

Indica la versión estable cubierta (ej: "Python 3.12", "Spring Boot 3.3", "Java 21 LTS").
Los ejemplos de código y las APIs de los ficheros generados por prompt2 deben ser
coherentes con esta versión.


6. Prefijo de ficheros

Deriva el prefijo para los nombres de fichero a partir del TEMA:
  - Usa las 2-3 iniciales o sílabas más reconocibles del nombre
  - El prefijo va seguido de guion y el nombre del subtema en kebab-case
  - Ejemplos:
      "Kubernetes"   → k8s-          → k8s-pod-lifecycle.md
      "Azure"        → az-           → az-iam-rbac.md
      "Python"       → py-           → py-concurrencia-async.md
      "Spring AI"    → spring-ai-    → spring-ai-chat-client.md
      "Terraform"    → tf-           → tf-state-remote.md
      "Apache Kafka" → kafka-        → kafka-consumer-groups.md

---

FASE 2 — ESTRATEGIA DE ORDENACIÓN

Aplica el patrón correspondiente al tipo clasificado en Fase 1.
Si el TEMA es híbrido, el tipo primario domina; el tipo secundario informa la profundidad.

LENGUAJE DE PROGRAMACIÓN
  Sintaxis y tipos básicos → Control de flujo → Funciones, lambdas y closures
  → Programación orientada a objetos / paradigmas propios del lenguaje
  → Módulos, paquetes y gestión de dependencias
  → Concurrencia y paralelismo → I/O: ficheros, red, serialización
  → Ecosistema y herramientas de desarrollo → Testing

FRAMEWORK / LIBRERÍA
  Arquitectura y conceptos core → Configuración y arranque
  → Componentes core (un sub-tema por subsistema funcional)
  → Integración con el host (lenguaje o plataforma base)
  → Configuración avanzada y tuning
  → Testing del framework → Patrones, extensibilidad y antipatrones

PLATAFORMA CLOUD
  Modelo de cuenta y gobernanza → Identidad y control de acceso
  → Cómputo → Red y conectividad → Almacenamiento
  → Bases de datos gestionadas → Observabilidad: logs, métricas, trazas
  → Automatización y DevOps → Seguridad y cumplimiento
  → Arquitecturas de referencia

CERTIFICACIÓN TÉCNICA
  Exactamente los dominios del examen oficial, en el mismo orden del temario oficial,
  con profundidad proporcional al peso de cada dominio (%).
  Un dominio con peso 30% tendrá más nodos hoja que uno con peso 5%.
  No inventar sub-dominios que el temario oficial no menciona.

HERRAMIENTA / CLI
  Concepto y arquitectura interna → Instalación y primeros pasos
  → Comandos y operaciones core → Workflows habituales
  → Integración con el ecosistema → Casos avanzados y configuración
  → Testing y validación

ORQUESTADOR / MIDDLEWARE
  Arquitectura interna → Instalación y modos de deployment
  → Primitivas y objetos core → Configuración del sistema
  → Networking y descubrimiento de servicios → Almacenamiento y persistencia
  → Seguridad → Observabilidad
  → Operaciones y mantenimiento → Extensibilidad y APIs
  → Testing y validación

---

FASE 3 — GENERACIÓN DEL ÍNDICE


1. Regla de profundidad

La profundidad emerge del contenido, no de una plantilla. Evalúa cada rama de forma
independiente usando estas dos reglas:

Cuándo añadir un nivel de agrupación (más profundidad):
- Un nodo tendría más de 7–8 hijos directos Y esos hijos pertenecen a sub-dominios
  claramente distintos → introduce un nivel intermedio que los agrupe por dominio
- Sin ese agrupador, la rama sería difícil de navegar

Cuándo NO añadir profundidad:
- Los hijos son homogéneos en tipo y nivel → mantén el nivel actual, sin agrupadores
- Un nivel intermedio no aportaría contexto real de navegación → no lo añadas

Ramas distintas pueden tener distinta profundidad. No hay mínimo ni máximo:
solo la que el contenido justifique.


2. Regla de granularidad y fichero único — la más importante

Cada nodo hoja recibe exactamente un fichero .md propio y exclusivo.
Ningún fichero es compartido por dos nodos. Esta es una regla sin excepciones.

Para decidir si un tema merece fichero propio, estima cuántas líneas ocuparía
desarrollado con la estructura de prompt2 (introducción, diagrama, ejemplo funcional,
tabla de parámetros, buenas/malas prácticas, comparación).
Rango de referencia: 300–700 líneas.

Cuándo dividir un nodo en hijos:
- Un concepto con más de 2 variantes de configuración → subtemas separados
- Un flujo con más de 1 ciclo de vida distinto → subtemas separados
- Una comparativa entre 2+ alternativas → subtema propio
- Un bloque que superaría 700 líneas → se divide en hijos más pequeños

Cuándo fusionar nodos hermanos en uno solo:
- Dos sub-temas que individualmente no superarían 150 líneas → un único nodo hoja
- Un sub-tema tan acotado que prompt2 generaría secciones artificiales o vacías
  → fusionar con el hermano más próximo

Test de viabilidad antes de asignar un nodo como hoja:
¿Puede este tema sostener de forma natural una introducción, un diagrama real, un ejemplo
funcional completo, una tabla de parámetros y una sección de buenas prácticas?
- Sí → nodo hoja con fichero propio
- No → demasiado pequeño: fusionar con hermano o subir al nodo padre

El objetivo es máxima granularidad dentro de lo necesario: cada fichero cubre exactamente
un tema cohesivo, ni más ni menos.


3. Convenciones de naming

- Numeración: 1, 1.1, 1.2, 1.2.1, 1.2.2, etc. — la profundidad refleja la jerarquía real
- Nodos intermedios (con hijos): NO tienen fichero → aparecen como agrupadores sin enlace
- Nodos hoja (sin hijos): SÍ tienen fichero → son los únicos que generan .md
- El último subtema de cada tema principal es siempre: X.N Testing / Verificación de [Nombre]
- Si un tema justifica extensión avanzada, añádela como X.N+1 Extensión Avanzada: [Nombre]
  con sus propios hijos, terminando siempre en X.N+1.M Antipatrones
- Nombres en términos del dominio real, nunca genéricos
  ("Pods y ciclo de vida", no "Configuración de objetos")
- Ficheros: [prefijo-de-fase1]-nombre-en-kebab-case.md
  (ej: k8s-pod-lifecycle.md, az-rbac-roles.md, py-async-await.md)


4. Orden didáctico dentro de cada tema

Concepto → Instalación/Setup → Configuración → Uso → Variantes/Casos edge → Testing

---

FASE 4 — VERIFICACIÓN DE COBERTURA

Antes de emitir la salida final, responde estas preguntas sobre el índice generado.
Si alguna respuesta revela una laguna, corrige el índice antes de continuar.
Escribe el resultado con el encabezado "## Verificación de cobertura":

1. ¿Están representados TODOS los sub-dominios que un profesional que trabaja con
   este TEMA necesita conocer?

2. ¿Hay algún tema estándar del campo que falta? (Contrasta con la fuente
   autoritativa definida en Fase 1.)

3. ¿El orden tiene sentido didáctico? ¿Cada tema puede comprenderse con los
   conocimientos que los temas anteriores aportan?

4. ¿Los nodos hoja son cohesivos? ¿Cada fichero cubre exactamente un concepto,
   ni dos mezclados ni uno dividido artificialmente?

5. ¿Hay nombres de fichero repetidos? Si los hay, la jerarquía está mal definida:
   corregir antes de continuar.

Si no se detectan lagunas, escribir: "Cobertura verificada: no se detectan lagunas."

---

FORMATO DE SALIDA

Para cada tema principal genera este bloque:

Tema X: [Nombre]

X.1     [Nombre subtema]                    (agrupa X.1.1 y X.1.2 — sin fichero)
X.1.1   [Nombre sub-subtema]                → [prefijo]-fichero-a.md     ← fichero único
X.1.2   [Nombre sub-subtema]                → [prefijo]-fichero-b.md     ← fichero único
X.2     [Nombre subtema]                    → [prefijo]-fichero-c.md     (es hoja: no tiene hijos)
...
X.N     Testing / Verificación de [Nombre]  → [prefijo]-testing.md
X.N+1   Extensión Avanzada: [Nombre]        (agrupa X.N+1.1 … X.N+1.M — sin fichero)
X.N+1.1   [técnica avanzada]              → [prefijo]-ext-a.md
...
X.N+1.M   Antipatrones                    → [prefijo]-antipatrones.md

NUNCA dos nodos distintos comparten el mismo nombre de fichero.


Al final de todos los temas, genera el bloque listo para pegar en README.md:

IMPORTANTE: en el README los nodos intermedios van como texto plano sin enlace;
solo los nodos hoja llevan enlace [Nombre](fichero.md).

Ejemplo correcto:
- 1.4 Herramientas de build: Maven y Gradle
    - [1.4.1 Maven con Java 21](maven-java21.md)
    - [1.4.2 Gradle con Java 21](gradle-java21.md)

Ejemplo incorrecto:
- [1.4 Herramientas de build: Maven y Gradle](build-tools.md)  ← NO, tiene hijos

Y una tabla resumen:

  Nro | Tema | Ficheros hoja | Líneas estimadas/fichero
  ----|------|---------------|-------------------------
   1  | ...  |       N       |         ~XXX
   2  | ...  |       N       |         ~XXX
Total       N ficheros

Nota: solo se contabilizan los nodos hoja. Los nodos intermedios no cuentan.
El total de ficheros = total de nodos hoja únicamente.

---

REGLAS ADICIONALES

- No generes contenido de los subtemas: solo nombres, numeración y ficheros
- Cada nombre de fichero aparece exactamente una vez en todo el índice; si al generar
  ves que dos nodos compartirían fichero, la jerarquía está mal definida: corrígela
- Cuando introduces un nodo intermedio de agrupación, indica entre paréntesis el motivo:
  cuántos hijos agrupa y por qué son un sub-dominio distinto
- Si el TEMA es una certificación, alinea exactamente con los dominios y porcentajes
  oficiales; no añadas ni elimines dominios que el temario no mencione
- Si el TEMA es un lenguaje o framework, genera el índice para la versión definida en
  Fase 1 punto 5; no mezcles APIs de versiones distintas en el mismo índice
- No generes más de 150 nodos hoja en total; si el dominio supera ese límite,
  el alcance está mal definido: reducir el scope en Fase 1 punto 3 antes de continuar

---

Cómo usarlo: escribe el TEMA y ejecuta. El modelo analiza el dominio, define los temas
de forma autónoma y genera la hoja de ruta completa de ficheros lista para alimentar
al PROMPT 2 uno a uno, rama a rama.
