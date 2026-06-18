# Gestión MP 2026 — Centro de control operacional

Aplicación web de un solo archivo (`index.html`) para gestionar el mantenimiento
del equipamiento clínico del HHHA: ~1.000 equipos, mantenimiento preventivo (MP),
correctivo, pendientes y reprogramaciones.

> **Stack:** HTML + CSS + JavaScript *vanilla* (sin build) + [SheetJS](https://sheetjs.com) vía CDN.
> Funciona offline tras la primera carga.

## Uso

1. Abre `index.html` en un navegador moderno (o sírvelo desde cualquier servidor estático).
2. Importa **`ProgramaciónMP2026.xlsm`** (arrastra el archivo o haz clic). Hay una copia
   de prueba en [`sample-data/`](sample-data/).
3. Trabaja desde el **Tablero**; los datos operativos se guardan solos en `localStorage`.
4. Usa **Exportar JSON** periódicamente como respaldo (un hospital no debe depender solo de `localStorage`).

## Reparto de responsabilidades

- **El `.xlsm` manda en el cumplimiento** (programación anual + MP hecha / no hecha / causal).
  La app lo **lee, nunca lo escribe**. Al re-importarlo, refresca el estado *Oficial*.
- **La app manda en lo operativo:** detalle de cada MP, gestión correctiva, pendientes y
  bitácora por equipo. Persiste en `localStorage` con respaldo JSON.

## Lectura del `.xlsm`

| Hoja | Contenido |
|---|---|
| `PMP_2026` | Programación anual (1 columna por mes: `X` `R` `RA` `PM`) |
| `Registro_MP-2026` | Registro de cumplimiento (pares **P/R** por mes) |

- Encabezados en la **fila 7**, datos desde la **fila 8**; columnas identificadas por encabezado.
- Se **excluyen** `Q` (Observación) y `S` (Responsable MP); se ignoran las auxiliares tras `AR`.
- **N° Inventario** y **N° Serie** se leen como *string* (conservan ceros a la izquierda).
- Identificador único = N° Serie o N° Inventario (no el ID).
- Resultados (`registro.R`): `Si` · `Si-RA` · `C1`–`C8` · `FS` · `No` · `NU` · `Baja`. Datos sucios → se ignoran.

## Las 6 vistas

1. **Tablero "¿qué hago hoy?"** — KPIs reactivos + listas accionables (MP del mes sin resultado,
   MP vencidas, equipos detenidos, pendientes, reprogramaciones por firmar).
2. **Equipos** — tabla densa de ~1.000 filas (sin virtualización): búsqueda global con *debounce*,
   filtros combinables (AND), ordenamiento por columna, *badges* de estado, ficha de detalle y
   exportación de la vista a CSV/XLSX.
3. **Preventivo** — registro del detalle de la MP del mes (fecha, ejecutor, tipo interno/externo,
   resultado, estado resultante, observaciones) con ciclo **Borrador → Oficial**.
4. **Correctivo** — apertura de OT (Folio SIGEM), rutas **A** Repuestos · **B** Visita técnica ·
   **C** Servicio técnico · **D** Baja, línea de compra (Trato Directo / Compra Ágil) y los dos cierres
   (operativo y documental).
5. **Pendientes** — cinco tipos en tres orígenes (correctivo · preventiva · administrativo) sobre un
   núcleo común, con tareas automáticas.
6. **Reprogramación** — ciclo de las causales sin falla `C1/C5/C6/C7/C8`: generar → imprimir →
   2 firmas → escribir el código en el Excel → recargar → *Oficial*.

## Reglas de negocio

- `C1/C5/C6/C7/C8` → reprogramación sin falla, ≤30 días, **no** cambian el estado.
- `C2/C3/C4 · FS · NU` → equipo no disponible, sin fecha
  (C2→Servicio Técnico · C3/FS→No Operativo · C4→Préstamo · NU→No Ubicable).
- `No` → no realizada sin causal: queda como *MP sin resultado por gestionar* (no reprograma).
- `Baja` → prioridad máxima.
- **Borrador → Oficial:** una MP `Si`/`Si-RA` queda Oficial cuando su resultado aparece en el
  `.xlsm` recargado.

## Persistencia

`localStorage`: `mp2026.equipos`, `mp2026.detalleMP`, `mp2026.pendientes`, `mp2026.correctivos`
(+ `mp2026.meta`). Botones **Exportar / Importar JSON** para respaldo y restauración.

## Documentación de referencia

En [`docs/`](docs/): la especificación de implementación, los diagramas del proceso (auditados) y
el diagrama del módulo de pendientes.
