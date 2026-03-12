# Portal de Citas - Gestión de Transporte

Prototipo de calendario para gestión de citas de transporte (Goods Entry / Goods Exit) con FullCalendar.

## Escenarios

### Escenario 1: Calendario sin citas

- Vista inicial al cargar la aplicación.
- Calendario vacío con toast "No appointments are registered".
- Para pasar al Escenario 2: pulsar **Enter** o **flecha derecha (→)**.

### Escenario 2: Calendario con citas pobladas

- Se activa al pulsar Enter o flecha derecha desde el Escenario 1.
- Muestra citas de ejemplo en el día actual y días adyacentes.
- Incluye 4 citas empalmadas (10:00–11:45) para probar el principio de "Espacio Compartido".
- Filtros por estado: All, Created, Confirmed, Rejected, Completed.

## Archivos principales

| Archivo | Descripción |
|---------|-------------|
| `PRUEBACURSOR.html` | Aplicación principal con ambos escenarios |
| `dashboard-citas-maqueta.html` | Maqueta alternativa del dashboard |

## Ejecución local

```bash
# Con Python
python3 -m http.server 8080

# Con Node.js
npx serve -p 8080
```

Abrir: http://localhost:8080/PRUEBACURSOR.html

## Principio de Espacio Compartido

Las citas que comparten horario se muestran lado a lado (sin solapamiento por Z-index):

- **Cita única**: 100% del ancho del carril.
- **Colisión**: ancho dividido entre el número de citas concurrentes.
