# DHL Menu Semanal — Especificaciones Tecnicas

> Documento actualizado con el estado real del proyecto.
> Usar como brief para continuar el desarrollo con cualquier IA o desarrollador.

---

## 1. Que es el proyecto?

Una herramienta interna para **DHL** que permite organizar el pedido de comida semanal de sus empleados. El admin carga el menu (comidas, bebidas y postre distintos para cada dia de la semana), comparte un link, cada persona elige su pedido para los 5 dias, y el admin ve el resumen agrupado por equipo en tiempo real.

---

## 2. Arquitectura

| Capa | Tecnologia | Detalle |
|------|-----------|---------|
| Frontend | HTML + CSS + JS vanilla | Archivo unico `index.html`, sin frameworks ni build tools |
| Backend | Supabase (PostgreSQL + REST API) | Acceso directo desde frontend via SDK JS |
| Hosting | GitHub Pages | Repo: `https://github.com/1kt0n/menu-semanal` |
| CDN | Supabase JS SDK UMD | `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js` |
| Fuente | Google Fonts — Inter (400-800) | Tipografia principal |

### Importante sobre el SDK
- Se usa la build **UMD** del SDK de Supabase (no ESM). La URL ESM default no funciona en `<script>` tags de browser.
- El cliente se instancia como `window.supabase.createClient(URL, KEY)` y se guarda en variable `db` (no `supabase`, para evitar colision con el namespace global del SDK).
- Todas las variables globales usan `var` en vez de `let` para evitar errores de Temporal Dead Zone cuando se referencian desde `onclick` inline.

---

## 3. Supabase — Credenciales y Base de datos

### Credenciales
```
SUPABASE_URL = 'https://ghxgamzvvgsootxxyeee.supabase.co'
SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImdoeGdhbXp2dmdzb290eHh5ZWVlIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzE4MDQyMTQsImV4cCI6MjA4NzM4MDIxNH0.wem5mZntSR-BB82ivuQ3lYC4AJHxvD2s7x-zIVXJ2zQ'
```

### SQL de creacion de tablas
```sql
CREATE TABLE menu (
  id SERIAL PRIMARY KEY,
  desde TEXT NOT NULL,
  hasta TEXT NOT NULL,
  dias JSONB NOT NULL DEFAULT '[]',
  creado BIGINT NOT NULL
);

CREATE TABLE respuestas (
  id SERIAL PRIMARY KEY,
  nombre TEXT NOT NULL,
  grupo TEXT NOT NULL,
  pedidos JSONB NOT NULL DEFAULT '[]',
  hora TEXT NOT NULL,
  menu_id INTEGER REFERENCES menu(id) ON DELETE CASCADE
);

CREATE TABLE admins (
  id SERIAL PRIMARY KEY,
  usuario TEXT NOT NULL UNIQUE,
  clave TEXT NOT NULL
);

-- RLS: acceso publico habilitado en las 3 tablas
ALTER TABLE menu ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_menu" ON menu FOR ALL USING (true) WITH CHECK (true);

ALTER TABLE respuestas ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_respuestas" ON respuestas FOR ALL USING (true) WITH CHECK (true);

ALTER TABLE admins ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_admins" ON admins FOR ALL USING (true) WITH CHECK (true);

-- Admin inicial
INSERT INTO admins (usuario, clave) VALUES ('admin', 'admin123');
```

### Modelo de datos

#### Tabla `menu` — Columna `dias` (JSONB)
Cada dia tiene sus propias opciones de comida, bebida y postre:
```json
[
  {
    "nombre": "Lunes",
    "comidas": ["Milanesa napolitana", "Pollo al horno", "Tarta de verdura"],
    "bebidas": ["Agua", "Gaseosa", "Jugo"],
    "postre": "Flan con dulce de leche"
  },
  {
    "nombre": "Martes",
    "comidas": ["Ravioles", "Suprema", "Ensalada cesar"],
    "bebidas": ["Agua", "Gaseosa"],
    "postre": "Fruta de estacion"
  }
]
```

#### Tabla `respuestas` — Columna `pedidos` (JSONB)
Cada empleado envia un pedido para los 5 dias de la semana en una sola accion:
```json
[
  { "dia": "Lunes", "comida": "Milanesa napolitana", "bebida": "Agua" },
  { "dia": "Martes", "comida": "Ravioles", "bebida": "Gaseosa" },
  { "dia": "Miercoles", "comida": "Hamburguesa", "bebida": "Jugo" },
  { "dia": "Jueves", "comida": "Pollo", "bebida": "Agua" },
  { "dia": "Viernes", "comida": "Pizza", "bebida": "Gaseosa" }
]
```

---

## 4. Constantes del negocio

```javascript
var DIAS = ['Lunes', 'Martes', 'Miercoles', 'Jueves', 'Viernes'];
var GRUPOS = ["HUB", "COURIER AM", "FORMAL", "DENUNCIAS", "AFORO", "COURIER PM"];
```

- **No hay limite** de cantidad de personas que pueden pedir. Antes era 30, fue eliminado.
- Los equipos estan hardcodeados en el array `GRUPOS`.

---

