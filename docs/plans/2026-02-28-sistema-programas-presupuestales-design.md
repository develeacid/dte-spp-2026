# Diseño del Sistema de Programas Presupuestales

**Fecha:** 2026-02-28
**Actualización:** 2026-03-02 — Incorpora correcciones del Análisis de Cumplimiento PbR-SED
**Enfoque:** Híbrido (flujo guiado para programas nuevos + importación flexible para existentes)
**Nivel de gobierno:** Estatal (alineado a PED)

---

## 1. Contexto y objetivo

Sistema que asiste a servidores públicos en la creación, seguimiento y evaluación de programas presupuestarios bajo la Metodología de Marco Lógico (MML), el Presupuesto basado en Resultados (PbR) y el Sistema de Evaluación del Desempeño (SED).

La IA actúa como asistente: sugiere durante la captura y valida coherencia metodológica. El usuario siempre decide y aprueba. Nada se publica sin confirmación humana.

---

## 2. Usuarios y roles

| Rol | Perfil | Responsabilidades |
|-----|--------|-------------------|
| **Planeador (DGPOP)** | Áreas de planeación estratégica | Diseña programas, administra MIR, configura matriz de alineación, revisa y aprueba reportes de avance, consulta evaluaciones transversales |
| **Operador (UR)** | Unidades Responsables ejecutoras | Captura avances periódicos de indicadores, adjunta medios de verificación, redacta justificaciones de desviaciones |

---

## 3. Alcance del ciclo presupuestario

El sistema cubre 6 de las 7 etapas del ciclo:

1. **Planeación** — Carga de PED, matriz de alineación, diagnóstico (MML)
2. **Programación** — Diseño de MIR, indicadores, fichas técnicas
3. **Presupuestación** — Asignación y calendarización de metas
4. **Ejercicio y Control** — Captura periódica de avances con máquina de estados
5. **Seguimiento** — Semaforización, tableros consolidados
6. **Evaluación** — Análisis de desviaciones, lógica vertical, índice de desempeño

**Excluido (fase futura):** Rendición de Cuentas / Portal de transparencia ciudadana. Los reportes exportables con diccionario de datos preparan esta transición.

---

## 4. Sección 1 — Estructura de datos: Cascada de planes

### 4.1 Jerarquía de planeación

```
CATÁLOGOS DE REFERENCIA (inmutables)
  Agenda 2030 (17 ODS + metas)
    └── Plan Nacional de Desarrollo (PND)
          └── Ejes, estrategias, objetivos

PLANEACIÓN ESTATAL (editable)
  Plan Estatal de Desarrollo (PED)
    ├── Eje
    │     └── Tema / Sub-eje
    │           └── Objetivo Estratégico
    │                 └── Estrategia
    │                       └── Línea de Acción
    └── Programas Derivados (Sectoriales, Especiales, Institucionales)
          └── Objetivos de programa derivado
```

### 4.2 Matriz de Alineación

Mapa precargado de correspondencias muchos-a-muchos entre todos los niveles de la cascada:

```
Línea de Acción ↔ Estrategia ↔ Obj. Estratégico ↔ Eje PED ↔ PND ↔ ODS
```

- Se captura una sola vez y queda como referencia para todos los programas
- Relaciones muchos-a-muchos: una Línea de Acción puede impactar múltiples ODS
- Al asignar una Línea de Acción a un programa, el sistema hereda automáticamente toda la cadena hacia arriba

### 4.3 Carga de datos

- ODS y PND se precargan como catálogos en Markdown estructurado
- PED se captura por el usuario o se importa desde Markdown
- La IA parsea documentos importados y sugiere estructura jerárquica para confirmación del usuario
- Para sugerencias de alineación semántica: embeddings vectoriales (búsqueda por similitud de cosenos), no prompts brutos

### 4.4 Ancla MIR ↔ Planes

| Nivel MIR | Se alinea con |
|-----------|---------------|
| **Fin** | Objetivo Estratégico del PED (hereda PND y ODS) |
| **Propósito** | Objetivo de Programa Derivado/Sectorial |
| **Componentes/Actividades** | Líneas de Acción |

---

## 5. Sección 2 — Flujo de creación desde cero (MML)

### 5.1 Las 6 etapas guiadas

**Etapa 1: Definición del problema**
- Usuario describe la situación no deseada
- IA sugiere redacción clara, sin verbos, sin soluciones
- Salida: Problema central definido

