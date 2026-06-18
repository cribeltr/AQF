# Sistema de Gestión MP 2026 — Aplicación HTML

Aplicación web **autónoma** (un solo archivo HTML) que reemplaza la captura de
datos del libro Excel `Sistema_Gestion_MP2026_Integrado.xlsx`. En lugar de
escribir directamente en las celdas, el equipamiento clínico del HHHA se
gestiona desde formularios con listas desplegables, autocompletado y tableros
que se calculan en tiempo real.

## Cómo usarla

1. Abre **`Sistema_Gestion_MP2026.html`** con doble clic en cualquier navegador
   (Chrome, Edge, Firefox). No necesita servidor ni instalación.
2. Trabaja desde el menú lateral. Todo lo que ingresas se **guarda solo** en el
   navegador (`localStorage`).
3. Para respaldar o compartir, usa los botones de la barra superior:
   - **📊 Exportar Excel** — genera un `.xlsx` con todas las hojas (requiere
     conexión la primera vez para cargar la librería; si no hay, descarga JSON).
   - **⬇ Respaldo JSON** / **⬆ Importar** — copia de seguridad completa.
   - **↺ Restablecer** — vuelve a los datos originales del Excel.
   - Cada módulo además exporta su tabla a **CSV** (abre en Excel).

> La app arranca precargada con los datos del Excel original: 893 equipos,
> catálogos, 1.104 registros de MP, la carta MP 2026 (888 equipos), pendientes
> y los registros de las 7 hojas operativas.

## Funciones incluidas (equivalentes al Excel)

| Módulo | Función |
|---|---|
| **Panel / Tablero** | KPIs en tiempo real: casos abiertos, no operativos, en servicio técnico, bajas, cerrados, pendientes documentales, avance del programa preventivo y cola de pendientes. Lista de casos correctivos abiertos. |
| **Ficha de Equipo** | Elige un N° de inventario y muestra identidad, estado actual e historial completo (OT, visitas, envíos, compras, terreno, bajas, registro MP). |
| **Pendientes** | Cola de acción con situación, estado, prioridad, responsables y días de atraso calculados. |
| **1 · Órdenes de Trabajo** | Folio SIGEM, técnico, requerimiento. Al elegir el inventario autocompleta equipo, serie y servicio. |
| **2 · Línea de Compra** | Cotizaciones, informes técnicos y órdenes de compra ligadas al folio. |
| **3 · Repuestos** | Reparaciones con repuestos y estado final. |
| **4 · Visitas Técnicas** | Visitas de proveedor, resultado, reporte y responsables. |
| **5 · Servicio Técnico (Envíos)** | Envíos a empresa externa, retorno y decisión de reevaluación. |
| **6 · Bajas** | Documentos de baja. |
| **7 · Reparación en Terreno** | Reparaciones en sitio. |
| **MP Carta 2026** | Carta de ejecución por equipo y mes (X/E/C2/C3/R/B) con totales y % de avance. |
| **Registro MP** | Registro detallado de cada mantención, con descripción del resultado automática. |
| **Equipos** | Maestro de equipos (alimenta todos los autocompletados). |
| **Catálogos** | Listas editables: ingenieros, empresas, servicios, estados, etc. |

Las fórmulas del Excel (VLOOKUP de identidad, COUNTIF/SUMPRODUCT de los
tableros, totales de la carta MP) están reimplementadas en JavaScript y se
recalculan automáticamente al ingresar o editar datos.

## Reconstruir el archivo

El HTML se genera combinando la plantilla con los datos del Excel:

```bash
pip install openpyxl
python3 build.py        # lee template.html + source.xlsx -> Sistema_Gestion_MP2026.html
```

- `template.html` — interfaz y lógica (sin datos).
- `source.xlsx` — libro Excel original (fuente de los datos semilla).
- `build.py` — extrae los datos e inyecta el JSON en la plantilla.
