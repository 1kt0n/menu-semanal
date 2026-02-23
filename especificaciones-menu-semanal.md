# 🍽️ Menú Semanal — Especificaciones del Proyecto

> Documento generado para continuar el desarrollo en Claude Code desde PC.
> El proyecto fue construido como un único archivo HTML con almacenamiento persistente.
> El objetivo a futuro es convertirlo en una app web real con backend propio.

---

## 1. ¿Qué es el proyecto?

Una herramienta interna para que una empresa organice el pedido de comida semanal de sus empleados. El admin carga el menú, comparte un link, cada persona elige su pedido, y el admin ve el resumen en tiempo real.

---

## 2. Estado actual

El sistema está funcionando como un **único archivo HTML** (`menu-semanal.html`) con:

- Frontend en HTML + CSS + JavaScript vanilla (sin frameworks)
- Almacenamiento mediante `window.storage` (API de persistencia de Claude.ai)
- Sin backend propio — limitación actual que impide compartir el link externamente

---

## 3. Funcionalidades implementadas

### Panel Admin (`/admin`)
- Carga de fechas de la semana (desde / hasta)
- Alta de opciones de **comida** (cantidad libre, sin límite)
- Alta de opciones de **bebida** (cantidad libre, sin límite)
- Campo de **postre del día** (texto libre, informativo, no se elige)
- Botón "Guardar y generar link" → guarda el menú y genera la URL del formulario
- Al reabrir, carga automáticamente el menú guardado anteriormente

### Formulario de empleado (`?vista=form`)
- Muestra la semana configurada
- Selección de **equipo** (6 equipos fijos, ver sección 5)
- Campo de **nombre y apellido**
- Selección de **comida** (botones tipo tarjeta)
- Selección de **bebida** (botones tipo tarjeta)
- Muestra el **postre del día** como información (no seleccionable)
- Validaciones: equipo → nombre → comida → bebida (en ese orden)
- Si la persona ya respondió (mismo nombre), actualiza su pedido
- Mensaje de confirmación al enviar

### Panel Resumen (`/resumen`)
- Contador de respuestas recibidas vs. pendientes (base: 30 personas)
- Postre del día destacado (si fue cargado)
- Barras de progreso por opción de **comida** (azul)
- Barras de progreso por opción de **bebida** (violeta)
- Barras de progreso por **equipo** (verde)
- Lista de personas ordenada por equipo, con nombre, equipo, comida y bebida
- Botón "Limpiar respuestas" para resetear al inicio de cada semana
- Botón "Actualizar" para refrescar los datos

---

## 4. Estructura del código actual

```
menu-semanal.html
├── <style>          → CSS completo (variables, componentes, responsive)
├── <nav>            → Navegación entre las 3 vistas
├── #vista-admin     → Panel de configuración del menú
├── #vista-form      → Formulario para empleados
├── #vista-resumen   → Dashboard de resultados
└── <script>
    ├── storageGet() / storageSet()   → Helpers de persistencia
    ├── irA(vista)                    → Navegación entre vistas
    ├── toast(msg)                    → Notificaciones flotantes
    ├── renderItems(tipo)             → Renderiza lista de comidas/bebidas en admin
    ├── agregarItem / eliminarItem    → CRUD de opciones de menú
    ├── guardarMenu()                 → Guarda menú en storage y genera link
    ├── cargarFormulario()            → Carga y renderiza el formulario
    ├── enviarPedido()                → Guarda la respuesta del empleado
    ├── cargarResumen()               → Calcula y renderiza el resumen
    ├── resetearRespuestas()          → Limpia todas las respuestas
    └── init()                        → Inicialización (detecta ?vista=form)
```

---

## 5. Datos fijos del negocio

### Equipos (6, fijos, hardcodeados)
```javascript
const GRUPOS = ["HUB", "COURIER AM", "FORMAL", "DENUNCIAS", "AFORO", "COURIER PM"];
```

### Total de personas esperadas
```javascript
const TOTAL_ESPERADO = 30;
```

---

## 6. Modelo de datos

### Menú semanal (key: `menu-semanal`)
```json
{
  "desde": "24 de junio",
  "hasta": "28 de junio",
  "comidas": ["Milanesa", "Pollo al horno", "..."],
  "bebidas": ["Agua", "Gaseosa", "..."],
  "postre": "Flan con dulce de leche",
  "creado": 1719187200000
}
```

### Respuestas (key: `respuestas`)
```json
[
  {
    "nombre": "Juan Pérez",
    "grupo": "HUB",
    "comida": "Milanesa",
    "bebida": "Agua",
    "hora": "12:30"
  }
]
```

---

## 7. Paleta de colores y diseño

```css
--bg: #f8f9fb;
--card: #ffffff;
--border: #e5e7eb;
--primary: #2563eb;       /* Azul — comidas, acciones principales */
--primary-light: #eff6ff;
--text: #111827;
--muted: #6b7280;
--success: #16a34a;       /* Verde — equipos, confirmación */
--success-light: #f0fdf4;
--danger: #dc2626;        /* Rojo — eliminar, limpiar */
--accent: #7c3aed;        /* Violeta — bebidas, pendientes */
```

- Fuente: **Inter** (Google Fonts)
- Responsive: breakpoint en 480px (grilla de opciones pasa a 1 columna)
- Navegación sticky en top

---

## 8. Limitación actual y próximo paso

### El problema
El sistema usa `window.storage`, una API exclusiva del entorno Claude.ai. Esto hace que:
- El menú y las respuestas **solo existen dentro de esa sesión de Claude**
- El link generado no es accesible desde otro dispositivo o navegador
- No es posible compartirlo con las 30 personas del equipo

### Solución propuesta: migrar a app web real

#### Stack sugerido
| Capa | Tecnología | Motivo |
|------|-----------|--------|
| Frontend | HTML + JS vanilla o React | Ya tenemos el diseño |
| Backend | Node.js + Express | Simple, liviano |
| Base de datos | Supabase (PostgreSQL) | Gratis, API REST lista |
| Deploy | Vercel (frontend) + Railway o Render (backend) | Gratis en tier inicial |

#### Alternativa más simple (sin backend propio)
Usar **Supabase directamente desde el frontend** con su SDK de JavaScript:
- Reemplaza `storageGet/storageSet` por llamadas a Supabase
- No requiere servidor propio
- El link generado funciona para cualquier persona

---

## 9. Próximas funcionalidades deseadas

- [ ] Link compartible que funcione fuera de Claude.ai
- [ ] Que el admin pueda ver el resumen en tiempo real mientras llegan respuestas
- [ ] Posibilidad de cerrar el formulario (deadline de pedidos)
- [ ] Historial de semanas anteriores
- [ ] Exportar resumen a texto plano para cargar en el sistema del comedor

---

## 10. Cómo continuar en Claude Code

1. Abrí Claude Code en tu PC
2. Pegá este documento como contexto inicial
3. Adjuntá el archivo `menu-semanal.html` como referencia del diseño
4. Pedile que migre el proyecto a una app con Supabase como base de datos

### Prompt sugerido para empezar:

```
Tengo una app de menú semanal construida en HTML + JS vanilla.
El diseño y la lógica ya están completos (te adjunto el archivo).
Quiero migrarla a una app web real usando Supabase como base de datos,
para que el formulario sea accesible desde cualquier dispositivo via link.
Mantené el diseño visual existente. Empecemos por configurar Supabase
y reemplazar las funciones storageGet/storageSet.
```

---

*Generado el 22 de febrero de 2026 — Proyecto desarrollado en Claude.ai*
