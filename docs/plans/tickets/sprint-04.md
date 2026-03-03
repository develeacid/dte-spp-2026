## Sprint 4: MIR y Validaciones

---

### S4-T1: Interfaz Etapa 5 — Estructura Analitica del Programa (EAP)

**Tipo:** feat
**Rama:** `feat/S4-T1-etapa5-eap`

**Descripcion:**
Mapeo automatico del arbol de objetivos filtrado a la estructura Fin/Proposito/Componentes/Actividades.

**Criterios de aceptacion:**
- [ ] Mapeo automatico: fines → Fin, objetivo central → Proposito, medios directos → Componentes, medios indirectos → Actividades
- [ ] Solo incluye nodos de la alternativa seleccionada
- [ ] Vista previa de la estructura resultante
- [ ] Usuario puede ajustar el mapeo manualmente
- [ ] Validacion de logica vertical basica

---

### S4-T2: Migraciones y modelos para MIR

**Tipo:** feat
**Rama:** `feat/S4-T2-modelos-mir`

**Descripcion:**
Crear tabla `mir_niveles` y `mir_versiones` (snapshots).

**Campos mir_niveles:**
- id, programa_presupuestario_id, tipo_nivel (enum), parent_id, resumen_narrativo, supuestos, arbol_nodo_id, orden
- FKs de alineacion: ped_objetivo_estrategico_id, programa_derivado_objetivo_id, ped_linea_accion_id

**Criterios de aceptacion:**
- [ ] Enum: fin, proposito, componente, actividad
- [ ] Relacion recursiva parent_id (Actividad → Componente)
- [ ] Trazabilidad a arbol_nodos via arbol_nodo_id
- [ ] mir_versiones guarda snapshots en JSONB
- [ ] Actualizar documentacion del esquema

---

### S4-T3: Migraciones y modelos para Indicadores

**Tipo:** feat
**Rama:** `feat/S4-T3-modelos-indicadores`

**Descripcion:**
Crear tablas: `indicadores`, `indicador_variables`, `catalogo_unidades_medida`, `medios_verificacion`, `cremaa_validaciones`. Los campos `tipo`, `dimension` y `frecuencia` deben almacenarse de forma que la capa de negocio pueda validar las restricciones por nivel.

**Restricciones por nivel (poka-yoke metodologico):**
- `tipo`: Solo editable en Componente. Fin/Proposito = Estrategico (forzado). Actividad = Gestion (forzado)
- `dimension`: Fin=solo Eficacia | Proposito=Eficacia,Eficiencia | Componente=Eficacia,Eficiencia,Calidad | Actividad=Eficacia,Eficiencia,Economia
- `frecuencia`: Fin=Anual/Bianual/Sexenal | Proposito=Semestral/Anual | Componente=Trimestral/Semestral | Actividad=Mensual/Trimestral

**Criterios de aceptacion:**
- [ ] `indicadores` con todos los campos de ficha tecnica (nombre, formula_texto, tipo, dimension, frecuencia, sentido, linea_base, meta, rangos semaforo)
- [ ] Enum `tipo`: estrategico, gestion
- [ ] Enum `dimension`: eficacia, eficiencia, calidad, economia
- [ ] Enum `frecuencia`: mensual, trimestral, semestral, anual, bianual, sexenal
- [ ] Validacion a nivel de Model: `validateTipoForNivel()`, `validateDimensionForNivel()`, `validateFrecuenciaForNivel()`
- [ ] `indicador_variables` con campo `simbolo` (string 5) para evaluacion matematica
- [ ] `catalogo_unidades_medida` como catalogo con clave y nombre (seeder CONAC)
- [ ] `medios_verificacion` vinculados a indicador
- [ ] `cremaa_validaciones` con 6 campos boolean + observacion por letra
- [ ] Relaciones Eloquent completas
- [ ] Actualizar documentacion del esquema

---

### S4-T4: Interfaz de captura MIR 4x4

**Tipo:** feat
**Rama:** `feat/S4-T4-interfaz-mir-4x4`

**Descripcion:**
Componente Livewire que renderiza la matriz 4x4. Columna 1 (Resumen Narrativo) viene prellenada desde la EAP. El usuario captura las columnas 2, 3 y 4. La UI aplica las reglas condicionales de la Seccion 5.5 del plan (el motor de poka-yoke se implementa en el ticket S4-T11).

**Criterios de aceptacion:**
- [ ] Matriz visual 4 filas x 4 columnas
- [ ] Col 1 prellenada, editable
- [ ] Col 2: formulario de indicador integrado con las restricciones dinamicas de S4-T11
- [ ] Col 3: formulario de medios de verificacion
- [ ] Col 4: textarea para supuestos
- [ ] Multiples componentes e indicadores por nivel (agregar/quitar)
- [ ] Campo de asignacion de UR Coadyuvante visible en filas de Componente y Actividad (ver S4-T12)
- [ ] Guardado automatico (no se pierde trabajo)
- [ ] Solo accesible con permiso `editar_mir`