**Etapa 2: Árbol del problema**
- Usuario captura causas (directas e indirectas) y efectos (directos e indirectos)
- IA sugiere causas/efectos adicionales, valida que no sean soluciones disfrazadas
- Salida: Árbol de causas-efectos validado

**Etapa 3: Árbol de objetivos**
- El sistema transforma: Causas → Medios, Efectos → Fines, Problema → Objetivo central
- IA sugiere redacción en positivo de cada nodo
- Usuario revisa y ajusta cada transformación
- Salida: Árbol de medios-fines

**Etapa 4: Selección de alternativas**
- El sistema agrupa los medios en posibles estrategias
- IA evalúa viabilidad técnica, institucional y presupuestal
- Usuario selecciona la(s) alternativa(s) a implementar
- Salida: Estrategia seleccionada

**FILTRO CRÍTICO entre Etapa 4 y 5:**
Solo las ramas del árbol que sobrevivieron a la selección de alternativas pasan a la Estructura Analítica del Programa. Las ramas no seleccionadas se "podan" visualmente. El Fin y Propósito se mantienen; Componentes y Actividades se derivan solo de los medios de la alternativa elegida.

**Etapa 5: Estructura Analítica del Programa (EAP)**
- Mapeo desde el árbol filtrado:
  - Fines superiores → FIN de la MIR
  - Objetivo central → PROPÓSITO
  - Medios directos → COMPONENTES
  - Medios indirectos → ACTIVIDADES
- IA valida lógica vertical y sugiere alineación con la Matriz de Alineación
- Salida: Estructura lista para la MIR

**Etapa 6: Elaboración de la MIR (4x4)**
- Columna 1 (Resumen Narrativo) se llena automáticamente desde la EAP
- Usuario captura por cada nivel:
  - Col 2: Indicadores (nombre, fórmula, variables, dimensión, frecuencia, línea base, meta)
  - Col 3: Medios de verificación (documento, área generadora, periodicidad, ubicación)
  - Col 4: Supuestos (factores externos, riesgos)
- Salida: MIR completa y validada

### 5.5 Motor de Reglas Condicionales para la Ficha Técnica del Indicador

El formulario de captura de indicadores es **dependiente del nivel (fila)** que se está editando. El frontend (Livewire) y el backend (Form Requests de Laravel) aplican restricciones automáticas que actúan como "poka-yoke" metodológico, evitando errores de captura sin necesidad de intervención de la IA.

#### 5.5.1 Auto-asignación del Tipo de Indicador

| Nivel | Comportamiento del campo |
|-------|-------------------------|
| **Fin** | Campo bloqueado → auto-asigna **Estratégico** |
| **Propósito** | Campo bloqueado → auto-asigna **Estratégico** |
| **Componente** | Campo **editable**: el usuario elige entre Estratégico (si impacta directamente a la sociedad) o De Gestión (si es servicio interno) |
| **Actividad** | Campo bloqueado → auto-asigna **De Gestión** |

#### 5.5.2 Filtrado de Dimensiones de Medición

| Nivel | Dimensiones disponibles |
|-------|------------------------|
| **Fin** | Solo **Eficacia** |
| **Propósito** | Eficacia, Eficiencia |
| **Componente** | Eficacia, Eficiencia, Calidad |
| **Actividad** | Eficacia, Eficiencia, **Economía** (exclusiva de este nivel) |

#### 5.5.3 Restricciones de Frecuencia de Medición

| Nivel | Opciones del dropdown |
|-------|----------------------|
| **Fin** | Anual, Bianual, Sexenal |
| **Propósito** | Semestral, Anual |
| **Componente** | Trimestral, Semestral |
| **Actividad** | Mensual, Trimestral |

> **Nota:** Las frecuencias diaria y semanal están excluidas del sistema en todos los niveles. La selección del dropdown activa automáticamente la calendarización correcta de metas en el módulo de seguimiento.

### 5.2 Validación sintáctica del Resumen Narrativo (SHCP)

La IA audita que la Columna 1 cumpla con las fórmulas obligatorias:

| Nivel | Sintaxis obligatoria |
|-------|---------------------|
| **Fin** | "Contribuir a [Impacto macro] mediante [Solución del programa]" |
| **Propósito** | "[Población Objetivo] + [Verbo en presente/participio] + [Condición lograda]" |
| **Componente** | "[Bien o Servicio] + [Participio: terminado en -ado/-ido]" |
| **Actividad** | "[Sustantivo deverbal] + [Complemento]" |

