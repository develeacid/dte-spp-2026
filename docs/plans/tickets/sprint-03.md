## Sprint 3: Metodologia de Marco Logico (Etapas 1-4)

---

### S3-T1: Migracion y modelo para Programas Presupuestarios

**Tipo:** feat
**Rama:** `feat/S3-T1-modelo-programa-presupuestario`

**Descripcion:**
Crear la tabla cabecera `programas_presupuestarios` y la tabla de alineacion `programa_alineacion`.

**Campos programas_presupuestarios:**
- id, team_id (FK), clave_programa, nombre, ejercicio_fiscal, origen (nuevo/importado), estado (borrador/activo/cerrado), created_by (FK), timestamps, softDeletes

**Criterios de aceptacion:**
- [ ] Migraciones completas con FK a teams y users
- [ ] Modelo con scope de team_id (aislamiento por UR)
- [ ] Relaciones: belongsTo Team, hasMany MirNiveles, hasMany Arboles
- [ ] Indice en team_id y ejercicio_fiscal
- [ ] Solo creacion con permiso `crear_programa`

---

### S3-T2: Migraciones y modelos para Arboles (Problema/Objetivos)

**Tipo:** feat
**Rama:** `feat/S3-T2-modelos-arboles`

**Descripcion:**
Crear tablas `arboles` y `arbol_nodos` con estructura Adjacency List para representar los arboles de problemas y objetivos.

**Campos arbol_nodos:**
- id, arbol_id (FK), parent_id (FK recursiva), tipo_nodo (enum), descripcion, nodo_origen_id (FK, vinculo problema→objetivo), orden, timestamps

**Criterios de aceptacion:**
- [ ] Enum tipo_nodo cubre: problema_central, causa_directa, causa_indirecta, efecto_directo, efecto_indirecto, objetivo_central, medio_directo, medio_indirecto, fin_directo, fin_indirecto
- [ ] Relacion recursiva: `ArbolNodo hasMany children`, `belongsTo parent`
- [ ] `nodo_origen_id` permite vincular un nodo del arbol de objetivos con su nodo origen del arbol de problemas
- [ ] Indice en parent_id
- [ ] Actualizar documentacion del esquema

---

### S3-T3: Migraciones y modelos para Alternativas

**Tipo:** feat
**Rama:** `feat/S3-T3-modelos-alternativas`

**Descripcion:**
Crear tablas `alternativas` y `alternativa_nodos` (pivote) para representar las estrategias candidatas y su seleccion.

**Criterios de aceptacion:**
- [ ] Alternativa belongsTo ProgramaPresupuestario
- [ ] Pivote vincula alternativa con nodos del arbol de objetivos
- [ ] Campo `seleccionada` (boolean) marca la alternativa elegida
- [ ] Campo `justificacion_seleccion` documenta la decision

---

### S3-T4: Interfaz Etapa 1 — Definicion del problema

**Tipo:** feat
**Rama:** `feat/S3-T4-etapa1-definicion-problema`

**Descripcion:**
Componente Livewire para capturar el problema central del programa. La IA sugiere mejoras de redaccion.

**Flujo:**
1. Usuario escribe la situacion no deseada en un textarea
2. Boton "Validar con IA" envia el texto al LLM via job en cola
3. IA retorna sugerencia de redaccion (sin verbos, sin soluciones, claro y concreto)
4. Usuario acepta la sugerencia o conserva su texto original
5. Se guarda como nodo `problema_central` en el arbol de problemas

**Criterios de aceptacion:**
- [ ] Textarea con contador de caracteres
- [ ] Boton de asistencia IA (no obligatorio, el usuario puede omitirlo)
- [ ] Respuesta de IA se muestra como sugerencia editable
- [ ] Al confirmar, se crea el arbol de tipo `problema` y el nodo central
- [ ] Estado guardado: el usuario puede salir y retomar

---

### S3-T5: Interfaz Etapa 2 — Arbol del problema

**Tipo:** feat
**Rama:** `feat/S3-T5-etapa2-arbol-problema`

