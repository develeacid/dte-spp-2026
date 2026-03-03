# Modelado Físico de Base de Datos

**Fecha:** 2026-02-28
**Motor:** PostgreSQL + pgvector
**ORM:** Eloquent (Laravel)
**Convenciones:** snake_case, plurales en inglés para tablas, `id` autoincremental como PK

---

## Capa 1: Identidad y aislamiento (Sprint 1 — Jetstream + Spatie)

Las tablas de Jetstream se crean automáticamente al instalar el paquete. Se listan aquí como referencia para entender las FK que dependen de ellas.

### Tablas Jetstream (auto-generadas)

```
users
├── id (PK)
├── name
├── email (unique)
├── password
├── two_factor_secret
├── two_factor_recovery_codes
├── two_factor_confirmed_at
├── current_team_id (FK → teams.id)  -- Team/UR activo
├── profile_photo_path
├── remember_token
├── timestamps

teams
├── id (PK)
├── user_id (FK → users.id)  -- Creador/dueño del team
├── name                      -- Nombre de la UR/Dependencia
├── personal_team (boolean)
├── timestamps

team_user (pivote)
├── id (PK)
├── team_id (FK → teams.id)
├── user_id (FK → users.id)
├── role (string)  -- Rol dentro del team (Jetstream básico)

team_invitations
├── id (PK)
├── team_id (FK → teams.id)
├── email
├── role
├── timestamps

sessions
├── id (PK, string)
├── user_id (FK → users.id, nullable, index)
├── ip_address (string, nullable)
├── user_agent (text, nullable)
├── payload (longText)
├── last_activity (integer, index)

personal_access_tokens
├── id (PK)
├── tokenable_type
├── tokenable_id
├── name
├── token (unique)
├── abilities (text, nullable)
├── last_used_at
├── expires_at
├── timestamps
```

### Tablas Spatie (auto-generadas)

```
roles
├── id (PK)
├── name (string)        -- planeador, operador, admin
├── guard_name (string)
├── timestamps

permissions
├── id (PK)
├── name (string)        -- gestionar_catalogos, crear_programa, etc.
├── guard_name (string)
├── timestamps

model_has_roles (pivote)
├── role_id (FK → roles.id)
├── model_type
├── model_id

model_has_permissions (pivote)
├── permission_id (FK → permissions.id)
├── model_type
├── model_id

role_has_permissions (pivote)
├── permission_id (FK → permissions.id)
├── role_id (FK → roles.id)
```

### Extensión custom: campos adicionales en `teams`

```
-- Migración custom para extender la tabla teams de Jetstream

ALTER teams ADD:
├── clave_ur (string, nullable, unique)     -- Clave oficial de la UR
├── titular (string, nullable)              -- Nombre del titular
├── tipo_ur (enum: sustantiva, apoyo)       -- Rol en ejecución presupuestal
├── activa (boolean, default: true)
```

---

## Capa 2: Catálogos de referencia inmutables (Sprint 2)

### Agenda 2030 — ODS

```
ods_objetivos
├── id (PK)
├── numero (tinyInteger, unique)            -- 1 a 17
├── nombre (string)                         -- "Fin de la pobreza"
├── descripcion (text, nullable)
├── embedding (vector(1536), nullable)      -- pgvector
├── timestamps

ods_metas
├── id (PK)
├── ods_objetivo_id (FK → ods_objetivos.id)
├── clave (string)                          -- "1.1", "1.2", etc.
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps
```

### Plan Nacional de Desarrollo (PND)

```
pnd_ejes
├── id (PK)
├── numero (tinyInteger)
├── nombre (string)
├── descripcion (text, nullable)
├── periodo_inicio (year)
├── periodo_fin (year)
├── embedding (vector(1536), nullable)
├── timestamps

pnd_objetivos
├── id (PK)
├── pnd_eje_id (FK → pnd_ejes.id)
├── clave (string)
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps

pnd_estrategias
├── id (PK)
├── pnd_objetivo_id (FK → pnd_objetivos.id)
├── clave (string)
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps
```

---

## Capa 3: Planeación estatal editable (Sprint 2)

### Plan Estatal de Desarrollo (PED)

