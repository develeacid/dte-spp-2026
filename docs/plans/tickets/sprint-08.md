## Sprint 8: Orquestacion IA (Transversal)

---

### S8-T1: Refactorizar LlmService con patrones avanzados (Alcance Reducido y Enfocado)

**Tipo:** refactor
**Rama:** `refactor/S8-T1-llm-service-avanzado`

**Descripcion:**
Consolidar el servicio de IA creado en S3-T8 con todos los metodos del dominio. **Nota importante:** gracias al motor de poka-yoke implementado en S4-T11, la IA ya NO necesita detectar errores de Tipo, Dimension o Frecuencia de indicadores, los cuales son bloqueados por el sistema antes de persistirse. El LlmService se concentra en auditoria semantica y causal de mayor valor.

**Metodos consolidados (alcance exclusivo de la IA):**
- `suggestNarrativeSyntax($nivel, $texto)` — Validacion sintactica SHCP (semantica, no estructura)
- `validateCremaa($indicador)` — Validacion CREMAA letra por letra (criterios de calidad del indicador)
- `validateVerticalLogic($mir)` — Congruencia causal entre niveles (causa-efecto)
- `validateHorizontalLogic($nivel)` — Indicador mide el objetivo; medio verifica el indicador
- `extractVariables($formula)` — Extraccion de variables de formulas matematicas
- `generateJustification($avance, $supuestos, $historial)` — Borradores de justificacion acotados a supuestos (Col. 4), magnitud de desviacion e historial
- `suggestAlignment($texto, $nivel)` — Sugerencias de alineacion semantica con la cascada de planes
- `detectCausalBreaks($evaluacion)` — Deteccion de rupturas causales entre niveles al cierre del ejercicio

**Lo que la IA ya NO valida (delegado al sistema en S4-T11):**
- Tipo de indicador por nivel (bloqueado por Form Request)
- Dimension de medicion por nivel (filtrado en UI y validado en backend)
- Frecuencia de medicion por nivel (dropdowns restringidos y validacion backend)

**Criterios de aceptacion:**
- [ ] Todos los metodos centralizados en un solo servicio
- [ ] Cada metodo tiene su prompt template versionado
- [ ] Los prompts NO incluyen instrucciones para validar Tipo, Dimension o Frecuencia (son responsabilidad del sistema)
- [ ] Tests unitarios con mocks para cada metodo
- [ ] Logging unificado de todas las interacciones
- [ ] Rate limiting por usuario/sesion

---

### S8-T2: Pipeline de embeddings batch

**Tipo:** feat
**Rama:** `feat/S8-T2-embeddings-batch`

**Descripcion:**
Comando artisan para generar/regenerar embeddings en batch para todos los registros que no los tengan o necesiten actualizacion.

**Criterios de aceptacion:**
- [ ] `php artisan embeddings:generate` procesa todos los registros pendientes
- [ ] Procesamiento en chunks para no saturar el API
- [ ] Reporte al finalizar: N generados, M fallidos, X omitidos
- [ ] Flag `--force` para regenerar todos
- [ ] Schedulable para ejecucion nocturna

---

### S8-T3: Monitoreo y metricas de uso de IA

**Tipo:** feat
**Rama:** `feat/S8-T3-monitoreo-ia`

**Descripcion:**
Dashboard para el administrador que muestre metricas de uso del servicio de IA.

**Metricas:**
- Llamadas por dia/semana/mes
- Tokens consumidos
- Tiempo promedio de respuesta
- Tasa de error
- Uso por tipo de operacion (validacion, sugerencia, justificacion)
- Uso por usuario/UR

**Criterios de aceptacion:**
- [ ] Tabla de logs de IA con campos: tipo, usuario, tokens, duracion, status
- [ ] Dashboard con graficas basicas
- [ ] Solo accesible para admin