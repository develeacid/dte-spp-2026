# Plan: S1-T6 — Seeders de datos de prueba para desarrollo

**Ticket:** S1-T6
**Tipo:** chore
**Rama:** `chore/S1-T6-seeders-desarrollo`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S1-T1, S1-T2, S1-T3 (roles, permisos y campos UR deben existir)

---

## Contexto

Los seeders de desarrollo generan un escenario realista completo que permite trabajar el sistema sin datos reales. Incluyen el escenario **multi-UR** crítico para probar el middleware de S1-T5:

- **Secretaría de Educación** = UR Coordinadora de un programa transversal
- **Secretaría de Salud** = UR Coadyuvante del mismo programa (responsable del Componente 2)
- **Secretaría de Seguridad** = UR sin participación en el programa transversal

Cada UR tiene 2 usuarios: 1 planeador y 1 operador. Más un admin global.

---

## Pre-requisitos

- S1-T1 completado (Jetstream con Teams)
- S1-T2 completado (campos `clave_ur`, `titular`, `tipo_ur`, `activa` en `teams`)
- S1-T3 completado (roles `admin`, `planeador`, `operador` y permisos)

---

## Pasos

### 1. Crear el seeder principal de desarrollo

```bash
sail artisan make:seeder DesarrolloSeeder
```

Editar `database/seeders/DesarrolloSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\Team;
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\PermissionRegistrar;

class DesarrolloSeeder extends Seeder
{
    public function run(): void
    {
        // Limpiar cache de permisos
        app()[PermissionRegistrar::class]->forgetCachedPermissions();

        // ── 1. ADMIN GLOBAL ──────────────────────────────────────
        $admin = User::create([
            'name'     => 'Administrador del Sistema',
            'email'    => 'admin@sistema.test',
            'password' => Hash::make('password'),
        ]);
        $admin->assignRole('admin');

        // ── 2. UR COORDINADORA: Secretaría de Educación ──────────
        $urEducacion = Team::create([
            'user_id'      => $admin->id,
            'name'         => 'Secretaría de Educación',
            'clave_ur'     => 'SE-001',
            'titular'      => 'Dr. Juan Pérez García',
            'tipo_ur'      => 'sustantiva',
            'activa'       => true,
            'personal_team' => false,
        ]);

        $planeadorEdu = $this->crearUsuario(
            'Planeador Educación',
            'planeador.edu@sistema.test',
            'planeador',
            $urEducacion
        );

        $operadorEdu = $this->crearUsuario(
            'Operador Educación',
            'operador.edu@sistema.test',
            'operador',
            $urEducacion
        );

        // ── 3. UR COADYUVANTE: Secretaría de Salud ───────────────
        $urSalud = Team::create([
            'user_id'      => $admin->id,
            'name'         => 'Secretaría de Salud',
            'clave_ur'     => 'SS-002',
            'titular'      => 'Dra. María López Hernández',
            'tipo_ur'      => 'apoyo',
            'activa'       => true,
            'personal_team' => false,
        ]);

        $planeadorSalud = $this->crearUsuario(
            'Planeador Salud',
            'planeador.salud@sistema.test',
            'planeador',
            $urSalud
        );

        $operadorSalud = $this->crearUsuario(
            'Operador Salud',
            'operador.salud@sistema.test',
            'operador',
            $urSalud
        );

        // ── 4. UR SIN PARTICIPACIÓN: Secretaría de Seguridad ─────
        $urSeguridad = Team::create([
            'user_id'      => $admin->id,
            'name'         => 'Secretaría de Seguridad',
            'clave_ur'     => 'SEG-003',
            'titular'      => 'Lic. Roberto Sánchez Cruz',
            'tipo_ur'      => 'sustantiva',
            'activa'       => true,
            'personal_team' => false,
        ]);

        $planeadorSeg = $this->crearUsuario(
            'Planeador Seguridad',
            'planeador.seg@sistema.test',
            'planeador',
            $urSeguridad
        );

        $operadorSeg = $this->crearUsuario(
            'Operador Seguridad',
            'operador.seg@sistema.test',
            'operador',
            $urSeguridad
        );

        $this->command->info('✓ Seeders de desarrollo completados.');
        $this->command->table(
            ['Usuario', 'Email', 'Rol', 'UR'],
            [
                ['Admin', 'admin@sistema.test', 'admin', '—'],
                ['Planeador Edu', 'planeador.edu@sistema.test', 'planeador', 'Educación'],
                ['Operador Edu', 'operador.edu@sistema.test', 'operador', 'Educación'],
                ['Planeador Salud', 'planeador.salud@sistema.test', 'planeador', 'Salud'],
                ['Operador Salud', 'operador.salud@sistema.test', 'operador', 'Salud'],
                ['Planeador Seg', 'planeador.seg@sistema.test', 'planeador', 'Seguridad'],
                ['Operador Seg', 'operador.seg@sistema.test', 'operador', 'Seguridad'],
            ]
        );
        $this->command->info('Contraseña de todos los usuarios: password');
    }

    private function crearUsuario(string $nombre, string $email, string $rol, Team $team): User
    {
        $user = User::create([
            'name'     => $nombre,
            'email'    => $email,
            'password' => Hash::make('password'),
        ]);

        $user->assignRole($rol);

        // Asignar team como equipo actual
        $user->teams()->attach($team->id);
        $user->forceFill(['current_team_id' => $team->id])->save();

        return $user;
    }
}
```