```
ped_planes
├── id (PK)
├── nombre (string)                         -- "Plan Estatal de Desarrollo 2022-2027"
├── nivel_gobierno (enum: estatal, municipal)
├── periodo_inicio (year)
├── periodo_fin (year)
├── activo (boolean, default: true)         -- Solo uno activo a la vez
├── timestamps

ped_ejes
├── id (PK)
├── ped_plan_id (FK → ped_planes.id)
├── numero (tinyInteger)
├── nombre (string)                         -- "Seguridad y Justicia"
├── descripcion (text, nullable)
├── embedding (vector(1536), nullable)
├── timestamps

ped_temas
├── id (PK)
├── ped_eje_id (FK → ped_ejes.id)
├── numero (tinyInteger)
├── nombre (string)                         -- "Prevención del delito"
├── descripcion (text, nullable)
├── embedding (vector(1536), nullable)
├── timestamps

ped_objetivos_estrategicos
├── id (PK)
├── ped_tema_id (FK → ped_temas.id)
├── clave (string)
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps

ped_estrategias
├── id (PK)
├── ped_objetivo_estrategico_id (FK → ped_objetivos_estrategicos.id)
├── clave (string)
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps

ped_lineas_accion
├── id (PK)
├── ped_estrategia_id (FK → ped_estrategias.id)
├── clave (string)
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps
```

### Programas Derivados

```
programas_derivados
├── id (PK)
├── ped_plan_id (FK → ped_planes.id)
├── tipo (enum: sectorial, especial, institucional, regional)
├── nombre (string)
├── descripcion (text, nullable)
├── timestamps

programas_derivados_objetivos
├── id (PK)
├── programa_derivado_id (FK → programas_derivados.id)
├── clave (string)
├── descripcion (text)
├── embedding (vector(1536), nullable)
├── timestamps
```

---

## Capa 4: Matriz de alineación (Sprint 2)

Relaciones muchos-a-muchos entre los niveles de la cascada.

```
-- Alineación PED ↔ PND
alineacion_ped_pnd
├── id (PK)
├── ped_objetivo_estrategico_id (FK → ped_objetivos_estrategicos.id)
├── pnd_objetivo_id (FK → pnd_objetivos.id)
├── timestamps
    UNIQUE(ped_objetivo_estrategico_id, pnd_objetivo_id)

-- Alineación PND ↔ ODS
alineacion_pnd_ods
├── id (PK)
├── pnd_objetivo_id (FK → pnd_objetivos.id)
├── ods_meta_id (FK → ods_metas.id)
├── timestamps
    UNIQUE(pnd_objetivo_id, ods_meta_id)

-- Alineación Líneas de Acción ↔ Programas Derivados
alineacion_linea_programa_derivado
├── id (PK)
├── ped_linea_accion_id (FK → ped_lineas_accion.id)
├── programa_derivado_objetivo_id (FK → programas_derivados_objetivos.id)
├── timestamps
    UNIQUE(ped_linea_accion_id, programa_derivado_objetivo_id)
```

**Herencia automática:** Cuando se vincula una Línea de Acción a un programa presupuestario, Eloquent recorre:

```
Línea de Acción → Estrategia → Obj. Estratégico → [alineacion_ped_pnd]
→ PND Objetivo → [alineacion_pnd_ods] → ODS Meta → ODS Objetivo
```

Esto se implementa como accessors/relationships en los modelos Eloquent, no como tablas adicionales.

---

## Capa 5: Programas presupuestarios y MIR (Sprint 4-5)

### Programa Presupuestario (cabecera)

```
programas_presupuestarios
├── id (PK)
├── team_id (FK → teams.id)                 -- UR responsable (Team de Jetstream)
├── clave_programa (string, unique per ejercicio)
├── nombre (string)
├── ejercicio_fiscal (year)
├── origen (enum: nuevo, importado)
├── estado (enum: borrador, activo, cerrado)
├── created_by (FK → users.id)
├── timestamps
├── softDeletes

-- Alineación directa del programa
programa_alineacion
├── id (PK)
├── programa_presupuestario_id (FK → programas_presupuestarios.id)
├── ped_eje_id (FK → ped_ejes.id)
├── ped_linea_accion_id (FK → ped_lineas_accion.id, nullable)
├── programa_derivado_objetivo_id (FK → programas_derivados_objetivos.id, nullable)
├── ods_meta_id (FK → ods_metas.id, nullable)  -- Herencia o manual
├── timestamps
```

