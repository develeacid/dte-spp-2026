## Sprint 5: Importacion de Programas Existentes

---

### S5-T1: Parser de Markdown para MIR existentes

**Tipo:** feat
**Rama:** `feat/S5-T1-parser-markdown-mir`

**Descripcion:**
Importador que acepta un archivo Markdown con la estructura de una MIR existente y lo mapea a las tablas del sistema.

**Formato esperado:** segun plantilla definida en el documento de diseno (con Frecuencia, Tipo/Dimension, Linea Base).

**Criterios de aceptacion:**
- [ ] Acepta Markdown con estructura de MIR
- [ ] Parsea los 4 niveles (Fin, Proposito, Componentes, Actividades)
- [ ] Extrae indicadores con todos los campos de ficha tecnica
- [ ] Previsualizacion antes de confirmar importacion
- [ ] Manejo de errores de formato con mensajes claros

---

### S5-T2: Parser de CSV/Excel para MIR existentes

**Tipo:** feat
**Rama:** `feat/S5-T2-parser-csv-excel-mir`

**Descripcion:**
Importador alternativo que acepta archivos CSV o Excel. Mapeo de columnas configurable.

**Criterios de aceptacion:**
- [ ] Acepta CSV y Excel (.xlsx)
- [ ] Paso de mapeo: usuario asocia columnas del archivo con campos del sistema
- [ ] Previsualizacion del resultado
- [ ] Misma logica de creacion de registros que el parser Markdown

---

### S5-T3: Diagnostico de completitud

**Tipo:** feat
**Rama:** `feat/S5-T3-diagnostico-completitud`

**Descripcion:**
Tras la importacion, generar reporte que clasifica huecos en criticos y menores.

**Huecos criticos (bloquean seguimiento del indicador):**
- Variables de formula no identificadas
- Frecuencia de medicion ausente
- Linea base y anio base faltantes

**Huecos menores (no bloquean):**
- Sintaxis del resumen narrativo
- Supuestos faltantes
- Medios de verificacion incompletos

**Criterios de aceptacion:**
- [ ] Reporte visual: tabla con cada elemento, estado (OK/faltante/critico) y accion sugerida
- [ ] Indicadores con huecos criticos marcados como `activo_seguimiento = false`
- [ ] Indicadores completos marcados como `activo_seguimiento = true`
- [ ] Enlace directo para completar cada hueco

---

### S5-T4: Flujo asistido para completar huecos

**Tipo:** feat
**Rama:** `feat/S5-T4-completar-huecos`

**Descripcion:**
Interfaz que guia al usuario por los huecos pendientes de un programa importado. La IA sugiere contenido basado en lo ya importado.

**Criterios de aceptacion:**
- [ ] Lista de huecos pendientes ordenados por prioridad (criticos primero)
- [ ] Al seleccionar un hueco, formulario de captura con sugerencia de IA
- [ ] Al completar huecos criticos de un indicador, se activa automaticamente para seguimiento
- [ ] Progreso visible: barra o porcentaje de completitud

---

### S5-T5: Vinculacion de programas importados con cascada de planes

**Tipo:** feat
**Rama:** `feat/S5-T5-vinculacion-importados-planes`

**Descripcion:**
Si el programa importado no tiene alineacion, el sistema sugiere vinculos usando la Matriz de Alineacion y busqueda semantica. Si ya tiene, valida contra la Matriz.

**Criterios de aceptacion:**
- [ ] Deteccion: programa sin alineacion → flujo de sugerencia
- [ ] Deteccion: programa con alineacion → flujo de validacion
- [ ] Sugerencias basadas en similitud semantica del Resumen Narrativo
- [ ] Advertencias si la alineacion existente contradice la Matriz

---

### S5-T6: Calendarizacion de metas al activar programa

**Tipo:** feat
**Rama:** `feat/S5-T6-calendarizacion-metas`

**Descripcion:**
Al activar un programa (nuevo o importado), el sistema genera los registros de `metas_periodo` distribuyendo la meta anual en los periodos segun la frecuencia de cada indicador.

**Criterios de aceptacion:**
- [ ] Generacion automatica de periodos con fechas de inicio/fin
- [ ] Distribucion de meta anual (el usuario puede ajustar la distribucion)
- [ ] Etiquetas legibles: "Q1 2026", "Ene 2026", "S1 2026"
- [ ] Validacion: la suma de metas por periodo = meta anual (para variables acumulables)

---