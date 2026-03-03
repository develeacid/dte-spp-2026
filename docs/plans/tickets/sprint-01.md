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