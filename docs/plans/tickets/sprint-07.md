## Sprint 7: Evaluacion y Reportes

---

### S7-T1: Migraciones para evaluacion

**Tipo:** feat
**Rama:** `feat/S7-T1-migraciones-evaluacion`

**Descripcion:**
Crear tablas: `evaluaciones_programa`, `anexos_transversales`, `indicador_anexo_transversal`.

**Criterios de aceptacion:**
- [ ] Migraciones completas
- [ ] Seeder para anexos transversales base (Genero, NNA, Cambio Climatico, Anticorrupcion)
- [ ] Pivote M:M entre indicadores y anexos transversales
- [ ] `evaluaciones_programa` con indice de desempeno y conteos de semaforo
- [ ] Actualizar documentacion del esquema

---

### S7-T2: Etiquetado de indicadores con Anexos Transversales

**Tipo:** feat
**Rama:** `feat/S7-T2-etiquetado-anexos-transversales`

**Descripcion:**
En la ficha del indicador (Sprint 4) o al importar, permitir etiquetar cada indicador con una o mas tematicas transversales.

**Criterios de aceptacion:**
- [ ] Checkboxes de anexos transversales en la ficha del indicador
- [ ] Guardado en tabla pivote
- [ ] Visible en la vista de MIR y en el panel de seguimiento

---

### S7-T3: Calculo del Indice de Desempeno General

**Tipo:** feat
**Rama:** `feat/S7-T3-indice-desempeno`

**Descripcion:**
Calcular el indice 0-100 por programa al cierre del ejercicio. Promedio ponderado del porcentaje de avance de metas, con mayor peso a niveles superiores.

**Pesos sugeridos:**
- Fin: 40%
- Proposito: 30%
- Componentes: 20%
- Actividades: 10%

**Criterios de aceptacion:**
- [ ] Comando artisan que calcula y guarda en `evaluaciones_programa`
- [ ] Conteo de semaforos (verde, amarillo, rojo, sin dato) incluido
- [ ] Pesos configurables (no hardcodeados)
- [ ] El calculo solo considera indicadores con `activo_seguimiento = true`

---

### S7-T4: Evaluacion por programa — Vista de cierre

**Tipo:** feat
**Rama:** `feat/S7-T4-evaluacion-programa`

**Descripcion:**
Pantalla de evaluacion integral al cierre del ejercicio fiscal para un programa especifico.

**Secciones:**
1. Resumen ejecutivo (alineacion, objetivo central)
2. Tablero de semaforos consolidado
3. Comparativa vs ejercicio anterior con tendencias
4. Analisis de desviaciones (justificaciones aprobadas, supuestos incumplidos)
5. Indicadores cronicos en rojo (2+ ejercicios)

**Criterios de aceptacion:**
- [ ] Vista completa con las 5 secciones
- [ ] Tendencia por indicador: mejoro / empeoro / estable
- [ ] Indicadores cronicos destacados visualmente
- [ ] Solo accesible con permiso `exportar_reportes`

---

### S7-T5: Validacion de logica vertical al cierre

**Tipo:** feat
**Rama:** `feat/S7-T5-logica-vertical-cierre`

**Descripcion:**
La IA analiza los semaforos por nivel y detecta rupturas en la cadena causal.

**Ejemplo de ruptura:**
- Actividades en verde + Componente en rojo = problema de diseno (la solucion no sirve)
- Componente en verde + Proposito en rojo = factores externos (supuestos no cumplidos)

**Criterios de aceptacion:**
- [ ] Analisis automatico al generar evaluacion
- [ ] Reporte de rupturas con explicacion por caso
- [ ] Sugerencia: problema de ejecucion vs problema de diseno
- [ ] La IA no emite juicios de valor sobre las UR

---

### S7-T6: Paneles de evaluacion transversal

**Tipo:** feat
**Rama:** `feat/S7-T6-evaluacion-transversal`

**Descripcion:**
Paneles que cruzan todos los programas del estado para dar vision global al planeador.

**Vistas:**
- Por Eje del PED: conteo de semaforos e indice promedio
- Por ODS: indicadores que contribuyen a cada ODS
- Por Unidad Responsable: desempeno por dependencia
- Por Anexo Transversal: filtro por tematica cruzando dependencias

**Criterios de aceptacion:**
- [ ] 4 vistas funcionales con filtros
- [ ] Ranqueo por indice de desempeno
- [ ] Datos derivados de la Matriz de Alineacion (herencia funcional)
- [ ] Solo accesible con permiso `exportar_reportes`

---

### S7-T7: Exportacion a PDF y Excel

**Tipo:** feat
**Rama:** `feat/S7-T7-exportacion-pdf-excel`

**Descripcion:**
Generar reportes exportables en formato PDF y Excel.

**Reportes:**
- MIR en formato oficial
- Fichas tecnicas de indicadores
- Reporte de avance trimestral por programa
- Reporte de evaluacion anual por programa
- Reporte transversal por Eje PED / ODS / Anexo

**Criterios de aceptacion:**
- [ ] Cada reporte incluye sello de tiempo y periodo evaluado
- [ ] PDF con formato profesional (encabezado institucional configurable)
- [ ] Excel con hojas separadas por seccion
- [ ] Descarga funcional desde la interfaz
- [ ] Solo accesible con permiso `exportar_reportes`

---

### S7-T8: Exportacion de datos abiertos con Diccionario

**Tipo:** feat
**Rama:** `feat/S7-T8-datos-abiertos`

**Descripcion:**
Exportar datos en CSV y JSON, empaquetados con un archivo de metadatos (Diccionario de Datos) que describe cada campo.

**Criterios de aceptacion:**
- [ ] Exportacion CSV con codificacion UTF-8
- [ ] Exportacion JSON estructurado
- [ ] Diccionario de datos generado automaticamente (nombre de campo, tipo, descripcion)
- [ ] Empaquetado en ZIP: datos + diccionario
- [ ] Cumple con requisitos de la Ley General de Transparencia

---