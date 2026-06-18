# Sistema de Gestión MP 2026 — Proceso en texto

**Equipamiento Clínico HHHA** · versión texto de los diagramas · Build 2026-06-17.51 (revisión auditada)

Notación: las **¿preguntas?** son nodos de decisión; cada rama indica la condición y `→` la acción o el nodo siguiente. Los estados de cierre son **Borrador** (provisional) y **Oficial** (confirmado tras recargar la planilla).

> **Cambios en esta revisión** (auditoría cruzada con `ProgramaciónMP2026.xlsm`):
> 1. Se agregan los resultados `Si-RA` (año anterior realizada) y `No` (no realizada), antes ausentes.
> 2. `FS` = Fuera de Servicio y `NU` = No Ubicable quedan definidas (antes "no definidas en el árbol").
> 3. Se documenta el ciclo `RA → Si-RA` (mantención arrastrada del año anterior).
> 4. El motor de estado (§1) filtra equipos no disponibles (Préstamo / No Ubicable) antes de intentar MP.
> 5. Se distingue celda **vacía** (no registrada) de `No` (registrada como no realizada).
> 6. Se define **"el maestro"** y se alinean §2 (rutas A/B/C/D) con §3.
> 
> Decisiones confirmadas: `No` **no reprograma** (queda como MP sin resultado por gestionar, visible en el Tablero); `NU` es **estado propio** ("No Ubicable"). El **módulo de pendientes** se reemplaza por el modelo de `arbol-pendientes.html` (5 tipos en 3 orígenes); se elimina el "pendiente de seguimiento".

---

## 0. Convenciones

> Alineado al modelo de datos de SIGEM (programación, registro mensual, estados y bitácora).

**Programación anual** (`prog`), por mes:
- `X` programada · `R` / `RA` reprogramada · `PM` puesta en marcha.

**Registro mensual** (`registro`), por mes guarda dos valores:
- `P` = lo programado · `R` = resultado.

**Resultado de una MP** (`registro.R`):
- `Si` realizada · `Si-RA` realizada de año anterior · `C1`–`C8` no se pudo (reprogramación con causal) · `No` no realizada (sin causal) · `FS` fuera de servicio · `NU` no ubicable · `Baja` equipo dado de baja.

**Resultados — comportamientos:**
- `C1` `C5` `C6` `C7` `C8` → **reprogramación sin falla** (`reprog30 = TRUE`): no cambia el estado del equipo, marca `R` en el mes siguiente y abre pendiente de reprogramar ≤ 30 días.
- `C2` `C3` `C4` (y `FS` / `NU`) → **equipo no disponible / falla** (`reprog30 = FALSE`): no se fija fecha; se espera el reintegro o la reparación.
- `No` → **no realizada sin causal**: no reprograma por sí sola; queda como **MP sin resultado por gestionar** (visible en el Tablero) hasta que se ejecute o se le asigne una causal.
- `Si-RA` → **cierra una programación `RA`** (mantención del año anterior ejecutada); se comporta como `Si`.
- `Baja` → **prioridad máxima**: prevalece sobre cualquier otro evento del mes.

**Estado del equipo según el resultado:**
- `C2` → En Servicio Técnico · `C3` → No Operativo (espera repuestos) · `C4` → **Préstamo** (el equipo está fuera del hospital; no se le asigna Operativo/No Operativo) · `FS` → No Operativo · `NU` → **No Ubicable** (estado propio: el equipo no se localiza).
- Con `Si`, el estado lo decide quien registra: Operativo por defecto, o No Operativo si lo marca.

**Ciclo Borrador → Oficial:**
- **El maestro** = la hoja oficial `Registro_MP-2026` una vez recargada; un evento "está en el maestro" cuando, tras recargar la planilla, su resultado aparece en esa hoja.
- Una MP con `Si` (o `Si-RA`) queda **Oficial** si el evento está en el maestro; si no, queda en **Borrador**.
- En reprogramaciones, tras escribir el código en el Excel se recarga la planilla: si aparece el resultado → **Oficial** (archivado en carpeta + en la carta) y se cierra el pendiente; si no → sigue en **Borrador** (revisar código / mes).

**Glosario de causales:**
- `C1` — Paciente no libera el equipo.
- `C2` — Equipo en servicio técnico (→ En Servicio Técnico).
- `C3` — Espera de repuestos (→ No Operativo).
- `C4` — Préstamo a otro hospital (no declara estado).
- `C5` — Sin HH funcionario SEC (carga).
- `C6` — Sin HH servicio técnico externo.
- `C7` — Ausencia funcionario SEC > 15 días.
- `C8` — Contingencia hospitalaria.
- `FS` — Fuera de Servicio (→ No Operativo).
- `NU` — **No Ubicable**: estado propio del equipo cuando no se localiza (distinto de No Operativo).
- `No` — No Realizada (sin causal): queda como MP sin resultado por gestionar; no reprograma ni crea un tipo de pendiente formal.
- `Si-RA` — Mantención del año anterior realizada (cierra una programación `RA`).
- `Baja` — Equipo dado de baja (prioridad máxima).

