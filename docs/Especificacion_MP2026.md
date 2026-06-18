# Especificación de implementación — Sistema de Gestión MP 2026

Complemento técnico de `Diagramas_MP2026_ajustado.md` y `arbol-pendientes.html`. **Fuente de datos: `ProgramaciónMP2026.xlsm`** (documento oficial). Define carga, persistencia, interfaz y dirección visual. Entregar todo junto a Claude Code.

---

## 1. Stack

- Un solo archivo `.html` (HTML + CSS + JS *vanilla*, sin paso de build).
- **+ SheetJS (`xlsx`) vía CDN** para leer el `.xlsm`.
- Funciona offline tras la primera carga.

---

## 2. Carga de datos — importar `ProgramaciónMP2026.xlsm`

> El `.xlsm` es el **registro oficial validado por resolución**: la programación anual y el cumplimiento formal (MP hecha / no hecha / causal). La app **lo lee, nunca lo escribe**. El usuario lo actualiza a mano; al **re-importarlo**, la app refresca el estado Oficial (ver ciclo Borrador→Oficial). La app aporta lo que el `.xlsm` no guarda: detalle de ejecución, lo correctivo, pendientes y bitácora.

Leer dos hojas:

| Hoja | Contenido | Meses |
|---|---|---|
| `PMP_2026` | Programación anual | **1 columna por mes** (solo `P`) |
| `Registro_MP-2026` | Registro de cumplimiento | **par P/R por mes** |

**Reglas de lectura (ambas hojas):** encabezados en la **fila 7**, datos desde la **fila 8**; identificar columnas por encabezado; **excluir** `Q` (Observación) y `S` (Responsable MP); en `Registro_MP-2026` **ignorar** las columnas auxiliares a la derecha de `AR`.

**Campos por equipo — columnas A–S:**

| Col | Campo | Notas |
|---|---|---|
| A | Fam | Familia/categoría (útil para agrupar) |
| B | ID | Orden en la planilla. **No** es identificador único |
| C | N° Carpeta | |
| D | N° Inventario | **TEXTO** (ej. `2-115361`) |
| E | Equipo | |
| F | Servicio | |
| G | Unidad | |
| H | Ubicación | |
| I | Procedencia | |
| J | Marca | |
| K | Modelo | |
| L | Serie | **TEXTO**, conserva ceros a la izquierda (`0024` ≠ `24`) |
| M | Año Instalación | |
| N | Vida Útil Residual | |
| O | Clasificación | |
| P | ENU / Baja | |
| ~~Q~~ | ~~Observación~~ | **EXCLUIR** |
| R | Frecuencia MP | Trimestral, Semestral, etc. |
| ~~S~~ | ~~Responsable MP~~ | **EXCLUIR** |

> **Identificador único:** N° de Serie o N° de Inventario (no el ID). Leer ambos como *string*.

**Programación — `PMP_2026` (columnas T–AE = Ene…Dic):** un valor por mes en la subcolumna `P`: `X` programada · `R` reprogramada · `RA` reprogramada de año anterior · `PM` puesta en marcha · *(vacío)* sin MP.

**Registro — `Registro_MP-2026` (pares P/R):**

| Mes | P/R | Mes | P/R |
|---|---|---|---|
| Ene | T / U | Jul | AF / AG |
| Feb | V / W | Ago | AH / AI |
| Mar | X / Y | Sep | AJ / AK |
| Abr | Z / AA | Oct | AL / AM |
| May | AB / AC | Nov | AN / AO |
| Jun | AD / AE | Dic | AP / AQ |

- **P** (Programa): `X` / `R` / `RA` / `PM`.
- **R** (Resultado): `Sí` realizada · `Si-RA` año anterior realizada · `C1`–`C8` reprogramada (causal) · `FS` fuera de servicio · `No` no realizada · `NU` no ubicable · `Baja` dada de baja.

> Datos sucios en celdas de mes → ignorar.

---

## 3. Causales de reprogramación

