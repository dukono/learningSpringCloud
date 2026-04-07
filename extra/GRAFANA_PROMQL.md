# 📊 Grafana + PromQL: Guía Completa de Queries

Guía para entender cómo construir y leer queries en Grafana usando **PromQL** (Prometheus Query Language).

**Query real que usaremos como hilo conductor en todo el documento:**
```promql
avg(increase(amiga_java_cache_getMulti_timer_seconds_sum{platform=~"$platform",tenant=~"$tenant",environment=~"$environment",slot=~"$slot",hostname=~"$Pod",projectkey=~"$jirakey"}[$__interval])) by (cache_name)
```

---

## Índice

1. [¿Qué es PromQL? — El flujo completo](#1-qué-es-promql--el-flujo-completo)
2. [Tipos de métricas en Prometheus](#2-tipos-de-métricas-en-prometheus)
3. [Anatomía de una métrica](#3-anatomía-de-una-métrica)
4. [Selectores de labels y filtros](#4-selectores-de-labels-y-filtros)
5. [Variables de Grafana](#5-variables-de-grafana)
6. [Rangos de tiempo (Range Vectors)](#6-rangos-de-tiempo-range-vectors)
7. [Funciones sobre counters: rate, increase, irate](#7-funciones-sobre-counters-rate-increase-irate)
8. [Funciones de agregación: avg, sum, max, min...](#8-funciones-de-agregación-avg-sum-max-min)
9. [Agrupamiento: by y without](#9-agrupamiento-by-y-without)
10. [Desglose visual capa por capa de la query real](#10-desglose-visual-capa-por-capa-de-la-query-real)
11. [Tipos de visualización en Grafana](#11-tipos-de-visualización-en-grafana)
12. [Opciones comunes del panel](#12-opciones-comunes-del-panel)
13. [Cheatsheet rápido](#13-cheatsheet-rápido)

---

## 1. ¿Qué es PromQL? — El flujo completo

PromQL es el lenguaje de consultas de **Prometheus**, la base de datos de series temporales más usada para monitoreo. Grafana lo usa para construir gráficos y dashboards.

### El flujo de extremo a extremo

```
┌─────────────────┐     expone métricas      ┌─────────────────┐
│   Tu aplicación │ ──────────────────────▶  │   /metrics      │
│   (Java, Node…) │     en texto plano        │  endpoint HTTP  │
└─────────────────┘                           └────────┬────────┘
                                                       │  scrape cada 15s
                                                       ▼
                                              ┌─────────────────┐
                                              │   Prometheus    │
                                              │  (guarda series │
                                              │   temporales)   │
                                              └────────┬────────┘
                                                       │  PromQL query
                                                       ▼
                                              ┌─────────────────┐
                                              │    Grafana      │
                                              │  (visualiza el  │
                                              │    resultado)   │
                                              └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │    Gráfico 📈   │
                                              └─────────────────┘
```

### ¿Qué es una métrica?

Una métrica es **un número que cambia en el tiempo**, identificado por un nombre y etiquetas (labels).

```
# Así se ven las métricas en el endpoint /metrics de tu app:

http_requests_total{method="GET",  status="200"} 1523
http_requests_total{method="POST", status="200"}  847
http_requests_total{method="GET",  status="500"}   12
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="users",   hostname="pod-1"} 4823.7
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="products",hostname="pod-1"}  991.2
```

Prometheus recoge estos números cada 15s y los almacena. Grafana los consulta con PromQL.

---

## 2. Tipos de métricas en Prometheus

Hay 4 tipos. Saber cuál es cada una determina **qué funciones puedes usar**.

### Visión general

```
┌──────────────────────────────────────────────────────────────────────┐
│                    TIPOS DE MÉTRICAS                                 │
├───────────────┬──────────────────────────────┬───────────────────────┤
│     TIPO      │       COMPORTAMIENTO          │  FUNCIONES A USAR     │
├───────────────┼──────────────────────────────┼───────────────────────┤
│  Counter      │  Solo sube ↑ (nunca baja)    │  rate() increase()    │
│  Gauge        │  Sube y baja ↑↓ libremente   │  avg() max() directo  │
│  Histogram    │  Distribución (_sum _count   │  histogram_quantile() │
│               │  _bucket)                    │  rate() sobre _bucket │
│  Summary      │  Como Histogram pero en      │  igual que Histogram  │
│               │  el cliente                  │  (menos flexible)     │
└───────────────┴──────────────────────────────┴───────────────────────┘
```

---

### Counter (Contador)

**Solo sube.** Acumula desde que arrancó la app. Se resetea a 0 si el proceso se reinicia.

```
Tiempo:  t=0    t=1min  t=2min  t=3min  t=4min  t=5min
Valor:     0      120     280     390     520     640    ← nunca baja

Sufijos típicos: _total  _count  _sum
Ejemplos:  http_requests_total,  errors_total,  bytes_sent_total
```

**El problema de usarlo directamente:**
```
# Sin rate() — solo ves una línea que sube sin parar (inútil para alertas)

  640 │                                          ╭──
  520 │                               ╭──────────╯
  390 │              ╭────────────────╯
  280 │    ╭─────────╯
  120 │────╯
    0 └──────────────────────────────────────────────▶ tiempo
       (números enormes que crecen sin parar)
```

**Con `rate()` — ves la tasa real de cambio (útil):**
```
# rate(http_requests_total[5m]) — peticiones por segundo

  2.5 │         ╭──╮
  2.0 │    ╭────╯  ╰──╮     ╭──╮
  1.5 │────╯          ╰─────╯  ╰────
  1.0 │
    0 └──────────────────────────────▶ tiempo
       ← pico de tráfico a las 10:15 →
```

```promql
# ❌ MAL — número enorme que solo sube (el total desde que arrancó la app)
http_requests_total

# ✅ BIEN — peticiones por segundo en los últimos 5 min
rate(http_requests_total[5m])
```

---

### Gauge (Medidor)

**Sube y baja libremente.** Representa el estado actual en cada momento.

```
Tiempo:  t=0    t=1min  t=2min  t=3min  t=4min  t=5min
Valor:    512     480     623     590     410     530   ← sube y baja

Ejemplos: node_memory_MemAvailable_bytes
          http_connections_active
          queue_size_current
```

**Gráfico típico de un Gauge (memoria libre):**
```
  623 │              ╭──╮
  530 │                  ╰──────────────╭──
  512 │────╮                            │
  480 │    ╰────╮              ╭────────╯
  410 │         ╰──────────────╯
    0 └──────────────────────────────────────▶ tiempo
```

```promql
# Directamente — muestra el valor actual
node_memory_MemAvailable_bytes

# Con agregación — promedio de memoria libre en todos los pods
avg(node_memory_MemAvailable_bytes) by (instance)
```

---

### Histogram (Histograma)

Mide la **distribución de valores**. Genera automáticamente **3 tipos de series**:

```
Una métrica Histogram llamada  http_request_duration_seconds
genera automáticamente estas series en Prometheus:

┌─────────────────────────────────────────────────────────────────┐
│  http_request_duration_seconds_bucket{le="0.05"}  = 240        │ ← cuántas peticiones
│  http_request_duration_seconds_bucket{le="0.1"}   = 412        │   tardaron MENOS
│  http_request_duration_seconds_bucket{le="0.25"}  = 580        │   que ese umbral (le)
│  http_request_duration_seconds_bucket{le="0.5"}   = 631        │
│  http_request_duration_seconds_bucket{le="1.0"}   = 650        │
│  http_request_duration_seconds_bucket{le="+Inf"}  = 650        │ ← total (siempre)
├─────────────────────────────────────────────────────────────────┤
│  http_request_duration_seconds_count              = 650         │ ← total observaciones
│  http_request_duration_seconds_sum                = 97.3        │ ← suma de todos los valores
└─────────────────────────────────────────────────────────────────┘
```

> **Tu métrica** `amiga_java_cache_getMulti_timer_seconds_sum` es la parte `_sum` de un Histogram. Acumula el **tiempo total en segundos** de todas las llamadas a caché desde que arrancó la app.

```promql
# Percentil 95 de latencia — "el 95% de las peticiones tardó menos que X"
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

### Summary (Resumen)

Similar al Histogram pero los percentiles los calcula la propia app (no Prometheus). Menos flexible: **no se pueden agregar entre instancias**.

```
Summary genera:
  metric_count   = 650
  metric_sum     = 97.3
  metric{quantile="0.5"}  = 0.12   ← percentil 50 calculado en la app
  metric{quantile="0.9"}  = 0.31   ← percentil 90 calculado en la app
  metric{quantile="0.99"} = 0.82   ← percentil 99 calculado en la app
```

---

## 3. Anatomía de una métrica

```
amiga_java_cache_getMulti_timer_seconds_sum { cache_name="users", hostname="pod-1", platform="web" }
│──────────────────────────────────────────│   │─────────────────────────────────────────────────│
              NOMBRE DE LA MÉTRICA                              LABELS (etiquetas)
│────│ │──────────────────────────│ │─────│ │──│
  app        subsistema            unidad  tipo

  amiga              → nombre de la aplicación
  java_cache_getMulti → subsistema: caché Java, operación getMulti
  timer_seconds       → unidad: tiempo medido en segundos
  sum                 → parte del histogram: suma acumulada de tiempos
```

### Los labels son dimensiones

Cada combinación única de labels es una **serie diferente**:

```
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="users",   hostname="pod-1"} = 4823.7s
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="users",   hostname="pod-2"} = 4711.2s
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="users",   hostname="pod-3"} = 4950.1s
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="products",hostname="pod-1"} =  991.2s
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="products",hostname="pod-2"} =  887.5s
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="products",hostname="pod-3"} = 1023.8s
amiga_java_cache_getMulti_timer_seconds_sum{cache_name="sessions",hostname="pod-1"} = 2341.6s
...
```

Cada fila = una serie temporal = una posible línea en el gráfico.

---

## 4. Selectores de labels y filtros

Los labels van entre llaves `{}` después del nombre de la métrica para **filtrar** qué series quieres.

### Operadores de filtro

| Operador | Significado | Ejemplo | Resultado |
|----------|-------------|---------|-----------|
| `=` | Igual exacto | `env="production"` | Solo prod |
| `!=` | Diferente | `env!="development"` | Todo menos dev |
| `=~` | Coincide con **regex** | `env=~"prod.*"` | prod, production, prod-eu... |
| `!~` | **No** coincide con regex | `env!~"dev\|test"` | Todo menos dev y test |

### ¿Por qué se usa `=~` con las variables de Grafana?

```
Cuando el usuario elige en el desplegable...

  "All"         →  Grafana pone  $platform = ".*"   (regex que coincide con todo)
  "web"         →  Grafana pone  $platform = "web"
  "web, mobile" →  Grafana pone  $platform = "web|mobile"

Con  =   solo funcionaría con valores exactos → ".*" nunca coincidiría con nada
Con  =~  funciona con regex → ".*" coincide con cualquier valor ✅
```

### Ilustración del filtrado

```
Todas las series disponibles en Prometheus:
  {cache_name="users",   hostname="pod-1", environment="prod",  ...} = 48230s
  {cache_name="users",   hostname="pod-2", environment="prod",  ...} = 47110s
  {cache_name="users",   hostname="pod-3", environment="prod",  ...} = 49500s
  {cache_name="users",   hostname="pod-4", environment="stage", ...} =   820s  ← excluida
  {cache_name="products",hostname="pod-1", environment="prod",  ...} =  9910s
  {cache_name="products",hostname="pod-2", environment="prod",  ...} =  8870s
  {cache_name="products",hostname="pod-3", environment="prod",  ...} = 10230s

Aplicando  {environment=~"$environment"}  donde $environment = "prod":

  {cache_name="users",   hostname="pod-1", environment="prod"} = 48230s  ✅
  {cache_name="users",   hostname="pod-2", environment="prod"} = 47110s  ✅
  {cache_name="products",hostname="pod-1", environment="prod"} =  9910s  ✅
```

### Los labels de tu query explicados

```promql
{
  platform=~"$platform",       # web / mobile / all
  tenant=~"$tenant",           # acme / beta / all  (sistema multi-cliente)
  environment=~"$environment", # prod / staging / dev
  slot=~"$slot",               # blue / green  (despliegue blue-green)
  hostname=~"$Pod",            # pod-1 / pod-2 / all  (pods de Kubernetes)
  projectkey=~"$jirakey"       # PROJECT-A / PROJECT-B  (proyecto Jira)
}
```

---

## 5. Variables de Grafana

Las variables hacen los dashboards **interactivos** — el usuario elige en desplegables y las queries se actualizan solas.

### Cómo se ve en Grafana

```
┌──────────────────────────────────────────────────────────────────────┐
│  Dashboard: Cache Metrics                                            │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ ┌──────┐ ┌──────────┐  │
│  │ Platform ▼│ │ Tenant  ▼│ │ Environment ▼│ │ Pod ▼│ │ Jirakey ▼│  │
│  │   web     │ │  acme   │ │    prod      │ │ All  │ │  PRJ-A   │  │
│  └──────────┘ └──────────┘ └─────────────┘ └──────┘ └──────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ 📈 Cache getMulti Time (avg por pod)                           │ │
│  │  12s ┤     ╭──╮  users                                        │ │
│  │   8s ┤╭────╯  ╰──╮  ──── users                               │ │
│  │   4s ┤│           ╰────  ···· products                        │ │
│  │   0s └─────────────────────────────────────────────           │ │
│  │       10:00    10:30    11:00    11:30                         │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

Cuando cambias "Platform" de `web` a `mobile`, Grafana sustituye `$platform` por `mobile` en la query y el gráfico se recarga.

### Variables de usuario (desplegables)

Se definen en **Dashboard Settings → Variables**:

| Tipo | Para qué sirve | Ejemplo |
|------|----------------|---------|
| `Query` | Los valores vienen de una query PromQL | Lista de pods activos |
| `Custom` | Lista fija que tú defines | `prod,staging,dev` |
| `Interval` | Intervalos de tiempo | `1m, 5m, 1h` |
| `Datasource` | Elegir fuente de datos | Prometheus A / B |
| `Constant` | Valor fijo, no cambia | URL de un servicio |
| `Text box` | El usuario escribe libre | Un ID concreto |

**Ejemplo: variable `$Pod` que lista los pods activos en el entorno seleccionado:**
```promql
# Query de la variable — obtiene los valores posibles del desplegable:
label_values(
  amiga_java_cache_getMulti_timer_seconds_sum{environment=~"$environment"},
  hostname
)
# Resultado del desplegable: pod-1, pod-2, pod-3, pod-4 (solo los de prod si environment=prod)
```

### Variables especiales del sistema (automáticas de Grafana)

No las defines tú — Grafana las calcula según el zoom del dashboard:

| Variable | Qué contiene | Ejemplo |
|----------|-------------|---------|
| `$__interval` | Intervalo entre puntos según el zoom | `15s`, `1m`, `5m` |
| `$__rate_interval` | Como `$__interval` + mínimo 4 scrapes | `1m` |
| `$__range` | Rango total visible en el dashboard | `6h`, `1d` |
| `$__from` | Timestamp inicio del rango (ms) | `1711234567000` |
| `$__to` | Timestamp fin del rango (ms) | `1711256167000` |

### Cómo cambia `$__interval` según el zoom

```
┌─────────────────────────────────────────────────────────────┐
│  Rango visible  │  Resolución  │  $__interval               │
├─────────────────┼──────────────┼────────────────────────────┤
│  Últimos 15min  │  alta        │  ~10s (muchos puntos)      │
│  Últimas 6h     │  media       │  ~30s                      │
│  Últimas 24h    │  baja        │  ~2m                       │
│  Últimos 7d     │  muy baja    │  ~15m (pocos puntos)       │
└─────────────────┴──────────────┴────────────────────────────┘

Al hacer zoom IN (ver menos tiempo) → $__interval baja → más resolución
Al hacer zoom OUT (ver más tiempo)  → $__interval sube → menos resolución
```

> **`$__interval` vs `$__rate_interval`**
> `$__rate_interval` garantiza al menos 4 intervalos de scrape de margen.
> **Regla práctica:** usa `$__rate_interval` con `rate()` y `$__interval` con `increase()`.

---

## 6. Rangos de tiempo (Range Vectors)

### Instant Vector vs Range Vector

```
INSTANT VECTOR — un valor por serie en este momento exacto
────────────────────────────────────────────────────────────
 http_requests_total{pod="pod-1"}
                                        │ ahora
 ────────────────────────────────────●──┼── tiempo
                                    1523│

 Resultado: {pod="pod-1"} → 1523   (un número)


RANGE VECTOR — todos los valores en una ventana de tiempo
────────────────────────────────────────────────────────────
 http_requests_total{pod="pod-1"}[5m]
                         │←── 5m ──│ ahora
 ──────────────────●──●──●──●──●──●──┼── tiempo
                  1200 1280 1350 1420 1480 1523│

 Resultado: {pod="pod-1"} → [1200, 1280, 1350, 1420, 1480, 1523]
            (una lista de puntos — solo útil dentro de rate/increase)
```

**Un Range Vector no se puede graficar directamente.** Solo sirve como argumento de `rate()`, `increase()`, etc.

### Sintaxis de duración

```promql
[30s]          # últimos 30 segundos
[5m]           # últimos 5 minutos
[1h]           # última hora
[2d]           # últimos 2 días
[$__interval]  # intervalo dinámico de Grafana (recomendado)
```

### Por qué necesitamos el rango — ilustración

```
Counter http_requests_total a lo largo del tiempo:

  t=10:00  →  1200  (total acumulado)
  t=10:01  →  1280
  t=10:02  →  1350
  t=10:03  →  1420
  t=10:04  →  1480
  t=10:05  →  1523  (ahora)

increase(...[5m]) = 1523 - 1200 = 323 peticiones en 5 minutos
rate(...[5m])     = 323 / 300s  = 1.07 peticiones/segundo
```

---

## 7. Funciones sobre counters: rate, increase, irate

Estas 3 funciones **solo funcionan con counters** (series que solo suben). Todas necesitan un range vector `[rango]`.

### `rate()` — Tasa por segundo (suavizado)

Calcula la **tasa de cambio por segundo** promediada sobre el rango.

```promql
rate(http_requests_total[5m])
```

```
Cómo funciona internamente:
  Toma el rango de 5 minutos (300 segundos)
  Calcula: (valor_ahora - valor_hace_5min) / 300 segundos
  Resultado: peticiones por segundo

  Si en 5 min pasó de 1200 a 1523:
  rate = (1523 - 1200) / 300 = 1.07 req/s

Gráfico resultante (suavizado):
  2.5 │    ╭─────╮
  2.0 │   ╭╯     ╰╮
  1.5 │───╯        ╰────────
  1.0 │
    0 └──────────────────────▶ tiempo
```

- ✅ Para tendencias generales y alertas
- ✅ Maneja reinicios del proceso (si el counter baja a 0 lo detecta)
- ✅ Resultado en **unidades/segundo** (decimales OK: 0.5 req/s = 1 req cada 2s)

### `increase()` — Incremento total en el rango

Calcula el **total acumulado** en el periodo. Equivale a `rate() × duración_en_segundos`.

```promql
increase(http_requests_total[5m])
```

```
Cómo funciona internamente:
  Igual que rate() pero multiplica por la duración:
  increase = rate([5m]) × 300s = 1.07 × 300 = 321 peticiones

  → No da por segundo, da el TOTAL en esos 5 minutos

Gráfico resultante (igual forma que rate, escala diferente):
  750 │    ╭─────╮
  500 │   ╭╯     ╰╮
  300 │───╯        ╰────────
    0 └──────────────────────▶ tiempo
    (mismo pico pero en "peticiones en 5min", no "peticiones/s")
```

- ✅ Para ver totales: "¿cuántas peticiones hubo en este intervalo?"
- ✅ Es el que usa tu query de caché: cuántos segundos se gastaron en total

### `irate()` — Tasa instantánea (más reactivo)

Como `rate()` pero solo usa los **últimos 2 puntos** del rango. Mucho más reactivo.

```promql
irate(http_requests_total[5m])
```

```
rate()  — promedia todos los puntos del rango (suave):
  2.5 │    ╭─────╮
  2.0 │   ╭╯     ╰╮
  1.5 │───╯        ╰────────
      └──────────────────────▶

irate() — solo usa los últimos 2 puntos (nervioso):
  4.0 │    ╭╮ ╭╮
  2.0 │  ╭─╯╰─╯╰╮ ╭─╮
  1.0 │──╯        ╰─╯ ╰──────
      └──────────────────────▶
      (mismos datos, pero irate detecta los picos instantáneos)
```

- ✅ Para detectar picos muy breves
- ❌ No usar para tendencias o alertas (demasiado nervioso)

### Tabla comparativa

```
┌─────────────┬─────────────────────────────┬────────────────────────┐
│  Función    │  Resultado                  │  Cuándo usarla         │
├─────────────┼─────────────────────────────┼────────────────────────┤
│  rate()     │  unidades/segundo (decimal) │  Tendencias, alertas   │
│  increase() │  total en el rango          │  Totales, tu query     │
│  irate()    │  unidades/segundo (brusco)  │  Detectar picos        │
└─────────────┴─────────────────────────────┴────────────────────────┘

Ejemplo con los mismos datos (counter pasó de 1200 a 1523 en 5min):
  rate([5m])     → 1.07 req/s
  increase([5m]) → 323 peticiones
  irate([5m])    → depende de los últimos 2 puntos (puede ser 2.5 req/s)
```

---

## 8. Funciones de agregación: avg, sum, max, min...

Las funciones de agregación **combinan múltiples series en una sola** (o en grupos).

Se aplican **por fuera** de las funciones de rango:
```promql
avg( rate(metric[5m]) )   ✅ correcto — avg envuelve a rate
rate( avg(metric)[5m] )   ❌ incorrecto — no existe esa sintaxis
```

### `avg()` — Promedio entre series

```promql
avg(rate(http_requests_total[5m])) by (service)
```

```
Series de entrada (varias instancias del mismo servicio):
  {service="api", pod="pod-1"} → 2.1 req/s
  {service="api", pod="pod-2"} → 1.9 req/s
  {service="api", pod="pod-3"} → 2.3 req/s

avg() by (service):
  {service="api"} → (2.1 + 1.9 + 2.3) / 3 = 2.1 req/s  ← una sola línea

Gráfico antes del avg (3 líneas, una por pod):        Después del avg (1 línea):
  3.0 ┤  pod-3 ·····╭·····                              3.0 ┤
  2.5 ┤        ─────╯   ╰──                             2.5 ┤      ╭────
  2.0 ┤ pod-1 ──────────────   pod-2 ·····              2.0 ┤──────╯
  1.5 ┤                                                 1.5 ┤
      └──────────────────────▶                              └──────────▶
```

- ✅ Para ver el comportamiento "típico" de una instancia
- ⚠️ Sensible a outliers (un pod muy lento sube el promedio)

### `sum()` — Suma de todas las series

```promql
sum(rate(http_requests_total[5m])) by (service)
```

```
Series de entrada:
  {service="api", pod="pod-1"} → 2.1 req/s
  {service="api", pod="pod-2"} → 1.9 req/s
  {service="api", pod="pod-3"} → 2.3 req/s

sum() by (service):
  {service="api"} → 2.1 + 1.9 + 2.3 = 6.3 req/s  ← throughput total del servicio
```

- ✅ Para ver el **throughput total** del sistema (carga global)

### `max()` — Máximo entre series

```promql
max(rate(cpu_usage_seconds_total[5m])) by (service)
```

```
Series de entrada:
  {pod="pod-1"} → 0.45 (45% CPU)
  {pod="pod-2"} → 0.72 (72% CPU)  ← el pod más cargado
  {pod="pod-3"} → 0.38 (38% CPU)

max():
  → 0.72  (el pod más saturado — útil para alertas)
```

- ✅ Para detectar el **peor caso** — el pod más saturado, el tiempo más lento

### `min()` — Mínimo entre series

```promql
min(node_memory_MemAvailable_bytes) by (node)
```

```
  {node="node-1"} → 4GB libre
  {node="node-2"} → 512MB libre  ← el nodo en más riesgo
  {node="node-3"} → 2GB libre

min():
  → 512MB  (el nodo con menos memoria — el que puede fallar primero)
```

- ✅ Para encontrar el **eslabón más débil** del sistema

### `count()` — Contar cuántas series hay

```promql
count(up{job="my-app"} == 1)
```

```
  {pod="pod-1"} → 1  (arriba)
  {pod="pod-2"} → 1  (arriba)
  {pod="pod-3"} → 0  (caído)  ← no pasa el filtro == 1
  {pod="pod-4"} → 1  (arriba)

count(... == 1):
  → 3  (hay 3 pods activos de 4)
```

- ✅ Para contar instancias activas, alertar si bajan de N

### `sum_over_time()`, `avg_over_time()`, `max_over_time()` — Para Gauges en el tiempo

A diferencia de las anteriores (que agregan **entre series**), estas agregan **una serie a lo largo del tiempo**.

```promql
# Promedio de memoria a lo largo de la última hora (para un gauge)
avg_over_time(node_memory_MemAvailable_bytes[1h])
```

```
Una sola serie a lo largo del tiempo:
  t=10:00 → 4.1GB
  t=10:15 → 3.8GB
  t=10:30 → 4.5GB
  t=10:45 → 3.9GB
  t=11:00 → 4.2GB

avg_over_time([1h]) → (4.1+3.8+4.5+3.9+4.2)/5 = 4.1GB  ← promedio de la hora
```

> ⚠️ Estas funciones son **solo para Gauges**. Para Counters usa `rate()`/`increase()`.

### `histogram_quantile()` — Percentiles

Calcula el percentil N de un Histogram.

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

```
Cómo leer el resultado:
  histogram_quantile(0.95, ...) = 0.42s

  Significa: "el 95% de las peticiones tardó MENOS de 0.42 segundos"
             "solo el 5% tardó MÁS de 0.42 segundos"

Percentiles comunes:
  0.50 → mediana: la mitad tardó menos, la mitad más
  0.90 → p90: el 90% tardó menos
  0.95 → p95: el 95% tardó menos  (el más usado en SLOs)
  0.99 → p99: el 99% tardó menos  (detecta los outliers más lentos)
```

---

## 9. Agrupamiento: `by` y `without`

Por defecto, las agregaciones eliminan **todos** los labels y producen un solo número.
Con `by` y `without` controlas qué labels conservas en el resultado.

### `by` — Conservar solo estos labels (agrupar por ellos)

```
ANTES del avg (muchas series, muchos labels):
  {cache_name="users",   hostname="pod-1", environment="prod"} → 10s
  {cache_name="users",   hostname="pod-2", environment="prod"} → 12s
  {cache_name="users",   hostname="pod-3", environment="prod"} →  9s
  {cache_name="products",hostname="pod-1", environment="prod"} →  3s
  {cache_name="products",hostname="pod-2", environment="prod"} →  4s
  {cache_name="products",hostname="pod-3", environment="prod"} →  3s

avg() sin by — pierde TODOS los labels:
  → 6.8s   (un solo número, mezcla de todo — poco útil)

avg(...) by (cache_name) — conserva solo cache_name:
  {cache_name="users"}    → (10+12+9)/3  = 10.3s  ← una línea por caché
  {cache_name="products"} → (3+4+3)/3   =  3.3s  ← una línea por caché

avg(...) by (cache_name, hostname) — conserva ambos:
  {cache_name="users",   hostname="pod-1"} → 10s  ← una línea por pod×caché
  {cache_name="users",   hostname="pod-2"} → 12s
  ...  (más líneas, más detalle)
```

**En el gráfico:**
```
Sin by (1 línea, todo mezclado):         Con by (cache_name) (1 línea por caché):
  10s ┤─────────────────────             12s ┤  users ──────────────
   7s ┤                                   8s ┤     ──────────────
   0s └──────────────────────▶            3s ┤  products ············
                                          0s └──────────────────────▶
```

### `without` — Eliminar solo estos labels (conservar el resto)

```promql
# Elimina hostname y pod, conserva todo lo demás
sum(rate(http_requests_total[5m])) without (hostname, pod)
```

- Usa `by` cuando sabes **qué labels quieres** → más explícito, recomendado
- Usa `without` cuando tienes muchos labels y quieres quitar solo algunos

---

## 10. Desglose visual capa por capa de la query real

```promql
avg(increase(amiga_java_cache_getMulti_timer_seconds_sum{platform=~"$platform",tenant=~"$tenant",environment=~"$environment",slot=~"$slot",hostname=~"$Pod",projectkey=~"$jirakey"}[$__interval])) by (cache_name)
```

### Vista de capas (de dentro hacia afuera)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  CAPA 3: avg( ... ) by (cache_name)                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  CAPA 2: increase( ... [$__interval])                                 │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  CAPA 1: amiga_java_cache_getMulti_timer_seconds_sum{           │  │  │
│  │  │            platform=~"$platform",                               │  │  │
│  │  │            tenant=~"$tenant",                                   │  │  │
│  │  │            environment=~"$environment",                         │  │  │
│  │  │            slot=~"$slot",                                       │  │  │
│  │  │            hostname=~"$Pod",                                    │  │  │
│  │  │            projectkey=~"$jirakey"                               │  │  │
│  │  │          }                                                      │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Capa 1 — Seleccionar y filtrar

```promql
amiga_java_cache_getMulti_timer_seconds_sum{environment=~"prod", ...}
```

**Qué hace:** selecciona solo las series que coincidan con los filtros del dashboard.

```
Todas las series en Prometheus:
  {cache_name="users",   hostname="pod-1", environment="prod",  ...} = 48230s
  {cache_name="users",   hostname="pod-2", environment="prod",  ...} = 47110s
  {cache_name="users",   hostname="pod-3", environment="prod",  ...} = 49500s
  {cache_name="users",   hostname="pod-4", environment="stage", ...} =   820s  ← excluida
  {cache_name="products",hostname="pod-1", environment="prod",  ...} =  9910s
  {cache_name="products",hostname="pod-2", environment="prod",  ...} =  8870s
  {cache_name="products",hostname="pod-3", environment="prod",  ...} = 10230s

Resultado de la Capa 1 (solo las de prod):
  {cache_name="users",   hostname="pod-1"} = 48230s  (acumulado desde arranque)
  {cache_name="users",   hostname="pod-2"} = 47110s
  {cache_name="users",   hostname="pod-3"} = 49500s
  {cache_name="products",hostname="pod-1"} =  9910s
  {cache_name="products",hostname="pod-2"} =  8870s
  {cache_name="products",hostname="pod-3"} = 10230s

⚠️ Estos son números ENORMES que solo crecen — son contadores acumulados.
   No se pueden graficar así directamente (solo verías líneas que suben).
```

---

### Capa 2 — Calcular el incremento por intervalo

```promql
increase( amiga_java_cache_getMulti_timer_seconds_sum{...} [$__interval] )
```

**Qué hace:** para cada punto del gráfico, calcula cuánto creció el contador en ese intervalo.

Supongamos `$__interval = 1m` (el gráfico muestra las últimas 6h con resolución de 1 minuto):

```
Para {cache_name="users", hostname="pod-1"}:

  t=10:00  contador = 48000s
  t=10:01  contador = 48012s  → increase = 12s  (gastó 12s en caché en ese minuto)
  t=10:02  contador = 48023s  → increase = 11s
  t=10:03  contador = 48041s  → increase = 18s  ← pico de carga
  t=10:04  contador = 48053s  → increase = 12s
  t=10:05  contador = 48065s  → increase = 12s

Gráfico de un pod (seconds gastados en caché por minuto):
  18s ┤         ╭──╮
  12s ┤─────────╯  ╰─────────  ← tiempo gastado en caché "users" en pod-1
   0s └────────────────────────▶ tiempo
```

El resultado son **6 series** (3 pods × 2 cachés), cada una con un punto cada minuto:
```
  {cache_name="users",   hostname="pod-1"} → [12, 11, 18, 12, 12, ...]
  {cache_name="users",   hostname="pod-2"} → [10, 13, 16, 11, 14, ...]
  {cache_name="users",   hostname="pod-3"} → [13, 12, 19, 13, 11, ...]
  {cache_name="products",hostname="pod-1"} → [ 3,  4,  5,  3,  4, ...]
  {cache_name="products",hostname="pod-2"} → [ 4,  3,  4,  4,  3, ...]
  {cache_name="products",hostname="pod-3"} → [ 3,  4,  5,  3,  4, ...]
```

---

### Capa 3 — Promediar por caché (colapsar pods)

```promql
avg( increase(...) ) by (cache_name)
```

**Qué hace:** agrupa las 6 series por `cache_name` y calcula el promedio entre pods. Los labels `hostname`, `environment`, etc. desaparecen.

```
En t=10:03 (el minuto de pico):
  users   pod-1 = 18s ┐
  users   pod-2 = 16s ├─ avg → {cache_name="users"}    = (18+16+19)/3 = 17.7s
  users   pod-3 = 19s ┘
  products pod-1 =  5s ┐
  products pod-2 =  4s ├─ avg → {cache_name="products"} = (5+4+5)/3   =  4.7s
  products pod-3 =  5s ┘

Resultado final — 2 series (una por caché):
  {cache_name="users"}    → [11, 12, 17.7, 12, 12.3, ...]
  {cache_name="products"} → [ 3,  3,  4.7,  3,  3.7, ...]
```

**Gráfico final en Grafana (Time Series):**
```
  18s ┤                users ──────
  12s ┤──────────╭──────────────────────  ← users (promedio entre pods)
   6s ┤────────────────────────────────────  ← products (promedio entre pods)
   0s └─────────────────────────────────────▶ tiempo
      10:00      10:03     10:06

Leyenda:
  ─────  cache_name="users"      (más tiempo = caché más usada)
  ─────  cache_name="products"
```

### Resumen del flujo completo

```
Capa 1: Filtrar           Capa 2: increase()           Capa 3: avg() by (cache_name)
─────────────────         ────────────────────          ──────────────────────────────
6 series filtradas   →    6 series con incrementos  →  2 series finales
(3 pods × 2 cachés)       por intervalo                 (una por caché)

{users, pod-1} =48230s    {users, pod-1} = 12s/min ─┐
{users, pod-2} =47110s    {users, pod-2} = 10s/min  ├─avg→ {users}    = 11s/min ─── 📈
{users, pod-3} =49500s    {users, pod-3} = 13s/min ─┘
{products,pod-1}= 9910s   {products,pod-1}= 3s/min ─┐
{products,pod-2}= 8870s   {products,pod-2}= 4s/min  ├─avg→ {products} =  3s/min ─── 📈
{products,pod-3}=10230s   {products,pod-3}= 3s/min ─┘
```

---

## 11. Tipos de visualización en Grafana

### Time Series — Gráfico de líneas

El panel más usado. Muestra cómo evoluciona una métrica en el tiempo.

```
┌─────────────────────────────────────────────────────────────┐
│  Cache getMulti Time                              [6h] ▼    │
│                                                             │
│  18s ┤              ╭───╮  users                           │
│  12s ┤──────────────╯   ╰──────────────────  ════ users    │
│   4s ┤──────────────────────────────────────  ···· products│
│   0s └─────────────────────────────────────▶ tiempo        │
│       10:00   11:00   12:00   13:00   14:00                 │
└─────────────────────────────────────────────────────────────┘

Cada línea = una serie = un valor de cache_name
```

**Opciones clave:**
- **Lines / Bars / Points** — tipo de trazo
- **Fill opacity** — rellena el área bajo la curva
- **Stacking** — apila las series (útil para ver totales acumulados)
- **Thresholds** — líneas de color verde/amarillo/rojo

**Cuándo usarlo:** `rate()`, `increase()`, `avg()` sobre el tiempo. Tu query de caché va aquí.

---

### Stat — Un número grande

Muestra un solo valor resaltado. Para KPIs y métricas "ahora mismo".

```
┌─────────────────────────┐  ┌─────────────────────────┐
│  Pods activos           │  │  Errores última hora     │
│                         │  │                         │
│       ● 12              │  │        🔴 47             │
│                         │  │                         │
│  ↑ 2 respecto a ayer    │  │  ↑ 12 respecto a ayer   │
└─────────────────────────┘  └─────────────────────────┘
    (verde = bien)                (rojo = umbral superado)
```

**Opciones clave:**
- **Calculation:** Last (último valor), Mean, Max, Sum
- **Color mode:** fondo o texto de color según thresholds
- **Sparkline:** mini gráfico de fondo

```promql
# Query típica para un Stat: total de peticiones en la última hora
sum(increase(http_requests_total[1h]))
```

---

### Gauge — Medidor circular / barra

Para porcentajes y valores con rango conocido (0-100%).

```
┌───────────────────────────────┐
│  CPU Usage                    │
│                               │
│        ╭───────────╮          │
│       ╱  🟢  72%   ╲         │
│      │   ────────   │         │
│       ╲             ╱         │
│        ╰───────────╯          │
│      0%    50%    100%        │
└───────────────────────────────┘
  Verde < 70%  Amarillo < 90%  Rojo > 90%
```

```promql
# Porcentaje de uso de CPU
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

---

### Bar chart — Barras comparativas

Compara valores entre categorías en un momento dado.

```
┌───────────────────────────────────────────┐
│  Peticiones por servicio (ahora)          │
│                                           │
│  api      ████████████████████ 6.3 req/s  │
│  frontend ██████████████       4.1 req/s  │
│  worker   ████                 1.2 req/s  │
│  batch    ██                   0.7 req/s  │
│                                           │
└───────────────────────────────────────────┘
```

```promql
sum(rate(http_requests_total[5m])) by (service)
```

---

### Table — Tabla de datos

Muestra las series como filas con sus labels como columnas.

```
┌──────────────┬──────────┬────────────┬────────────┐
│  cache_name  │  hostname│  Last      │  Max       │
├──────────────┼──────────┼────────────┼────────────┤
│  users       │  pod-1   │  12.3s     │  18.7s     │
│  users       │  pod-2   │  10.1s     │  16.2s     │
│  users       │  pod-3   │  13.5s     │  19.1s     │
│  products    │  pod-1   │   3.2s     │   5.1s     │
│  products    │  pod-2   │   3.8s     │   4.9s     │
└──────────────┴──────────┴────────────┴────────────┘
```

**Cuándo usarlo:** debugging, ver todos los labels, combinar varias métricas en columnas.

---

### Heatmap — Mapa de calor

Para visualizar distribuciones de histogramas a lo largo del tiempo.

```
┌───────────────────────────────────────────────────┐
│  Latencia HTTP — Distribución                     │
│                                                   │
│  >1s   │░░░░░░░░░░░░▓▓░░░░░░░░░░░░░░░░░░░         │
│  500ms │░░░░░░▓▓▓▓▓▓▓▓▓▓▓░░░░░░░▓░░░░░░░         │
│  100ms │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓         │
│  50ms  │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓         │
│        └──────────────────────────────────▶       │
│         ░ pocas peticiones  ▓ muchas peticiones    │
└───────────────────────────────────────────────────┘
La mayoría tarda 50-100ms pero hay picos ocasionales >500ms
```

```promql
rate(http_request_duration_seconds_bucket[5m])
```

---

### Resumen: ¿qué panel usar?

```
┌─────────────────────────────────────────────────────────────────┐
│  ¿Qué quiero ver?                    →  Panel recomendado       │
├─────────────────────────────────────────────────────────────────┤
│  Evolución en el tiempo               →  Time Series            │
│  Un número importante ahora mismo     →  Stat                   │
│  % dentro de un rango (CPU, mem)      →  Gauge                  │
│  Comparar categorías (ranking)        →  Bar chart              │
│  Ver todos los labels / debugging     →  Table                  │
│  Distribución de latencias            →  Heatmap                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. Opciones comunes del panel

### Legend (Leyenda) — Nombrar las líneas

Por defecto Grafana muestra todos los labels. Con el formato `{{label}}` lo personalizas:

```
Query devuelve: {cache_name="users", environment="prod", hostname="pod-1"}

Sin formato:    {cache_name="users", environment="prod", hostname="pod-1"}  ← largo y feo
Con {{cache_name}}:  users                                                  ← limpio ✅
Con {{cache_name}} - {{hostname}}:  users - pod-1                           ← más detalle
```

Configuración en **Panel → Legend → Legend mode: Table** añade columnas con estadísticas:
```
┌──────────────┬─────────┬─────────┬─────────┐
│  Name        │  Last   │  Mean   │  Max    │
├──────────────┼─────────┼─────────┼─────────┤
│  users       │  12.1s  │  11.8s  │  18.7s  │
│  products    │   3.2s  │   3.5s  │   5.1s  │
└──────────────┴─────────┴─────────┴─────────┘
```

---

### Axes (Ejes) — Unidades y escala

La opción **Unit** es muy importante — Grafana formatea automáticamente:

```
Sin unidad:    12345678   ← número crudo, sin contexto
Unit = bytes:  11.77 MiB  ← formateado automáticamente ✅
Unit = seconds: 3.2 s     ← formateado ✅
Unit = req/s:   1.07 req/s ✅

Unidades útiles en Grafana:
  seconds              → tiempo de respuesta
  bytes                → tráfico, memoria
  percent (0-100)      → uso de CPU, disco
  requests/sec         → throughput HTTP
  short (1K = 1000)    → contadores genéricos
```

---

### Thresholds (Umbrales) — Colores de alerta

```
Panel → Thresholds:
  Base  → verde  (todo bien)
  70    → amarillo (Warning)
  90    → rojo    (Critical)

En el gráfico:
  100 ┤              ╭──╮
   90 ┤- - - - - - - ┤  ├ - -  ← línea roja (Critical)
   72 ┤          ╭───╯  ╰──────
   70 ┤- - - - - ┤  - - - - -  ← línea amarilla (Warning)
   50 ┤──────────╯
    0 └──────────────────────▶
```

---

### Transformations — Manipular datos después de la query

Se aplican en Grafana **después** de recibir los datos de Prometheus. No son PromQL.

```
┌───────────────────┬────────────────────────────────────────────────┐
│  Transformación   │  Para qué sirve                                │
├───────────────────┼────────────────────────────────────────────────┤
│  Reduce           │  Colapsa serie → un solo valor (para Stat)     │
│  Filter by name   │  Ocultar series o columnas                     │
│  Rename by regex  │  Renombrar series con regex                    │
│  Merge            │  Unir varias queries en una tabla              │
│  Calculate field  │  Añadir columna calculada (A / B * 100)        │
│  Sort by          │  Ordenar filas de una tabla                    │
└───────────────────┴────────────────────────────────────────────────┘

Ejemplo: mostrar error rate en tabla con Merge + Calculate field
  Query A: sum(rate(errors_total[5m]))   by (service)  → columna "errors"
  Query B: sum(rate(requests_total[5m])) by (service)  → columna "requests"
  Merge → une por service
  Calculate field → errors / requests * 100 → columna "error_rate %"

  ┌──────────────┬─────────┬──────────┬──────────────┐
  │  service     │ errors  │ requests │ error_rate % │
  ├──────────────┼─────────┼──────────┼──────────────┤
  │  api         │  0.12   │  6.30    │  1.9%        │
  │  frontend    │  0.05   │  4.10    │  1.2%        │
  └──────────────┴─────────┴──────────┴──────────────┘
```

---

### Query Options

```
Min interval:     15s   ← nunca calcular con $__interval < scrape interval
Max data points:  300   ← límite de puntos por serie (afecta resolución)
Relative time:    1h    ← este panel siempre muestra 1h, sin importar el zoom global
```

---

## 13. Cheatsheet rápido

### Estructura de una query

```promql
AGREGACIÓN( FUNCIÓN_RANGO( MÉTRICA{FILTROS}[RANGO] ) ) by (LABEL)
   avg         increase        _sum     env="prod"  [$__interval]    cache_name
```

### Funciones sobre counters

```promql
rate(metric[5m])          # tasa/segundo, suavizado → tendencias
irate(metric[5m])         # tasa/segundo, brusco    → picos
increase(metric[5m])      # incremento total en 5min → totales
```

### Funciones de agregación

```promql
avg(metric) by (label)    # promedio  → comportamiento típico
sum(metric) by (label)    # suma      → throughput total
max(metric) by (label)    # máximo    → peor caso, saturación
min(metric) by (label)    # mínimo    → eslabón más débil
count(metric) by (label)  # contar    → instancias activas
```

### Filtros de labels

```promql
{label="valor"}           # exacto
{label!="valor"}          # diferente
{label=~"regex"}          # coincide con regex (obligatorio con variables Grafana)
{label!~"regex"}          # no coincide con regex
{label=~"val1|val2"}      # OR entre valores
{label=~".*"}             # cualquier valor (cuando se selecciona "All")
```

### Variables especiales de Grafana

```promql
[$__interval]             # intervalo dinámico según zoom → usar con increase()
[$__rate_interval]        # como $__interval + mínimo seguro → usar con rate()
[$__range]                # rango total visible en el dashboard
```

### Percentiles (requiere histogram `_bucket`)

```promql
histogram_quantile(0.50, rate(metric_bucket[5m]))   # mediana
histogram_quantile(0.95, rate(metric_bucket[5m]))   # p95 (el más usado en SLOs)
histogram_quantile(0.99, rate(metric_bucket[5m]))   # p99 (outliers)
```

### Operaciones aritméticas

```promql
metric_a / metric_b * 100                # porcentaje
rate(errors[5m]) / rate(requests[5m])    # error rate
```

### Patrones comunes

```promql
# Error rate %
rate(errors_total[5m]) / rate(requests_total[5m]) * 100

# Latencia media (de histogram)
rate(request_duration_seconds_sum[5m]) / rate(request_duration_seconds_count[5m])

# Disponibilidad
avg_over_time(up{job="my-app"}[1h]) * 100

# Top 5 pods por carga
topk(5, rate(cpu_seconds_total[5m]))

# Pods caídos
count(up{job="my-app"} == 0)
```

---

## Deconstrucción final: tu query en una frase

```promql
avg(increase(amiga_java_cache_getMulti_timer_seconds_sum{...filtros...}[$__interval])) by (cache_name)
```

> **"Para cada intervalo de tiempo del gráfico y cada caché diferente, muéstrame el tiempo promedio (entre todos los pods) que se ha gastado en llamadas getMulti a esa caché — filtrando solo los pods, entornos y tenants que el usuario tiene seleccionados."**

```
Resultado en el gráfico:
  Una línea por cada cache_name diferente.

  Si "users" tiene la línea más alta:
    → la caché de usuarios recibe más carga O cada llamada es más lenta
    → puede indicar un problema de rendimiento en esa caché concreta

  Si una línea sube de repente:
    → pico de uso de esa caché en ese momento
    → puede correlacionarse con un despliegue, un evento de tráfico, etc.
```