### 5.3 Validación CREMAA desglosada

Cada indicador se evalúa letra por letra:

- **C**laro — ¿Preciso e inequívoco?
- **R**elevante — ¿Información sustancial sobre el objetivo?
- **E**conómico — ¿Datos disponibles a costo razonable?
- **M**onitoreable — ¿Comprobación independiente posible?
- **A**decuado — ¿Base suficiente para juzgar desempeño?
- **A**portación marginal — ¿No repite información de otro indicador del mismo nivel?

La interfaz muestra cada letra en verde/rojo con explicación específica del fallo.

### 5.4 Persistencia y versionado

- Cada etapa guarda su estado; el usuario puede salir y retomar
- Al regresar a etapas anteriores, el sistema ofrece: crear borrador alterno o sobreescribir
- Snapshots permiten bifurcaciones temporales (explorar MIR alternativas sin perder trabajo)
- Los cambios propagan advertencias hacia adelante

---

## 6. Sección 3 — Flujo de importación de programas existentes

### 6.1 Proceso de importación

**Paso 1: Carga de datos**
- Formatos: Markdown estructurado (estándar), CSV, Excel
- El usuario sube lo disponible: MIR, fichas técnicas, avances, medios de verificación
- La IA parsea, mapea a la estructura del sistema y extrae variables de las fórmulas

**Paso 2: Diagnóstico de completitud**
- Reporte de huecos por elemento (presente/faltante/hueco crítico)
- Huecos críticos (bloquean seguimiento a nivel de indicador): variables de fórmula no identificadas, frecuencia de medición ausente, línea base y año base faltantes
- Huecos menores (no bloquean): sintaxis del resumen narrativo, supuestos faltantes

**Paso 3: Completar información faltante**
- El usuario completa a su ritmo
- La IA sugiere contenido basándose en lo ya importado
- Mismas validaciones que programas nuevos (CREMAA, sintaxis, lógica)

**Paso 4: Vinculación con la cascada de planes**
- Si no tiene alineación: el sistema sugiere vínculos usando la Matriz de Alineación
- Si ya tiene: se valida contra la Matriz y se señalan inconsistencias

**Paso 5: Activación para seguimiento**
- El programa queda activo; indicadores sin huecos críticos habilitan captura
- Se configuran frecuencias de medición y calendarización de metas

### 6.2 Plantilla Markdown de importación

```markdown
# Programa: [Nombre del programa]
## Unidad Responsable: [Nombre de la UR]
## Ejercicio Fiscal: [Año]

### FIN
- **Resumen Narrativo:** [Texto]
- **Indicador:** [Nombre del indicador]
- **Fórmula:** [Expresión matemática]
- **Frecuencia:** [Mensual/Trimestral/Semestral/Anual]
- **Tipo / Dimensión:** [Estratégico|Gestión] / [Eficacia|Eficiencia|Calidad|Economía]
- **Línea Base:** [Valor] ([Año])
- **Meta:** [Valor]
- **Medio de Verificación:** [Documento, Área, Periodicidad]
- **Supuesto:** [Texto]

### PROPÓSITO
[Misma estructura]

### COMPONENTE 1
[Misma estructura]

### ACTIVIDAD 1.1
[Misma estructura]
```

---

## 7. Sección 4 — Seguimiento y captura periódica

### 7.1 Calendarización de metas

Al activar un programa, la meta anual de cada indicador se distribuye en los periodos de captura según su frecuencia. El semáforo compara **avance del periodo vs meta del periodo**, no vs meta anual.

### 7.2 Naturaleza de variables

Cada variable se clasifica al definirse:

| Tipo | Comportamiento | Ejemplo |
|------|---------------|---------|
| **Acumulable** | Se suman entre periodos | Becas entregadas, talleres realizados |
| **Continua / de saldo** | Se promedia o se toma último valor | Tasa de desempleo, % de cobertura |

El motor de cálculo aplica la reducción matemática correcta en tableros anuales.

### 7.3 Flujo de captura

1. El sistema genera periodos de captura según frecuencia de cada indicador
2. Notifica a la UR responsable cuando se abre un periodo
3. El operador captura valores de cada variable de la fórmula
4. El sistema calcula el resultado aplicando la fórmula registrada
5. Semaforización sugerida según rangos y sentido del indicador
6. Justificación cualitativa:
   - Verde: opcional
   - Amarillo/rojo: obligatoria
   - La IA genera borrador basado en supuestos (Col. 4), magnitud de desviación e historial
   - El usuario edita y aprueba
