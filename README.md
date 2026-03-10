# Fleet Command — Trailer Logistics Pro

Calendario de planificación de conductores con vistas mensual, semanal, diaria y timeline (Gantt). Conectado a Supabase en tiempo real. Desplegado automáticamente en Vercel desde este repositorio.

**URL producción:** https://planificacion-choferes.vercel.app/

---

## Stack

| Capa | Tecnología |
|------|-----------|
| Frontend | HTML + CSS + JS vanilla (un solo archivo `index.html`) |
| Backend / DB | Supabase (PostgreSQL + PostgREST REST API) |
| Deploy | Vercel (auto-deploy desde rama `main`) |
| Auth | Supabase `anon` key (solo lectura pública) |

No hay bundler, no hay framework, no hay dependencias npm en runtime. Todo vive en `index.html`.

---

## Estructura del proyecto

```
index.html          ← toda la app (HTML + CSS + JS, ~1900 líneas)
README.md           ← este archivo
```

---

## Vistas disponibles

| Vista | Descripción |
|-------|-------------|
| **Mes** | Calendario mensual, clic en día para crear evento |
| **Semana** | Grid de 7 columnas con eventos que se expanden horizontalmente (event spanning) |
| **Día** | Vista de 24 horas con bloques posicionados por hora |
| **Timeline** | Gantt: conductores en el eje Y, días del mes en el eje X, barras de colores por estado |

---

## Fuentes de datos (Supabase)

### Tablas / Vistas utilizadas

```
b_conductores          → conductores activos (estado = 1)
b_vehiculos            → vehículos activos (estado = 1)
v_viajes_inteligentes  → vista de viajes con datos enriquecidos
```

### Columnas de `v_viajes_inteligentes` usadas

```
viaje_id, nro_viaje, cliente_estandar, tipo_operacion,
patente_tracto, patente_rampla,
conductor_principal, rut_conductor_uno,
conductor_dos, rut_conductor_dos,
estado_viaje_estandar,
fecha_entrada_origen, fecha_salida_origen,
fecha_entrada_destino, fecha_salida_destino,
hora_salida_origen, hora_entrada_destino
```

### Permisos requeridos en Supabase

`v_viajes_inteligentes` es una **vista**, no una tabla, por lo que RLS no aplica directamente. Se configuró con:

```sql
-- En SQL Editor de Supabase:
GRANT SELECT ON b_conductores TO anon;
GRANT SELECT ON b_vehiculos TO anon;
GRANT SELECT ON v_viajes_inteligentes TO anon;
```

Para `b_conductores` y `b_vehiculos` también se creó una política RLS:
- Tipo: PERMISSIVE
- Comando: SELECT
- `using(true)` — permite lectura pública anónima

---

## Configuración de Supabase en el código

```javascript
const SB_URL = 'https://cfcgkeexjewhijajltwg.supabase.co';
const SB_KEY = 'sb_publishable_ABThEK5aihgvRmxe1wLMwA_ZbxBKbcF'; // anon/publishable key
const SB_H   = { 'apikey': SB_KEY, 'Authorization': 'Bearer ' + SB_KEY };

async function sbGet(table, params = '') {
  const r = await fetch(`${SB_URL}/rest/v1/${table}?${params}`, { headers: SB_H });
  if (!r.ok) throw new Error(`Supabase ${table}: ${r.status}`);
  return r.json();
}
```

> **Nota:** Se intentó usar `supabase-js` v2 vía CDN pero no existe build UMD para v2. Se reemplazó completamente por `fetch()` nativo contra la REST API de PostgREST.

---

## Carga de datos (`loadSupabase`)

```javascript
async function loadSupabase() {
  // 1. Conductores activos → llena array global C[]
  // 2. Vehículos activos   → llena arrays globales T[] (tractos) y R[] (ramplas)
  // 3. Viajes del mes actual ± 1 mes, paginados en bloques de 1000
  //    (Supabase/PostgREST tiene límite de 1000 filas por request)
}
```

### Paginación de viajes

Supabase devuelve máximo 1000 filas por request. Con rangos grandes (4+ meses) hay más de 1000 viajes y muchos quedan fuera. La solución es paginar:

```javascript
let viajes = [], offset = 0, chunk;
do {
  chunk = await sbGet('v_viajes_inteligentes',
    `select=${cols}&fecha_salida_origen=gte.${from}&...&order=fecha_salida_origen.asc&limit=1000&offset=${offset}`
  );
  if (Array.isArray(chunk)) viajes.push(...chunk);
  offset += 1000;
} while (Array.isArray(chunk) && chunk.length === 1000);
```

### Matching conductor ↔ viaje

Los conductores se identifican por RUT. La unión es:
```javascript
const cond = C.find(c => c.rut === v.rut_conductor_uno);
```
Si los RUTs no coinciden exactamente (espacios, guiones), el conductor no se asocia y el viaje no aparece en el Timeline.

### Fechas

Los campos de fecha en Supabase son timestamps completos (`2026-03-06T08:00:00`). Se extrae solo la parte de fecha:
```javascript
const dateStart = (v.fecha_salida_origen || '').substring(0, 10); // → "2026-03-06"
const dateEnd   = (rawEnd || '').substring(0, 10) || dateStart;
```
Sin esto, al hacer `dateStart + 'T00:00:00'` en los cálculos de barras del Timeline se producía una fecha inválida y las barras no aparecían.