## 5. Vistas y funcionalidades

### 5.1 Login (`#vista-login`)
- Formulario con usuario y contraseña (texto plano)
- Valida contra tabla `admins` en Supabase
- Al autenticar, setea `esAdmin = true` y muestra tabs Admin y Resumen
- Ruta por defecto cuando se accede sin `?vista=form`

### 5.2 Admin (`#vista-admin`) — Requiere login
- Campos "Desde" y "Hasta" para la semana (texto libre, ej: "24 de febrero")
- **Tabs por dia** (Lunes a Viernes), cada uno con:
  - Lista de comidas (agregar/eliminar, cantidad libre)
  - Lista de bebidas (agregar/eliminar, cantidad libre)
  - Campo de postre (texto libre, informativo)
- Boton "Guardar y generar link" → inserta nuevo registro en tabla `menu` y muestra URL
- La URL del formulario es: `{dominio}?vista=form`
- Validacion: cada dia debe tener al menos 1 comida y 1 bebida

### 5.3 Formulario (`#vista-form`) — Acceso publico
- Acceso via URL `?vista=form` (oculta tabs de Admin/Resumen/Login)
- Muestra semana configurada
- Seleccion de equipo (botones tipo tarjeta, 6 opciones)
- Campo nombre y apellido
- **5 secciones, una por dia**, cada una con:
  - Header amarillo con nombre del dia
  - Opciones de comida (botones, seleccion unica)
  - Opciones de bebida (botones, seleccion unica)
  - Postre del dia (informativo, no seleccionable)
- Validacion: equipo, nombre, y comida+bebida de cada dia son obligatorios
- Si la persona ya respondio (mismo nombre, case-insensitive via `ilike`), actualiza su pedido
- Mensaje de confirmacion con detalle del pedido

### 5.4 Resumen (`#vista-resumen`) — Requiere login
- Titulo de la semana + contador de respuestas recibidas (sin limite/pendientes)
- **Tabs por dia** para filtrar
- Postre del dia destacado (card con icono)
- **Agrupado por equipo**: para cada equipo con pedidos ese dia, una card con:
  - Nombre del equipo + badge con cantidad de personas (fondo amarillo)
  - Subtotales de comida en pills (fondo amarillo claro, ej: "3x Milanesa")
  - Subtotales de bebida en pills (fondo rojo claro, ej: "4x Agua")
  - Lista de personas: avatar con inicial, nombre, comida + bebida
- Equipos sin pedidos ese dia no se muestran
- Boton "Limpiar respuestas" (rojo, con confirmacion)
- Boton "Actualizar" para refrescar datos

---

## 6. Estructura del archivo `index.html`

```
index.html (1200 lineas, archivo unico)
├── <head>
│   ├── Google Fonts (Inter 400-800)
│   ├── Supabase JS SDK UMD (CDN)
│   └── <style> — CSS completo
│       ├── :root variables (DHL theme)
│       ├── Navegacion (nav roja sticky)
│       ├── Cards, inputs, botones
│       ├── Tabs de dias
│       ├── Opciones del formulario
│       ├── Resumen (stats, personas)
│       ├── Toast, success-box, empty states
│       └── Media query 480px
├── <body>
│   ├── <nav> — Logo "DHL Menu" + tabs (Login/Admin/Formulario/Resumen)
│   ├── #vista-login — Formulario de credenciales
│   ├── #vista-admin — Panel de configuracion del menu
│   ├── #vista-form — Formulario del empleado
│   ├── #vista-resumen — Dashboard agrupado por equipo
│   └── #toast — Notificaciones flotantes
└── <script>
    ├── CONSTANTES (DIAS, GRUPOS)
    ├── SUPABASE CONFIG (URL, KEY, cliente db)
    ├── HELPERS (getMenu, saveMenu, getRespuestas, saveRespuesta, deleteRespuestas)
    ├── AUTENTICACION (verificarAdmin, loginAdmin, esAdmin flag)
    ├── NAVEGACION (irA con proteccion de rutas)
    ├── TOAST (toast)
    ├── ADMIN (menuDias, renderAdminTabs, renderAdminItems, guardarMenu, copiarLink)
    ├── FORMULARIO (cargarFormulario, resaltarOpcion, enviarPedido)
    ├── RESUMEN (cargarResumen, renderResumenDia con agrupacion por equipo)
    ├── RESET (resetearRespuestas)
    └── INIT (detecta ?vista=form o muestra login)
```

---

## 7. Paleta de colores — Branding DHL

```css
:root {
  --bg: #faf8f5;           /* Fondo general (crema suave) */
  --card: #ffffff;          /* Fondo de cards */
  --border: #e8e0d4;        /* Bordes */
  --primary: #D40511;       /* Rojo DHL — acciones principales, titulos */
  --primary-light: #fff1f1; /* Rojo claro — fondos hover/seleccion */
  --primary-hover: #b30410; /* Rojo oscuro — hover de botones */
  --yellow: #FFCC00;        /* Amarillo DHL — nav activo, tabs, badges, avatares */
  --yellow-light: #fff9e0;  /* Amarillo claro — fondos, pills, link box */
  --text: #2d2d2d;          /* Texto principal */
  --muted: #7a7268;         /* Texto secundario */
  --success: #16a34a;       /* Verde — boton enviar pedido */
  --danger: #D40511;        /* Rojo — eliminar, limpiar */
}
```

