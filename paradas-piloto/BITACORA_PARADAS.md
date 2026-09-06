# BITÁCORA — Dashboard de Paradas TL1 / TL2 / TL3

**Proyecto:** Dashboard de Paradas — Laminación  
**Estado:** piloto local validado / preparación de publicación externa protegida  
**Última actualización:** 2026-09-06  
**Repositorio:** `Hidroraices/Ingles`  
**Independencia:** este proyecto es independiente de JARVIS / DEPROETAID. No usar ni modificar su backend, Supabase ni repositorios.

---

## 1. Objetivo

Construir un sistema para consultar y analizar las paradas de los trenes laminadores TL1, TL2 y TL3 combinando histórico consolidado y paradas del día, sin exigir trabajo adicional al operador.

El producto final debe permitir consulta gerencial dentro y fuera de planta, pero la publicación externa debe estar protegida y no debe convertir los archivos operativos ni la infraestructura de planta en recursos accesibles desde Internet.

---

## 2. Principio de seguridad del piloto

El piloto se divide en tres capas:

1. **Captura:** los agentes leen los reportes operativos en solo lectura.
2. **Consolidación:** una central combina histórico + eventos del día.
3. **Publicación protegida:** el usuario autorizado accede desde fuera de planta mediante un canal autenticado y cifrado.

Reglas obligatorias:

- No publicar los Excel fuente.
- No publicar rutas internas, credenciales, nombres de archivos operativos ni datos personales innecesarios.
- No abrir acceso entrante directo desde Internet hacia las PCs de los trenes.
- No almacenar datos productivos reales en GitHub.
- La publicación externa debe requerir identidad/autorización.
- El tráfico externo debe viajar cifrado.
- Para el piloto, preferir acceso privado sobre una red segura antes que exponer una URL pública.
- Registrar qué campos salen de planta y mantener una matriz de minimización de datos para elevar a TI/Seguridad de Información.

---

## 3. Fuente histórica consolidada

Durante el piloto se utiliza un Excel maestro corporativo sincronizado localmente.

Hoja fuente: `REGISTRO_BD`.

Solo se leen las columnas A:V. Las columnas posteriores contienen fórmulas y no forman parte de la ingestión.

**Regla de seguridad:** el maestro se trata como SOLO LECTURA. El proceso abre binariamente el archivo, valida que sea un XLSX/ZIP válido, crea una copia local validada y únicamente esa copia se abre con `openpyxl` en `read_only=True, data_only=True`. No usar Excel COM sobre el maestro.

### Incidente documentado

En pruebas V06/V07 el archivo maestro llegó a quedar ilegible y Excel reportó formato/extensión inválidos. Se recuperó mediante historial de versiones. A partir de este incidente se abandonó COM para el maestro y se implementó el esquema seguro de V08.

---

## 4. Base histórica validada en V08

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

La clasificación del tren se obtiene del prefijo de la O/P conforme a las reglas validadas en el código operativo. Los prefijos concretos se mantienen fuera de esta bitácora pública.

Cualquier valor que no cumpla una regla conocida se marca `SIN CLASIFICAR`; nunca se adivina.

En la carga completa actual: **0 sin clasificar**.

---

## 5. Reglas de negocio validadas

### Barras trabadas — BT

La fuente de verdad es `CLASIFICACIÓN`.

Se considera BT cuando la clasificación corresponde a las categorías operativas A o F validadas con el usuario.

Por tanto: **BT = A OR F**.

No inferir BT mediante búsqueda de palabras en descripción, causa o tipo.

### Parada >20 min

La regla vigente es estricta:

`duracion_min > 20`

No usar `>=20` para este radar.

### Radar gerencial

El dashboard conserva todo el universo de paradas y permite priorizar:

- Todas las paradas.
- BT.
- >20 min.
- BT o >20 min.

No se ocultan las paradas no críticas.

---

## 6. V08 — arquitectura vigente

V08 es la central local validada:

- FastAPI.
- SQLite local.
- Dashboard HTML.
- Excel maestro en solo lectura mediante copia local validada.
- sincronización histórica prevista 08:30 y 20:30;
- reintentos posteriores si falla una sincronización;
- actualización manual disponible;
- servidor LAN preparado para recibir agentes y servir el dashboard.

V08 no depende de JARVIS/DEPROETAID.

### Dashboard

Incluye:

- Fecha inicio / fecha fin.
- Planta / Tren.
- Producto.
- Producto dependiente del tren seleccionado.
- Filtros reactivos.
- KPIs de eventos, horas, BT, >20 min y toneladas B/T.
- Tabla Producto / O.P / Equipo.
- ranking de horas por equipo.
- detalle de eventos.

---

## 7. Integración de “paradas de hoy”

Se añadió una tabla separada `paradas_hoy` para que los reportes operativos puedan aparecer antes de la consolidación del Excel maestro.

Principio:

- `paradas` = histórico consolidado.
- `paradas_hoy` = información operacional aún no consolidada.
- el dashboard consulta ambos universos.
- cuando el mismo evento llega posteriormente al histórico, debe evitarse mostrarlo duplicado.

---

## 8. Agente TL1 — prueba real validada

El 06/09/2026 se probó el agente TL1 contra un reporte operacional real.

Resultado:

- eventos detectados: **4**;
- listos: **4**;
- incompletos: **0**;
- central recibió: **4**;
- insertó: **4**;
- rechazó: **0**;
- errores: **0**.

