# Tickets para Notion — Sistema de Programas Presupuestales

**Fecha:** 2026-02-28
**Actualizacion:** 2026-03-02 — Incorpora cambios del Analisis de Cumplimiento PbR-SED
**Formato:** Cada ticket incluye ID, titulo, descripcion, criterios de aceptacion y rama Git sugerida.
**Estados Kanban:** Backlog | In Progress | In Review | Done

---

## Sprint 0: Infraestructura y Entorno

---

### S0-T0: Inicializar proyecto Laravel 12 en raíz

**Tipo:** chore
**Rama:** `chore/S0-T0-laravel-init`

**Descripcion:**
Ejecutar `composer create-project laravel/laravel .` directamente en la raiz del repositorio para que la estructura del framework sea el proyecto principal.

**Criterios de aceptacion:**
- [ ] Directorio raiz contiene la estructura completa de Laravel 12 (app, artisan, composer.json, etc.)
- [ ] `.env` inicial generado correctamente
- [ ] Git detecta los nuevos archivos de Laravel

---

### S0-T1: Configurar Laravel Sail con PostgreSQL y pgvector

**Tipo:** chore
**Rama:** `chore/S0-T1-sail-config`

**Descripcion:**
Instalar Laravel Sail y configurar el archivo `docker-compose.yml` para usar la imagen de PostgreSQL compatible con `pgvector` y el servicio de Redis.

**Servicios requeridos:**
- Laravel Sail (PHP 8.3+)
- PostgreSQL 16 (Imagen: `pgvector/pgvector:pg16`)
- Redis

**Criterios de aceptacion:**
- [ ] `php artisan sail:install` configurado para pgsql y redis
- [ ] `docker-compose.yml` modificado para usar `pgvector/pgvector:pg16`
- [ ] `./vendor/bin/sail up -d` levanta los servicios correctamente
- [ ] Conexión a base de datos PostgreSQL exitosa desde Sail

---

### S0-T2: Habilitar extensión pgvector en PostgreSQL

**Tipo:** chore
**Rama:** `chore/S0-T2-pgvector-setup`

**Descripcion:**
Crear una migración inicial en Laravel para habilitar la extensión `pgvector` en la base de datos PostgreSQL.

**Criterios de aceptacion:**
- [ ] Migración generada con `CREATE EXTENSION IF NOT EXISTS vector`
- [ ] `sail artisan migrate` ejecuta la migración sin errores
- [ ] La extensión `vector` está activa en el esquema de PostgreSQL
- [ ] Prueba manual: se permite crear una tabla con columna tipo `vector(1536)`

---

### S0-T4: Configurar Redis como driver de colas y caché

**Tipo:** chore
**Rama:** `chore/S0-T4-redis-setup`

**Descripcion:**
Configurar las variables de entorno para que Laravel utilice Redis para manejar el sistema de colas, caché y sesiones.

**Criterios de aceptacion:**
- [ ] `.env` configurado con `QUEUE_CONNECTION=redis` y `CACHE_STORE=redis`
- [ ] `sail artisan queue:work` procesa jobs de prueba
- [ ] Cache de Laravel operativa sobre Redis
- [ ] Sesiones de usuario persistidas en Redis

---

## Sprint 1: Identidad y Aislamiento

---

### S1-T1: Instalar Laravel Jetstream con Teams

**Tipo:** feat
**Rama:** `feat/S1-T1-jetstream-teams`

**Descripcion:**
Instalar Jetstream con el stack Livewire y la funcionalidad de Teams activada. Ejecutar migraciones base. Verificar que el flujo de registro, login y creacion de equipos funcione.

**Criterios de aceptacion:**
- [ ] `composer require laravel/jetstream` ejecutado
- [ ] Jetstream instalado con `--teams` flag
- [ ] `php artisan migrate` crea las tablas: users, teams, team_user, team_invitations, sessions, personal_access_tokens
- [ ] Flujo de registro crea usuario + team personal
- [ ] Login y logout funcionan correctamente
- [ ] Interfaz de gestion de equipos accesible

---

### S1-T2: Extender tabla teams con campos de Unidad Responsable

**Tipo:** feat
**Rama:** `feat/S1-T2-teams-campos-ur`

**Descripcion:**
Crear migracion para agregar campos especificos de Unidad Responsable a la tabla `teams` de Jetstream.

**Campos a agregar:**
- `clave_ur` (string, nullable, unique) — Clave oficial de la UR
- `titular` (string, nullable) — Nombre del titular
- `tipo_ur` (enum: sustantiva, apoyo) — Rol en ejecucion presupuestal
- `activa` (boolean, default: true)

