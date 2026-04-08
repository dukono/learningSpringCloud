PROMPT 1 — GENERADOR DE ÍNDICE DESGLOSADO

Actúa como arquitecto de documentación técnica. Tu única tarea es generar el índice
jerárquico completo para un curso o guía de estudio sobre el tema indicado.

---

ENTRADA

TEMA = [escribe aquí el tema, ej: "Java 21", "Kubernetes CKA", "AZ-900: Azure Fundamentals"]

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


2. Regla de granularidad — la más importante

Cada nodo hoja (el que no tiene hijos) debe poder desarrollarse en un único fichero .md
de entre 300 y 700 líneas. Esta regla determina cuándo dividir:

- Un concepto con más de 2 variantes de configuración → subtemas separados
- Un flujo con más de 1 ciclo de vida distinto → subtemas separados
- Una comparativa entre 2+ alternativas → subtema propio
- Un bloque que cubriría más de 700 líneas → se divide en hijos más pequeños

El objetivo es que ningún fichero generado posteriormente provoque timeout por exceso de contenido.


3. Convenciones de naming

- Numeración: 1, 1.1, 1.2, 1.2.1, 1.2.2, etc. — la profundidad refleja la jerarquía real
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
  Fichero principal: [nombre-fichero].md

  X.1     [Nombre subtema]                    → [fichero].md
  X.1.1   [Nombre sub-subtema]                → [fichero].md   (solo si la rama lo necesita)
  X.1.2   [Nombre sub-subtema]                → [fichero].md
  X.2     [Nombre subtema]                    → [fichero].md
  ...
  X.N     Testing / Verificación de [Nombre]  → [fichero]-testing.md
  X.N+1   Extensión Avanzada: [Nombre]        → [fichero]-extension.md  (si aplica)
    X.N+1.1   [técnica avanzada]
    ...
    X.N+1.N   Antipatrones


Al final de todos los temas, genera el bloque listo para pegar en README.md con
el índice jerárquico completo como lista de enlaces markdown.

Y una tabla resumen:

  Nro | Tema | Ficheros hoja | Líneas estimadas/fichero
  ----|------|---------------|-------------------------
   1  | ...  |       N       |         ~XXX
   2  | ...  |       N       |         ~XXX
  Total       N ficheros

---

REGLAS ADICIONALES

- No generes contenido de los subtemas, solo nombres, numeración y ficheros
- Si el tema es una certificación (ej: CKA, AZ-900), alinea los temas con el temario oficial
- Si el tema es un lenguaje o framework, alinea con el flujo natural de aprendizaje
- Indica explícitamente cuando una rama tiene más de 2 niveles de profundidad y por qué

---

Cómo usarlo: escribe el TEMA y ejecuta. El resultado es la hoja de ruta completa de ficheros
lista para alimentar al PROMPT 2 uno a uno, rama a rama.