### MML: Árboles y alternativas (Sprint 3)

```
arboles
├── id (PK)
├── programa_presupuestario_id (FK → programas_presupuestarios.id)
├── tipo (enum: problema, objetivos)
├── version (integer, default: 1)           -- Snapshots
├── es_actual (boolean, default: true)
├── timestamps

arbol_nodos
├── id (PK)
├── arbol_id (FK → arboles.id)
├── parent_id (FK → arbol_nodos.id, nullable)  -- Adjacency List
├── tipo_nodo (enum: problema_central, causa_directa, causa_indirecta,
│              efecto_directo, efecto_indirecto,
│              objetivo_central, medio_directo, medio_indirecto,
│              fin_directo, fin_indirecto)
├── descripcion (text)
├── nodo_origen_id (FK → arbol_nodos.id, nullable)  -- Vínculo problema→objetivo
├── orden (integer)
├── timestamps

alternativas
├── id (PK)
├── programa_presupuestario_id (FK → programas_presupuestarios.id)
├── nombre (string)
├── seleccionada (boolean, default: false)
├── justificacion_seleccion (text, nullable)
├── timestamps

alternativa_nodos (pivote: qué nodos del árbol incluye cada alternativa)
├── id (PK)
├── alternativa_id (FK → alternativas.id)
├── arbol_nodo_id (FK → arbol_nodos.id)
```

### MIR: Matriz 4x4

```
mir_niveles
├── id (PK)
├── programa_presupuestario_id (FK → programas_presupuestarios.id)
├── tipo_nivel (enum: fin, proposito, componente, actividad)
├── parent_id (FK → mir_niveles.id, nullable)  -- Actividad → Componente
├── resumen_narrativo (text)
├── supuestos (text, nullable)
├── arbol_nodo_id (FK → arbol_nodos.id, nullable)  -- Trazabilidad a la EAP
├── orden (integer)
├── -- FK de alineación por nivel:
├── ped_objetivo_estrategico_id (FK, nullable)  -- Solo para tipo=fin
├── programa_derivado_objetivo_id (FK, nullable) -- Solo para tipo=proposito
├── ped_linea_accion_id (FK, nullable)          -- Para componente/actividad
├── timestamps

mir_versiones (snapshots)
├── id (PK)
├── programa_presupuestario_id (FK → programas_presupuestarios.id)
├── datos_snapshot (jsonb)              -- Serialización completa de la MIR
├── etiqueta (string, nullable)         -- "Borrador alternativo A"
├── created_by (FK → users.id)
├── timestamps
```

### Indicadores (Columna 2)

```
indicadores
├── id (PK)
├── mir_nivel_id (FK → mir_niveles.id)
├── nombre (string)                     -- Max ~10 palabras
├── formula_texto (text)                -- Expresión legible "(A / B) × 100"
├── tipo_indicador (enum: estrategico, gestion)
├── dimension (enum: eficacia, eficiencia, calidad, economia)
├── frecuencia_medicion (enum: mensual, trimestral, semestral, anual)
├── sentido (enum: ascendente, descendente)
├── linea_base_valor (decimal(15,4), nullable)
├── linea_base_anio (year, nullable)
├── meta_anual (decimal(15,4))
├── -- Rangos de semaforización (porcentaje de desviación):
├── rango_verde_min (decimal(5,2))      -- ej. -15.00
├── rango_verde_max (decimal(5,2))      -- ej. +15.00
├── rango_amarillo_min (decimal(5,2))   -- ej. -25.00
├── rango_amarillo_max (decimal(5,2))   -- ej. -15.01
├── -- Todo fuera de verde y amarillo = rojo
├── activo_seguimiento (boolean, default: false)  -- Se activa cuando no hay huecos críticos
├── timestamps

indicador_variables
├── id (PK)
├── indicador_id (FK → indicadores.id)
├── nombre (string)                     -- "Alumnos inscritos"
├── simbolo (string, length: 5)         -- 'A', 'B', 'V1' — para evaluación matemática (MathExecutor)
├── descripcion (text, nullable)
├── tipo_comportamiento (enum: acumulable, continua)
├── unidad_medida_id (FK → catalogo_unidades_medida.id)
├── orden_en_formula (tinyInteger)      -- Posición en la fórmula
├── timestamps
```