| Código | Descripción | Comportamiento |
|---|---|---|
| C1 | No se puede desocupar el equipo del paciente (indicación clínica) | reprogramar ≤30 d |
| C2 | Equipo en servicio técnico | sin fecha |
| C3 | No operativo, espera repuestos/accesorios | sin fecha |
| C4 | En préstamo a otro hospital | sin fecha |
| C5 | Sin HH del funcionario SEC (carga) | reprogramar ≤30 d |
| C6 | Sin HH del servicio técnico externo | reprogramar ≤30 d |
| C7 | Ausencia justificada del funcionario SEC > 15 días | reprogramar ≤30 d |
| C8 | Contingencia hospitalaria | reprogramar ≤30 d |

- **C2, C3, C4** → `reprog30 = FALSE`: sin nueva fecha; se registra en el mes real al reintegrar el equipo.
- **C1, C5, C6, C7, C8** → `reprog30 = TRUE`: reprogramar dentro de 30 días.

---

## 4. Ejecutores (lista cerrada)

Carlos Bahamondes Seguel · Cristián Beltrán Oviedo · Cristina Rozas Urrutia · Daniel Díaz Neira · Ignacio Berner Bergara · Macarena Toledo · Marco Ulloa · Matías Soazo Garrido · Ricardo Matus Aroca · Tito Millapán Riquelme · Personal Externo.

---

## 5. Almacenamiento (persistencia)

- El `.xlsm` es la **fuente de carga** (solo lectura); lo que el usuario genera en la app (detalle de MP, correctivos, pendientes, bitácora) vive en `localStorage`.
- Claves: `mp2026.equipos`, `mp2026.detalleMP`, `mp2026.pendientes`, `mp2026.correctivos`.
- Guardado automático **+ Exportar / Importar JSON** para respaldo (un hospital no debe depender solo de `localStorage`).

---

## 6. Modelo de Pendientes

> Según `arbol-pendientes.html`. **Cinco tipos en tres orígenes**, sobre un núcleo común.

**Núcleo común:** `equipo · serie · inventario · servicio · tipo · fechaCompromiso · prioridad · situación · estado · respEjecutivo · respAdministrativo · enEsperaDe · descripcion · tareas[] · fechaCompletado · creado`.
El **N° Inventario autocompleta** equipo, serie y servicio; `creado` y `díasDeAtraso` se calculan solos.

- **Origen `correctivo`** (hay OT en SIGEM):
  - **`documental`** — el equipo ya opera pero falta el Reporte de Servicio. Campos extra: `folioSIGEM · empresa · origen (visita | servicio_técnico)`. Tareas: gestionar → recibir/verificar → archivar. Resp. ejecutivo = quien consigue el reporte; administrativo por defecto = Cristián.
- **Origen `preventiva`** (sin Folio SIGEM):
  - **`protocolo_interno`** — falta el reporte interno de la MP.
  - **`protocolo_externo`** — lo completa el proveedor. Campo extra: `empresa`.
  - **`reprogramacion`** — la MP no se ejecutó. Tareas: generar e imprimir + firma del supervisor y del jefe de equipos.
- **Origen `administrativo`**:
  - **`otro`** — solo el núcleo común, sin campos extra.

> Solo `documental` añade `folioSIGEM`, `empresa` y `origen`. Una MP **vacía** o con resultado **`No`** **no** crea pendiente formal: queda como *MP sin resultado por gestionar*, visible en el Tablero (no reprograma).

---

## 7. Interfaz — vistas y navegación

1. **Tablero "¿qué hago hoy?" / Centro de control** — KPIs + accesos: MP del mes sin resultado, MP vencidas, pendientes, equipos detenidos, reprogramaciones por firmar.
2. **Equipos** — tabla densa filtrable + ficha de detalle.
3. **Mantenimiento Preventivo** — registro del **detalle** de la MP del mes (fecha, ejecutor, tipo interno/externo, estado resultante, observaciones).
4. **Mantenimiento Correctivo** — apertura OT + ruta A/B/C/D + línea de compra + los dos cierres.
5. **Pendientes** — cinco tipos en tres orígenes (§6).
6. **Reprogramación** — ciclo C1/C5/C6/C7/C8 (generar → imprimir → 2 firmas → escribir el código en el `.xlsm` → recargar → Oficial).