### 2. Actualizar DatabaseSeeder

Editar `database/seeders/DatabaseSeeder.php`:

```php
public function run(): void
{
    $this->call([
        RolesAndPermissionsSeeder::class,
        DesarrolloSeeder::class,
    ]);
}
```

### 3. Ejecutar los seeders

```bash
sail artisan migrate:fresh --seed
```

### 4. Verificar el escenario

```bash
sail artisan tinker
```

```php
use App\Models\User;

// Verificar admin
$admin = User::where('email', 'admin@sistema.test')->first();
$admin->hasRole('admin'); // true

// Verificar planeador de Educación
$planeador = User::where('email', 'planeador.edu@sistema.test')->first();
$planeador->hasRole('planeador');          // true
$planeador->currentTeam->name;             // "Secretaría de Educación"
$planeador->currentTeam->clave_ur;         // "SE-001"
$planeador->hasPermissionTo('crear_programa'); // true

// Verificar operador de Salud
$operador = User::where('email', 'operador.salud@sistema.test')->first();
$operador->hasRole('operador');                 // true
$operador->currentTeam->name;                   // "Secretaría de Salud"
$operador->hasPermissionTo('capturar_avance');  // true
$operador->hasPermissionTo('crear_programa');   // false

exit
```

### 5. Probar login en el navegador

Acceder a `http://localhost/login` con cualquier usuario de prueba:
- `admin@sistema.test` / `password`
- `planeador.edu@sistema.test` / `password`
- `operador.salud@sistema.test` / `password`

---

## Criterios de aceptación

- [ ] `sail artisan db:seed` ejecuta sin errores
- [ ] Se crean 7 usuarios con roles y teams correctamente asignados
- [ ] Login con cualquier usuario de prueba funciona en `http://localhost/login`
- [ ] `$user->currentTeam` retorna la UR correcta para cada usuario
- [ ] `$user->hasPermissionTo(...)` respeta los permisos por rol
- [ ] 2FA puede configurarse para usuarios de prueba (Jetstream lo permite en `/user/profile`)

---

## Usuarios de prueba

| Email                          | Password   | Rol       | UR                       |
|--------------------------------|------------|-----------|--------------------------|
| admin@sistema.test             | password   | admin     | —                        |
| planeador.edu@sistema.test     | password   | planeador | Secretaría de Educación  |
| operador.edu@sistema.test      | password   | operador  | Secretaría de Educación  |
| planeador.salud@sistema.test   | password   | planeador | Secretaría de Salud      |
| operador.salud@sistema.test    | password   | operador  | Secretaría de Salud      |
| planeador.seg@sistema.test     | password   | planeador | Secretaría de Seguridad  |
| operador.seg@sistema.test      | password   | operador  | Secretaría de Seguridad  |

---

## Notas

- La asociación del escenario multi-UR (Educación=coordinadora, Salud=coadyuvante en el programa transversal) se completa en **S3-T1** cuando exista la tabla `programa_team`
- Todos los usuarios usan la misma contraseña `password` — **solo para desarrollo local**, nunca en staging/producción
- El 2FA se puede configurar manualmente en `/user/profile` para cada usuario de prueba
- `migrate:fresh --seed` borra todos los datos — usar solo en desarrollo local
