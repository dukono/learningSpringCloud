PROMPT 2 — GENERADOR DE SUBTEMA INDIVIDUAL

Actúa como redactor técnico experto. Vas a desarrollar UN único subtema de documentación
didáctica y estructurada. El prompt contiene todas las reglas y patrones a seguir.
Tu única tarea es recibir el subtema asignado y generar el fichero correspondiente.

---

ENTRADA

SUBTEMA = [pega aquí la línea exacta generada por el PROMPT 1, ej:
           "3.2 Configuración de Eureka Server → eureka-server-config.md"]

El SUBTEMA contiene: identificador numérico, nombre, y nombre del fichero a crear.
El identificador determina el tema padre (ej: 3.2 pertenece al tema 3).
Usa ese contexto para inferir el dominio, el nivel de profundidad y los vecinos de navegación.

---

ESTRUCTURA DEL FICHERO A GENERAR

Sigue este orden exacto de secciones:


SECCION 1 — Cabecera y navegación (obligatoria)

  # [ID] [Nombre del subtema]

  <- [subtema anterior] | Índice (README.md) | [subtema siguiente] ->

  ---


SECCION 2 — Introducción

Un único párrafo que responda:
- ¿Qué problema concreto resuelve este subtema?
- ¿Por qué existe? ¿Qué pasaría sin él?
- ¿Cuándo se necesita específicamente?

Escrito en términos del dominio real, sin abstracciones vacías.
Si el subtema depende de algo externo que debe estar listo antes, incluye al final:

  > [PREREQUISITO] descripción concreta de lo que debe estar configurado o instalado antes.


SECCION 3 — Representación visual

Incluye uno según el tipo de contenido del subtema:
- Diagrama de flujo en ASCII o Mermaid para procesos y secuencias
- Tabla comparativa para configuraciones o alternativas
- Árbol de componentes para estructuras jerárquicas

No uses listas como sustituto. El diagrama debe representar relaciones o flujos reales.
Si el subtema no tiene flujo ni estructura que visualizar, omite esta sección.


SECCION 4 — Ejemplo completo y funcional

El ejemplo debe ser directamente ejecutable o aplicable, sin placeholders ni "...".
Incluye: imports necesarios, configuración mínima, código o pasos completos.
Añade comentarios en el código solo donde el comportamiento no sea evidente.
Si hay varias formas de hacer lo mismo, muestra primero la recomendada.
Las variantes van en la sección de comparación.


SECCION 5 — Parámetros y propiedades (solo si el subtema introduce configuración)

Tabla obligatoria cuando hay propiedades configurables:

  Parámetro | Tipo | Valor por defecto | Descripción


SECCION 6 — Buenas y malas prácticas

  Hacer:
  - [práctica concreta + razón específica en este dominio]
  - [práctica concreta + razón específica en este dominio]

  Evitar:
  - [antipatrón concreto + consecuencia real]
  - [antipatrón concreto + consecuencia real]


SECCION 7 — Comparación o variante (solo si aplica)

Diferencias entre opciones del subtema, versiones legacy vs modernas, o casos edge.
Usa tabla cuando hay 2+ alternativas a comparar.
Si alguna variante está obsoleta, añade [LEGACY] en el texto al mencionarla.


SECCION 8 — Pie y navegación (obligatoria)

  ---

  <- [subtema anterior] | Índice (README.md) | [subtema siguiente] ->

---

MARCADORES DE IMPORTANCIA

Aplica donde corresponda, siempre dentro de blockquote (>):

  > [CONCEPTO] cuando el párrafo define el concepto central del subtema
  > [PREREQUISITO] cuando el subtema requiere algo externo previo
  > [EXAMEN] cuando es fuente frecuente de confusión o pregunta de evaluación
  > [ADVERTENCIA] cuando hay un error frecuente o comportamiento inesperado

El marcador [LEGACY] es el único que va fuera del blockquote, integrado en el texto.

---

INSTRUCCIONES ESPECIALES SEGÚN TIPO DE SUBTEMA

Si el ID termina en .N y el nombre contiene "Testing" o "Verificación":
- Mínimo 3 estrategias de verificación, cada una como:
    Estrategia N: [nombre]
    Descripción + ejemplo funcional completo
- Cierra con tabla resumen:
    Situación | Estrategia recomendada

Si el nombre contiene "Extensión Avanzada":
- Primera sección sin número: "Límites del enfoque estándar"
  - 1 párrafo sobre qué casos no cubre el enfoque básico
  - Tabla: Técnica/Concepto | Para qué sirve
- Cada bloque avanzado: párrafo → ejemplo completo → nota técnica en blockquote
- Penúltima sección "Antipatrones": exactamente 3 marcadores [ADVERTENCIA], sin excepción
- Última sección: tabla "Enfoque estándar vs Enfoque avanzado"
  (columnas: Caso de uso | Estándar suficiente | Requiere extensión)
- En este tipo de fichero, el único marcador permitido es [ADVERTENCIA]

---

REGLAS DE CALIDAD

R1. Ninguna sección empieza con código o tabla. Siempre hay un párrafo antes.
R2. Todo ejemplo es completo y funcional. Sin fragmentos incompletos ni "...".
R3. Los diagramas representan relaciones o flujos reales, no listas con formato visual.
R4. Los marcadores van siempre en blockquote (>), excepto [LEGACY] que va en el texto.
R5. La navegación aparece exactamente dos veces: tras el título y al final del fichero.
R6. No hay índice interno. El README.md es la única tabla de contenidos del proyecto.
R7. El fichero tiene entre 300 y 700 líneas.
    Si el desarrollo natural superaría 700 líneas, avisa antes de escribir
    y propón cómo dividirlo en dos subtemas separados.

---

Cómo usarlo: pega la línea del subtema en ENTRADA y ejecuta. Genera exactamente un
fichero .md coherente con el patrón del proyecto, sin necesidad de rellenar nada más.