### Elementos de branding
- **Navbar**: fondo rojo DHL (`#D40511`) con logo amarillo (`#FFCC00`), sombra roja
- **Tabs activos**: fondo amarillo DHL con texto oscuro
- **Day headers** del formulario: fondo amarillo DHL
- **Avatares** de personas: fondo amarillo DHL
- **Toast**: fondo oscuro con texto amarillo
- **Inputs focus**: borde amarillo con sombra amarilla sutil
- **Body**: gradientes sutiles radiales rojo/amarillo en esquinas
- **Cards**: sombra sutil (`box-shadow: 0 1px 3px rgba(0,0,0,0.04)`)

---

## 8. Flujo de uso

### Admin
1. Accede a la URL principal (sin parametros)
2. Ve pantalla de login → ingresa usuario y contraseña
3. En Admin: configura fechas, agrega comidas/bebidas/postre por dia
4. Guarda → obtiene link `{url}?vista=form`
5. Comparte el link con los empleados
6. En Resumen: ve pedidos agrupados por equipo, filtra por dia

### Empleado
1. Recibe link `{url}?vista=form`
2. Selecciona equipo, ingresa nombre
3. Para cada dia (Lunes a Viernes): elige comida y bebida
4. Confirma pedido → se guarda en Supabase
5. Si vuelve a entrar con el mismo nombre, actualiza su pedido

---

## 9. Funciones JS principales

| Funcion | Descripcion |
|---------|-------------|
| `getMenu()` | Obtiene el menu mas reciente de Supabase (ordenado por `creado` desc) |
| `saveMenu(menu)` | Inserta un nuevo menu en Supabase |
| `getRespuestas(menuId)` | Obtiene todas las respuestas para un menu especifico |
| `saveRespuesta(resp)` | Inserta o actualiza respuesta (busca por nombre case-insensitive + menu_id) |
| `deleteRespuestas(menuId)` | Elimina todas las respuestas de un menu |
| `verificarAdmin(user, pass)` | Verifica credenciales contra tabla `admins` |
| `loginAdmin()` | Lee inputs, verifica, setea `esAdmin=true`, muestra tabs |
| `irA(vista)` | Navega entre vistas con proteccion de rutas admin |
| `renderAdminTabs()` | Renderiza tabs de dias + contenido de comidas/bebidas/postre |
| `guardarMenu()` | Valida, limpia items vacios, guarda en Supabase |
| `cargarFormulario()` | Carga menu desde Supabase, renderiza opciones para los 5 dias |
| `enviarPedido()` | Valida todo, construye array de pedidos, guarda en Supabase |
| `cargarResumen()` | Carga menu + respuestas, renderiza tabs y llama `renderResumenDia()` |
| `renderResumenDia()` | Agrupa pedidos por equipo, genera cards con subtotales y lista de personas |
| `resetearRespuestas()` | Elimina todas las respuestas con confirmacion |
| `init()` | Detecta `?vista=form` (empleado) o muestra login (admin) |

---

## 10. Notas tecnicas importantes

1. **Archivo unico**: Todo esta en `index.html`. No hay archivos JS/CSS separados.
2. **`var` obligatorio**: Todas las variables globales deben ser `var`, no `let`/`const`, porque se referencian desde `onclick` inline y pueden causar errores de Temporal Dead Zone si el SDK falla al cargar.
3. **Cliente `db`**: El cliente Supabase se guarda en `var db` (no `supabase`) para evitar colision con `window.supabase` del SDK. Todas las funciones helper hacen `if (!db) return` como null-check.
4. **`createClient` en try/catch**: La inicializacion esta envuelta en try/catch para que si el SDK no carga, el resto del script siga funcionando.
5. **Deteccion de duplicados**: `saveRespuesta` busca por `nombre` (case-insensitive con `ilike`) + `menu_id`. Si existe, hace update; si no, insert.
6. **GitHub Pages**: El archivo se llama `index.html` (no `menu-semanal.html`) para que GitHub Pages lo sirva como pagina principal.
7. **RLS publico**: Las 3 tablas tienen Row Level Security habilitado con politicas que permiten todo (`FOR ALL USING (true)`). No hay autenticacion a nivel de Supabase — la autenticacion es a nivel de app (tabla `admins`).

---

## 11. Posibles mejoras futuras

- [ ] Cerrar/abrir formulario (deadline de pedidos)
- [ ] Historial de semanas anteriores
- [ ] Exportar resumen a texto plano o PDF
- [ ] Tiempo real con Supabase Realtime (subscripcion a cambios)
- [ ] Contraseña hasheada en tabla `admins` (actualmente es texto plano)
- [ ] Administrar equipos desde el panel admin (en vez de hardcodeados)
- [ ] Input type="password" para la contraseña (actualmente es type="text")

---

*Actualizado: febrero 2026 — Proyecto desplegado en GitHub Pages con Supabase como backend*