7. Adjuntar medios de verificación (documento, área, fecha)

### 7.4 Configuración de semáforos

- Rangos de tolerancia por indicador (ej. verde: ±15%, amarillo: -15% a -25%, rojo: <-25% y >+25%)
- Sentido del indicador: ascendente ("más es mejor") o descendente ("menos es mejor")
- Reglas estrictas: sin solapamiento numérico, meta siempre dentro del rango verde, no inicia en 0

### 7.5 Máquina de estados del reporte

```
En Captura (UR) → En Revisión (DGPOP) → Aprobado y Congelado
                         ↓
                   Observado (retorno a UR con comentarios)
```

- **En Captura:** Operador ingresa variables, adjunta evidencia, genera justificación
- **En Revisión:** Reporte bloqueado para edición. Planeador valida evidencia vs números
- **Observado:** Planeador rechaza con comentarios. Regresa a la UR para corrección
- **Aprobado y Congelado:** Avance inmutable. Alimenta tableros

### 7.6 Integridad documental

- Sello de tiempo inalterable al adjuntar archivos
- Archivos congelados al pasar a "Aprobado"
- Desbloqueo excepcional requiere solicitud formal con registro de auditoría

### 7.7 Vistas de seguimiento

**Para el Operador (UR):** Sus indicadores pendientes de captura, historial de reportes, estados.

**Para el Planeador (DGPOP):** Panel consolidado por programa con todos los indicadores, metas, avances, semáforos. Vista expandible por indicador. Vista comparativa ejercicio actual vs anterior.

---

## 8. Sección 5 — Evaluación y salidas

### 8.1 Evaluación por programa

Al cierre del ejercicio fiscal:

1. **Resumen ejecutivo:** Alineación (PED → PND → ODS), objetivo central, recursos
2. **Tablero de semáforos consolidado:** Todos los indicadores con color final, comparativa vs ejercicio anterior, tendencia (mejoró/empeoró/estable)
3. **Análisis de desviaciones:** Indicadores en amarillo/rojo con justificaciones aprobadas, supuestos incumplidos, indicadores crónicamente en rojo (2+ ejercicios)
4. **Validación de lógica vertical al cierre:** La IA contrasta semáforos por nivel y señala rupturas causales (ej. Actividades en verde pero Componente en rojo = problema de diseño, no de ejecución)

### 8.2 Evaluación transversal

Paneles que cruzan todos los programas del estado:

- **Por Eje del PED:** Conteo de semáforos e índice de desempeño por eje
- **Por ODS:** Indicadores que contribuyen a cada ODS y su estado agregado
- **Por Unidad Responsable:** Desempeño agrupado por dependencia
- **Por Anexos Transversales:** Filtro por temáticas (Perspectiva de Género, NNA, Cambio Climático, Anticorrupción) cruzando todas las dependencias

### 8.3 Índice de Desempeño General (0-100)

Promedio ponderado del porcentaje de avance de metas por programa:

| Nivel | Peso |
|-------|------|
| Fin | Mayor peso |
| Propósito | Alto |
| Componentes | Medio |
| Actividades | Menor peso |

Permite ranqueo entre programas y entre dependencias.

### 8.4 Rol de la IA en evaluación

- Detecta rupturas en lógica vertical (resultados contradictorios entre niveles)
- Señala indicadores crónicamente en rojo
- Sugiere si la desviación es de ejecución o de diseño
- **No emite juicios de valor** sobre el desempeño de las UR. Presenta datos y patrones. El planeador interpreta.

### 8.5 Reportes exportables

| Reporte | Formato |
|---------|---------|
| MIR en formato oficial | PDF, Excel |
| Fichas técnicas de indicadores | PDF |
| Reporte de avance trimestral | PDF, Excel |
| Reporte de evaluación anual | PDF, Excel |
| Reporte transversal (Eje PED / ODS / Anexo) | PDF, Excel |
| Datos estructurados para reutilización | Markdown, CSV, JSON |

Todos los reportes incluyen sello de tiempo, periodo evaluado y nota de fecha de aprobación.

### 8.6 Datos abiertos

Toda exportación CSV/JSON se empaqueta con un **Diccionario de Datos** generado automáticamente que describe cada campo, cumpliendo con la Ley General de Transparencia para futura publicación en portales ciudadanos o Plataforma Nacional de Transparencia.