---

## 1. Flujo general del programa

- Abrir la aplicación.
- **¿Planilla (Excel) cargada?**
  - **No** → cargar planilla (equipos + programación anual).
  - **Sí** → Tablero "¿qué hago hoy?" (resumen accionable del día).
- Por cada equipo, el **motor de estado** define la siguiente acción:

- **¿Equipo disponible en el hospital? (no en Préstamo ni No Ubicable)**
  - **No** → no se le programa MP este mes; esperar reintegro / localización (si toca MP, se marca con su causal o resultado).
  - **Sí** → sigue ↓
- **¿Equipo detenido? (No Operativo / Servicio Técnico)**
  - **Sí** → **¿Tiene expediente correctivo abierto?**
    - **No** → Abrir correctivo (OT).
    - **Sí** → Avanzar el expediente.
    - → entra al proceso **Mantenimiento Correctivo** (ver §3).
  - **No** → **¿MP del mes programada y sin resultado?**
    - **No** → **¿Pendiente vencido o protocolo sin resolver?**
      - **Sí** → Gestionar el pendiente.
      - **No** → ✓ Equipo al día.
    - **Sí** → Ejecutar **Mantenimiento Preventivo** (ver §4).

Cierre transversal: todo registro pasa por el ciclo **Borrador → Oficial** (§0) al recargar la planilla / al cerrar la gestión.

---

## 2. Árbol unificado — "¿qué hago con este equipo?"

> Vista resumida. El detalle de las **cuatro rutas** del expediente correctivo está en §3: **A** Repuestos · **B** Visita técnica externa · **C** Servicio técnico (envío) · **D** Baja. La etapa "Falta resolver la compra" de abajo agrupa A y B; "Enviado a servicio técnico" es la ruta C; la **baja (D)** es también una salida posible del expediente.

- **¿El equipo está detenido? (No Operativo / Servicio Técnico)**
  - **Sí** → **¿Tiene expediente correctivo abierto?**
    - **No** → Abrir Orden de Trabajo (Folio SIGEM).
    - **Sí** → **¿En qué etapa va el expediente?**
      - **Enviado a servicio técnico** → **¿El equipo retornó?**
        - **No** → Seguir el envío (N° de envío · responsable).
        - **Sí, sin su reporte** → Pendiente: gestionar el reporte de reparación.
        - **Sí, con su reporte** → Registrar Reporte de Servicio (reparación / diagnóstico externo).
      - **Falta resolver la compra** → Trato Directo / Compra Ágil → informe técnico → orden de compra.
      - **Ya reparado** → **¿Quedó operativo?**
        - **Sí** → Cerrar expediente.
        - **No** → Seguir abierto (cuenta días detenido).
  - **No** → **¿Le toca MP este mes y no tiene resultado?**
    - **Sí** → **¿Cómo resultó la mantención?**
      - **Realizada (`Si` / `Si-RA`)** → **¿Quedó gestión pendiente?**
        - **Sí** → Crear protocolo (Interno/Externo): Sí / No / Imprimir / Gestión.
        - **No** → MP en Borrador → oficial al recargar la carta.
      - **No se pudo (causal C1–C8)** → **¿Qué causal?**
        - **C2 / C3 / C4** → No se fija fecha; se registra al reintegrar el equipo.
        - **C1 / C5 / C6 / C7 / C8** → Reprogramar dentro de 30 días → generar → imprimir → 2 firmas → oficializar.
      - **`FS`** → No Operativo (fuera de servicio) · **`NU`** → No Ubicable · **`No`** → MP sin resultado por gestionar (no reprograma) · **`Baja`** → dar de baja (prioridad máxima).
    - **No** → **¿Tiene pendientes por resolver?** (5 tipos en 3 orígenes — ver `arbol-pendientes.html`)
      - **Correctivo · documental** → conseguir el Reporte de Servicio del proveedor → verificar → archivar.
      - **Preventiva · reprogramación** → generar · imprimir · 2 firmas (supervisor + jefe de equipos) · oficializar · cerrar.
      - **Preventiva · protocolo interno / externo** → completar el reporte de la MP (el externo lo llena el proveedor).
      - **Administrativo · otro** → gestión puntual sin campos extra.
      - **No** → ✓ Equipo al día.

---

## 3. Mantención Correctiva (OT · Folio SIGEM)