**Comportamientos de la vista Equipos** (JS *vanilla*, sin frameworks):
- **Búsqueda global** con *debounce* ~300 ms sobre varias columnas (inventario, equipo, serie, servicio, marca, modelo).
- **Filtros combinables (AND)**: familia · servicio · estado · mes; se acumulan con la búsqueda.
- **Tabla densa** (padding mínimo); ~1.000 filas **sin virtualización** (scroll/paginación simple).
- **Ordenamiento por columna** (asc → desc → sin orden).
- **Estado y tareas pendientes con *badge* de color** por fila.
- **Exportar la vista** filtrada a CSV/XLSX (además del respaldo JSON del §5).

**KPIs reactivos**: las tarjetas del Tablero se recalculan al aplicar filtros.

**Navegación:** el botón **Volver** regresa a la vista de origen (la navegación conserva el contexto de origen).

---

## 8. Dirección visual / UX — "centro de control operacional"

Herramienta de trabajo diario para gestionar 800–1.000 equipos. **Eficiencia y claridad por sobre lo decorativo.** (Claude Code puede apoyarse en el skill `frontend-design` para los tokens.)

- **Pantalla principal = centro de control:** el estado global de los equipos se ve de inmediato, sin saltar entre pantallas.
- **Búsqueda siempre visible** arriba: encontrar cualquier equipo en segundos.
- **Banda de KPIs** bajo la búsqueda: % operativos · fuera de servicio · tareas pendientes · MP vencidas · equipos en servicio técnico (con tiempo detenido).
- **Tabla grande y densa** como zona principal: una fila por equipo (inventario, nombre, ubicación, servicio, estado, tareas pendientes); ver muchos a la vez.
- **Filtros in-situ** (botones, desplegables, búsqueda), **sin modales ni ventanas**: pasar de 1.000 equipos a los relevantes en segundos.
- **Color con moderación:** sobre todo para alertas y estados; el resto neutro y discreto, para trabajar horas **sin fatiga visual**.
- **Mínimos clics** en las acciones frecuentes; sensación de panel de operaciones, no de sitio web.

---

## 9. Reglas de negocio a no perder

- **C1/C5/C6/C7/C8** → reprogramación sin falla, ≤30 días, no cambian el estado.
- **C2/C3/C4 · FS · NU** → equipo no disponible, sin fecha. Estados: C2→Servicio Técnico · C3→No Operativo · C4→Préstamo · FS→No Operativo · NU→No Ubicable (estado propio).
- **Baja** → prioridad máxima sobre cualquier otro evento del mes.
- **Borrador → Oficial:** el detalle se trabaja en la app (Borrador); una MP queda **Oficial** cuando su resultado se escribe en el `.xlsm` y, al recargarlo, la app lo refleja.

---

## 10. Checklist para Claude Code

- [ ] Importar `ProgramaciónMP2026.xlsm` (solo lectura); leer `PMP_2026` y `Registro_MP-2026`.
- [ ] Encabezados fila 7, datos fila 8; excluir Q y S; ignorar auxiliares tras `AR`.
- [ ] N° Inventario y Serie como **string** (ceros a la izquierda) en todo el flujo.
- [ ] Meses: 1 col/mes en `PMP_2026`; pares P/R (T/U…AP/AQ) en `Registro_MP-2026`.
- [ ] Resultados: `Sí`, `Si-RA`, `C1–C8`, `FS`, `No`, `NU`, `Baja`.
- [ ] Pendientes con los 5 tipos / 3 orígenes (§6).
- [ ] `localStorage` + Exportar/Importar JSON; la app **no** escribe el `.xlsm`.
- [ ] UI tipo centro de control (§8): búsqueda fija, KPIs reactivos, tabla densa, filtros in-situ, color solo para alertas; sin virtualización.
- [ ] Las 6 vistas con navegación que conserva el origen + motor de estado del §1 del `.md`.