---

## 9. Arquitectura técnica

### 9.1 Stack tecnológico

**TALL Stack:** Tailwind CSS + Alpine.js + Laravel + Livewire

### 9.2 Autenticación y estructura organizacional: Laravel Jetstream + Multi-UR

Jetstream con módulo **Teams** activado proporciona la base de autenticación. El modelo de datos se extiende para soportar programas transversales con múltiples Unidades Responsables (UR).

**Tablas core de autenticación:**
- `users` — Servidor público (nombre, email, contraseña, token 2FA, foto)
- `sessions` — Sesiones activas con capacidad de cierre remoto
- `personal_access_tokens` — Tokens API (para futura fase de transparencia/portal ciudadano)
- `password_reset_tokens` — Recuperación de contraseñas

**Tablas de Teams (mapeo a estructura burocrática):**
- `teams` — Representa una Unidad Responsable (UR) o Dependencia
- `team_user` — Tabla pivote usuario ↔ equipos (un usuario puede pertenecer a varias UR)
- `team_invitations` — Invitaciones por correo para unirse a una UR

**Modelo Multi-UR: UR Coordinadora y UR Coadyuvante**

Para soportar programas transversales operados por más de una dependencia:

- `programas_presupuestarios.team_id` → **UR Coordinadora** (dueña del programa, administra la MIR completa)
- `mir_niveles.team_id` (FK nullable) → **UR Coadyuvante** asignada al Componente o Actividad específico
- `programa_team` — Tabla pivote `programa_id ↔ team_id` con columna `rol` (ENUM: `coordinadora`, `coadyuvante`) para registrar todos los equipos participantes

**Reglas de acceso con middleware flexible:**

| Situación | Acceso al programa | Acceso al componente/indicador |
|-----------|--------------------|--------------------------------|
| Operador de la UR Coordinadora | Completo | Completo |
| Operador de UR Coadyuvante | Solo lectura (read-only) en el programa general | Edición/captura solo en sus Componentes/Actividades asignados |
| Planeador (DGPOP) | Completo en todos los programas | Completo |

> El middleware global ya **no bloquea por `team_id` del programa de forma rígida**. Primero verifica si el usuario es Coordinador; si no, verifica si es Coadyuvante con permisos a nivel de componente. El aislamiento estricto (Salud no ve Educación) se mantiene para programas sin relación de coadyuvancia.

### 9.3 Roles y permisos: Jetstream + Spatie

Jetstream controla **quién eres** y **a qué UR perteneces**.
Spatie (`spatie/laravel-permission`) controla **qué puedes hacer**:

| Permiso | Planeador (DGPOP) | Operador (UR) | Admin |
|---------|-------------------|---------------|-------|
| `gestionar_catalogos` | Si | No | Si |
| `crear_programa` | Si | No | Si |
| `editar_mir` | Si | No | Si |
| `capturar_avance` | No | Si | Si |
| `revisar_avance` | Si | No | Si |
| `aprobar_avance` | Si | No | Si |
| `exportar_reportes` | Si | Limitado | Si |
| `administrar_usuarios` | No | No | Si |

### 9.4 Seguridad gubernamental

- Autenticación de doble factor (2FA) obligatoria
- Gestión de sesiones de navegador (cierre remoto)
- Aislamiento de datos por Team/UR vía middleware
- Registro de auditoría en todas las acciones críticas

---

## 10. Plan de implementación por Sprints

Gestión mediante tablero Kanban en Notion. Cada sprint entrega funcionalidad verificable.

### Sprint 0: Infraestructura y entorno

Objetivo: Levantar la base inmutable antes de escribir lógica de negocio.

| Tarea | Detalle |
|-------|---------|
| **Contenerización** | Docker: servicios para servidor web, PHP, Redis (colas y caché), PostgreSQL |
| **Base de datos** | PostgreSQL con extensión pgvector para búsquedas por similitud de cosenos |
| **Inicialización** | Proyecto base TALL (Tailwind, Alpine.js, Laravel, Livewire) |

### Sprint 1: Identidad y aislamiento (Sección 9)

Objetivo: Control de acceso y partición de datos por UR, con soporte para programas multi-UR.