---

### S4-T5: Validacion sintactica SHCP del Resumen Narrativo

**Tipo:** feat
**Rama:** `feat/S4-T5-validacion-sintaxis-shcp`

**Descripcion:**
La IA audita que el Resumen Narrativo de cada nivel cumpla con la sintaxis obligatoria de la SHCP.

**Reglas:**
- Fin: "Contribuir a [Impacto] mediante [Solucion]"
- Proposito: "[Poblacion] + [Verbo presente/participio] + [Condicion]"
- Componente: "[Bien/Servicio] + [Participio -ado/-ido]"
- Actividad: "[Sustantivo deverbal] + [Complemento]"

**Criterios de aceptacion:**
- [ ] Al guardar/validar un nivel, la IA analiza la sintaxis
- [ ] Resultado: cumple / no cumple con explicacion especifica
- [ ] Sugerencia de reescritura que respeta la formula
- [ ] El usuario decide si acepta la sugerencia
- [ ] No bloquea el guardado (es advertencia, no error)

---

### S4-T6: Validacion CREMAA desglosada

**Tipo:** feat
**Rama:** `feat/S4-T6-validacion-cremaa`

**Descripcion:**
Al crear o editar un indicador, la IA evalua los 6 criterios CREMAA individualmente y guarda el resultado en `cremaa_validaciones`.

**Criterios de aceptacion:**
- [ ] Boton "Validar CREMAA" en la ficha del indicador
- [ ] Resultado visual: 6 letras, cada una en verde (cumple) o rojo (no cumple)
- [ ] Cada letra tiene explicacion especifica del fallo
- [ ] Sugerencias de mejora por criterio
- [ ] Resultado se persiste en la tabla cremaa_validaciones
- [ ] No bloquea guardado

---

### S4-T7: Validacion de logica vertical y horizontal

**Tipo:** feat
**Rama:** `feat/S4-T7-validacion-logica`

**Descripcion:**
Validacion automatica de la coherencia interna de la MIR.

**Logica vertical:**
- Actividades producen Componentes
- Componentes logran Proposito
- Proposito contribuye al Fin
- La IA analiza si la relacion causal es coherente

**Logica horizontal:**
- El indicador realmente mide el objetivo (Resumen Narrativo)
- El medio de verificacion puede proporcionar los datos del indicador
- La frecuencia del medio coincide con la frecuencia del indicador

**Criterios de aceptacion:**
- [ ] Boton "Validar MIR completa"
- [ ] Reporte de validacion con hallazgos por nivel
- [ ] Cada hallazgo clasificado: error critico / advertencia / sugerencia
- [ ] No bloquea guardado pero se muestra prominentemente

---

### S4-T8: Alineacion automatica MIR ↔ Matriz de Alineacion

**Tipo:** feat
**Rama:** `feat/S4-T8-alineacion-mir`

**Descripcion:**
Al crear una MIR, el sistema sugiere la alineacion con la cascada de planes usando la Matriz de Alineacion y busqueda semantica.

**Flujo:**
1. Al capturar el Fin, el sistema busca por similitud los Objetivos Estrategicos del PED mas cercanos
2. Al seleccionar uno, hereda automaticamente: PND y ODS
3. Al capturar Componentes/Actividades, sugiere Lineas de Accion
4. Se llenan las FKs de alineacion en mir_niveles

**Criterios de aceptacion:**
- [ ] Sugerencias de alineacion usando busqueda semantica
- [ ] Herencia automatica al seleccionar un nivel
- [ ] Usuario puede aceptar, rechazar o buscar manualmente
- [ ] FKs de alineacion guardadas en mir_niveles
- [ ] Vista de cadena completa: Linea de Accion → ... → ODS

---

### S4-T9: Extraccion de variables de formulas

**Tipo:** feat
**Rama:** `feat/S4-T9-extraccion-variables`

**Descripcion:**
Al capturar la formula de un indicador, la IA extrae las variables y crea registros en `indicador_variables` con nombre y simbolo.

**Ejemplo:**
- Formula: `(Alumnos inscritos / Egresados secundaria) x 100`
- Variables extraidas: A = "Alumnos inscritos", B = "Egresados secundaria"

