# Plan: S1-T1 — Instalar Laravel Jetstream con Teams

**Ticket:** S1-T1
**Tipo:** feat
**Rama:** `feat/S1-T1-jetstream-teams`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S0-T0, S0-T1 (Sail corriendo con PostgreSQL)

---

## Contexto

Jetstream es el starter kit oficial de Laravel para autenticación y gestión de usuarios. Se instala sobre una aplicación **nueva** de Laravel (S0-T0 ya lo cumple). El stack **Livewire** con `--teams` es el requerido por el sistema para soporte multi-UR.

La flag `--teams` genera las tablas `teams`, `team_user`, `team_invitations` y configura el modelo `User` con el trait `HasTeams`, que habilita `$user->currentTeam`, `$user->allTeams()`, etc.

---

## Pre-requisitos

- S0-T0 completado (Laravel 12 en raíz)
- S0-T1 completado (`sail up -d` corriendo con PostgreSQL pgvector y Redis)
- S0-T4 completado (Redis como driver de colas)

---

## Pasos

### 1. Instalar el paquete Jetstream

```bash
sail composer require laravel/jetstream
```

### 2. Ejecutar el instalador con Livewire y Teams

```bash
sail artisan jetstream:install livewire --teams
```

Esto genera:
- Rutas de autenticación en `routes/web.php`
- Vistas Blade en `resources/views/`
- Componentes Livewire en `app/Livewire/`
- Acciones en `app/Actions/Jetstream/`
- Políticas en `app/Policies/`
- Configuración en `config/jetstream.php`

### 3. Instalar dependencias de frontend y compilar assets

```bash
sail npm install
sail npm run build
```

### 4. Ejecutar las migraciones

```bash
sail artisan migrate
```

Tablas creadas:
- `users` (con campos de Jetstream: two_factor_secret, two_factor_recovery_codes, profile_photo_path, current_team_id)
- `teams`
- `team_user`
- `team_invitations`
- `sessions`
- `personal_access_tokens`
- `password_reset_tokens`

### 5. Verificar el flujo de registro

Abrir en el navegador: `http://localhost`

Flujo a verificar:
1. Click en **Register** — formulario de registro accesible
2. Crear usuario de prueba (ej. `admin@test.com` / `password`)
3. Al registrar, se crea automáticamente un equipo personal (ej. "Admin's Team")
4. Verificar que el dashboard carga correctamente
5. Click en el menú de equipo (top-right) — debe mostrar el selector de equipos
6. Ir a **Team Settings** — gestión de miembros accesible

### 6. Verificar login y logout

```bash
# Login
http://localhost/login

# Logout (desde el menú de usuario)
http://localhost/logout
```

### 7. Verificar la interfaz de gestión de equipos

Desde el dashboard, ir a `http://localhost/teams/create` — debe mostrar formulario de creación de equipo.

### 8. Verificar tests de Jetstream

Jetstream instala tests automáticamente:

```bash
sail artisan test
```

Los tests de Jetstream (`tests/Feature/`) deben pasar.

---

## Criterios de aceptación

- [ ] `composer require laravel/jetstream` ejecutado
- [ ] Jetstream instalado con `--teams` flag
- [ ] `sail artisan migrate` crea las tablas: users, teams, team_user, team_invitations, sessions, personal_access_tokens
- [ ] Flujo de registro crea usuario + team personal
- [ ] Login y logout funcionan correctamente
- [ ] Interfaz de gestión de equipos accesible en `/teams`

---

## Archivos clave generados por Jetstream

```
app/
├── Actions/Jetstream/
│   ├── AddTeamMember.php
│   ├── CreateTeam.php
│   ├── DeleteTeam.php
│   ├── DeleteUser.php
│   ├── InviteTeamMember.php
│   ├── RemoveTeamMember.php
│   └── UpdateTeamName.php
├── Models/
│   ├── Team.php         ← Modelo de equipo/UR
│   ├── TeamInvitation.php
│   └── User.php         ← Con trait HasTeams
├── Policies/
│   └── TeamPolicy.php
config/
└── jetstream.php        ← Configuración de features
```

---

## Notas

- Las migraciones de Jetstream reemplazarán la tabla `users` base de Laravel — ejecutar `migrate:fresh` si ya existía
- El equipo "personal" se crea automáticamente en el registro; en el sistema final representará la UR del usuario
- En S1-T2 se extenderá la tabla `teams` con campos de Unidad Responsable (`clave_ur`, `titular`, etc.)
- No habilitar el feature de `invitations` en `config/jetstream.php` por ahora — se evaluará en sprints posteriores