**Criterios de aceptacion:**
- [ ] Migracion con `up()` y `down()` completos
- [ ] `migrate:fresh` ejecuta sin errores
- [ ] Modelo `Team` extendido con `$fillable` actualizado
- [ ] Seeder de prueba crea 3 UR de ejemplo (1 sustantiva, 1 apoyo, 1 inactiva)
- [ ] Actualizar documentacion del esquema de BD

---

### S1-T3: Integrar Spatie/laravel-permission

**Tipo:** feat
**Rama:** `feat/S1-T3-integracion-spatie-roles`

**Descripcion:**
Instalar el paquete `spatie/laravel-permission`. Crear los roles base y permisos granulares del sistema. Configurar seeders.

**Roles:**
- `admin`
- `planeador`
- `operador`

**Permisos:**
- `gestionar_catalogos`
- `crear_programa`
- `editar_mir`
- `capturar_avance`
- `revisar_avance`
- `aprobar_avance`
- `exportar_reportes`
- `administrar_usuarios`

**Asignacion:**
- admin: todos los permisos
- planeador: gestionar_catalogos, crear_programa, editar_mir, revisar_avance, aprobar_avance, exportar_reportes
- operador: capturar_avance, exportar_reportes (limitado)

**Criterios de aceptacion:**
- [ ] Paquete instalado y migraciones ejecutadas
- [ ] Seeder `RolesAndPermissionsSeeder` crea roles y permisos
- [ ] Modelo `User` usa el trait `HasRoles`
- [ ] Test: usuario con rol planeador tiene permiso `crear_programa`
- [ ] Test: usuario con rol operador NO tiene permiso `crear_programa`

---

### S1-T4: Activar 2FA obligatorio

**Tipo:** feat
**Rama:** `feat/S1-T4-2fa-obligatorio`

**Descripcion:**
Configurar Jetstream para que la autenticacion de doble factor sea obligatoria para todos los usuarios. Redirigir a configuracion de 2FA si el usuario no lo ha activado.

**Criterios de aceptacion:**
- [ ] Middleware que detecta si el usuario no tiene 2FA configurado
- [ ] Redireccion automatica a pagina de configuracion de 2FA
- [ ] Usuario no puede acceder a ninguna ruta protegida sin 2FA activo
- [ ] Flujo de activacion de 2FA funcional (QR + codigos de respaldo)

---

### S1-T5: Middleware de aislamiento Multi-UR (UR Coordinadora / Coadyuvante)

**Tipo:** feat
**Rama:** `feat/S1-T5-middleware-multi-ur`

**Descripcion:**
Crear middleware global con logica de aislamiento en dos niveles para soportar programas transversales con multiples Unidades Responsables. El middleware ya NO bloquea rigidamente por `team_id` del programa, sino que verifica el rol de la UR en cada programa.

**Logica del middleware:**
1. Si el usuario pertenece a la **UR Coordinadora** del programa: acceso completo (lectura + escritura en todo el programa)
2. Si el usuario pertenece a una **UR Coadyuvante** (registrada en `programa_team`): acceso read-only al programa general + acceso de captura/edicion exclusivamente en Componentes/Actividades asignados a su UR
3. Si no tiene ninguna relacion con el programa: acceso denegado (403)

**Criterios de aceptacion:**
- [ ] Middleware registrado globalmente para rutas autenticadas
- [ ] Consulta a `programa_team` para determinar el rol (coordinadora/coadyuvante) del equipo activo del usuario
- [ ] UR Coordinadora: acceso sin restricciones al programa
- [ ] UR Coadyuvante: solo lectura en el programa; escritura limitada a sus mir_niveles asignados (verificado por `mir_niveles.team_id`)
- [ ] Aislamiento total para usuarios sin relacion con el programa (403)
- [ ] Admin puede acceder a todos los programas sin restriccion
- [ ] Test: operador de UR Coadyuvante puede capturar avance en su componente asignado
- [ ] Test: operador de UR Coadyuvante recibe 403 al intentar editar componente de otra UR
- [ ] Test: operador de UR sin relacion recibe 403 al intentar acceder al programa

---

### S1-T7: Migracion tabla pivote programa_team (Multi-UR)

**Tipo:** feat
**Rama:** `feat/S1-T7-tabla-programa-team`

