## Sprint 2: Cascada de Planes y Matriz de Alineacion

---

### S2-T1: Migraciones y modelos para catalogos ODS

**Tipo:** feat
**Rama:** `feat/S2-T1-catalogos-ods`

**Descripcion:**
Crear migraciones, modelos Eloquent y seeders para la Agenda 2030. Tablas: `ods_objetivos` y `ods_metas`. Estos son catalogos inmutables (no editables por el usuario).

**Campos clave:**
- `ods_objetivos`: id, numero (1-17), nombre, descripcion, embedding (vector 1536)
- `ods_metas`: id, ods_objetivo_id (FK), clave ("1.1", "1.2"), descripcion, embedding

**Criterios de aceptacion:**
- [ ] Migraciones con `up()` y `down()` completos
- [ ] Columnas `embedding` de tipo `vector(1536)` creadas correctamente
- [ ] Modelos con relaciones: `OdsObjetivo hasMany OdsMeta`
- [ ] Seeder carga los 17 ODS y sus metas desde archivo Markdown fuente
- [ ] `migrate:fresh --seed` ejecuta sin errores
- [ ] Actualizar documentacion del esquema

---

### S2-T2: Migraciones y modelos para catalogos PND

**Tipo:** feat
**Rama:** `feat/S2-T2-catalogos-pnd`

**Descripcion:**
Crear migraciones, modelos y seeders para el Plan Nacional de Desarrollo. Tablas: `pnd_ejes`, `pnd_objetivos`, `pnd_estrategias`. Catalogos inmutables.

**Relaciones Eloquent:**
- `PndEje hasMany PndObjetivo`
- `PndObjetivo hasMany PndEstrategia`
- `PndObjetivo belongsTo PndEje`

**Criterios de aceptacion:**
- [ ] Migraciones con embeddings en cada tabla
- [ ] Seeder carga datos del PND vigente desde Markdown
- [ ] Relaciones Eloquent definidas y funcionales
- [ ] `migrate:fresh --seed` sin errores
- [ ] Actualizar documentacion del esquema

---

### S2-T3: Migraciones y modelos para PED

**Tipo:** feat
**Rama:** `feat/S2-T3-modelos-ped`

**Descripcion:**
Crear migraciones y modelos para toda la jerarquia del Plan Estatal de Desarrollo. Estructura anidada con Adjacency List.

**Tablas:**
- `ped_planes` (id, nombre, nivel_gobierno, periodo_inicio, periodo_fin, activo)
- `ped_ejes` (id, ped_plan_id, numero, nombre, descripcion, embedding)
- `ped_temas` (id, ped_eje_id, numero, nombre, descripcion, embedding)
- `ped_objetivos_estrategicos` (id, ped_tema_id, clave, descripcion, embedding)
- `ped_estrategias` (id, ped_objetivo_estrategico_id, clave, descripcion, embedding)
- `ped_lineas_accion` (id, ped_estrategia_id, clave, descripcion, embedding)

**Relaciones Eloquent (cascada HasMany):**
- Plan → Ejes → Temas → Objetivos → Estrategias → Lineas de Accion

**Criterios de aceptacion:**
- [ ] 6 migraciones con `up()` y `down()`
- [ ] 6 modelos con `$fillable`, relaciones `hasMany`/`belongsTo`
- [ ] Constraint: solo un `ped_planes` puede tener `activo = true`
- [ ] Seeder de ejemplo con datos ficticios (1 plan, 3 ejes, 2 temas por eje)
- [ ] `migrate:fresh --seed` sin errores
- [ ] Actualizar documentacion del esquema

---

### S2-T4: Migraciones y modelos para Programas Derivados

**Tipo:** feat
**Rama:** `feat/S2-T4-programas-derivados`

**Descripcion:**
Crear tablas para programas sectoriales, especiales, institucionales y regionales que emanan del PED.

**Tablas:**
- `programas_derivados` (id, ped_plan_id, tipo [enum], nombre, descripcion)
- `programas_derivados_objetivos` (id, programa_derivado_id, clave, descripcion, embedding)

**Criterios de aceptacion:**
- [ ] Migraciones completas
- [ ] Enum: sectorial, especial, institucional, regional
- [ ] Relaciones: ProgramaDerivado belongsTo PedPlan, hasMany Objetivos
- [ ] Seeder de ejemplo
- [ ] Actualizar documentacion del esquema

---

### S2-T5: Tablas pivote para Matriz de Alineacion

**Tipo:** feat
**Rama:** `feat/S2-T5-matriz-alineacion`

**Descripcion:**
Crear las tablas pivote muchos-a-muchos que representan la Matriz de Alineacion entre niveles de la cascada de planes.

**Tablas:**
- `alineacion_ped_pnd` (ped_objetivo_estrategico_id ↔ pnd_objetivo_id)
- `alineacion_pnd_ods` (pnd_objetivo_id ↔ ods_meta_id)
- `alineacion_linea_programa_derivado` (ped_linea_accion_id ↔ programa_derivado_objetivo_id)

**Todas con:**
- Constraint UNIQUE en la combinacion de FKs
- Timestamps

**Criterios de aceptacion:**
- [ ] 3 migraciones con indices unicos compuestos
- [ ] Relaciones `belongsToMany` configuradas en los modelos correspondientes
- [ ] Seeder que crea alineaciones de ejemplo
- [ ] Test: no se permite duplicar una alineacion
- [ ] Test: al consultar una Linea de Accion, se puede recorrer toda la cadena hasta ODS via relaciones Eloquent
- [ ] Actualizar documentacion del esquema

---

### S2-T6: CRUD de PED con interfaz Livewire

**Tipo:** feat
**Rama:** `feat/S2-T6-crud-ped-livewire`