### Catálogo de unidades de medida (CONAC)

```
catalogo_unidades_medida
├── id (PK)
├── clave (string, unique)              -- "PER", "MXN", "PCT", etc.
├── nombre (string)                     -- "Personas", "Pesos", "Porcentaje"
├── timestamps
```

### Medios de verificación (Columna 3)

```
medios_verificacion
├── id (PK)
├── indicador_id (FK → indicadores.id)
├── nombre_documento (string)
├── area_generadora (string)
├── periodicidad (string)
├── ubicacion_url (string, nullable)
├── timestamps
```

### Validación CREMAA

```
cremaa_validaciones
├── id (PK)
├── indicador_id (FK → indicadores.id)
├── claro (boolean, nullable)
├── claro_observacion (text, nullable)
├── relevante (boolean, nullable)
├── relevante_observacion (text, nullable)
├── economico (boolean, nullable)
├── economico_observacion (text, nullable)
├── monitoreable (boolean, nullable)
├── monitoreable_observacion (text, nullable)
├── adecuado (boolean, nullable)
├── adecuado_observacion (text, nullable)
├── aportacion_marginal (boolean, nullable)
├── aportacion_marginal_observacion (text, nullable)
├── validado_por (FK → users.id, nullable)
├── timestamps
```

---

## Capa 6: Seguimiento (Sprint 6)

### Calendarización de metas

```
metas_periodo
├── id (PK)
├── indicador_id (FK → indicadores.id)
├── ejercicio_fiscal (year)
├── periodo_numero (tinyInteger)        -- 1, 2, 3... según frecuencia
├── periodo_etiqueta (string)           -- "Q1 2026", "Ene 2026", "S1 2026"
├── fecha_inicio (date)
├── fecha_fin (date)
├── meta_periodo (decimal(15,4))        -- Meta distribuida para este periodo
├── timestamps
    UNIQUE(indicador_id, ejercicio_fiscal, periodo_numero)
```

### Captura de avances

```
avances
├── id (PK)
├── meta_periodo_id (FK → metas_periodo.id)
├── estado (enum: en_captura, en_revision, observado, aprobado)
├── resultado_calculado (decimal(15,4), nullable)  -- Resultado de la fórmula
├── semaforo_sugerido (enum: verde, amarillo, rojo, nullable)
├── semaforo_final (enum: verde, amarillo, rojo, nullable)
├── justificacion_ia (text, nullable)   -- Borrador generado por IA
├── justificacion_final (text, nullable) -- Editada y aprobada por usuario
├── justificacion_requerida (boolean, default: false)
├── historial_observaciones (jsonb, nullable) -- Array de {fecha, usuario_id, comentario} — bitácora de revisiones
├── capturado_por (FK → users.id)
├── revisado_por (FK → users.id, nullable)
├── fecha_captura (timestamp, nullable)
├── fecha_envio_revision (timestamp, nullable)
├── fecha_aprobacion (timestamp, nullable)
├── congelado_at (timestamp, nullable)  -- Sello de inmutabilidad
├── timestamps

avance_variables (valores capturados por variable)
├── id (PK)
├── avance_id (FK → avances.id)
├── indicador_variable_id (FK → indicador_variables.id)
├── valor (decimal(15,4))
├── timestamps
    UNIQUE(avance_id, indicador_variable_id)
```

### Medios de verificación adjuntos (evidencia por avance)

```
avance_evidencias
├── id (PK)
├── avance_id (FK → avances.id)
├── nombre_archivo (string)
├── ruta_almacenamiento (string)        -- Storage path
├── mime_type (string)
├── tamano_bytes (bigInteger)
├── area_generadora (string)
├── fecha_documento (date, nullable)
├── hash_archivo (string)               -- SHA-256 para integridad
├── congelado_at (timestamp, nullable)
├── timestamps
```