**Descripcion:**
Crear la tabla pivote `programa_team` que registra todos los equipos participantes en un programa presupuestario, diferenciando entre UR Coordinadora y UR Coadyuvante. Tambien agregar FK nullable `team_id` en `mir_niveles` para asignar componentes/actividades a URs especificas.

**Campos programa_team:**
- `programa_presupuestario_id` (FK)
- `team_id` (FK)
- `rol` (ENUM: `coordinadora`, `coadyuvante`)
- `timestamps`

**Cambio en mir_niveles:**
- Agregar columna `team_id` (FK nullable a teams) — indica la UR Coadyuvante responsable del nivel

**Criterios de aceptacion:**
- [ ] Migracion `programa_team` con constraint UNIQUE en `(programa_presupuestario_id, team_id)`
- [ ] Migracion alter table `mir_niveles` agrega `team_id` nullable con FK
- [ ] Enum `rol` con valores `coordinadora` y `coadyuvante`
- [ ] Modelo `ProgramaPresupuestario` tiene relacion `belongsToMany Team` via `programa_team` con campo pivot `rol`
- [ ] Seeder de prueba: Programa X con Educacion como coordinadora y Salud como coadyuvante del Componente 2
- [ ] `migrate:fresh --seed` sin errores
- [ ] Actualizar documentacion del esquema de BD

---

### S1-T6: Seeders de datos de prueba para desarrollo

**Tipo:** chore
**Rama:** `chore/S1-T6-seeders-desarrollo`

**Descripcion:**
Crear seeders completos para desarrollo local que generen un escenario realista de prueba, incluyendo un programa transversal multi-UR.

**Datos a generar:**
- 3 Teams/UR: "Secretaria de Educacion" (coordinadora), "Secretaria de Salud" (coadyuvante), "Secretaria de Seguridad"
- 2 usuarios por team: 1 planeador + 1 operador
- 1 usuario admin global
- 1 programa presupuestario con Educacion como UR Coordinadora y Salud como UR Coadyuvante del Componente 2

**Criterios de aceptacion:**
- [ ] `php artisan db:seed` ejecuta sin errores
- [ ] Cada usuario tiene rol y team asignado correctamente
- [ ] Login con cualquier usuario de prueba funciona
- [ ] 2FA puede configurarse para usuarios de prueba
- [ ] Escenario multi-UR: el operador de Salud puede ver el programa pero solo capturar en su componente

---

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

## Sprint 6: Seguimiento y Captura Periodica

---

### S6-T1: Migraciones para tablas de seguimiento

**Tipo:** feat
**Rama:** `feat/S6-T1-migraciones-seguimiento`

**Descripcion:**
Crear tablas: `metas_periodo`, `avances`, `avance_variables`, `avance_evidencias`, `desbloqueos`.

**Criterios de aceptacion:**
- [ ] Todas las migraciones con `up()` y `down()`
- [ ] `avances.historial_observaciones` como JSONB
- [ ] `avance_evidencias.hash_archivo` para integridad SHA-256
- [ ] `desbloqueos` con relacion polimorfica
- [ ] Indices en campos de consulta frecuente
- [ ] `migrate:fresh --seed` sin errores
- [ ] Actualizar documentacion del esquema

---

### S6-T2: Calendario de captura y notificaciones

**Tipo:** feat
**Rama:** `feat/S6-T2-calendario-notificaciones`

**Descripcion:**
El sistema genera automaticamente los periodos de captura abiertos segun la frecuencia de cada indicador y notifica a la UR responsable.

**Criterios de aceptacion:**
- [ ] Comando artisan schedulable que detecta periodos por abrir
- [ ] Notificacion a operadores de la UR cuando se abre un periodo
- [ ] Vista del operador: "Mis indicadores pendientes de captura"
- [ ] Vista del planeador: "Indicadores con captura vencida"

---

### S6-T3: Formulario de captura de avance

**Tipo:** feat
**Rama:** `feat/S6-T3-formulario-captura-avance`

**Descripcion:**
Componente Livewire para que el operador capture los valores de las variables de un indicador en un periodo especifico.

**Flujo:**
1. Operador selecciona indicador y periodo
2. Sistema muestra campos por cada variable (con nombre y simbolo)
3. Al capturar valores, el sistema calcula el resultado aplicando la formula
4. Se muestra el semaforo sugerido
5. Si amarillo/rojo: formulario de justificacion obligatorio

