# Análisis: Calendario y Principio de "Espacio Compartido"

## Resumen ejecutivo

Este documento analiza en profundidad el **segundo escenario** (registros poblados de citas) del archivo `PRUEBACURSOR.html`, enfocándose en el componente de calendario, el espacio para slots de cita y el contenedor de cita, para identificar por qué **no es posible mostrar las citas de forma separada** con el comportamiento esperado del "Espacio Compartido".

---

## 1. Comportamiento esperado (lógica de negocio)

| Regla | Descripción |
|------|-------------|
| **Cita única** | Si no hay conflictos, la cita reclama el 100% del ancho del carril del día |
| **Colisión de horarios** | Cuando dos o más eventos comparten el mismo rango de minutos, el ancho se divide: \( W = \frac{\text{Ancho Total del Día}}{N} \) |
| **Prioridad de apilamiento** | Citas ordenadas de izquierda a derecha, generalmente por hora de inicio |
| **Márgenes** | Pequeño margen/padding entre citas para visualizar entidades separadas |

---

## 2. Arquitectura del componente actual

### 2.1 Jerarquía de componentes

```
#calendar (FullCalendar root)
└── .fc-view-harness
    └── .fc-timegrid (vista timeGridDay / timeGridWeek)
        ├── .fc-timegrid-axis (columna de horas)
        ├── .fc-timegrid-axis (columna all-day)
        └── .fc-timegrid-body
            └── .fc-timegrid-cols
                └── .fc-timegrid-col (columna del día)
                    ├── .fc-timegrid-col-frame (slots de tiempo)
                    │   └── .fc-timegrid-slot (división de horarios)
                    └── .fc-timegrid-events (contenedor de eventos)
                        └── .fc-timegrid-event-harness (por cada cita)
                            └── .fc-timegrid-event / .fc-event
                                └── .fc-event-main (contenido custom vía eventContent)
```

### 2.2 Responsabilidades

| Componente | Función |
|------------|---------|
| **Calendario** | Contenedor principal, gestiona vistas (Día/Semana/Mes) y eventos |
| **Slot de tiempo** | `.fc-timegrid-slot` — define las franjas horarias (eje vertical) |
| **Harness de evento** | `.fc-timegrid-event-harness` — posiciona y dimensiona cada cita (left/width en %) |
| **Contenedor de cita** | `.fc-timegrid-event` + `.fc-event-minimal` — renderiza el contenido visual |

---

## 3. Configuración actual de FullCalendar

### 3.1 Opciones relevantes (PRUEBACURSOR.html, líneas 460-491)

```javascript
slotEventOverlap: false,   // ✅ Correcto: evita Z-index overlap, dispone en columnas
eventDisplay: 'block',
displayEventTime: true,
displayEventEnd: true,
events: [],                 // Inicialmente vacío; se cargan con Enter/ArrowRight
dayMaxEvents: 5,            // Solo afecta vista dayGrid (Mes), no timeGrid
```

### 3.2 Interpretación

- **`slotEventOverlap: false`**: FullCalendar debe mostrar eventos superpuestos **en columnas lado a lado**, sin solapamiento visual.
- **`eventMaxStack`**: No definido (default `null`) — sin límite de columnas.
- **`dayMaxEvents`**: Solo aplica a la vista de mes, no a timeGrid.

---

## 4. Causas raíz identificadas

### 4.1 CSS que interfiere con el layout de columnas

#### A) Margen fijo en el harness (línea 209)

```css
.fc .fc-timegrid-event-harness { margin: 0 2px; overflow: hidden !important; min-width: 0 !important; }
```

**Problema**: FullCalendar posiciona cada harness con `left` y `width` en **porcentaje** (ej. 0–33%, 33–66%, 66–100% para 3 eventos). Añadir `margin: 0 2px` implica:

- Cada harness necesita 2px a izquierda y derecha.
- Con 4 eventos: 4 × 4px = 16px de margen extra.
- El ancho total efectivo supera el 100% del contenedor → desbordamiento o solapamiento.