**Criterios de aceptacion:**
- [ ] Al escribir la formula, boton "Extraer variables"
- [ ] IA identifica variables y asigna simbolos (A, B, C...)
- [ ] Se crean registros en indicador_variables
- [ ] Usuario puede editar nombres, simbolos y clasificar comportamiento (acumulable/continua)
- [ ] Se selecciona unidad de medida del catalogo CONAC
- [ ] Si la IA no puede extraer variables, se marca como hueco critico

---

### S4-T10: Snapshots y versionado de MIR

**Tipo:** feat
**Rama:** `feat/S4-T10-snapshots-mir`

**Descripcion:**
Implementar el sistema de snapshots que permite crear borradores alternos de la MIR sin destruir el trabajo actual.

**Criterios de aceptacion:**
- [ ] Boton "Crear snapshot" guarda estado completo en mir_versiones (JSONB)
- [ ] Listado de versiones con etiqueta y fecha
- [ ] Restaurar una version reemplaza la MIR actual (con confirmacion)
- [ ] Al regresar a etapas anteriores (arbol), advertencia de impacto en MIR
- [ ] Opcion: "Crear borrador alterno" vs "Sobreescribir actual"

---

### S4-T11: Form Requests y UI dinamica condicional — Motor Poka-Yoke del Indicador

**Tipo:** feat
**Rama:** `feat/S4-T11-form-requests-condicional-indicador`

**Descripcion:**
Implementar el motor de reglas condicionales de la Seccion 5.5 del plan. El formulario de la ficha tecnica del indicador debe mostrar campos y opciones diferentes segun el nivel (fila) de la MIR que se esta editando. La validacion se aplica tanto en el frontend (Livewire/Alpine.js) como en el backend (Laravel Form Requests).

**Reglas a implementar por nivel:**

| Campo | Fin | Proposito | Componente | Actividad |
|-------|-----|-----------|------------|----------|
| Tipo de indicador | Estrategico (bloqueado) | Estrategico (bloqueado) | Editable (Estrategico/Gestion) | Gestion (bloqueado) |
| Dimensiones disponibles | Solo Eficacia | Eficacia, Eficiencia | Eficacia, Eficiencia, Calidad | Eficacia, Eficiencia, Economia |
| Frecuencias disponibles | Anual, Bianual, Sexenal | Semestral, Anual | Trimestral, Semestral | Mensual, Trimestral |

**Criterios de aceptacion:**
- [ ] Livewire: el campo Tipo se renderiza como label de solo lectura en Fin/Proposito/Actividad, y como select en Componente
- [ ] Alpine.js: el dropdown de Dimension filtra dinamicamente sus opciones segun la propiedad `nivel` del indicador
- [ ] Alpine.js: el dropdown de Frecuencia muestra solo las opciones validas segun el nivel
- [ ] Laravel Form Request `StoreIndicadorRequest`: regla condicional que valida `tipo` segun `nivel` antes de persistir
- [ ] Laravel Form Request: valida que `dimension` este en el set permitido para el `nivel` recibido
- [ ] Laravel Form Request: valida que `frecuencia` este en el set permitido para el `nivel` recibido
- [ ] Tests de Form Request: intentar guardar Economia en nivel Fin retorna error de validacion 422
- [ ] Tests de Form Request: intentar guardar frecuencia Mensual en nivel Fin retorna error 422
- [ ] Tests de Form Request: guardar Calidad en nivel Componente es exitoso
- [ ] UI no muestra mensajes de error cuando las opciones invalidas simplemente no aparecen en el dropdown

---

### S4-T12: Asignacion de UR Coadyuvante por Componente/Actividad

**Tipo:** feat
**Rama:** `feat/S4-T12-asignacion-ur-coadyuvante`

**Descripcion:**
En la interfaz de la MIR (Columna 1 del nivel Componente y Actividad), el Planeador puede asignar una UR Coadyuvante responsable de ese nivel especifico. El sistema registra la relacion en `programa_team` y en `mir_niveles.team_id`, activando los permisos del middleware Multi-UR.

**Criterios de aceptacion:**
- [ ] Selector de UR (teams disponibles en el sistema) visible en filas de Componente y Actividad
- [ ] El campo es opcional: si no se asigna, la responsabilidad recae en la UR Coordinadora
- [ ] Al asignar una UR: se actualiza `mir_niveles.team_id` y se crea/actualiza registro en `programa_team` con `rol = coadyuvante`
- [ ] Al quitar la asignacion: se elimina `mir_niveles.team_id` (null) y se verifica si la UR tiene otros componentes; si no, se elimina de `programa_team`
- [ ] Solo el Planeador puede asignar/quitar UR Coadyuvante (`permiso editar_mir`)
- [ ] La UR Coadyuvante asignada aparece como etiqueta visible junto al Componente en la vista de MIR
- [ ] Notificacion opcional al operador de la UR Coadyuvante cuando se le asigna un componente

---