| Tarea | Detalle |
|-------|---------|
| **Jetstream & Teams** | Instalación con funcionalidad de equipos. Team = Unidad Responsable |
| **Roles y permisos** | Spatie/laravel-permission. Seeders con perfiles base (Planeador, Operador, Admin) y permisos granulares |
| **Seguridad** | 2FA forzoso. |
| **Middleware Multi-UR** | El middleware ya **no filtra rígidamente por `team_id` del programa**. Implementar lógica en dos niveles: (1) si el usuario pertenece a la UR Coordinadora → acceso completo; (2) si pertenece a UR Coadyuvante → acceso read-only al programa + acceso de captura solo en Componentes/Actividades asignados. |
| **Tabla pivote `programa_team`** | Migración con columna `rol` (ENUM: `coordinadora`, `coadyuvante`). Permite que múltiples equipos participen como ejecutores en una misma MIR. |

### Sprint 2: Cascada de planes y matriz de alineación (Sección 4)

Objetivo: Modelar la jerarquía de planeación y las correspondencias entre niveles.

| Tarea | Detalle |
|-------|---------|
| **Catálogos inmutables** | Migraciones y seeders para ODS (17 objetivos + metas) y PND (ejes, estrategias, objetivos). Carga desde Markdown |
| **PED editable** | CRUD completo: Eje → Tema/Sub-eje → Objetivo Estratégico → Estrategia → Línea de Acción. Estructura anidada (Adjacency List con `parent_id`) |
| **Programas Derivados** | CRUD: tipo (Sectorial, Especial, Institucional, Regional), objetivos de programa derivado |
| **Matriz de alineación** | Tablas pivote muchos-a-muchos. Interfaz para vincular niveles con herencia automática hacia arriba |
| **Importación Markdown** | Parser que convierte Markdown estructurado del PED en registros de BD. Usuario confirma estructura sugerida |
| **Embeddings** | Al guardar cualquier objetivo/línea de acción, generar embedding vectorial vía job en cola Redis. Almacenar en pgvector |

### Sprint 3: Metodología de Marco Lógico — Etapas 1 a 4 (Sección 5)

Objetivo: Flujo guiado de diagnóstico hasta selección de alternativas.

| Tarea | Detalle |
|-------|---------|
| **Etapa 1: Problema** | Formulario de captura. IA sugiere redacción clara (job en cola, respuesta asíncrona vía Livewire) |
| **Etapa 2: Árbol del problema** | Interfaz visual de árbol (causas/efectos). IA sugiere nodos adicionales |
| **Etapa 3: Árbol de objetivos** | Transformación automática (negativo → positivo). IA sugiere redacción. Usuario aprueba cada nodo |
| **Etapa 4: Alternativas** | Agrupación de medios en estrategias. IA evalúa viabilidad. Usuario selecciona |
| **Filtro de poda** | Solo ramas seleccionadas pasan a la EAP. Visualización de ramas descartadas |
| **Persistencia** | Estado por etapa. Snapshots para bifurcaciones temporales |

### Sprint 4: MIR y validaciones (Sección 5 cont.)

Objetivo: Construcción de la matriz 4x4 con todas las validaciones normativas y el motor de reglas condicionales.

| Tarea | Detalle |
|-------|---------|
| **Etapa 5: EAP** | Mapeo automático del árbol filtrado a Fin/Propósito/Componentes/Actividades. Validación de lógica vertical |
| **Etapa 6: MIR 4x4** | Interfaz de matriz. Col 1 (Resumen Narrativo) prellenada desde EAP |
| **Indicadores** | Ficha técnica completa: nombre, fórmula, extracción de variables, tipo/dimensión, frecuencia, línea base, meta, sentido, rangos de semáforo |
| **Medios de verificación** | Documento, área generadora, periodicidad, ubicación |
| **Supuestos** | Captura por nivel |
| **Validación sintáctica SHCP** | IA audita fórmulas de Resumen Narrativo por nivel (Fin: "Contribuir a...", etc.) |
| **Validación CREMAA** | Desglose letra por letra con explicación de cada fallo |
| **Validación lógica** | Vertical (causa-efecto entre niveles) y horizontal (indicador mide el objetivo, medio verifica el indicador) |
| **Alineación** | Al crear MIR, el sistema sugiere vínculos con la Matriz de Alineación usando embeddings. Herencia automática (Línea de Acción → PED → PND → ODS) |
| **Variables** | Extracción de variables de fórmulas. Clasificación: acumulable vs continua |
| **⚠️ Form Requests y UI dinámica condicional para la Ficha Técnica del Indicador** | Implementar las matrices de restricción de la Sección 5.5: (1) Auto-asignación de Tipo según nivel; (2) Filtrado de Dimensiones según nivel; (3) Dropdown de Frecuencia restringido por nivel. Validación duplicada en frontend (Livewire/Alpine.js) y backend (Laravel Form Requests), aplicando el principio de poka-yoke metodológico. |
| **Asignación de UR Coadyuvante por Componente** | Al definir cada Componente en la MIR, el Planeador puede asignar un `team_id` de UR Coadyuvante. El sistema registra la relación en `programa_team` y actualiza los permisos del middleware. |

