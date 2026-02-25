# DHL Menu Semanal — Especificaciones Tecnicas

> Documento actualizado con el estado real del proyecto.
> Usar como brief para continuar el desarrollo con cualquier IA o desarrollador.

---

## 1. Que es el proyecto?

Herramienta interna para **DHL** que organiza pedidos de comida por bloques semanales.
El admin configura el menu, comparte un link publico y el equipo carga pedidos por dia.
El resumen se ve agrupado por equipo y se puede exportar a CSV.

---

## 2. Arquitectura

| Capa | Tecnologia | Detalle |
|------|-----------|---------|
| Frontend | HTML + CSS + JS vanilla | Archivo unico `index.html`, sin frameworks ni build tools |
| Backend | Supabase (PostgreSQL + REST API) | Acceso directo via SDK JS |
| Hosting | GitHub Pages | Repo: `https://github.com/1kt0n/menu-semanal` |
| CDN | Supabase JS SDK UMD | `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js` |

### Importante sobre el SDK
- Se usa build **UMD** (no ESM) para navegador.
- Cliente creado como `window.supabase.createClient(URL, KEY)` en variable global `db`.
- Variables globales con `var` para evitar problemas con `onclick` inline.

---

## 3. Supabase — Base de datos

### Credenciales usadas por la app
```txt
SUPABASE_URL = 'https://ghxgamzvvgsootxxyeee.supabase.co'
SUPABASE_ANON_KEY = '...'
```

### Tablas base
```sql
CREATE TABLE IF NOT EXISTS public.menu (
  id BIGSERIAL PRIMARY KEY,
  desde TEXT NOT NULL,
  hasta TEXT NOT NULL,
  dias JSONB NOT NULL DEFAULT '[]',
  creado BIGINT NOT NULL
);

CREATE TABLE IF NOT EXISTS public.respuestas (
  id BIGSERIAL PRIMARY KEY,
  nombre TEXT NOT NULL,
  grupo TEXT NOT NULL,
  pedidos JSONB NOT NULL DEFAULT '[]',
  hora TEXT NOT NULL,
  menu_id BIGINT NOT NULL REFERENCES public.menu(id) ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS public.admins (
  id BIGSERIAL PRIMARY KEY,
  usuario TEXT NOT NULL UNIQUE,
  clave TEXT NOT NULL
);
```

### RLS (estado actual de la app)
```sql
ALTER TABLE public.menu ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.respuestas ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.admins ENABLE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS public_menu ON public.menu;
CREATE POLICY public_menu ON public.menu FOR ALL USING (true) WITH CHECK (true);

DROP POLICY IF EXISTS public_respuestas ON public.respuestas;
CREATE POLICY public_respuestas ON public.respuestas FOR ALL USING (true) WITH CHECK (true);

DROP POLICY IF EXISTS public_admins ON public.admins;
CREATE POLICY public_admins ON public.admins FOR ALL USING (true) WITH CHECK (true);
```

### Recomendado para evitar duplicados de pedidos
```sql
CREATE UNIQUE INDEX IF NOT EXISTS respuestas_menu_nombre_unique
ON public.respuestas (menu_id, lower(trim(nombre)));
```

### Cierre/deadline
La app intenta usar columnas nativas en `menu`:
```sql
ALTER TABLE public.menu
  ADD COLUMN IF NOT EXISTS fecha_limite TIMESTAMPTZ,
  ADD COLUMN IF NOT EXISTS cerrado BOOLEAN NOT NULL DEFAULT false;
```

Si REST responde `42703` (columna inexistente), la app usa **fallback** y guarda meta de cierre dentro de `menu.dias` en clave `_meta_cierre`.

---

## 4. Constantes de negocio

```javascript
var DIAS = ['Lunes', 'Martes', 'Miércoles', 'Jueves', 'Viernes', 'Sabado', 'Domingo'];
var GRUPOS = [
  'HUB AM', 'HUB PM', 'COURIER AM', 'COURIER PM',
  'FORMAL AM', 'FORMAL PM', 'AFORO AM', 'AFORO PM',
  'DENUNCIAS', 'SEGURIDAD'
];
var BEBIDAS_FIJAS = ['PEPSI', 'PEPSI BLACK', '7UP', '7UP LIGHT', 'PASO DE LOS TOROS', 'MIRINDA', 'JUGO POMELO', 'JUGO MANZANA', 'AGUA', 'AGUA CON GAS'];
```

- Sin limite de cantidad de personas.
- Equipos hardcodeados en `GRUPOS`.

---

## 5. Funcionalidades actuales

### 5.1 Login (`#vista-login`)
- Login admin por credenciales definidas en frontend.
- Al autenticar muestra tabs de Admin y Resumen.

### 5.2 Admin (`#vista-admin`)
- Carga semana con inputs `type=date`: `Desde` y `Hasta`.
- Tiene toggle `Limitar tiempo de pedidos`; al activarlo habilita **Fecha limite para tomar pedidos** (`datetime-local`).
- Carga comidas/postre por dia (bebidas fijas globales).
- Guarda menu y genera link `?vista=form`.
- Ordena dias segun fecha `Desde` para soportar bloques no estandar (ej: sabado-viernes).
- Render de tabs con etiqueta tipo almanaque: `Dia dd-mm`.

