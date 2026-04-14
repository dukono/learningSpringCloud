PROMPT 1 — GENERADOR DE ÍNDICE DESGLOSADO

Actúa como arquitecto de documentación técnica. Tu única tarea es generar el índice
jerárquico completo para un curso o guía de estudio sobre el tema indicado.

---

ENTRADA

TEMA = "[INSERTA EL TEMÁtica]"

---

INSTRUCCIONES

1. Determina la estructura automáticamente

A partir del TEMA, tú decides:
- Cuántos temas principales hay (numerados 1, 2, 3...)
- Cuántos subtemas tiene cada tema principal
- La profundidad del árbol en cada rama

La profundidad es variable y determinada por el contenido, no por una plantilla fija.
Una rama puede tener 2 niveles (tema > subtema), otra puede tener 3 (tema > bloque > subtema).
No apliques la misma profundidad a todas las ramas: cada rama tiene la profundidad que necesita.


2. Regla de granularidad y fichero único — la más importante

Cada nodo hoja recibe exactamente un fichero .md propio y exclusivo.
Ningún fichero es compartido por dos nodos. Esta es una regla sin excepciones.

El tamaño objetivo de cada fichero es 300–700 líneas desarrollando el contenido completo
con la estructura de prompt2 (introducción, diagrama, ejemplo funcional, tabla de parámetros,
buenas/malas prácticas, comparación). Esta estimación determina cuándo dividir y cuándo fusionar:

Cuándo dividir un nodo en hijos:
- Un concepto con más de 2 variantes de configuración → subtemas separados
- Un flujo con más de 1 ciclo de vida distinto → subtemas separados
- Una comparativa entre 2+ alternativas → subtema propio
- Un bloque que superaría 700 líneas con prompt2 completo → se divide en hijos más pequeños

Cuándo fusionar nodos hermanos en uno solo:
- Dos sub-temas que individualmente no superarían 150 líneas con prompt2 → un único nodo hoja
- Un sub-tema tan acotado que prompt2 generaría secciones artificiales o vacías → fusionar con el hermano más próximo

Test de viabilidad antes de asignar un nodo como hoja:
¿Puede este tema sostener de forma natural una introducción, un diagrama real, un ejemplo
funcional completo, una tabla de parámetros y una sección de buenas prácticas?
- Sí → nodo hoja con fichero propio
- No → demasiado pequeño, fusionar con hermano o subir al nodo padre

El objetivo es máxima granularidad dentro de lo necesario: cada fichero cubre exactamente
un tema cohesivo, ni más ni menos, sin comprimir contenido para no pasarse del límite.


3. Convenciones de naming

- Numeración: 1, 1.1, 1.2, 1.2.1, 1.2.2, etc. — la profundidad refleja la jerarquía real
- Nodos intermedios (aquellos que tienen hijos): NO se les asigna fichero → aparecen en el
  índice solo como agrupadores, sin enlace ni nombre de fichero
- Nodos hoja (aquellos sin hijos): SÍ se les asigna fichero → son los únicos que generan .md
- El último subtema de cada tema principal es siempre: X.N Testing / Verificación de [nombre]
- Si un tema justifica extensión avanzada, añádela como X.N+1 Extensión Avanzada: [nombre]
  con sus propios hijos, terminando siempre en X.N+1.N Antipatrones
- Nombres en términos del dominio real, nunca genéricos ("Configuración de Eureka", no "Configuración")
- Ficheros: kebab-case, sin espacios (eureka-server-config.md)


4. Orden didáctico dentro de cada tema

Concepto → Instalación/Setup → Configuración → Uso → Variantes/Casos edge → Testing

---

FORMATO DE SALIDA

Para cada tema principal genera este bloque:

Tema X: [Nombre]

X.1     [Nombre subtema]                    (agrupa X.1.1 y X.1.2 — sin fichero)
X.1.1   [Nombre sub-subtema]                → [fichero-a].md        ← fichero único exclusivo
X.1.2   [Nombre sub-subtema]                → [fichero-b].md        ← fichero único exclusivo
X.2     [Nombre subtema]                    → [fichero-c].md        (es hoja: no tiene hijos)
...
X.N     Testing / Verificación de [Nombre]  → [fichero]-testing.md
X.N+1   Extensión Avanzada: [Nombre]        (agrupa X.N+1.1 … X.N+1.N — sin fichero)
X.N+1.1   [técnica avanzada]              → [fichero-ext-1].md     ← fichero único exclusivo
...
X.N+1.N   Antipatrones                    → [fichero]-antipatrones.md

NUNCA dos nodos distintos comparten el mismo nombre de fichero.


Al final de todos los temas, genera el bloque listo para pegar en README.md con
el índice jerárquico completo como lista de enlaces markdown.

IMPORTANTE: en el README los nodos intermedios se renderizan como texto plano sin enlace;
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

Nota: los nodos intermedios no cuentan como ficheros hoja. Solo se contabilizan
los nodos sin hijos. El total de ficheros = total de nodos hoja únicamente.

---

REGLAS ADICIONALES

- No generes contenido de los subtemas, solo nombres, numeración y ficheros
- Cada nombre de fichero aparece exactamente una vez en todo el índice — si al generar ves que
  dos nodos compartirían fichero, es señal de que la jerarquía está mal definida: revísala
- Si el tema es una certificación (ej: CKA, AZ-900), alinea los temas con el temario oficial
- Si el tema es un lenguaje o framework, alinea con el flujo natural de aprendizaje
- Indica explícitamente cuando una rama tiene más de 2 niveles de profundidad y por qué

---

Cómo usarlo: escribe el TEMA y ejecuta. El resultado es la hoja de ruta completa de ficheros
lista para alimentar al PROMPT 2 uno a uno, rama a rama.