### Registro de auditoría para desbloqueos

```
desbloqueos
├── id (PK)
├── desbloqueble_type (string)          -- Polymorphic: avances, avance_evidencias
├── desbloqueble_id (unsignedBigInteger)
├── solicitado_por (FK → users.id)
├── aprobado_por (FK → users.id, nullable)
├── motivo (text)
├── estado (enum: solicitado, aprobado, rechazado)
├── timestamps
```

---

## Capa 7: Evaluación (Sprint 7)

### Índice de desempeño

```
evaluaciones_programa
├── id (PK)
├── programa_presupuestario_id (FK → programas_presupuestarios.id)
├── ejercicio_fiscal (year)
├── indice_desempeno (decimal(5,2))     -- 0.00 a 100.00
├── total_indicadores (integer)
├── indicadores_verde (integer)
├── indicadores_amarillo (integer)
├── indicadores_rojo (integer)
├── indicadores_sin_dato (integer)
├── generado_at (timestamp)
├── timestamps
    UNIQUE(programa_presupuestario_id, ejercicio_fiscal)
```

### Anexos transversales (etiquetado)

```
anexos_transversales
├── id (PK)
├── nombre (string)                     -- "Perspectiva de Género", "NNA", etc.
├── clave (string, unique)
├── descripcion (text, nullable)
├── timestamps

indicador_anexo_transversal (pivote M:M)
├── id (PK)
├── indicador_id (FK → indicadores.id)
├── anexo_transversal_id (FK → anexos_transversales.id)
├── timestamps
    UNIQUE(indicador_id, anexo_transversal_id)
```

---

## Diagrama de relaciones principales

```
teams (UR)
  │
  └──< programas_presupuestarios
         │
         ├── programa_alineacion ──> ped_*, ods_*, programas_derivados_*
         │
         ├──< arboles
         │      └──< arbol_nodos (Adjacency List)
         │
         ├──< alternativas
         │      └──< alternativa_nodos ──> arbol_nodos
         │
         └──< mir_niveles (Adjacency List: actividad→componente)
                │
                └──< indicadores
                       │
                       ├──< indicador_variables
                       ├──< medios_verificacion
                       ├── cremaa_validaciones
                       ├──< indicador_anexo_transversal ──> anexos_transversales
                       │
                       └──< metas_periodo
                              │
                              └──< avances
                                     ├──< avance_variables
                                     └──< avance_evidencias


ped_planes
  └──< ped_ejes
         └──< ped_temas
                └──< ped_objetivos_estrategicos ──< alineacion_ped_pnd ──> pnd_objetivos
                       └──< ped_estrategias
                              └──< ped_lineas_accion

pnd_ejes
  └──< pnd_objetivos ──< alineacion_pnd_ods ──> ods_metas
         └──< pnd_estrategias

ods_objetivos
  └──< ods_metas
```

---

## Índices recomendados

```sql
-- Filtro por team (aislamiento de UR) - el más crítico
CREATE INDEX idx_programas_team ON programas_presupuestarios(team_id);
CREATE INDEX idx_programas_ejercicio ON programas_presupuestarios(ejercicio_fiscal);

-- Navegación jerárquica (Adjacency List)
CREATE INDEX idx_arbol_nodos_parent ON arbol_nodos(parent_id);
CREATE INDEX idx_mir_niveles_parent ON mir_niveles(parent_id);
CREATE INDEX idx_mir_niveles_programa ON mir_niveles(programa_presupuestario_id);

-- Seguimiento
CREATE INDEX idx_metas_periodo_indicador ON metas_periodo(indicador_id, ejercicio_fiscal);
CREATE INDEX idx_avances_estado ON avances(estado);
CREATE INDEX idx_avances_meta_periodo ON avances(meta_periodo_id);

-- Búsqueda semántica (pgvector - HNSW para performance)
CREATE INDEX idx_ods_embedding ON ods_objetivos USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_pnd_obj_embedding ON pnd_objetivos USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_ped_obj_embedding ON ped_objetivos_estrategicos USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_ped_lineas_embedding ON ped_lineas_accion USING hnsw (embedding vector_cosine_ops);
```