### 5.3 Formulario (`#vista-form`)
- Vista publica por `?vista=form`.
- Seleccion de equipo, nombre, y pedidos por dia.
- Permite dias sin pedido (franco/vacaciones), pero valida pares comida+bebida.
- Muestra etiqueta de dia en formato `Dia dd-mm`.
- Bloquea envio si:
  - semana cerrada manualmente, o
  - fecha limite vencida.
- Proteccion anti doble click (`envioEnCurso` + boton deshabilitado).

### 5.4 Resumen (`#vista-resumen`)
- Estadisticas y agrupacion por equipo.
- Filtro por equipo.
- Exportacion `CSV` semanal con filtro aplicado.
- Boton `Cerrar semana (sin borrar)`:
  - bloquea nuevos pedidos,
  - mantiene historial,
  - pensado para exportar y pasar a la semana siguiente.
- Muestra estado: `Abierto/Cerrado` y fecha limite cuando existe.

---

## 6. Estructura de datos

### `menu.dias` (JSONB)
Cada dia contiene:
```json
{
  "nombre": "Lunes",
  "comidas": ["..."],
  "bebidas": ["..."],
  "postre": "..."
}
```

Fallback de cierre (solo cuando no hay columnas nativas visibles por REST):
```json
{
  "_meta_cierre": {
    "fecha_limite": "2026-02-25T20:00:00.000Z",
    "cerrado": true
  }
}
```

### `respuestas.pedidos` (JSONB)
```json
[
  { "dia": "Lunes", "comida": "Milanesa", "bebida": "AGUA" },
  { "dia": "Martes", "comida": "Ravioles", "bebida": "PEPSI" }
]
```

---

## 7. Cambios recientes aplicados (brief de cambios)

1. Equipo nuevo `SEGURIDAD` agregado en `GRUPOS`.
2. Anti-duplicado robusto en `saveRespuesta`:
   - merge de pedidos por dia,
   - consolidacion y limpieza de filas duplicadas por `nombre + menu_id`.
3. Etiquetas de dias tipo almanaque `Dia dd-mm` en Admin, Formulario y Resumen.
4. Resumen mejorado:
   - filtro por equipo,
   - exportacion CSV semanal.
5. Flujo de semana:
   - se reemplaza logica de “limpiar respuestas” por “cerrar semana” sin borrar historial.
6. Deadline de pedidos:
   - campo de fecha limite en Admin,
   - bloqueo de formulario/envio al vencer fecha.
7. Fallback de cierre:
   - si `menu.fecha_limite/cerrado` no esta disponible por REST, persiste en `menu.dias._meta_cierre`.
8. Aviso de deadline en formulario usuario:
   - banner amarillo visible mientras el formulario esta abierto con la fecha/hora de cierre.
   - mensaje de bloqueo claro al vencer (con fecha exacta).
9. Auto-calculo de fecha limite en Admin:
   - al ingresar `Desde`, se auto-sugiere el Viernes anterior al mediodia (patron operativo habitual).
   - boton `Recalcular` para forzar sugerencia si el campo ya tiene valor.
   - admin puede sobreescribir manualmente en cualquier momento.

---

## 8. Funciones JS clave

| Funcion | Descripcion |
|---------|-------------|
| `getMenu()` | Obtiene menu mas reciente y aplica meta de cierre (nativa o fallback) |
| `saveMenu(menu)` | Inserta/actualiza menu activo |
| `saveRespuesta(respuesta)` | Upsert logico por nombre/menu, merge de pedidos y limpieza de duplicados |
| `guardarMenu()` | Valida, ordena dias, guarda menu y fecha limite |
| `calcularFechaLimiteSugerida(desdeTexto)` | Calcula el Viernes anterior al mediodia dado un texto de fecha |
| `sugerirFechaLimite()` | Rellena el campo fecha_limite con la sugerencia (llamada por boton Recalcular) |
| `cargarFormulario()` | Render formulario y bloquea si semana cerrada/vencida |
| `enviarPedido()` | Envio validado + proteccion anti doble click |
| `cargarResumen()` | Render resumen semanal, estado y tabs por dia |
| `renderResumenDia()` | Agrupa por equipo y aplica filtro |
| `exportarResumenCSV()` | Descarga CSV semanal con columnas de detalle |
| `cerrarSemana()` | Cierra semana sin borrar pedidos |

---

## 9. Flujo operativo recomendado

1. Admin carga menu de bloque semanal.
2. Define fecha limite de pedidos.
3. Comparte link de formulario.
4. Al vencer deadline (o cerrar manualmente), se bloquean nuevos envios.
5. Admin exporta CSV.
6. Admin carga siguiente bloque semanal.

---

## 10. Pendientes sugeridos

- Reabrir semana desde UI (boton admin).
- Historial navegable de semanas (no solo menu mas reciente).
- Migrar cierre/deadline 100% a columnas nativas `menu.fecha_limite/cerrado` cuando el endpoint REST las exponga consistentemente.
- Mover auth admin fuera de credenciales hardcodeadas en frontend.

---

*Actualizado: 25 febrero 2026 — Incluye toggle de deadline y fechas `type=date` en Admin, con estado alineado a produccion*

### Flujo operativo real (referencia para fecha limite)
- El admin recibe el menu del comedor los jueves/viernes para la semana siguiente.
- Los bloques semanales son Sabado→Viernes o Lunes→Viernes segun la semana.
- Patron habitual: pedidos cierran el **Viernes anterior al inicio del bloque a las 12:00**.
- Ejemplo: menu desde Sab 28/2 → fecha limite sugerida = Vie 27/2 12:00.