**Descripcion:**
Crear la interfaz de administracion del PED. El planeador debe poder capturar y editar toda la jerarquia: Plan → Ejes → Temas → Objetivos Estrategicos → Estrategias → Lineas de Accion.

**Componentes Livewire:**
- Vista de arbol colapsable para navegar la jerarquia
- Formularios inline para crear/editar nodos
- Validacion de campos requeridos

**Criterios de aceptacion:**
- [ ] Solo accesible con permiso `gestionar_catalogos`
- [ ] Arbol colapsable renderiza toda la jerarquia del PED
- [ ] CRUD completo en cada nivel (crear, editar, eliminar)
- [ ] Eliminar un nodo padre advierte sobre hijos dependientes
- [ ] Validacion: campos requeridos, longitud maxima
- [ ] Responsive (Tailwind)

---

### S2-T7: CRUD de Programas Derivados con interfaz Livewire

**Tipo:** feat
**Rama:** `feat/S2-T7-crud-programas-derivados`

**Descripcion:**
Interfaz para gestionar los programas derivados (sectoriales, especiales, institucionales, regionales) y sus objetivos.

**Criterios de aceptacion:**
- [ ] Listado de programas derivados filtrado por tipo
- [ ] CRUD de programas derivados y sus objetivos
- [ ] Solo accesible con permiso `gestionar_catalogos`
- [ ] Vinculacion al PED activo

---

### S2-T8: Interfaz de Matriz de Alineacion

**Tipo:** feat
**Rama:** `feat/S2-T8-interfaz-matriz-alineacion`

**Descripcion:**
Crear la interfaz para que el planeador configure la Matriz de Alineacion, vinculando niveles de PED con PND y ODS. La interfaz debe mostrar la cascada y permitir crear/eliminar vinculos muchos-a-muchos.

**Criterios de aceptacion:**
- [ ] Vista que muestra los 3 niveles de alineacion (PED↔PND, PND↔ODS, Linea↔Programa Derivado)
- [ ] Interfaz para agregar/quitar vinculos (selects o drag-and-drop)
- [ ] Al vincular, se muestra la cadena completa resultante (Linea → Estrategia → Obj. PED → PND → ODS)
- [ ] Solo accesible con permiso `gestionar_catalogos`
- [ ] Test: herencia automatica funciona (al seleccionar una Linea de Accion, el sistema muestra los ODS heredados)

---

### S2-T9: Importador de PED desde Markdown

**Tipo:** feat
**Rama:** `feat/S2-T9-importador-markdown-ped`

**Descripcion:**
Crear un importador que acepte un archivo Markdown con la estructura del PED y lo convierta en registros de la base de datos. El usuario debe confirmar la estructura sugerida antes de guardar.

**Formato Markdown esperado:**
```markdown
# Plan Estatal de Desarrollo 2022-2027

## Eje 1: Seguridad y Justicia
### Tema 1.1: Prevencion del delito
#### Objetivo 1.1.1: Reducir la incidencia delictiva juvenil
##### Estrategia 1.1.1.1: Programas de intervencion temprana
- Linea de Accion 1.1.1.1.1: Implementar talleres en zonas de riesgo
- Linea de Accion 1.1.1.1.2: Crear centros comunitarios
```

**Criterios de aceptacion:**
- [ ] Acepta archivo Markdown con la estructura definida
- [ ] Parsea y muestra previsualizacion del arbol resultante
- [ ] Usuario puede editar/corregir antes de confirmar
- [ ] Al confirmar, crea todos los registros en cascada
- [ ] Maneja errores de formato con mensajes claros
- [ ] Solo accesible con permiso `gestionar_catalogos`

---

### S2-T10: Pipeline de generacion de embeddings

**Tipo:** feat
**Rama:** `feat/S2-T10-pipeline-embeddings`

**Descripcion:**
Crear el servicio y los jobs de Laravel para generar embeddings vectoriales al guardar o actualizar textos de planes/objetivos. Los embeddings se almacenan en las columnas `vector(1536)` de pgvector.

**Implementacion:**
- Servicio `EmbeddingService` que encapsula la llamada al API del LLM
- Job `GenerateEmbedding` que se despacha a la cola Redis
- Observer en los modelos de planes que despacha el job al crear/actualizar `descripcion`

**Criterios de aceptacion:**
- [ ] Servicio genera embedding y lo guarda en la columna correspondiente
- [ ] Job se ejecuta de forma asincrona (no bloquea la interfaz)
- [ ] Al crear un registro de PED, el embedding se genera automaticamente
- [ ] Al actualizar la descripcion, el embedding se regenera
- [ ] Manejo de errores: si el API falla, el registro se guarda sin embedding y se reintenta
- [ ] Test con mock del API de embeddings

---

### S2-T11: Servicio de busqueda semantica por similitud

**Tipo:** feat
**Rama:** `feat/S2-T11-busqueda-semantica`

**Descripcion:**
Crear servicio que realiza busquedas por similitud de cosenos usando pgvector. Se usara en la Matriz de Alineacion para sugerir correspondencias entre niveles de planes.

**Implementacion:**
- Metodo `findSimilar($texto, $tabla, $limite)` que:
  1. Genera embedding del texto de entrada
  2. Ejecuta query con `ORDER BY embedding <=> $vector LIMIT $limite`
  3. Retorna los N registros mas similares con su score

**Criterios de aceptacion:**
- [ ] Servicio retorna los registros mas similares ordenados por score
- [ ] Funciona contra todas las tablas con embeddings (ODS, PND, PED)
- [ ] Indices HNSW creados para performance
- [ ] Test: buscar "reducir pobreza" retorna ODS 1 como primer resultado
- [ ] Configurable: umbral minimo de similitud, limite de resultados

---