> Cada **registro** lista sus campos como `Campo · Tipo`. Tipos: Fecha · Texto · Texto largo · Lista · Casilla · Monto. Marcas: `(cond.)` = condicional (se exige según la decisión previa); `(opc.)` = opcional; el resto se registra de forma estándar. La **línea de compra** y los **dos cierres** son transversales: se comparten entre rutas.

**Tronco común**

- El equipo clínico falla.
- **Registro · Apertura — Orden de Trabajo (SIGEM)** (raíz del expediente):
  - Folio SIGEM · Texto
  - Fecha de la OT · Fecha
  - Técnico asignado · Lista
  - Estado inicial del equipo · Lista
  - Requerimiento / falla · Texto largo
  - N° de Inventario · Texto
  - Equipo · Texto
  - N° de Serie · Texto
  - Servicio clínico de origen · Lista
  - *Equipo, Serie y Servicio se autocompletan al elegir el N° de Inventario; Folio, Inventario y Serie se conservan como texto (ceros a la izquierda).*
- El jefe asigna la OT · el ingeniero intenta repararla en terreno (primer intento, siempre).
- **DECISIÓN 1 · ¿Se solucionó en terreno?**
  - **Sí** → **Registro · Reparación en terreno (cierre directo):**
    - Fecha de reparación / cierre · Fecha
    - Estado final (Operativo) · Lista
    - Descripción de la tarea realizada · Texto largo
    - → Cierre operativo de la OT.
  - **No** → **DECISIÓN 2 · ¿Qué requiere el equipo?** → el ingeniero deriva a UNA de las 4 rutas ↓.

**Ruta A · Repuestos** (Estado → No Operativo · pasa por la línea de compra)

- **Registro · Instalación del repuesto:**
  - Fecha reparación · Fecha
  - Descripción tarea · Texto largo
  - Estado final (Operativo) · Lista
- → Cierre.

**Ruta B · Visita técnica** (Estado → No Operativo · pasa por la línea de compra)

- **Registro · Visita del técnico externo:**
  - Fecha visita · Fecha
  - Proveedor / empresa · Lista
  - Ingeniero externo · Texto (opc.)
  - N° Reporte de Servicio · Texto
  - Fecha del reporte · Fecha
  - Archivado en carpeta · Casilla
  - ¿Reparó? · Lista
  - Descripción tarea · Texto largo (cond.)
  - Estado final · Lista (cond.)
  - Fecha de cierre · Fecha (cond.)
  - ¿Reporte recibido? · Casilla
  - Resp. conseguir reporte · Lista
  - Resp. administrativo · Lista
- **DECISIÓN 3 · ¿Reparó?**
  - **Sí** → Cierre.
  - **No** → cotiza el repuesto ↻ coordina otra visita (bucle).

**Ruta C · Servicio técnico** (Hoja de envío → estado Servicio Técnico · pasa por la línea de compra)

- **Registro · Envío y retorno del equipo:**
  - N° Hoja de Envío · Texto
  - Folio de la OT · Texto
  - Responsable que envía · Lista
  - Fecha de envío · Fecha
  - Empresa destino · Lista
  - N° Reporte de Servicio · Texto
  - Fecha del reporte · Fecha
  - Archivado en carpeta · Casilla
  - Fecha de retorno · Fecha
  - Folio guía de despacho · Texto
  - Estado al retorno · Lista
  - Decisión reevaluación · Lista (cond.)
  - ¿Reporte recibido? · Casilla
  - Resp. conseguir reporte · Lista
  - Resp. administrativo · Lista
- **DECISIÓN 4 · ¿Operativo al retorno?**
  - **Sí** → Cierre.
  - **No** → reevaluar: repuestos / nueva visita / reenvío / baja → reentra al árbol (bucle).

**Ruta D · Baja** (el equipo no tiene reparación)

- **Registro · Documento de baja:**
  - Folio documento de baja · Texto
  - Responsable interno · Lista
  - Fecha de la baja · Fecha
  - Motivo de la baja · Texto largo
- → Equipo retirado · fin. *Única ruta que no termina con equipo operativo.*

**Línea de compra** (subproceso compartido por A · B · C)

- **Registro 1 · Cotización:**
  - Fecha de cotización · Fecha
  - Proveedor cotizado · Lista
  - N° de cotización · Texto
  - Monto cotizado · Monto (opc.)
- **DECISIÓN 5 · ¿Trato Directo o Compra Ágil?** (depende de la empresa, no del monto: ¿es representante exclusiva de la marca?)
  - **Trato Directo** (empresa exclusiva) → requiere **Informe Técnico**:
    - N° Informe Técnico · Texto (cond.)
    - Fecha del informe · Fecha (cond.)
    - Responsable del informe · Lista (cond.)
  - **Compra Ágil** (no exclusiva) → sin informe; pasa directo a la OC.