---

## Mapeo de estados

`estado_viaje_estandar` en Supabase puede tener estos valores reales:

| Valor en DB | Color en app | Clase CSS |
|-------------|-------------|-----------|
| `Planificado` | Morado `#7c3aed` | `vacaciones` |
| `Asignado` | Rojo `#dc2626` | `licencia` |
| `En Ruta` | Dorado `#d4911a` | `ambos` |
| `Pendiente de Destino` | Azul `#2563eb` | `descarga` |
| `Finalizado` | Verde `#10b981` | `carga` |

> El campo `tipo_operacion` tiene valores de tipo de trailer (Cortina, Burgers, Furgón, etc.), **no** tipos de operación logística. No se usa para colorear.

---

## Persistencia de eventos manuales

Los viajes de Supabase son de solo lectura. Los eventos creados manualmente desde la app se guardan en `localStorage`:

```javascript
function lsSave() { /* guarda solo eventos con _manual: true */ }
function lsLoad() { /* recupera al iniciar */ }
```

Al intentar eliminar un viaje de Supabase, la app muestra un toast: *"Los viajes de Supabase no se pueden eliminar aquí"*.

---

## Vista Timeline — detalles técnicos

- **Eje Y:** Lista de conductores activos (de `b_conductores`)
- **Eje X:** Días del mes actual
- **Barras:** Posicionadas con `left` y `width` en porcentaje:
  ```javascript
  const startDay  = Math.floor((clampS - mS) / 864e5);
  const spanDays  = Math.max(1, Math.floor((clampE - clampS) / 864e5) + 1);
  const leftPct   = (startDay / dim) * 100;
  const widthPct  = (spanDays / dim) * 100;
  ```
- **Conductor disponible:** Fila marcada en verde si no tiene ningún viaje en el mes
- **Tooltip:** Al hacer hover muestra conductor, estado, fechas, patente, cliente

---

## Vista Semana — event spanning

Los eventos multi-día se despliegan horizontalmente usando CSS Grid:

```javascript
ev.colStart = Math.max(0, sOff) + 1;   // columna inicio (1-7)
ev.colEnd   = Math.min(6, eOff) + 2;   // columna fin (exclusiva)
```

```html
<div style="grid-column: 2 / 5; grid-row: 1;">...</div>
```

Se asignan "lanes" (filas) para que los eventos solapados no se encimen.

---

## Deploy

El proyecto usa Vercel con auto-deploy desde GitHub:

```
git add index.html
git commit -m "descripción"
git push origin main
# → Vercel detecta el push y redespliega automáticamente (~60 segundos)
```

**Repositorio:** https://github.com/Trailer-Logistics/planificacion-choferes

---

## Historial de cambios importantes

| Commit | Cambio |
|--------|--------|
| Inicial | Calendario base con Firebase + datos demo hardcodeados |
| `feat: week spanning grid + timeline/gantt view` | Event spanning en semana, vista Timeline, primer intento con Supabase CDN |
| `d4819ad` | Eliminado Firebase y CDN de supabase-js (no existe build UMD en v2). Reemplazado por `fetch()` nativo contra PostgREST |
| `7cf812c` | Fix: `.substring(0,10)` en fechas para evitar `'2026-03-06T08:00:00T00:00:00'` (fecha inválida) |
| `a0a42aa` | Debug: console logs para trazar por qué viaje 7073 no aparecía |
| `b7b5e1e` | Fix: paginación de viajes para superar límite de 1000 filas de Supabase |

---

## Problemas conocidos resueltos

**1. Datos demo reaparecen al recargar**
- Causa: Firebase seed() recreaba datos al detectar DB vacía. El listener de Firebase los restauraba al eliminar.
- Solución: Firebase eliminado completamente. Supabase + localStorage.

**2. Error 403 / permission denied en Supabase**
- Causa: RLS habilitado sin política para el rol `anon`.
- Solución: Política SELECT con `using(true)` en tablas + `GRANT SELECT` en la vista.

**3. CDN 404 — supabase-js v2**
- Causa: `@supabase/supabase-js@2` no tiene distribución UMD. El CDN de jsdelivr devuelve 404.
- Solución: Usar `fetch()` nativo contra la REST API de PostgREST directamente.

**4. Barras del Timeline no aparecen (NaN)**
- Causa: Supabase devuelve timestamps (`2026-03-06T08:00:00`). Al hacer `+' T00:00:00'` → fecha inválida → `NaN`.
- Solución: `.substring(0,10)` antes de cualquier cálculo de fechas.

**5. Viaje 7073 no aparece en Timeline**
- Causa: Supabase limita a 1000 filas por request. Con rango Feb-Mayo hay >1000 viajes; 7073 quedaba fuera del corte.
- Solución: Paginación con `limit=1000&offset=N` + `order=fecha_salida_origen.asc`.

**6. Todos los eventos se ven del mismo color**
- Causa: `tipo_operacion` contiene tipos de trailer, no categorías logísticas.
- Solución: Usar `estadoToOp(estado_viaje_estandar)` para asignar el color.
