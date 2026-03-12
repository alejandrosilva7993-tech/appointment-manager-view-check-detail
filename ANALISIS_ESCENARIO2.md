# Análisis: Escenario 2 no se muestra al pulsar Enter o flecha derecha

## Resumen del problema

Al pulsar Enter o la tecla "siguiente" (ArrowRight), el escenario 2 (calendario con citas pobladas) no se visualiza correctamente.

---

## Causas identificadas

### 1. **Rango de fechas visible vs fechas de los eventos (CAUSA PRINCIPAL)**

- **Eventos mock:** Las citas están en marzo 2025 (03-03 a 03-06).
- **Vista inicial:** `timeGridDay` muestra el día actual (por ejemplo, 11 de marzo de 2025 o la fecha del sistema).
- **Efecto:** Los eventos se añaden al calendario, pero la vista sigue en el día actual. Si el día actual no coincide con las fechas de las citas, el usuario no ve ningún evento.

**Solución:** Tras añadir los eventos, navegar a una fecha con citas, por ejemplo:
```javascript
calendar.gotoDate('2025-03-04');
```

---

### 2. **FullCalendar y la tecla ArrowRight**

- FullCalendar usa ArrowRight para navegar al día/semana/mes siguiente.
- Aunque el listener usa `capture: true` y `preventDefault()`/`stopPropagation()`, en algunos entornos el orden de ejecución puede variar.
- **Mitigación:** El uso de `capture: true` debería ejecutar nuestro handler antes que el de FullCalendar.

---

### 3. **Elemento con foco**

- Se excluyen INPUT y TEXTAREA.
- Si un botón tiene foco (p. ej. tras tabular), el handler sigue ejecutándose.
- **Conclusión:** No parece la causa principal.

---

### 4. **Compatibilidad de teclas**

- Se comprueban `e.key` y `e.keyCode` para ArrowRight (39) y Enter (13).
- **Conclusión:** La detección de teclas es correcta.

---

### 5. **Formato de eventos para addEvent**

- Los objetos tienen: `id`, `title`, `start`, `end`, `extendedProps`, `classNames`.
- FullCalendar acepta este formato.
- **Conclusión:** El formato es válido.

---

## Recomendación

Añadir `calendar.gotoDate('2025-03-04')` justo después de cargar los eventos para que la vista muestre un día con citas y el usuario vea el escenario 2.
