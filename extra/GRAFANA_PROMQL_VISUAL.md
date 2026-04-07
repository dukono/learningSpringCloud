# 📊 Grafana + PromQL — Guía Visual con Ilustraciones

Documento complementario a `GRAFANA_PROMQL.md`. Aquí **cada concepto tiene su propio panel ASCII** mostrando cómo se ve el resultado real en Grafana, con datos concretos.

**Query hilo conductor:**
```promql
avg(increase(amiga_java_cache_getMulti_timer_seconds_sum{environment=~"$environment"}[$__interval])) by (cache_name)
```

---

## Índice

1. [Anatomía visual de un panel Grafana](#1-anatomía-visual-de-un-panel-grafana)
2. [Counter crudo vs transformado — antes y después](#2-counter-crudo-vs-transformado--antes-y-después)
3. [rate() vs increase() vs irate() — mismos datos, tres resultados](#3-rate-vs-increase-vs-irate--mismos-datos-tres-resultados)
4. [avg() sum() max() min() — mismo counter, cuatro resultados distintos](#4-avg-sum-max-min--mismo-counter-cuatro-resultados-distintos)
5. [Efecto del by (label) — colapsando y desagregando](#5-efecto-del-by-label--colapsando-y-desagregando)
6. [Desglose capa a capa con gráficos intermedios](#6-desglose-capa-a-capa-con-gráficos-intermedios)
7. [Cada tipo de panel con datos reales](#7-cada-tipo-de-panel-con-datos-reales)
8. [histogram_quantile() — percentiles visualizados](#8-histogram_quantile--percentiles-visualizados)
9. [Dashboard completo — ejemplo de layout real](#9-dashboard-completo--ejemplo-de-layout-real)
10. [Alertas — cómo se ven los umbrales disparados](#10-alertas--cómo-se-ven-los-umbrales-disparados)

---

## 1. Anatomía visual de un panel Grafana

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│  Título del panel ──────────────────────────────────────────────────── [i] [⚙] [⛶]  │
│                                                                                      │
│  18s ┤                     ╭───────╮                                                 │
│      │                    ╭╯       ╰╮                  ← Línea series "users"        │
│  12s ┤────────────────────╯         ╰──────────────                                  │
│      │  ← Eje Y (unidad)                                                             │
│   4s ┤· · · · · · · · · · · · · · · · · · · · · · ·  ← Línea series "products"      │
│      │                                                                               │
│   0s └──────────────────────────────────────────────────────────────────────────▶   │
│       10:00     10:30     11:00     11:30     12:00     12:30    ← Eje X (tiempo)    │
│                                                                                      │
│  ─────  cache_name="users"    →  Last: 12.1s  Mean: 11.8s  Max: 18.7s               │
│  · · ·  cache_name="products" →  Last:  3.2s  Mean:  3.5s  Max:  5.1s               │
│  └──── Leyenda (tabla con estadísticas) ────────────────────────────────────────┘   │
│                                                                                      │
│  Tooltip al pasar el ratón: muestra el valor exacto de cada serie en ese instante:  │
│  ┌──────────────────────┐                                                            │
│  │ 10:32:15             │                                                            │
│  │ users     ●  14.3s   │                                                            │
│  │ products  ●   4.1s   │                                                            │
│  └──────────────────────┘                                                            │
└──────────────────────────────────────────────────────────────────────────────────────┘
     ↑                    ↑                              ↑
  Eje Y                Área del                       Zoom:
  (unidad              gráfico                      scroll en el área
  en Panel →                                        o arrastrar para
  Unit)                                             seleccionar rango
```

### Qué hace cada parte

| Elemento | Dónde se configura | Qué controla |
|----------|-------------------|--------------|
| **Título** | Panel → Title | El texto de cabecera |
| **Eje Y — unidad** | Panel → Axes → Unit | Cómo formatea los números (`s`, `bytes`, `req/s`…) |
| **Eje X** | Automático | El rango de tiempo del dashboard |
| **Líneas** | Cada query devuelve una serie = una línea | |
| **Leyenda** | Panel → Legend | Muestra labels y estadísticas |
| **Tooltip** | Panel → Tooltip | Al pasar el ratón muestra valores |
| **Thresholds** | Panel → Thresholds | Líneas horizontales de color |

---

## 2. Counter crudo vs transformado — antes y después

### Los datos de entrada (lo que hay en Prometheus)

```
amiga_java_cache_getMulti_timer_seconds_sum — contador acumulado desde que arrancó la app:

  t=09:00  →  45.200s  (acumulado)
  t=09:15  →  46.800s
  t=09:30  →  48.600s
  t=09:45  →  51.200s
  t=10:00  →  53.100s
  t=10:15  →  55.700s
  t=10:30  →  60.900s  ← pico de carga
  t=10:45  →  64.100s
  t=11:00  →  65.900s
  t=11:15  →  67.200s
```

---

### ❌ SIN transformar — counter crudo (casi inútil para monitoreo)

```promql
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="users"}
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  cache sum (crudo)                                                   │
│                                                                      │
│  67k ┤                                                  ╭───────    │
│      │                                               ╭──╯           │
│  60k ┤                                        ╭──────╯              │
│      │                               ╭────────╯                     │
│  53k ┤                       ╭───────╯                              │
│      │              ╭────────╯                                      │
│  46k ┤     ╭────────╯                                               │
│      │─────╯                                                        │
│  45k └──────────────────────────────────────────────────────────▶  │
│       09:00   09:30   10:00   10:30   11:00                         │
│                                                                      │
│  ─────  users                                                        │
└──────────────────────────────────────────────────────────────────────┘

Problemas:
  1. El número (67.000 segundos) no significa nada sin contexto
  2. Siempre sube → no puedes ver picos de carga
  3. Si el pod se reinicia, el contador baja a 0 → línea confusa
  4. Ver "¿cuándo fue el momento de más carga?" es imposible
```

---

### ✅ CON `increase()` — incremento por intervalo (útil)

```promql
increase(amiga_java_cache_getMulti_timer_seconds_sum{cache_name="users"}[$__interval])
```

*Con `$__interval = 15min` (dashboard mostrando las últimas 6h):*

```
Cálculo de increase() en cada punto:
  t=09:15 → 46.800 - 45.200 = 1.6s  (cuánto creció en 15min)
  t=09:30 → 48.600 - 46.800 = 1.8s
  t=09:45 → 51.200 - 48.600 = 2.6s  ← empezando a subir
  t=10:00 → 53.100 - 51.200 = 1.9s
  t=10:15 → 55.700 - 53.100 = 2.6s
  t=10:30 → 60.900 - 55.700 = 5.2s  ← PICO (visible ahora!)
  t=10:45 → 64.100 - 60.900 = 3.2s
  t=11:00 → 65.900 - 64.100 = 1.8s
  t=11:15 → 67.200 - 65.900 = 1.3s
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  Cache getMulti — tiempo por intervalo de 15min                      │
│                                                                      │
│  5.2s ┤                    ╭──╮                                      │
│       │                   ╭╯  ╰╮                                     │
│  3.2s ┤    ╭──────────────╯    ╰──────────────────                   │
│       │   ╭╯                                                         │
│  1.6s ┤───╯                                                          │
│       │                                                              │
│   0s  └──────────────────────────────────────────────────────────▶  │
│        09:00   09:30   10:00   10:30   11:00                         │
│                              ↑                                       │
│                         Pico de carga                                │
│                         claramente visible                           │
│                                                                      │
│  ─────  users                                                        │
└──────────────────────────────────────────────────────────────────────┘

Ventajas ahora:
  1. Ves cuándo hubo más carga (pico a las 10:30)
  2. La escala es entendible (segundos por intervalo)
  3. Si el pod se reinicia → increase() lo detecta y no genera pico falso
```

---

## 3. rate() vs increase() vs irate() — mismos datos, tres resultados

**Datos de entrada:** `http_requests_total{pod="pod-1"}` durante 5 minutos

```
t=10:00:00  →  1.200 peticiones (acumulado)
t=10:01:00  →  1.280    (+ 80 peticiones en 1min)
t=10:02:00  →  1.350    (+ 70)
t=10:03:00  →  1.490    (+ 140) ← pico
t=10:04:00  →  1.560    (+ 70)
t=10:05:00  →  1.630    (+ 70)
```

---

### Panel A — `rate([5m])` → peticiones **por segundo**, suavizado

```promql
rate(http_requests_total{pod="pod-1"}[5m])
```

```
Cálculo: (1630 - 1200) / 300 segundos = 1.43 req/s  (promedia todo el rango)

┌──────────────────────────────────────────────────────────────────────┐
│  HTTP requests — rate (req/s)                                        │
│                                                                      │
│  2.0 ┤                                                               │
│      │                                                               │
│  1.5 ┤──────────────────────────────────────────────────────────    │
│      │ ← línea casi plana porque PROMEDIA los 5 minutos             │
│  1.0 ┤                                                               │
│      │                                                               │
│  0.5 ┤                                                               │
│      │                                                               │
│   0  └──────────────────────────────────────────────────────────▶  │
│       10:01   10:02   10:03   10:04   10:05                          │
│                                                                      │
│  ─────  pod-1  (1.43 req/s promedio)                                 │
└──────────────────────────────────────────────────────────────────────┘

✅ Uso: tendencias, alertas, SLOs ("¿cuántas req/s en promedio?")
⚠️ El pico de 10:03 está suavizado — no lo verás tan pronunciado
```

---

### Panel B — `increase([5m])` → peticiones **totales** en el rango

```promql
increase(http_requests_total{pod="pod-1"}[5m])
```

```
Cálculo: 1630 - 1200 = 430 peticiones en 5 minutos

┌──────────────────────────────────────────────────────────────────────┐
│  HTTP requests — increase (total en ventana)                         │
│                                                                      │
│  500 ┤                                                               │
│      │                                                               │
│  430 ┤──────────────────────────────────────────────────────────    │
│      │ ← misma forma suavizada que rate(), escala diferente         │
│  300 ┤                                                               │
│      │   Nota: 430 = rate(1.43) × 300 segundos                      │
│  150 ┤   La FORMA es idéntica a rate(), solo cambia la unidad       │
│      │                                                               │
│   0  └──────────────────────────────────────────────────────────▶  │
│       10:01   10:02   10:03   10:04   10:05                          │
│                                                                      │
│  ─────  pod-1  (430 peticiones en 5min)                              │
└──────────────────────────────────────────────────────────────────────┘

✅ Uso: "¿cuántas peticiones hubo en este periodo?" — conteos, facturación
✅ Es lo que usa la query de caché: segundos totales gastados en ese intervalo
```

---

### Panel C — `irate([5m])` → tasa **instantánea** (últimos 2 puntos)

```promql
irate(http_requests_total{pod="pod-1"}[5m])
```

```
Cálculo: solo usa los 2 últimos puntos del rango:
  t=10:04 → 1560
  t=10:05 → 1630    irate = (1630 - 1560) / 60s = 1.17 req/s (en este momento)

  En t=10:03 (el pico): irate = (1490 - 1350) / 60s = 2.33 req/s ← pico visible!

┌──────────────────────────────────────────────────────────────────────┐
│  HTTP requests — irate (tasa instantánea)                            │
│                                                                      │
│  2.5 ┤              ╭──╮                                             │
│      │             ╭╯  ╰╮                                            │
│  1.5 ┤─────────────╯    ╰──────────────────────────────────         │
│      │ ← la curva es MÁS PRONUNCIADA — detecta el pico de 10:03    │
│  0.5 ┤                                                               │
│      │                                                               │
│   0  └──────────────────────────────────────────────────────────▶  │
│       10:01   10:02   10:03   10:04   10:05                          │
│                              ↑                                       │
│                         Pico visible                                 │
│                         (irate lo detecta)                           │
│  ─────  pod-1                                                        │
└──────────────────────────────────────────────────────────────────────┘

✅ Uso: detectar picos muy breves (duración de 1-2 scrapes)
❌ No usar para alertas ni tendencias (demasiado nervioso)
```

---

### Comparativa en una tabla

```
┌──────────────┬─────────────────────────────┬────────────────────────────────────┐
│  Función     │  Resultado con estos datos  │  Cuándo usarla                     │
├──────────────┼─────────────────────────────┼────────────────────────────────────┤
│  rate([5m])  │  1.43 req/s (plano)         │  Tendencias, SLOs, alertas         │
│  increase()  │  430 peticiones (plano)      │  Totales, conteos, facturación     │
│  irate([5m]) │  1.17 req/s (con pico 2.33) │  Detectar microbursts              │
└──────────────┴─────────────────────────────┴────────────────────────────────────┘
```

---

## 4. avg() sum() max() min() — mismo counter, cuatro resultados distintos

### Datos de entrada: 3 pods, 2 cachés — en un minuto concreto

```
Resultado de increase() para cada serie en t=10:30:

  {cache_name="users",   hostname="pod-1"}  →  18s  ← pod más lento
  {cache_name="users",   hostname="pod-2"}  →  10s
  {cache_name="users",   hostname="pod-3"}  →  14s
  {cache_name="products",hostname="pod-1"}  →   5s  ← pod más lento en products
  {cache_name="products",hostname="pod-2"}  →   3s
  {cache_name="products",hostname="pod-3"}  →   4s
```

---

### avg() by (cache_name) — comportamiento típico

```promql
avg(increase(...)) by (cache_name)
```

```
Cálculo:
  users    → (18 + 10 + 14) / 3 = 14.0s
  products → ( 5 +  3 +  4) / 3 =  4.0s

┌──────────────────────────────────────────────────────────────────────┐
│  avg — tiempo promedio entre pods                                    │
│                                                                      │
│  18s ┤                                                               │
│      │                                                               │
│  14s ┤──────────────────────────────────────────────────────────    │← users (14s)
│      │                                                               │
│   8s ┤                                                               │
│      │                                                               │
│   4s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·  │← products (4s)
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ─────  users     (14.0s)  → "en promedio, cada pod gasta 14s"      │
│  · · ·  products   (4.0s)  → "en promedio, cada pod gasta 4s"       │
│                                                                      │
│  ✅ Responde: "¿cuánto gasta en cache UNA instancia típica?"         │
└──────────────────────────────────────────────────────────────────────┘
```

---

### sum() by (cache_name) — throughput total del sistema

```promql
sum(increase(...)) by (cache_name)
```

```
Cálculo:
  users    → 18 + 10 + 14 = 42s  (tiempo TOTAL de todos los pods juntos)
  products →  5 +  3 +  4 = 12s

┌──────────────────────────────────────────────────────────────────────┐
│  sum — tiempo total de todos los pods                                │
│                                                                      │
│  42s ┤──────────────────────────────────────────────────────────    │← users (42s)
│      │                                                               │
│  30s ┤                                                               │
│      │                                                               │
│  20s ┤                                                               │
│      │                                                               │
│  12s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·  │← products (12s)
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ─────  users     (42s)  → "todos los pods juntos gastan 42s"       │
│  · · ·  products  (12s)  → "todos los pods juntos gastan 12s"       │
│                                                                      │
│  ✅ Responde: "¿cuánta carga total soporta el sistema?"              │
│  ⚠️  Si añades un pod más, sum() sube aunque cada pod trabaje igual  │
└──────────────────────────────────────────────────────────────────────┘
```

---

### max() by (cache_name) — el peor caso

```promql
max(increase(...)) by (cache_name)
```

```
Cálculo:
  users    → max(18, 10, 14) = 18s  ← el pod más lento
  products → max( 5,  3,  4) =  5s

┌──────────────────────────────────────────────────────────────────────┐
│  max — el pod más lento de cada caché                                │
│                                                                      │
│  18s ┤──────────────────────────────────────────────────────────    │← users (18s = pod-1)
│      │                                                               │
│  12s ┤                                                               │
│      │                                                               │
│   8s ┤                                                               │
│      │                                                               │
│   5s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·  │← products (5s = pod-1)
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ─────  users     (18s)  → "el pod más lento de users es pod-1"     │
│  · · ·  products   (5s)  → "el pod más lento de products es pod-1"  │
│                                                                      │
│  ✅ Responde: "¿hay algún pod con problemas? ¿cuál es el peor caso?" │
│  ✅ Útil para alertas: si max() > umbral, al menos un pod está mal   │
└──────────────────────────────────────────────────────────────────────┘
```

---

### min() by (cache_name) — el eslabón más fuerte

```promql
min(increase(...)) by (cache_name)
```

```
Cálculo:
  users    → min(18, 10, 14) = 10s  ← el pod más rápido
  products → min( 5,  3,  4) =  3s

┌──────────────────────────────────────────────────────────────────────┐
│  min — el pod más rápido de cada caché                               │
│                                                                      │
│  18s ┤                                                               │
│      │                                                               │
│  12s ┤                                                               │
│      │                                                               │
│  10s ┤──────────────────────────────────────────────────────────    │← users (10s = pod-2)
│      │                                                               │
│   3s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·  │← products (3s = pod-2)
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ✅ Responde: "¿cuál es el mejor rendimiento posible? (línea base)"  │
│  ✅ Si min() y max() se separan mucho → los pods se comportan distinto│
└──────────────────────────────────────────────────────────────────────┘
```

---

### Cuatro funciones en un solo panel — comparativa visual

```
┌──────────────────────────────────────────────────────────────────────┐
│  Cache users — comparativa de agregaciones en t=10:30                │
│                                                                      │
│  42s ┤═══════════════════════════════════════════════════════════    │← sum=42s (carga total)
│      │                                                               │
│  18s ┤──────────────────────────────────────────────────────────    │← max=18s (peor pod)
│      │                                                               │
│  14s ┤─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │← avg=14s (típico)
│      │                                                               │
│  10s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·  │← min=10s (mejor pod)
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ═════  sum (42s) → carga total del sistema (todos los pods)         │
│  ─────  max (18s) → el pod más lento (para alertas)                  │
│  ─ ─ ─  avg (14s) → comportamiento típico                            │
│  · · ·  min (10s) → el pod más rápido (línea base)                   │
│                                                                      │
│  Brecha max-min = 18-10 = 8s → los pods se comportan de forma        │
│  desigual. Investigar por qué pod-1 es tan lento.                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Efecto del by (label) — colapsando y desagregando

### Los datos sin ningún `by` — sin agregación

```promql
increase(amiga_java_cache_getMulti_timer_seconds_sum{environment="prod"}[$__interval])
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  Sin by — 6 líneas (3 pods × 2 cachés)                               │
│                                                                      │
│  20s ┤                  ╭──╮                                         │← users-pod-3
│      │           ╭──────╯  ╰──────────╮                             │← users-pod-1
│  14s ┤    ╭──────╯                    ╰──────────────                │← users-pod-2 (aprox)
│      │                                                               │
│   6s ┤╭───╮  ╭───╮  ╭───╮                                           │← products-pod-1
│   4s ┤╯   ╰──╯   ╰──╯                                               │← products-pod-2,3
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  Leyenda:                                                            │
│  ─────  {cache_name="users",   hostname="pod-1", environment="prod"} │
│  ─────  {cache_name="users",   hostname="pod-2", environment="prod"} │
│  ─────  {cache_name="users",   hostname="pod-3", environment="prod"} │
│  · · ·  {cache_name="products",hostname="pod-1", environment="prod"} │
│  · · ·  {cache_name="products",hostname="pod-2", environment="prod"} │
│  · · ·  {cache_name="products",hostname="pod-3", environment="prod"} │
│                                                                      │
│  Problema: difícil de leer. Los labels llenan la leyenda.            │
│  Útil solo para debugging detallado de un pod concreto.              │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Con `by (cache_name)` — 2 líneas, una por caché

```promql
avg(increase(...)) by (cache_name)
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  by (cache_name) — 2 líneas limpias                                  │
│                                                                      │
│  18s ┤              ╭───────╮                                        │
│      │         ╭────╯       ╰─────────────────                       │
│  12s ┤─────────╯                                                     │
│      │                                                               │
│   5s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·  │
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ─────  cache_name="users"      → promedio de los 3 pods             │
│  · · ·  cache_name="products"   → promedio de los 3 pods             │
│                                                                      │
│  ✅ Respuesta clara: "¿qué caché consume más tiempo?"                │
│  La caché "users" consume ~3x más que "products"                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Con `by (hostname)` — 3 líneas, una por pod

```promql
avg(increase(...)) by (hostname)
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  by (hostname) — 3 líneas, una por pod                               │
│                                                                      │
│  23s ┤              ╭──╮                                             │← pod-1 (más lento)
│      │         ╭────╯  ╰──────                                       │
│  17s ┤── ── ───╯                                                     │← pod-2
│      │                                                               │
│  14s ┤─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                │← pod-3
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ─────  hostname="pod-1"   (promedio de users+products = 23s / 2 ≈ 11.5s por caché)│
│  ── ──  hostname="pod-2"                                             │
│  ─ ─ ─  hostname="pod-3"                                             │
│                                                                      │
│  ✅ Respuesta: "¿hay algún pod que se comporta diferente?"           │
│  pod-1 tiene picos más altos → ¿problema de red? ¿hotspot de datos? │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Sin `by` y con `avg()` — 1 línea, todo mezclado

```promql
avg(increase(...))
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  Sin by — 1 línea (promedio de todo)                                 │
│                                                                      │
│  12s ┤              ╭──╮                                             │
│      │         ╭────╯  ╰─────────────────────────────               │
│   9s ┤─────────╯                                                     │
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ─────  {}   (sin labels — promedio de las 6 series)                 │
│                                                                      │
│  ⚠️  Mezcla "users" (14s) con "products" (4s) → resultado: 9s        │
│  Pierde toda la información sobre qué caché es más lenta             │
│  Útil solo para un KPI de "salud general del sistema de caché"       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Desglose capa a capa con gráficos intermedios

### La query completa

```promql
avg(increase(amiga_java_cache_getMulti_timer_seconds_sum{environment=~"prod"}[$__interval])) by (cache_name)
```

---

### Capa 1 — datos crudos filtrados

```
amiga_java_cache_getMulti_timer_seconds_sum{environment="prod"}
```

```
Estado de los datos después de la Capa 1:
  Series seleccionadas (filtradas por environment="prod"):
  ┌───────────────────────────────────┬──────────────────────────────────┐
  │  {cache_name="users",  pod="pod-1"}│  acumulado: 48.230s (y subiendo)│
  │  {cache_name="users",  pod="pod-2"}│  acumulado: 47.110s             │
  │  {cache_name="users",  pod="pod-3"}│  acumulado: 49.500s             │
  │  {cache_name="products",pod="pod-1"}│ acumulado:  9.910s             │
  │  {cache_name="products",pod="pod-2"}│ acumulado:  8.870s             │
  │  {cache_name="products",pod="pod-3"}│ acumulado: 10.230s             │
  └───────────────────────────────────┴──────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  Gráfico de la Capa 1 (si lo pintaras directamente — NO HAGAS ESTO)  │
│                                                                      │
│  50k ┤                                               ╭──────────    │
│  48k ┤                                         ╭─────╯              │
│  47k ┤                                   ╭─────╯                    │
│      │                             ╭─────╯                          │
│  10k ┤  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─╭─╯                                │
│   9k ┤· · · · · · · · · · · · · · ·                                 │
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ❌ Ilegible: números enormes que solo crecen                         │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Capa 2 — con increase() aplicado

```
increase(amiga_java_cache_getMulti_timer_seconds_sum{environment="prod"}[$__interval])
```

*Con `$__interval = 1 minuto`:*

```
Estado después de la Capa 2 — cada serie tiene el incremento por minuto:
  ┌───────────────────────────────────┬─────────────────────────────────────────┐
  │  {users,   pod-1}                 │  [12, 11, 18, 12, 12, 14, ...]  s/min  │
  │  {users,   pod-2}                 │  [10, 13, 16, 11, 14, 10, ...]          │
  │  {users,   pod-3}                 │  [13, 12, 19, 13, 11, 15, ...]          │
  │  {products,pod-1}                 │  [ 3,  4,  5,  3,  4,  3, ...]          │
  │  {products,pod-2}                 │  [ 4,  3,  4,  4,  3,  4, ...]          │
  │  {products,pod-3}                 │  [ 3,  4,  5,  3,  4,  3, ...]          │
  └───────────────────────────────────┴─────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  Gráfico de la Capa 2 (6 líneas — útil pero ruidoso)                 │
│                                                                      │
│  19s ┤              ╭╮                                               │← users-pod-3
│  18s ┤            ╭─╯╰╮                                             │← users-pod-1
│  16s ┤         ╭──╯   ╰──────────────────────                       │← users-pod-2
│  14s ┤──╮ ╭────╯                                                    │
│  12s ┤  ╰─╯                                                         │← users (los 3 pods)
│      │                                                               │
│   5s ┤         ╭╮   ╭╮                                              │← products-pod-1,3
│   4s ┤╭──╮╭────╯╰───╯╰──╮╭──────                                   │← products-pod-2
│   3s ┤╯  ╰╯              ╰╯                                          │
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│                                                                      │
│  ✅ Los valores son manejables (10-19s) y los picos son visibles      │
│  ⚠️  6 líneas superpuestas son difíciles de leer                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Capa 3 — resultado final con avg() by (cache_name)

```
avg(increase(...)) by (cache_name)
```

```
Operación en cada punto de tiempo — tomemos t=10:03 (el pico):
  users   pod-1=18, pod-2=16, pod-3=19 → avg = (18+16+19)/3 = 17.7s
  products pod-1=5, pod-2=4,  pod-3=5  → avg = ( 5+ 4+ 5)/3 =  4.7s

  → Las 6 series se colapsan en 2 series

┌──────────────────────────────────────────────────────────────────────┐
│  Gráfico FINAL — 2 líneas limpias                                    │
│                                                                      │
│  18s ┤              ╭────╮                                           │
│      │         ╭────╯    ╰────────────────────                       │
│  12s ┤─────────╯                                                     │
│      │                                                               │
│  4.7s┤· · · · ·╭· · · ·╮· · · · · · · · · · · · · · · · · · ·     │
│   3s ┤· · · · ·╯        ╰                                           │
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│       10:00   10:02   10:04   10:06   10:08   10:10                  │
│                       ↑                                              │
│                  Pico de carga                                       │
│                  (visible y claro)                                   │
│                                                                      │
│  ─────  cache_name="users"    → Promedio entre pods (≈12-18s)        │
│  · · ·  cache_name="products" → Promedio entre pods (≈3-5s)          │
│                                                                      │
│  Conclusión visual: la caché "users" tarda 3-4x más que "products"  │
│  El pico a las 10:03 afecta más a "users" que a "products"           │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Flujo completo resumido

```
  Capa 1: Filtrar                Capa 2: increase()            Capa 3: avg() by (cache_name)
  ─────────────────────          ──────────────────────         ─────────────────────────────
  6 series con números  →        6 series con incrementos  →    2 series — RESULTADO FINAL
  enormes y crecientes           por minuto (manejables)

  {users,  pod-1}: 48230s        {users,  pod-1}: 12s/min  ─┐
  {users,  pod-2}: 47110s        {users,  pod-2}: 10s/min   ├─ avg → {users}    = 11.3s/min
  {users,  pod-3}: 49500s        {users,  pod-3}: 13s/min  ─┘
  {products,pod-1}: 9910s        {products,pod-1}: 3s/min  ─┐
  {products,pod-2}: 8870s        {products,pod-2}: 4s/min   ├─ avg → {products} = 3.3s/min
  {products,pod-3}:10230s        {products,pod-3}: 3s/min  ─┘

       ❌ gráfico inútil              ⚠️ ruidoso (6 líneas)        ✅ claro (2 líneas)
```

---

## 7. Cada tipo de panel con datos reales

### Time Series — múltiples líneas con leyenda completa

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Cache getMulti — tiempo promedio por caché               Last 6h  ▼  [🔔] [⚙] │
│                                                                                  │
│  20s ┤                                                                           │
│      │                                                                           │
│  16s ┤              ╭──────╮                    ╭──╮                             │
│      │         ╭────╯      ╰──────────────╭─────╯  ╰──────                      │
│  12s ┤─────────╯                         ╭╯                                     │
│      │                                   │  ← subida gradual a las 14:30        │
│   8s ┤                                                                           │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  WARNING (7s)│
│   4s ┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·               │
│   0s └──────────────────────────────────────────────────────────────────────▶  │
│       09:00    10:00    11:00    12:00    13:00    14:00    15:00                 │
│                                                                                  │
│  ─────  users     │ Last: 12.1s │ Mean: 11.8s │ Max: 18.7s │ Min: 10.2s         │
│  · · ·  products  │ Last:  3.2s │ Mean:  3.5s │ Max:  5.1s │ Min:  2.9s         │
└──────────────────────────────────────────────────────────────────────────────────┘

Panel → Legend → Mode: Table
Panel → Axes → Unit: seconds
Panel → Thresholds → 7 (warning, amarillo)
```

---

### Stat — número grande con sparkline

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  Pods activos        │  │  Cache miss rate      │  │  Errores (1h)        │
│                      │  │                       │  │                      │
│  ● 12                │  │  🟡 8.3%              │  │  🔴 47               │
│                      │  │                       │  │                      │
│  ↗ +2 vs ayer        │  │  ↑ +2.1% vs hace 1h   │  │  ↑ +12 vs hace 1h   │
│                      │  │                       │  │                      │
│  ▁▂▂▃▃▃▃▃▃▄▄▄ (12)  │  │  ▂▃▃▃▄▅▅▅▆▇▇▇ ↑       │  │  ▁▁▁▁▁▁▂▃▅▇▇▇ ↑     │
│  └── sparkline ──┘   │  │   └─ tendencia ─┘     │  │  └─ pico ─┘         │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
    (verde = ok)               (amarillo = warning)       (rojo = crítico)

Panel → Stat → Calculation: Last
Panel → Thresholds:
  Base → verde
  5    → amarillo  (warning: más del 5% de miss rate)
  10   → rojo      (critical: más del 10%)
Panel → Sparkline: show
```

---

### Gauge — medidor con zonas de color

```
┌───────────────────────────────────────┐
│  CPU Usage — pod-1                    │
│                                       │
│           ╭─────────────╮             │
│         ╱               ╲            │
│        ╱    🟡 72%        ╲           │
│       │  ╭────────────╮   │          │
│       │  │            │   │          │
│        ╲ ╰────────────╯  ╱           │
│    🟢   ╲               ╱  🔴        │
│           ╰─────────────╯            │
│    0%       50%        100%          │
│   ════════════╪═══════════           │
│   verde    amarillo    rojo           │
│   (< 70%)  (70-90%)   (> 90%)        │
└───────────────────────────────────────┘

Panel → Gauge → Min: 0, Max: 100
Panel → Thresholds: 70 (amarillo), 90 (rojo)
Panel → Unit: percent (0-100)

Query:
  100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

---

### Bar chart — ranking de servicios

```
┌──────────────────────────────────────────────────────────────────────┐
│  Peticiones por servicio — ahora mismo                               │
│                                                                      │
│  api         ████████████████████████████████████ 6.3 req/s          │
│  frontend    ████████████████████████  4.1 req/s                     │
│  auth        ████████████  2.0 req/s                                 │
│  worker      ███████  1.2 req/s                                      │
│  batch       ████  0.7 req/s                                         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

Query:
  sum(rate(http_requests_total[5m])) by (service)

Panel → Bar chart → Orientation: Horizontal
Panel → Standard options → Unit: req/s
```

---

### Table — todos los labels visibles

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Cache detail — por pod y caché                                          │
│                                                                          │
│  cache_name  │  hostname  │  environment │  Last    │  Mean   │  Max    │
│ ─────────────┼────────────┼──────────────┼──────────┼─────────┼──────── │
│  users       │  pod-1     │  prod        │  12.3s   │  11.9s  │  18.7s  │
│  users       │  pod-2     │  prod        │  10.1s   │   9.8s  │  16.2s  │
│  users       │  pod-3     │  prod        │  13.5s   │  12.1s  │  19.1s  │
│  products    │  pod-1     │  prod        │   3.2s   │   3.1s  │   5.1s  │
│  products    │  pod-2     │  prod        │   3.8s   │   3.4s  │   4.9s  │
│  products    │  pod-3     │  prod        │   4.1s   │   3.7s  │   5.3s  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

Panel → Table
Panel → Legend → Mode: Table, Placement: Bottom
Panel → Transformations → Merge (si usas varias queries)
```

---

### Heatmap — distribución de latencias a lo largo del tiempo

```
┌──────────────────────────────────────────────────────────────────────┐
│  HTTP request duration — distribución                                │
│                                                                      │
│  > 1s  │░░░░░░░░░░░░░░░░▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░        │
│  500ms │░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░▓░░░░░░░░░░░░░░░░░         │
│  250ms │░░░░░▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░            │
│  100ms │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓           │
│   50ms │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓           │
│   10ms │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓           │
│        └───────────────────────────────────────────────▶ tiempo    │
│         09:00   10:00   11:00   12:00   13:00                       │
│                         ↑                                           │
│                    Pico de latencia                                 │
│                    a las 11:00                                      │
│                                                                     │
│  ░ = pocas peticiones en esa latencia                               │
│  ▓ = muchas peticiones en esa latencia                              │
│                                                                     │
│  Lectura: la mayoría tarda 50-100ms (zona más densa)               │
│           pico ocasional >500ms a las 11:00 (incidente)             │
└──────────────────────────────────────────────────────────────────────┘

Query:
  rate(http_request_duration_seconds_bucket[5m])

Panel → Heatmap
Panel → Data format: Time series buckets
```

---

## 8. histogram_quantile() — percentiles visualizados

### Qué mide cada percentil

```
Imagina 100 peticiones HTTP. Ordenadas por duración, de más rápida a más lenta:

  1, 2, 3... 45ms (p50: la mitad tardó menos, la mitad más)
                          ... 89ms (p90: el 90% tardó menos que esto)
                                     ... 120ms (p95: el 95% tardó menos)
                                                        ... 450ms (p99: el 99% tardó menos)
                                                                           ... 1200ms (la más lenta)

En diagrama de cajas:
                        p50        p90  p95             p99
  0ms──────────────────[─┼──────────┼────┼───────────────┼──────────────────]1200ms
                       45ms       89ms 120ms           450ms

  ← las más rápidas ──────────────────────────────────── las más lentas →
```

---

### Cuatro líneas de percentiles en el mismo panel

```promql
# Query A: histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
# Query B: histogram_quantile(0.90, rate(http_request_duration_seconds_bucket[5m]))
# Query C: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
# Query D: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  HTTP request latency — percentiles                                  │
│                                                                      │
│  500ms┤                    ╭────────╮                                │← p99 (los más lentos 1%)
│       │                   ╭╯        ╰──────────────                  │
│  200ms┤            ╭──────╯                                          │
│       │       ╭────╯                                  ╭──           │← p95 (los más lentos 5%)
│  120ms┤ ── ───╯                                ───────╯  ─          │
│       │                                ╭───────╯                     │← p90 (los más lentos 10%)
│   90ms┤─────────────────────────╭──────╯                             │
│       │                                                               │
│   45ms┤· · · · · · · · · · · · · · · · · · · · · · · · · · · · · · │← p50 (la mediana)
│       │                                                               │
│   10ms└──────────────────────────────────────────────────────────▶  │
│        09:00   10:00   11:00   12:00   13:00                          │
│                         ↑                                            │
│                    Incidente a las 11:00:                            │
│                    p99 sube a 500ms, p95 a 200ms                    │
│                    p50 apenas se mueve → no afecta a la mayoría     │
│                                                                      │
│  ─ ─ ─  p50 (mediana)  ~45ms  → la mitad de peticiones              │
│  ─────  p90            ~90ms  → el 90% de peticiones                 │
│  ─────  p95           ~120ms  → el 95% de peticiones                 │
│  ─────  p99           ~200ms  → el 99% de peticiones (los lentos)    │
└──────────────────────────────────────────────────────────────────────┘

Cómo leer el incidente de las 11:00:
  p50 apenas subió  → la mayoría de usuarios no lo notó
  p95 subió a 200ms → el 5% más lento tuvo mala experiencia
  p99 subió a 500ms → el 1% tuvo peticiones muy lentas

  Conclusión: el incidente afectó a peticiones pesadas (p99/p95)
              pero el tráfico general (p50) siguió normal.
              Posible causa: una query lenta o un lock de BD intermitente.
```

---

### Diferencia entre p50, p95, p99

```
┌─────────┬──────────────────────────────────┬──────────────────────────────────────┐
│ Métrica │ Significado                      │ Cuándo usarla                        │
├─────────┼──────────────────────────────────┼──────────────────────────────────────┤
│  p50    │ La mitad tardó menos que esto    │ Latencia "típica" del usuario medio  │
│  p90    │ El 90% tardó menos que esto      │ SLO estándar en muchas empresas      │
│  p95    │ El 95% tardó menos que esto      │ SLO estricto, experiencia de usuario │
│  p99    │ El 99% tardó menos que esto      │ Detectar outliers y casos extremos   │
└─────────┴──────────────────────────────────┴──────────────────────────────────────┘

Ejemplo real con SLO:
  "El 95% de las peticiones deben responder en menos de 200ms"
  → query de alerta: histogram_quantile(0.95, ...) > 0.2
  → si la línea p95 cruza la línea roja de 200ms → alerta
```

---

## 9. Dashboard completo — ejemplo de layout real

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  📊 Application Dashboard                              Last 6h ▼   🔄 30s               │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ ┌──────┐ ┌──────────┐                       │
│  │ Platform ▼│ │ Tenant  ▼│ │ Environment ▼│ │ Pod ▼│ │ Jirakey ▼│                       │
│  │   web     │ │  acme   │ │    prod      │ │ All  │ │  PRJ-A   │                       │
│  └──────────┘ └──────────┘ └─────────────┘ └──────┘ └──────────┘                       │
│                                                                                          │
│  ┌─────────────────────┐  ┌────────────────────┐  ┌─────────────────────┐              │
│  │  Pods activos  [Stat]│  │  CPU avg   [Gauge] │  │  Error rate  [Stat] │              │
│  │                      │  │                    │  │                     │              │
│  │       ● 12           │  │    🟢 45%          │  │    🟢 0.8%          │              │
│  │  ↑ +2 vs ayer        │  │  ────────          │  │  ↓ -0.2% vs 1h     │              │
│  │  ▁▂▃▃▃▄▄▄▄▄ (12)    │  │  0%  50% 100%      │  │  ▁▁▁▁▁▁▁▁▁▁ (0.8%) │              │
│  └─────────────────────┘  └────────────────────┘  └─────────────────────┘              │
│                                                                                          │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  Cache getMulti — tiempo promedio por caché                     [Time Series]      │ │
│  │                                                                                    │ │
│  │  18s ┤              ╭───╮                    ╭──╮                                 │ │
│  │      │         ╭────╯   ╰──────────────╭─────╯  ╰─────                           │ │
│  │  12s ┤─────────╯                       ╭╯                                        │ │
│  │   4s ┤· · · · · · · · · · · · · · · · ·╯ · · · · · · · · · · · · ·              │ │
│  │   0s └────────────────────────────────────────────────────────────────▶           │ │
│  │       09:00    10:00    11:00    12:00    13:00    14:00    15:00                  │ │
│  │  ─────  users (12.1s)    · · ·  products (3.2s)                                  │ │
│  └────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                          │
│  ┌──────────────────────────────────────────┐  ┌──────────────────────────────────────┐ │
│  │  HTTP Latency p95  [Time Series]         │  │  Cache detail  [Table]               │ │
│  │                                          │  │                                      │ │
│  │  200ms┤       ╭──────╮                  │  │  cache_name│ pod  │ Last │ Max       │ │
│  │  120ms┤───────╯      ╰─────────────     │  │  users     │pod-1 │12.3s │18.7s      │ │
│  │   50ms┤· · · · · · · · · · · · · · ·    │  │  users     │pod-2 │10.1s │16.2s      │ │
│  │   0ms └──────────────────────────────▶  │  │  products  │pod-1 │ 3.2s │ 5.1s      │ │
│  │       09:00   12:00   15:00             │  │  products  │pod-2 │ 3.8s │ 4.9s      │ │
│  └──────────────────────────────────────────┘  └──────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────┘

Organización del dashboard:
  Fila 1: KPIs (Stat + Gauge) → "¿el sistema está sano?"
  Fila 2: Tendencias (Time Series grande) → "¿qué está pasando en el tiempo?"
  Fila 3: Detalle (Time Series pequeño + Table) → "¿dónde exactamente?"
```

---

## 10. Alertas — cómo se ven los umbrales disparados

### Panel normal vs panel con alerta disparada

```
ESTADO NORMAL — todo en verde:
┌──────────────────────────────────────────────────────────────────────┐
│  Cache getMulti time  ● OK                                           │
│                                                                      │
│  20s ┤- - - - - - - - - - - - - - - - - - - - - - - - - -  CRIT(20s)│
│      │                                                               │
│  15s ┤─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ WARN(15s)│
│      │                                                               │
│  12s ┤──────────────────────────────────────────────────────────    │← métrica (12s, OK)
│      │                                                               │
│   0s └──────────────────────────────────────────────────────────▶  │
│       (el área bajo la curva es verde)                               │
└──────────────────────────────────────────────────────────────────────┘


ALERTA DISPARADA — la métrica cruza el umbral WARNING:
┌──────────────────────────────────────────────────────────────────────┐
│  Cache getMulti time  🔴 FIRING                         [ver alerta]│
│                                                                      │
│  20s ┤- - - - - - - - - - - - - - - - - - - - - - - - - -  CRIT(20s)│
│      │                                             ╭──────────────   │← ¡superó WARNING!
│  16s ┤                               ╭─────────────╯                 │
│ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  WARN(15s)│
│  12s ┤──────────────────────────────╭╯                               │
│      │                              │                                │
│   0s └──────────────────────────────┼───────────────────────────▶  │
│                                     ↑                                │
│                            [Anotación: "Alerta disparada 14:32"]     │
│                            (área sobre umbral en rojo/amarillo)      │
└──────────────────────────────────────────────────────────────────────┘


ALERTA CRÍTICA — supera el umbral CRITICAL:
┌──────────────────────────────────────────────────────────────────────┐
│  Cache getMulti time  🚨 CRITICAL                       [ver alerta]│
│                                                                      │
│  24s ┤                                    ╭──────╮                  │← ¡superó CRITICAL!
│ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┼─ ─ ─ ─┼─ ─  CRIT(20s)   │
│  18s ┤                          ╭─────────╯      ╰──────            │
│ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  WARN(15s)   │
│  12s ┤──────────────────────────╯                                    │
│      │                          ↑                                   │
│   0s └──────────────────────────┼───────────────────────────────▶  │
│                      [💬 "Alerta disparada 14:45"]                   │
│                      [💬 "Resuelto 15:02"]  ←─ anotaciones automáticas│
└──────────────────────────────────────────────────────────────────────┘
```

### Cómo configurar los umbrales

```
Panel → Thresholds:

  Base  →  🟢 verde  (todo bien)
  15    →  🟡 amarillo  (Warning: más de 15s por intervalo)
  20    →  🔴 rojo      (Critical: más de 20s por intervalo)

Grafana Alerting → Alert rules → New alert rule:
  Condition: avg(increase(...)) by (cache_name)  >  15   → Warning
  Condition: avg(increase(...)) by (cache_name)  >  20   → Critical
  Evaluation interval: 1m
  Pending period: 5m  (debe superar el umbral 5min seguidos antes de disparar)
  Notification: Slack / PagerDuty / email
```

### Diagrama de estados de una alerta

```
         métrica sube       sube   métrica baja
              ↓              ↓           ↓
  Normal ──► Pending ──► Firing ──► Resolved
    🟢          🟡           🔴          🟢

  Normal:   por debajo del umbral
  Pending:  por encima, pero aún dentro del "pending period" (evita falsas alarmas)
  Firing:   confirmado — se envía la notificación
  Resolved: bajó del umbral — se envía notificación de recuperación
```

---

## Resumen: guía de lectura rápida de un gráfico

```
Al ver un panel Grafana con tu query de caché, respóndete estas preguntas:

1. ¿Cuántas líneas hay?
   → Una por cada cache_name diferente (gracias al  by (cache_name))

2. ¿Qué mide el eje Y?
   → Segundos gastados en caché por intervalo de tiempo (gracias a increase())
   → Si la unidad pone "s" → está en segundos ✅

3. ¿Qué mide el eje X?
   → El tiempo (el rango seleccionado en el dashboard: 6h, 24h, etc.)

4. ¿Por qué sube una línea?
   → Más tiempo gastado en esa caché = más llamadas O llamadas más lentas
   → Puede ser tráfico normal o un problema de rendimiento

5. ¿Qué significa si las líneas se separan mucho?
   → Una caché es mucho más usada/lenta que otra
   → Investigar si "users" siempre es más alta que "products" (normal) o si es nueva

6. ¿Qué significa un pico repentino?
   → Evento de tráfico (deploy, cron, evento externo)
   → Problema de rendimiento puntual (query lenta, lock, GC pause)
   → Si coincide con un deploy: posiblemente introducido en esa versión
```

