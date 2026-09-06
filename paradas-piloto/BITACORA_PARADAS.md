# BITÁCORA — Dashboard de Paradas TL1 / TL2 / TL3

**Proyecto:** Dashboard de Paradas — Aceros Arequipa  
**Estado:** piloto local validado / preparación para despliegue operativo  
**Última actualización:** 2026-09-06  
**Repositorio:** `Hidroraices/Ingles`  
**Independencia:** este proyecto es independiente de JARVIS / DEPROETAID. No usar ni modificar su backend, Supabase ni repositorios.

---

## 1. Objetivo

Construir un sistema local para consultar y analizar las paradas de los trenes laminadores TL1, TL2 y TL3, combinando:

1. histórico consolidado;
2. paradas del día todavía no consolidadas;
3. agentes operativos que lean los reportes XLSM de cada tren sin modificarlos;
4. una PC central que mantenga la base SQLite y publique el dashboard en la red local.

El dashboard debe responder rápidamente: qué ocurrió, en qué tren/equipo/producto, cuánto tiempo se perdió, si hubo barras trabadas, si la parada superó 20 minutos y cuál fue la causa.

---

## 2. Fuente histórica consolidada

Durante el piloto se utilizó el Excel maestro de OneDrive:

`D:\OneDrive - CORPORACIÓN ACEROS AREQUIPA SA\2.0 Laminación\1. Informes Laminación\1.0 BASE DE DATOS _ BO\Base de datos paradas.xlsx`

Hoja fuente: `REGISTRO_BD`.

Solo se leen las columnas A:V. Las columnas posteriores contienen fórmulas y no forman parte de la ingestión.

**Regla de seguridad:** el maestro se trata como SOLO LECTURA. El proceso abre binariamente el archivo, valida que sea un XLSX/ZIP válido, crea una copia local validada y únicamente esa copia se abre con `openpyxl` en `read_only=True, data_only=True`. No usar Excel COM sobre el maestro.

### Incidente documentado

En pruebas V06/V07 el archivo maestro llegó a quedar ilegible y Excel reportó formato/extensión inválidos. Se recuperó mediante historial de versiones de OneDrive. A partir de este incidente se abandonó COM para el maestro y se implementó el esquema seguro de V08.

---

## 3. Base histórica validada en V08

Carga completa validada el 06/09/2026:

- Excel: **35,747 registros**.
- SQLite: **35,747 registros**.
- Sin fecha: **0**.
- Sin clasificar: **0**.
- Una segunda sincronización produjo **0 nuevas** y **0 duplicadas**.

Distribución histórica validada antes de integrar `paradas_hoy`:

- TL1: **9,760**.
- TL2: **19,021**.
- TL3: **6,966**.

Rangos observados:

- TL1: 02/01/2023 – 27/08/2026.
- TL2: 03/01/2023 – 30/08/2026.
- TL3: 01/02/2025 – 31/08/2026.

### Regla de identificación del tren

Se obtiene desde el prefijo de O/P:

- TL1 → `32150...`
- TL2 → `32230...`
- TL3 → `3245...`
- cualquier otro prefijo → `SIN CLASIFICAR`; nunca adivinar.

En la carga completa actual: **0 sin clasificar**.

---

## 4. Reglas de negocio validadas

### Barras trabadas — BT

La fuente de verdad es `CLASIFICACIÓN`.

Se considera BT cuando la clasificación es:

- `A: Imprevista con b/t`
- `F: Imprevista con b/Chatarra`

Por tanto: **BT = A OR F**.

No inferir BT mediante búsqueda de palabras en descripción, causa o tipo.

### Parada >20 min

La regla vigente es estricta:

`duracion_min > 20`

No usar `>=20` para este radar.

### Radar gerencial

El dashboard conserva todo el universo de paradas y permite priorizar:

- Todas las paradas.
- BT · Clasificación A + F.
- >20 min.
- BT o >20 min.

No se ocultan las paradas no críticas.

---

## 5. V08 — arquitectura vigente

V08 reemplazó la dependencia de Supabase por arquitectura local:

- FastAPI.
- SQLite local.
- Dashboard HTML.
- Excel maestro en solo lectura mediante copia local validada.
- sincronización histórica prevista 08:30 y 20:30;
- reintentos posteriores si falla una sincronización;
- actualización manual disponible;
- servidor preparado en `0.0.0.0:8787`;
- acceso local actual: `http://127.0.0.1:8787/`.

V08 no depende de JARVIS/DEPROETAID.

### Dashboard

Incluye:

- Fecha inicio / fecha fin.
- Planta / Tren.
- Producto.
- Producto dependiente del tren seleccionado; no debe mostrar productos de otros trenes.
- Sin botón `Aplicar`: los filtros deben reaccionar directamente.
- KPIs de eventos, horas, BT, >20 min y toneladas B/T.
- Tabla Producto / O.P / Equipo.
- ranking de horas por equipo.
- detalle de eventos.

---

## 6. Integración de “paradas de hoy”

Se añadió una tabla separada `paradas_hoy` para que los reportes operativos puedan aparecer antes de la consolidación del Excel maestro.

Principio:

- `paradas` = histórico consolidado.
- `paradas_hoy` = información operacional aún no consolidada.
- el dashboard consulta ambos universos.
- cuando el mismo evento llega posteriormente al histórico, debe evitarse mostrarlo duplicado.

La integración añadió un endpoint de ingestión local para agentes y mantiene separado el histórico.

---

## 7. Agente TL1 — prueba real validada

Ruta operacional TL1:

`T:\2026`

El 06/09/2026 se probó el agente contra:

`T:\2026\Setiembre\06092026-domingo.xlsm`

Resultado real inicial:

- eventos detectados: **4**;
- listos: **4**;
- incompletos: **0**;
- central recibió: **4**;
- insertó: **4**;
- rechazó: **0**;
- errores: **0**;
- `total_paradas_hoy = 4`.

El agente no modifica el XLSM y no escribe directamente SQLite; envía los eventos a la central.

Después de la integración, TL1 pasó visualmente de 9,760 históricos a **9,764** eventos al incluir los cuatro eventos operativos, sin duplicarlos.

---

## 8. Fecha operativa y turno nocturno — hotfix validado

Las pruebas con archivos reales de setiembre demostraron que el nombre del archivo no puede tomarse como fecha de verdad y que algunos F7 pueden estar vacíos.

Se implementó la siguiente regla:

### TURNO 1 — nocturno 20:00–08:00

Si la fecha operativa base es D:

- inicio >=20:00 → fecha calendario D;
- inicio <08:00 → fecha calendario D+1.

### TURNO 2

Conserva la fecha operativa base D.

### F7 vacío

Si F7 de un turno está vacío, el agente intenta heredar la fecha válida del otro turno del mismo XLSM.

**Nunca usar el nombre del archivo como fecha de verdad.**

### Reconciliación

La central reconcilia el evento operacional por origen físico `archivo + hoja + fila`, además de la huella lógica, para que una corrección de fecha no genere un duplicado.

Prueba posterior al hotfix:

- recibidos: **4**;
- insertados: **2**;
- actualizados: **2**;
- rechazados: **0**;
- `total_paradas_hoy`: **4**;
- errores: **0**.

Por tanto, la corrección temporal no produjo ocho eventos.

### Validación visual definitiva

Para O/P `321500014413`, producto `REDONDO LISO MOLI 1105 65 MM X 7.00M`:

**05/09/2026 — TL1**
- 2 eventos.
- Equipo: Todo el tren.
- 2.92 h.

**06/09/2026 — TL1**
- 2 eventos.
- Equipo: Cizalla volante N.º 4.
- ~0.97 h.

Total operacional: **4 eventos / ~3.9 h**.

Esto valida el cruce de medianoche del turno nocturno.

---

## 9. Reportes operativos y macro existente

Se revisó la macro `ConsolidarParadasYGenerarYEnviarReportePDFinal2` utilizada en los reportes operativos.

La macro actualmente:

- valida hoja TURNO 1 / TURNO 2;
- revisa barras trabadas;
- valida análisis/corrección;
- puede completar cantidad B/T faltante con 1;
- copia registros a `Registro_Histórico.xlsx` / `REGISTRO_BD`;
- genera PDF;
- envía correo mediante Outlook;
- al cerrar TURNO 2 puede generar el archivo del día siguiente;
- limpia rangos del nuevo reporte.

La nueva arquitectura NO debe depender de modificar esta macro para leer paradas. El agente debe ser observador de solo lectura.

---

## 10. Rutas operativas previstas

- TL1 → `T:\2026`
- TL2 → `S:\2026`
- TL3 → `L:\2026`

Los agentes de cada tren deben leer únicamente su fuente operacional y enviar eventos a la central.

No deben construir el histórico ni modificar los XLSM.

---

## 11. Arquitectura objetivo para despliegue

### PC central — siempre encendida

Responsabilidades:

- FastAPI.
- SQLite.
- Dashboard.
- recibir eventos TL1/TL2/TL3.
- consolidar histórico desde una fuente estable.
- publicar `:8787` en la LAN.

La central no debe depender de una ruta personal como:

`D:\OneDrive - ...`

que solo existe en la PC del desarrollador/usuario.

Antes del despliegue definitivo se debe definir una fuente histórica accesible desde la PC central, preferentemente:

- sincronización corporativa de OneDrive/SharePoint bajo la cuenta de la PC central, o
- una ruta de red corporativa estable autorizada.

No es necesario compartir el OneDrive personal con las PCs TL1/TL2/TL3: esos equipos solo necesitan leer sus rutas T:/S:/L: y alcanzar la IP/puerto de la central.

### PCs TL1 / TL2 / TL3

Cada una tendrá un agente local que:

- identifica el reporte activo;
- lee TURNO 1/TURNO 2;
- calcula fecha calendario correctamente;
- detecta eventos completos/incompletos;
- no modifica Excel;
- envía los eventos a la central;
- posteriormente deberá alertar localmente al operador sobre información crítica/incompleta.

---

## 12. Pendientes inmediatos

1. Construir paquete definitivo de PC central.
2. Definir la ubicación definitiva del Excel maestro para que la central pueda sincronizarlo sin depender de la PC personal.
3. Construir agentes definitivos TL1, TL2 y TL3 usando T:/S:/L:.
4. Configurar dirección/IP de la central en los agentes.
5. Configurar Windows Firewall para acceso LAN a TCP 8787.
6. Configurar inicio automático del servicio central y agentes.
7. Definir frecuencia de lectura/envío operacional.
8. Incorporar alerta local de reporte/parada incompleta.
9. Probar conectividad desde cada PC de tren hacia la central.
10. Probar dashboard desde una PC distinta a la central.
11. Validar TL2 y TL3 con archivos reales antes de declararlos productivos.
12. Endurecer identidad de eventos para escenarios de inserción/reordenamiento de filas del Excel.
13. Reimplementar, después de estabilizar lo anterior, el radar de eventos repetitivos/acumulados del diseño anterior.

---

## 13. Criterio de continuidad

No crear nuevas versiones por cada ajuste menor. Trabajar sobre el estado validado y generar backup antes de cambios de código.

Antes de cualquier modificación futura:

1. verificar esta bitácora;
2. verificar el código realmente desplegado;
3. verificar conteos de SQLite/dashboard;
4. preservar el Excel maestro en solo lectura;
5. no mezclar este proyecto con JARVIS/DEPROETAID.