**Criterios de aceptacion:**
- [ ] Campos dinamicos segun variables del indicador
- [ ] Calculo automatico usando MathExecutor con simbolos mapeados
- [ ] Semaforo calculado segun rangos y sentido del indicador
- [ ] El usuario puede ajustar el semaforo con justificacion
- [ ] Solo accesible con permiso `capturar_avance`
- [ ] Solo para indicadores con `activo_seguimiento = true`

---

### S6-T4: Generacion de justificaciones con IA

**Tipo:** feat
**Rama:** `feat/S6-T4-justificaciones-ia`

**Descripcion:**
Cuando un indicador cae en amarillo o rojo, la IA genera un borrador de justificacion que el usuario edita y aprueba.

**Fuentes de la IA (exclusivamente):**
- Supuestos del nivel correspondiente (Col. 4 de la MIR)
- Magnitud numerica de la desviacion respecto a la meta del periodo
- Historial del indicador (ejercicio anterior si existe)

**Criterios de aceptacion:**
- [ ] Borrador generado automaticamente al detectar amarillo/rojo
- [ ] El borrador cita explicitamente los supuestos de la MIR
- [ ] El usuario puede editar libremente antes de guardar
- [ ] El texto final se guarda en `avances.justificacion_final`
- [ ] El borrador de IA se guarda en `avances.justificacion_ia` (para auditoria)
- [ ] La IA nunca inventa contexto externo

---

### S6-T5: Adjuntar medios de verificacion (evidencia)

**Tipo:** feat
**Rama:** `feat/S6-T5-adjuntar-evidencia`

**Descripcion:**
Funcionalidad para que el operador adjunte archivos de evidencia por cada avance reportado.

**Criterios de aceptacion:**
- [ ] Upload de archivos (PDF, Excel, imagenes)
- [ ] Registro de: nombre, area generadora, fecha del documento
- [ ] Generacion automatica de hash SHA-256 al subir
- [ ] Validacion: el medio adjuntado corresponde al registrado en la MIR (advertencia si difiere)
- [ ] Almacenamiento seguro (no accesible publicamente)

---

### S6-T6: Maquina de estados del reporte de avance

**Tipo:** feat
**Rama:** `feat/S6-T6-maquina-estados-avance`

**Descripcion:**
Implementar el flujo de estados: En Captura → En Revision → Observado → Aprobado y Congelado.

**Criterios de aceptacion:**
- [ ] Estado inicial: `en_captura` (editable por operador)
- [ ] Transicion a `en_revision`: operador "envia" el reporte. Se bloquea para edicion
- [ ] Transicion a `observado`: planeador rechaza con comentarios (se agregan al historial JSONB)
- [ ] Transicion a `aprobado`: planeador aprueba. Se registra `congelado_at`
- [ ] Post-aprobacion: ni variables, ni justificacion, ni archivos son editables
- [ ] Notificacion al operador cuando su reporte es observado
- [ ] Notificacion al planeador cuando hay reportes en revision
- [ ] Solo planeador puede aprobar (`aprobar_avance`)
- [ ] Historial de observaciones visible como timeline

---

### S6-T7: Congelamiento y desbloqueo excepcional

**Tipo:** feat
**Rama:** `feat/S6-T7-congelamiento-desbloqueo`

**Descripcion:**
Implementar la inmutabilidad post-aprobacion y el flujo de desbloqueo excepcional con registro de auditoria.

**Criterios de aceptacion:**
- [ ] Avance aprobado: todos los campos inmutables
- [ ] Archivo adjunto congelado: no se puede reemplazar ni eliminar
- [ ] Flujo de desbloqueo: operador solicita → admin aprueba/rechaza
- [ ] Registro en tabla `desbloqueos` con motivo, solicitante, aprobador
- [ ] Al desbloquear, se crea entrada en el historial de observaciones
- [ ] Solo admin puede aprobar desbloqueos

---

### S6-T8: Vista de seguimiento para planeadores

**Tipo:** feat
**Rama:** `feat/S6-T8-vista-seguimiento-planeador`

**Descripcion:**
Panel consolidado donde el planeador ve todos los programas de su UR con sus indicadores, metas, avances y semaforos.

**Criterios de aceptacion:**
- [ ] Tabla resumen: Nivel | Indicador | Meta periodo | Avance | Semaforo
- [ ] Expandible: al clic, muestra historial de capturas, variables, justificacion, evidencia
- [ ] Filtros: por programa, por estado de avance, por semaforo
- [ ] Vista comparativa: ejercicio actual vs anterior (side by side)
- [ ] Solo accesible con permiso `revisar_avance`

---

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