### Sprint 5: Importación de programas existentes (Sección 6)

Objetivo: Migrar programas ya operativos sin bloquear su seguimiento.

| Tarea | Detalle |
|-------|---------|
| **Parser Markdown/CSV/Excel** | Importador que mapea datos a la estructura del sistema |
| **Extracción de variables** | IA parsea fórmulas para identificar variables. Huecos críticos si no puede |
| **Diagnóstico de completitud** | Reporte de huecos: críticos (bloquean indicador) vs menores (no bloquean) |
| **Completar huecos** | Flujo asistido por IA para llenar información faltante |
| **Vinculación con planes** | Sugerencia de alineación o validación contra Matriz de Alineación existente |
| **Activación** | Indicadores sin huecos críticos habilitan captura. Configuración de frecuencias y calendarización de metas |

### Sprint 6: Seguimiento y captura periódica (Sección 7)

Objetivo: Ciclo operativo de reporte de avances con control de calidad.

| Tarea | Detalle |
|-------|---------|
| **Calendario de captura** | Generación automática de periodos según frecuencia de cada indicador |
| **Calendarización de metas** | Distribución de meta anual en periodos. Semáforo compara avance vs meta del periodo |
| **Captura de avance** | Formulario por indicador: valores de variables, cálculo automático, semaforización sugerida |
| **Justificaciones** | Obligatoria en amarillo/rojo. IA genera borrador desde supuestos + desviación + historial. Usuario edita y aprueba |
| **Medios de verificación** | Adjuntar archivos con sello de tiempo |
| **Máquina de estados** | En Captura → En Revisión → Observado (retorno) → Aprobado y Congelado |
| **Congelamiento** | Inmutabilidad post-aprobación. Desbloqueo excepcional con registro de auditoría |
| **Notificaciones** | Alertas a UR cuando se abre periodo de captura. Alertas a DGPOP cuando hay reportes pendientes de revisión |

### Sprint 7: Evaluación y reportes (Sección 8)

Objetivo: Cierre de ciclo con inteligencia analítica y salidas documentales.

| Tarea | Detalle |
|-------|---------|
| **Evaluación por programa** | Resumen ejecutivo, tablero de semáforos, comparativa vs ejercicio anterior, tendencias |
| **Análisis de desviaciones** | Justificaciones aprobadas, supuestos incumplidos, indicadores crónicos en rojo |
| **Lógica vertical al cierre** | IA detecta rupturas causales entre niveles (Actividades verdes + Componente rojo = problema de diseño) |
| **Evaluación transversal** | Paneles por Eje PED, por ODS, por UR, por Anexos Transversales |
| **Índice de Desempeño (0-100)** | Promedio ponderado (Fin > Propósito > Componente > Actividad). Ranqueo entre programas y dependencias |
| **Anexos Transversales** | Etiquetado de indicadores con temáticas (Género, NNA, Cambio Climático, Anticorrupción). Filtro transversal |
| **Exportación** | MIR oficial (PDF/Excel), fichas técnicas, reportes trimestrales/anuales, reportes transversales |
| **Datos abiertos** | CSV/JSON empaquetados con Diccionario de Datos automático |

### Sprint 8: Orquestación IA (transversal)

Objetivo: Consolidar todas las integraciones de IA del sistema. Gracias al motor de reglas condicionales del Sprint 4, la IA **no necesita auditar errores de dimensión, frecuencia o tipo**, los cuales son bloqueados por diseño. La IA se concentra en auditoría semántica y causal.