**Impacto**: Alto. Puede romper el cálculo de columnas y provocar que los eventos se superpongan o se salgan del área visible.

#### B) `width: 100%` y `max-width: 100%` en el evento (líneas 265-266)

```css
.fc .fc-timegrid-event { width: 100% !important; max-width: 100% !important; }
```

**Análisis**: El evento ocupa el 100% de su harness. Si el harness tiene el ancho correcto (p. ej. 25% para 4 eventos), esto es adecuado. El problema no está aquí, sino en el harness.

#### C) `overflow: hidden` en múltiples niveles

```css
.fc .fc-timegrid-event-harness { overflow: hidden !important; }
.fc .fc-timegrid-event, .fc-event, ... { overflow: hidden !important; }
```

**Análisis**: Oculta contenido que se desborda (texto largo), pero no debería impedir el layout en columnas. Puede contribuir a que el texto se corte cuando el slot es estrecho.

#### D) `container-type: inline-size` en harness (líneas 289-291)

```css
.fc-timegrid-event-harness { container-type: inline-size; container-name: event-slot; }
@container event-slot (max-width: 80px) {
  .fc-event-minimal .fc-event-time { display: none !important; }
}
```

**Análisis**: Oculta la hora cuando el slot es estrecho. No afecta al posicionamiento en columnas.

---

### 4.2 Flujo de carga del escenario 2

El calendario arranca con `events: []`. Las citas se cargan al pulsar **Enter** o **ArrowRight**:

```javascript
window.addEventListener('keydown', function(e) {
  if (e.key === 'ArrowRight' || e.key === 'Enter' || ...) {
    cargarEscenario2();
  }
}, true);
```

**Posible confusión**: Si el usuario no pulsa Enter/ArrowRight, nunca ve citas. Además, FullCalendar usa ArrowRight para avanzar al día siguiente; aunque se usa `capture: true` y `preventDefault()`, puede haber conflictos en el orden de ejecución.

---

### 4.3 Datos de prueba (citas empalmadas)

Las 4 citas superpuestas del día actual (líneas 419-438):

| Cita | Inicio | Fin | Cliente |
|------|--------|-----|---------|
| 1 | 10:00 | 11:00 | Cliente A |
| 2 | 10:15 | 11:15 | Cliente B |
| 3 | 10:30 | 11:30 | Cliente C |
| 4 | 10:45 | 11:45 | Cliente D |

Todas comparten el rango 10:15–11:00. Con `slotEventOverlap: false`, FullCalendar debería asignar 4 columnas (~25% de ancho cada una).

---

### 4.4 Contenido custom (`eventContent`)

```javascript
eventContent: function(arg) {
  return {
    html: '<div class="fc-event-minimal"><span class="fc-event-status-bar status-' + estado + '"></span><span class="fc-event-time">' + timeText + '</span><span class="fc-event-title">' + title + '</span></div>'
  };
}
```

**Análisis**: El HTML es correcto. El layout en columnas lo gestiona FullCalendar a nivel de harness; el contenido solo debe respetar el contenedor. No se identifica causa directa aquí.

---

## 5. Resumen de causas

| # | Causa | Severidad | Componente afectado |
|---|-------|-----------|---------------------|
| 1 | `margin: 0 2px` en harness rompe el layout de columnas | **Alta** | Slot de cita / Harness |
| 2 | Posible conflicto Enter/ArrowRight con FullCalendar | Media | Calendario |
| 3 | Usuario puede no saber que debe pulsar Enter para cargar | Media | UX |
| 4 | Falta de separación visual explícita (bordes punteados como en la referencia) | Baja | Contenedor de cita |

---

## 6. Recomendaciones

### 6.1 Corregir el harness (prioridad alta) — APLICADO

**Eliminar o reducir el margen fijo** en `.fc-timegrid-event-harness`:

```css
/* ANTES (problemático) */
.fc .fc-timegrid-event-harness { margin: 0 2px; overflow: hidden !important; min-width: 0 !important; }

/* DESPUÉS (aplicado en PRUEBACURSOR.html) */
.fc .fc-timegrid-event-harness { 
  padding: 0 2px;  /* padding en lugar de margin: no afecta el cálculo de ancho */
  overflow: hidden !important; 
  min-width: 0 !important;
}
```

Se usó **padding** en lugar de margin para mantener la separación visual entre citas sin romper el layout de columnas.

### 6.2 Separación visual entre citas (principio de márgenes)

Para lograr el efecto de la imagen de referencia (bordes punteados, entidades separadas):

- Añadir `border` o `box-shadow` al contenedor de cita.
- Usar `gap` o `padding` en el contenedor de harnesses si FullCalendar lo permite.
- Evitar `margin` en el harness cuando el ancho es porcentual.

### 6.3 Carga del escenario 2 — APLICADO

- **Principio fijo**: El escenario 2 se carga **únicamente** con la tecla Enter o la tecla flecha derecha (→).
- No se muestra botón "Load demo" al usuario; la acción es exclusivamente por teclado.

### 6.4 Verificación de `slotEventOverlap`

Confirmar en el navegador que `slotEventOverlap: false` está activo y que FullCalendar genera harnesses con `left` y `width` en porcentaje. Si no es así, revisar la versión de FullCalendar y la documentación de la vista timeGrid.

---

## 7. Estructura de referencia (imagen objetivo)

La imagen de referencia muestra:

- Cita única: 100% de ancho.
- Bloque con colisiones: 3+ citas en columnas (~33% o menos cada una).
- Bordes naranjas punteados para separación visual.
- Icono de refresh en cada cita.
- Fondo crema y barra lateral naranja con rayas diagonales.

Para acercarse a ese diseño, además de corregir el layout:

1. Ajustar colores y bordes del contenedor de cita.
2. Añadir el icono de refresh si se requiere.
3. Mantener `text-overflow: ellipsis` para slots estrechos.

---

## 8. Checklist de validación

Tras aplicar las correcciones:

- [ ] Las 4 citas empalmadas (10:00–11:45) se muestran en 4 columnas visibles.
- [ ] No hay solapamiento por Z-index.
- [ ] Cita sin colisiones ocupa el 100% del ancho del día.
- [ ] Hay separación visual clara entre citas adyacentes.
- [ ] El texto se trunca con ellipsis cuando el ancho es insuficiente.
- [ ] El escenario 2 se carga correctamente (Enter/ArrowRight o método alternativo).

---

## 9. Análisis: Solapamiento inconsistente (4 citas empalmadas)

### 9.1 Patrón observado

- **Funciona**: Franjas con 2 citas superpuestas (6:00-7:00, 7:00-8:00, etc.) se muestran lado a lado.
- **Falla**: Franja 10:00-11:00 con 4 citas empalmadas: 2 se ven lado a lado y 2 se enciman (Z-index overlap).

### 9.2 Causas identificadas y correcciones aplicadas

| # | Causa | Corrección aplicada |
|---|-------|---------------------|
| 1 | Bug fc-timegrid-event-harness-inset (Issue #6512): aplica estilos de solapamiento a todos los eventos | CSS: `.fc-timegrid-event-harness-inset .fc-timegrid-event { box-shadow: none !important; }` |
| 2 | slotDuration 30 min: granularidad insuficiente para segmentos 10:00, 10:15, 10:30, 10:45 | `slotDuration: '00:15:00'` |
| 3 | eventOrderStrict false: prioriza compactación y puede reorganizar columnas mal con 4+ eventos | `eventOrderStrict: true` |
| 4 | container-type en harness: puede afectar el cálculo de left/width en % con position absolute | `container-type` removido del harness; fallback con media query para viewport estrecho |

### 9.3 Configuración actual aplicada

```javascript
slotDuration: '00:15:00',
eventOrderStrict: true,
views: { timeGridDay: { slotEventOverlap: false }, timeGridWeek: { slotEventOverlap: false } },
eventMaxStack: 99,
eventOrder: 'start',
```
