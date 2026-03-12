# Análisis: Estilos no aplicados (Today button stroke y días no disponibles)

## 1. Botón Today: stroke no se visualiza correctamente con el redondeo

### Causas probables

#### A) Orden de carga de estilos y FullCalendar v6

FullCalendar v6 **inyecta CSS dinámicamente** desde JavaScript en tiempo de ejecución (no usa un archivo `.css` externo). El orden efectivo de estilos es:

1. Estilos en `<style>` del documento
2. Tema PrimeVue (`theme.css`)
3. `<link>` a `index.global.min.css` de FullCalendar — **posible 404**: el paquete npm de FullCalendar v6 no incluye este archivo; el CSS va embebido en el JS
4. Al ejecutar `calendar.render()`, FullCalendar inyecta sus estilos en un `<style>` nuevo
5. `injectFcHeaderStyles()` añade nuestros overrides después de `render()`

Si el `<link>` de FullCalendar devuelve 404, el calendario depende solo de los estilos inyectados por JS. Nuestros overrides se inyectan después, pero FullCalendar podría volver a inyectar o actualizar estilos en cambios de vista.

#### B) Estructura DOM y `fc-button-group`

Los botones del toolbar suelen estar dentro de `.fc-button-group`. Si el grupo tiene `overflow: hidden` o `border-radius` en los extremos, puede recortar el stroke del botón Today.

Estructura típica:
```html
<div class="fc-toolbar-chunk">
  <div class="fc-button-group">
    <button class="fc-button fc-hoy-button">Today</button>
  </div>
  <div class="fc-button-group">
    <button class="fc-prev-button">...</button>
    <button class="fc-next-button">...</button>
  </div>
</div>
```

Si `.fc-button-group` tiene `overflow: hidden`, el `box-shadow` del botón Today puede quedar cortado en los bordes.

#### C) Conflicto con `.fc .fc-button`

Regla base:
```css
.fc .fc-button { border: 1px solid #ced4da !important; box-shadow: none !important; }
```

Nuestra regla:
```css
.fc .fc-hoy-button { border: none !important; box-shadow: 0 0 0 1px #ced4da !important; }
```

`.fc-hoy-button` es más específico que `.fc-button`, pero si FullCalendar aplica estilos inline o usa selectores más específicos, pueden ganar sobre nuestro CSS.

#### D) Estilos inline por JavaScript

Si aplicamos estilos con `element.style.boxShadow` después de `render()`, los inline tienen prioridad sobre el CSS. Si FullCalendar vuelve a renderizar el toolbar (por ejemplo en `datesSet`), puede eliminar esos estilos inline.

### Recomendaciones

1. **Comprobar el DOM** en DevTools: inspeccionar el botón Today y ver si tiene estilos inline o está dentro de un contenedor con `overflow: hidden`.
2. **Ajustar el contenedor**: si `.fc-button-group` tiene `overflow: hidden`, añadir `overflow: visible` para el grupo del botón Today.
3. **Usar variables CSS de FullCalendar**: si FC v6 expone variables como `--fc-button-border-color`, usarlas para evitar conflictos.
4. **Aplicar estilos en `datesSet`**: reaplicar los estilos del botón Today en cada `datesSet` por si el toolbar se re-renderiza.

---

## 2. Días no disponibles: color surface/300 no se aplica

### Causas probables

#### A) Estructura distinta en timeGrid vs dayGrid

En **dayGrid** (vista Month), las celdas son `<td>` con clases como `fc-day-sun`, `fc-daygrid-day`.

En **timeGrid** (vista Day/Week), la estructura es distinta:
- Columnas por día: `fc-timegrid-col`
- Cada columna puede tener `fc-day-sun`, etc.
- Las celdas de hora son `fc-timegrid-slot` dentro de cada columna

`dayCellClassNames` aplica a la “celda de día”. En timeGrid, eso puede ser:
- el contenedor de la columna, o
- solo la cabecera del día,

y no todas las celdas de hora de esa columna. Si el color se aplica solo a un elemento que no cubre toda el área visible, el fondo gris no se verá en toda la columna.

#### B) Prioridad de `fc-day-today`

Regla actual:
```css
.fc .fc-day-today { background: #eff6ff !important; }
```

Para un domingo que es hoy, el elemento puede tener `fc-day-sun` y `fc-day-today`. Ambas reglas usan `!important`; gana la que aparece después en el CSS. Si `fc-day-today` está definida después en los estilos de FullCalendar, el azul puede tapar nuestro gris.

#### C) Selectores que no coinciden con el DOM real

Usamos:
```css
.day-unavailable-cell,
.fc .fc-day-sun,
.fc-day-sun,
.fc-scrollgrid .fc-day-sun,
td.fc-day-sun { background: #d1d5db !important; }
```

En timeGrid, las columnas pueden ser `<div>` en lugar de `<td>`, así que `td.fc-day-sun` no aplicaría. Además, la clase `fc-day-sun` podría estar en un elemento padre o en la cabecera, no en las celdas de slots.

#### D) `dayCellClassNames` y timeGrid

La documentación indica que `dayCellClassNames` funciona en dayGrid y timeGrid. En timeGrid, la “celda de día” puede ser un contenedor que no pinta el fondo de toda la columna. Si el fondo se aplica solo a ese contenedor y las celdas de hora tienen su propio fondo, el color no se propagará a toda la columna.

### Recomendaciones

1. **Inspeccionar el DOM** en vista Week: localizar el elemento que representa un domingo y ver qué clases tiene y en qué nivel del árbol.
2. **Usar `dayCellDidMount`**: en el callback, aplicar el color directamente al elemento (`arg.el`) y a sus hijos si hace falta.
3. **Añadir selectores para timeGrid**: por ejemplo `.fc-timegrid-col.fc-day-sun` si esa es la estructura real.
4. **Revisar orden de reglas**: asegurarse de que la regla de días no disponibles vaya después de `fc-day-today` en nuestro CSS inyectado, para que domine en domingos que son hoy.

---

## Resumen de acciones sugeridas

| Problema | Acción |
|----------|--------|
| Today stroke | Revisar `overflow` en `.fc-button-group`; reaplicar estilos en `datesSet`; comprobar que no haya estilos inline que los borren |
| Días no disponibles | Inspeccionar DOM en timeGrid; usar `dayCellDidMount` si hace falta; añadir selectores para `.fc-timegrid-col`; verificar orden respecto a `fc-day-today` |