**Descripcion:**
Interfaz visual de arbol para capturar causas y efectos del problema central. La IA sugiere nodos adicionales.

**Componentes:**
- Visualizacion de arbol (problema central al centro, causas abajo, efectos arriba)
- Botones para agregar causa directa/indirecta y efecto directo/indirecto
- Formulario inline para describir cada nodo
- Boton "Sugerir con IA" que propone causas/efectos adicionales

**Criterios de aceptacion:**
- [ ] Arbol visual renderiza correctamente con nodos colapsables
- [ ] CRUD de nodos: agregar, editar, eliminar, reordenar
- [ ] Causas indirectas se anidan bajo causas directas
- [ ] Efectos indirectos se anidan bajo efectos directos
- [ ] IA sugiere nodos y valida que no sean soluciones disfrazadas
- [ ] Estado persistente entre sesiones

---

### S3-T6: Interfaz Etapa 3 — Arbol de objetivos

**Tipo:** feat
**Rama:** `feat/S3-T6-etapa3-arbol-objetivos`

**Descripcion:**
Transformacion automatica del arbol de problemas a arbol de objetivos. Cada nodo negativo se convierte en positivo. La IA sugiere la redaccion.

**Flujo:**
1. Al entrar, el sistema genera automaticamente el arbol de objetivos desde el de problemas
2. Cada nodo muestra: texto original (tachado) → texto sugerido (positivo)
3. Usuario revisa y aprueba/edita cada transformacion
4. Los nodos se vinculan via `nodo_origen_id`

**Criterios de aceptacion:**
- [ ] Transformacion automatica genera todos los nodos
- [ ] Vinculacion `nodo_origen_id` entre arbol de problemas y objetivos
- [ ] Usuario puede editar cada nodo transformado
- [ ] IA sugiere redaccion positiva cuando la transformacion automatica es ambigua
- [ ] Vista lado a lado: arbol de problemas vs arbol de objetivos

---

### S3-T7: Interfaz Etapa 4 — Seleccion de alternativas

**Tipo:** feat
**Rama:** `feat/S3-T7-etapa4-alternativas`

**Descripcion:**
Interfaz para agrupar los medios del arbol de objetivos en estrategias candidatas, evaluar su viabilidad y seleccionar la alternativa final.

**Flujo:**
1. Se muestran los medios (directos e indirectos) del arbol de objetivos
2. Usuario agrupa medios en 2-3 alternativas (drag-and-drop o checkboxes)
3. IA evalua viabilidad tecnica, institucional y presupuestal de cada agrupacion
4. Usuario selecciona la alternativa ganadora y documenta justificacion
5. Las ramas no seleccionadas se marcan como "podadas"

**Criterios de aceptacion:**
- [ ] Medios del arbol de objetivos listados para agrupacion
- [ ] Crear/nombrar alternativas y asignarles nodos
- [ ] IA genera evaluacion de viabilidad por alternativa
- [ ] Seleccion de alternativa final con justificacion obligatoria
- [ ] Visualizacion: ramas podadas se muestran tachadas/grises
- [ ] Solo los nodos de la alternativa seleccionada pasan a la Etapa 5

---

### S3-T8: Servicio centralizado de llamadas al LLM

**Tipo:** feat
**Rama:** `feat/S3-T8-servicio-llm`

**Descripcion:**
Crear la clase `LlmService` que encapsula todas las llamadas al modelo de lenguaje. Todas las interacciones de IA del sistema pasan por este servicio.

**Metodos iniciales:**
- `suggest($prompt, $context): string` — Sugerencia de texto
- `validate($text, $rules): ValidationResult` — Validacion contra reglas
- `transform($text, $instruction): string` — Transformacion de texto

**Criterios de aceptacion:**
- [ ] Servicio registrado en el Service Container de Laravel
- [ ] Llamadas ejecutadas via jobs en cola Redis
- [ ] Rate limiting configurable por usuario/sesion
- [ ] Manejo de errores: timeout, API caido, respuesta invalida
- [ ] Logging de todas las llamadas (prompt, respuesta, tokens, duracion)
- [ ] Interfaz facilmente mockeable para tests

---