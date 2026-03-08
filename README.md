# Fleet Command — Planificación Choferes
**Trailer Logistics Pro**

Aplicación web de una sola página (`index.html`) para gestionar la planificación diaria de conductores y vehículos de la flota Trailer Logistics.

---

## Arquitectura

- **Tecnología:** HTML + CSS + JavaScript vanilla. Sin frameworks, sin backend, sin dependencias externas (solo Google Fonts).
- **Persistencia:** Los datos viven en memoria (objeto `evs`). No hay base de datos ni localStorage; al recargar se ejecuta `seed()` con datos de demo.
- **Archivo único:** Todo el código (HTML, CSS, JS) está en `index.html` (~1540 líneas).

---

## Datos del dominio

| Variable | Descripción |
|----------|-------------|
| `C[]` | Conductores: `{id, n (nombre), a (apellido), rut, lic (vencimiento licencia)}` |
| `T[]` | Tractos (camiones): `{id, p (patente), m (marca), mo (modelo)}` |
| `R[]` | Ramplas (semi-remolques): `{id, p (patente), m (marca), mo (modelo)}` |
| `CLIENTES[]` | 40 clientes (Walmart, Falabella, Sodimac, Jumbo, etc.) |
| `evs{}` | Estado global de asignaciones: `{ "YYYY-MM-DD": [{id, cId, tId, rId, cli, op, hi, hf, obs}] }` |

### Tipos de operación (`op`)
| Código | Label | Color |
|--------|-------|-------|
| `carga` | Carga | Verde |
| `descarga` | Descarga | Azul |
| `ambos` | Carga + Descarga | Dorado |
| `mantencion` | Mantención | Naranja |
| `vacaciones` | Vacaciones | Púrpura |
| `licencia` | Licencia médica | Rojo |

---

## Vistas del calendario

| Vista | Función |
|-------|---------|
| Mes | `renderMonth()` — grilla mensual con chips de asignaciones |
| Semana | `renderWeek()` — columnas lunes a domingo |
| Día | `renderDay()` — franjas horarias del día |

---

## Funciones clave (JS)

| Función | Descripción |
|---------|-------------|
| `render()` | Re-renderiza la vista activa |
| `renderSidebar()` | Lista conductores (`sTab='c'`) o vehículos (`sTab='v'`) |
| `updateKPIs()` | Actualiza: libres, asignados, hoy, licencias próximas (<90 días) |
| `openCreate(key)` | Abre modal para crear asignación en fecha `YYYY-MM-DD` |
| `saveEv()` | Guarda asignación validando conflictos de conductor/tracto/rampla |
| `showDet(key, evId)` | Abre modal de detalle/edición |
| `delEv()` | Elimina asignación seleccionada |
| `busyC/busyT/busyR(id, key)` | Verifica si conductor/tracto/rampla está ocupado en esa fecha |
| `seed()` | Carga asignaciones de demo para el mes actual |
| `dk(date)` | Date → `"YYYY-MM-DD"` |
| `pk(key)` | `"YYYY-MM-DD"` → Date |

---

## Flota actual

- **39 conductores** (Walter Jimenez, Javier Valdes, Rodrigo Romero, Marcos Riquelme, Claudio Riveros, etc.)
- **13 tractos:** RENAULT T480/T520/C430/D-Wide, SCANIA R500, IVECO NEW ACTROS/SWAY
- **9 ramplas:** LECITRAILER, KRONE PROFI LINER, BURGERS DOUBLEDECK/FRIGORIFICO, SDC CURTAINSIDER

---

## Cómo modificar

### Agregar conductor
```js
// En C[]:
{id: <único>, n: "Nombre", a: "Apellido", rut: "XX.XXX.XXX-X", lic: "YYYY-MM-DD"}
```

### Agregar tracto o rampla
```js
// En T[] o R[]:
{id: <único>, p: "PATENTE", m: "MARCA", mo: "MODELO"}
```

### Persistencia real (localStorage)
```js
// Guardar: localStorage.setItem('evs', JSON.stringify(evs));
// Cargar:  evs = JSON.parse(localStorage.getItem('evs') || '{}');
```

---

## Repositorios relacionados

| Repo | URL |
|------|-----|
| Este (Trailer Logistics) | https://github.com/Trailer-Logistics/planificacion-choferes |
| Copia 203104 | https://github.com/203104/planificacion-choferes |
| Ticketera | https://github.com/Trailer-Logistics/Ticketera_TrailerLogistics |