| Tarea | Detalle |
|-------|---------|
| **Servicio centralizado** | Clase/servicio Laravel que encapsula todas las llamadas a LLM vía SDK/HTTP client |
| **Jobs asíncronos** | Todas las llamadas a LLM se ejecutan como jobs en cola Redis para no bloquear la interfaz |
| **Embeddings** | Pipeline: al guardar textos de planes/objetivos → generar embedding → almacenar en pgvector |
| **Búsqueda semántica** | Servicio de similitud de cosenos para sugerencias de alineación |
| **Validaciones IA (alcance reducido y enfocado)** | La IA audita exclusivamente: Sintaxis SHCP del Resumen Narrativo, criterios CREMAA (semántica), congruencia causal de la lógica vertical/horizontal, y sugerencias de redacción. Los errores de dimensión, frecuencia y tipo de indicador ya son prevenidos por el sistema (poka-yoke), liberando a la IA para tareas de mayor valor semántico. |
| **Justificaciones IA** | Generación de borradores acotada a supuestos (Col. 4), magnitud de desviación e historial del indicador |
| **Rate limiting** | Control de frecuencia de llamadas al LLM por usuario/sesión |

### Dependencias entre Sprints

```
Sprint 0 (Infra)
  └── Sprint 1 (Auth/Teams)
        └── Sprint 2 (Planes/Alineación)
              ├── Sprint 3 (MML Etapas 1-4)
              │     └── Sprint 4 (MIR/Validaciones)
              │           └── Sprint 5 (Importación)
              │                 └── Sprint 6 (Seguimiento)
              │                       └── Sprint 7 (Evaluación)
              └── Sprint 8 (IA - transversal, se desarrolla en paralelo desde Sprint 2)
```

---

## 11. Flujo de trabajo Git

### 11.1 Nomenclatura de ramas

Formato: `tipo/sprint-ticket-descripcion-corta`

| Tipo | Uso |
|------|-----|
| `feat/` | Nuevas características |
| `fix/` | Corrección de errores |
| `chore/` | Mantenimiento (dependencias, Docker) |
| `refactor/` | Mejoras de código sin alterar funcionalidad |

Ejemplos:
- `feat/S1-T4-integracion-spatie-roles`
- `fix/S2-T12-error-llave-foranea-pnd`
- `chore/S0-T1-configuracion-docker-pgvector`

Nunca se trabaja directamente en `main` o `develop`.

### 11.2 Conventional Commits

Formato: `tipo(alcance): descripcion en infinitivo`

- `feat(ped): crear migraciones y modelos para la jerarquia del plan estatal`
- `fix(auth): corregir redireccion tras login en jetstream`
- `chore(docker): agregar extension pgvector a postgres`

### 11.3 Plantilla de Pull Request

Archivo: `.github/PULL_REQUEST_TEMPLATE.md`

Incluye checklist obligatorio de validación TALL:
- Migraciones con `up()` y `down()` completos
- `migrate:fresh` sin errores
- Modelos con `$fillable` y relaciones definidas
- Documentación actualizada si se alteró el esquema
- Componentes Livewire sin exposición de datos sensibles
- `npm run build` sin errores
- Evidencia visual obligatoria para cambios de frontend

### 11.4 Flujo de trabajo

1. Tomar ticket en Notion → mover a "In Progress"
2. Crear rama: `git checkout -b feat/S1-T2-modelos-ped`
3. Desarrollar con commits granulares y descriptivos
4. Push: `git push origin feat/S1-T2-modelos-ped`
5. Abrir PR (plantilla aparece automáticamente)
6. Rellenar plantilla, adjuntar evidencia, solicitar revisión
7. Al aprobarse y hacer merge, borrar rama local y remota

---

## 12. Decisiones de diseño transversales

| Decisión | Detalle |
|----------|---------|
| **Ejercicios fiscales** | Vigente + anterior como referencia |
| **Escala** | <30 programas presupuestarios |
| **IA: rol** | Sugiere en captura, valida coherencia. Nunca decide. |
| **IA: alineación** | Embeddings vectoriales para sugerencias semánticas |
| **IA: justificaciones** | Se limita a supuestos (Col. 4), magnitud de desviación e historial |
| **Formato estándar** | Markdown estructurado como puente de importación/exportación |
| **Versionado** | Snapshots por etapa con bifurcaciones temporales |
| **Auditoría** | Sellos de tiempo, congelamiento post-aprobación, registro de desbloqueos |
| **Transparencia** | Fase futura; reportes exportables con diccionario de datos preparan la transición |
| **Auth** | Laravel Jetstream con Teams (2FA, sesiones, aislamiento por UR) |
| **Permisos** | Spatie/laravel-permission para granularidad de acciones |
| **Stack** | TALL (Tailwind, Alpine.js, Laravel, Livewire) |