El agente no modifica el XLSM y no escribe directamente SQLite; envía los eventos a la central.

Después de la integración, TL1 pasó visualmente de 9,760 históricos a **9,764** eventos al incluir los cuatro eventos operativos, sin duplicarlos.

---

## 9. Fecha operativa y turno nocturno — hotfix validado

Las pruebas con archivos reales demostraron que el nombre del archivo no puede tomarse como fecha de verdad y que algunos campos de fecha pueden estar vacíos.

Regla validada:

### TURNO 1 — nocturno 20:00–08:00

Si la fecha operativa base es D:

- inicio >=20:00 → fecha calendario D;
- inicio <08:00 → fecha calendario D+1.

### TURNO 2

Conserva la fecha operativa base D.

### Fecha vacía

Si la fecha de un turno está vacía, el agente intenta heredar la fecha válida del otro turno del mismo XLSM.

**Nunca usar el nombre del archivo como fecha de verdad.**

### Reconciliación

La central reconcilia el evento operacional por origen físico `archivo + hoja + fila`, además de la huella lógica, para que una corrección de fecha no genere un duplicado.

Prueba posterior al hotfix:

- recibidos: **4**;
- insertados: **2**;
- actualizados: **2**;
- rechazados: **0**;
- total operacional: **4**;
- errores: **0**.

Validación visual: los cuatro eventos quedaron distribuidos correctamente entre dos fechas calendario por el cruce de medianoche del turno nocturno.

---

## 10. Reportes operativos y macro existente

Se revisó la macro de consolidación utilizada en los reportes operativos.

La macro actualmente:

- valida hojas de turno;
- revisa barras trabadas;
- valida análisis/corrección;
- consolida registros;
- genera PDF;
- envía correo mediante Outlook;
- puede generar el archivo operativo del día siguiente.

La nueva arquitectura NO depende de modificar esta macro para leer paradas. El agente debe ser observador de solo lectura.

---

## 11. Rutas operativas

Cada tren utiliza una ruta corporativa propia. Las letras y rutas concretas se mantienen fuera de esta bitácora pública.

Los agentes de cada tren deben leer únicamente su fuente operacional y enviar eventos a la central.

No deben construir el histórico ni modificar los XLSM.

---

## 12. Arquitectura objetivo

### Capa de captura

Cada tren tendrá un agente local que:

- identifica el reporte activo;
- lee los turnos;
- calcula fecha calendario correctamente;
- detecta eventos completos/incompletos;
- no modifica Excel;
- envía los eventos a la central.

### Capa central

Una PC central siempre encendida:

- recibe TL1/TL2/TL3;
- mantiene SQLite;
- consolida histórico + día;
- sirve el dashboard dentro de planta.

La central debe usar una fuente histórica corporativa estable y no depender de una ruta personal.

### Capa de publicación externa protegida

Para el piloto, la publicación externa debe realizarse mediante un canal privado autenticado y cifrado que no abra directamente la red de planta a Internet.

Objetivo de experiencia:

- usuario autorizado → accede desde celular/laptop fuera de planta;
- usuario no autorizado → no puede descubrir ni consultar los datos;
- archivos Excel y rutas internas → permanecen en planta;
- PCs TL1/TL2/TL3 → no reciben conexiones directas desde Internet.

Antes de una publicación corporativa definitiva, elevar el piloto a TI/Seguridad de Información con la matriz de campos y controles implementados.

---

## 13. Datos permitidos para publicación piloto

Por defecto, la capa externa debe limitarse a datos necesarios para análisis gerencial:

- fecha;
- tren;
- producto o categoría autorizada;
- O/P solo si TI/negocio la considera necesaria;
- equipo;
- duración;
- clasificación BT/no BT;
- indicador >20 min;
- causa/categoría de causa si está autorizada;
- toneladas B/T si se requieren en los KPIs.

Excluir por defecto de la publicación externa:

- nombres de trabajadores;
- rutas internas;
- nombres físicos de archivos;
- credenciales;
- información de configuración de red;
- campos del Excel que no aporten al análisis gerencial.

---

## 14. Pendientes inmediatos

1. Validar agentes TL2 y TL3 con archivos reales.
2. Dejar la central en una PC siempre encendida.
3. Implementar acceso externo privado, autenticado y cifrado para el piloto.
4. Validar acceso desde un equipo externo sin abrir acceso directo a las PCs de los trenes.
5. Activar registro de accesos y usuarios autorizados.
6. Validar la matriz de datos publicados y excluir datos personales/no necesarios.
7. Preparar resumen de arquitectura y controles para elevar a TI/Seguridad de Información.
8. Configurar inicio automático de central y agentes.
9. Reimplementar, después de estabilizar lo anterior, el radar de eventos repetitivos/acumulados.

---

## 15. Criterio de continuidad

No crear nuevas versiones por cada ajuste menor. Trabajar sobre el estado validado y generar backup antes de cambios de código.

Antes de cualquier modificación futura:

1. verificar esta bitácora;
2. verificar el código realmente desplegado;
3. verificar conteos de SQLite/dashboard;
4. preservar el Excel maestro en solo lectura;
5. no mezclar este proyecto con JARVIS/DEPROETAID;
6. no publicar información real fuera de planta sin los controles de acceso definidos.