- **Registro 2 · Orden de Compra** (Leslie tramita · Finanzas emite):
  - N° de OC · Texto
  - Fecha emisión OC · Fecha
  - Descripción de la OC · Texto largo (opc.)
  - Fecha envío a proveedor · Fecha

**Los dos cierres** (transversales · no siempre coinciden en el tiempo)

- **Cierre operativo** — el ingeniero interno cierra la OT en SIGEM cuando el equipo vuelve a funcionar (es lo que importa para el servicio clínico y suele ocurrir primero):
  - Fecha cierre OT · Fecha (obligatorio)
- **Cierre documental** — llega el Reporte de Servicio (de visita o de servicio técnico) y se archiva en la carpeta física:
  - ¿Reporte recibido? · Casilla
  - Resp. conseguir · Lista
  - Resp. administrativo · Lista
  - Si la OT ya está cerrada pero el reporte no llega → **pendiente documental**: operativamente cerrado, administrativamente abierto.

**Regla base:** todo el expediente cuelga del folio SIGEM · solo el ingeniero interno asignado cierra la OT · el Reporte de Servicio lo archiva Cristián Beltrán.

---

## 4. Mantenimiento Preventivo

- **¿Le toca MP a este equipo este mes?** (programación `prog` ∈ `X` / `R` / `RA` / `PM`)
  - **No programada** → sin acción (si es mes destino de una reprogramación, llegará marcado `R`).
  - **Sí** → registrar la MP del mes: guarda `P` (lo programado) y `R` (resultado); fecha · ejecutor · tipo.
    - **¿Tipo de mantenimiento?**
      - **Interno**
      - **Externo** → + ingeniero externo + empresa (se precargan los últimos usados).
    - **¿Qué resultado se registró? (`registro.R`)**
      - **Vacío** → **NO REGISTRADO** (aún no se gestiona) → queda como **MP sin resultado por gestionar** (visible en el Tablero).
      - **`No` (registrada como no realizada)** → queda como **MP sin resultado por gestionar**; se distingue del vacío porque ya se revisó. No reprograma por sí sola.
      - **`Si` o `Si-RA` (ejecutada)** → **¿El evento está en el maestro?**
        - **Sí** → **OFICIAL** · estado Operativo\*.
        - **No** → **BORRADOR** · estado Operativo\* (pasa a Oficial al aparecer en el maestro).
        - `Si-RA` cierra además la programación `RA` arrastrada del año anterior.
        - Luego: **¿Quedó gestión pendiente?** Sí → crear protocolo (Interno/Externo): Sí / No / Imprimir / Gestión (+ subtareas).
        - \* Con `Si` / `Si-RA`, el estado lo decide quien registra: Operativo por defecto, o No Operativo si lo marca.
      - **`Baja`** → **EQUIPO DE BAJA** (prioridad máxima: prevalece sobre cualquier otro evento del mes).
      - **`C1` `C5` `C6` `C7` `C8` — reprogramación sin falla** → `reprog30 = TRUE`: no cambia el estado, marca `R` en el mes siguiente + pendiente reprogramar ≤ 30 días → **Ciclo de Reprogramación** (§5).
      - **`C2` `C3` `C4` · `FS` · `NU` — equipo no disponible** → `reprog30 = FALSE`: esperar reintegro / reparación / localización. Estado: `C2`→En Servicio Técnico · `C3`→No Operativo · `C4`→Préstamo · `FS`→No Operativo · `NU`→No Ubicable.

---

## 5. Ciclo de Reprogramación (causal C1 · C5 · C6 · C7 · C8)

> Solo las causales **sin falla** entran aquí. `C2` / `C3` / `C4` (y `FS` / `NU`) **no** se reprograman: `reprog30 = FALSE`, se espera el reintegro del equipo.

- **¿Causal `C2` / `C3` / `C4`?**
  - **Sí** → no se fija fecha nueva; se registra al reintegrar el equipo.
  - **No (`C1` / `C5` / `C6` / `C7` / `C8`)** → `reprog30 = TRUE`: marca `R` en el **mes siguiente**, reprogramar ≤ 30 días (origen: `X` + causal · destino: `R`).
    - **¿Ya generaste el reporte de reprogramación?**
      - **No** → generar el reporte (lleva la causal y la fecha de la MP).
      - **Sí** → **¿Ya lo imprimiste?**
        - **No** → imprimir el reporte.
        - **Sí** → **¿Tiene las dos firmas? (supervisor de servicio + jefe de equipos médicos)**
          - **No** → conseguir las firmas (agrupar por servicio clínico ayuda).
          - **Sí** → escribir el código en el Excel (mes y columna que indica la app).
            - **Recargar la planilla: ¿aparece con su resultado?**
              - **Sí** → Oficial → cerrar el pendiente.
              - **No** → sigue en Borrador (revisar el código / mes).
