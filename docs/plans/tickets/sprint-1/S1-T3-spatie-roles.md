# Plan: S1-T3 — Integrar Spatie/laravel-permission

**Ticket:** S1-T3
**Tipo:** feat
**Rama:** `feat/S1-T3-integracion-spatie-roles`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S1-T1

---

## Contexto

`spatie/laravel-permission` es el paquete estándar de la comunidad Laravel para roles y permisos. Se instala sobre el modelo `User` existente de Jetstream.

**Decisión de diseño importante:** Este sistema **coexiste** con el sistema de permisos de Teams de Jetstream. Los roles de Spatie (`admin`, `planeador`, `operador`) controlan el **qué puede hacer** el usuario en el sistema. Los equipos de Jetstream controlan **a qué UR pertenece**. Son complementarios.

El paquete usa un trait `HasRoles` en el modelo `User` que agrega métodos como `$user->hasPermissionTo()`, `$user->assignRole()`, etc.

---

## Pre-requisitos

- S1-T1 completado (Jetstream instalado con tabla `users`)
- `config/permission.php` no debe existir previamente

---

## Pasos

### 1. Instalar el paquete

```bash
sail composer require spatie/laravel-permission
```

### 2. Publicar migraciones y configuración

```bash
sail artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
```

Esto genera:
- `config/permission.php` — configuración del paquete
- Migración para tablas: `roles`, `permissions`, `model_has_roles`, `model_has_permissions`, `role_has_permissions`

### 3. Limpiar caché de configuración

```bash
sail artisan optimize:clear
```

### 4. Ejecutar migraciones

```bash
sail artisan migrate
```

### 5. Agregar el trait HasRoles al modelo User

Abrir `app/Models/User.php` y agregar el trait:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Jetstream\HasTeams;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasTeams;
    use HasRoles;

    // ... resto del modelo sin cambios
}
```

### 6. Crear el Seeder de Roles y Permisos

```bash
sail artisan make:seeder RolesAndPermissionsSeeder
```

Editar `database/seeders/RolesAndPermissionsSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;
use Spatie\Permission\PermissionRegistrar;

class RolesAndPermissionsSeeder extends Seeder
{
    public function run(): void
    {
        // Limpiar cache de permisos antes de crear
        app()[PermissionRegistrar::class]->forgetCachedPermissions();

        // Crear permisos
        $permisos = [
            'gestionar_catalogos',
            'crear_programa',
            'editar_mir',
            'capturar_avance',
            'revisar_avance',
            'aprobar_avance',
            'exportar_reportes',
            'administrar_usuarios',
        ];

        foreach ($permisos as $permiso) {
            Permission::create(['name' => $permiso]);
        }

        // Crear roles y asignar permisos
        $admin = Role::create(['name' => 'admin']);
        $admin->givePermissionTo(Permission::all());

        $planeador = Role::create(['name' => 'planeador']);
        $planeador->givePermissionTo([
            'gestionar_catalogos',
            'crear_programa',
            'editar_mir',
            'revisar_avance',
            'aprobar_avance',
            'exportar_reportes',
        ]);

        $operador = Role::create(['name' => 'operador']);
        $operador->givePermissionTo([
            'capturar_avance',
            'exportar_reportes',
        ]);
    }
}
```

### 7. Registrar el seeder en DatabaseSeeder

Abrir `database/seeders/DatabaseSeeder.php` y agregar:

```php
public function run(): void
{
    $this->call([
        RolesAndPermissionsSeeder::class,
    ]);
}
```

### 8. Ejecutar el seeder

```bash
sail artisan db:seed --class=RolesAndPermissionsSeeder
```

### 9. Verificar con Tinker

```bash
sail artisan tinker
```

```php
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

// Verificar roles creados
Role::all()->pluck('name');
// => ["admin", "planeador", "operador"]

// Verificar permisos del planeador
Role::findByName('planeador')->permissions->pluck('name');
// => ["gestionar_catalogos", "crear_programa", "editar_mir", "revisar_avance", "aprobar_avance", "exportar_reportes"]

// Verificar que operador NO tiene crear_programa
Role::findByName('operador')->hasPermissionTo('crear_programa');
// => false

exit
```

### 10. Crear tests de verificación

```bash
sail artisan make:test RolesAndPermissionsTest
```

Editar `tests/Feature/RolesAndPermissionsTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Spatie\Permission\Models\Role;
use Tests\TestCase;

class RolesAndPermissionsTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        $this->seed(\Database\Seeders\RolesAndPermissionsSeeder::class);
    }

    public function test_planeador_tiene_permiso_crear_programa(): void
    {
        $user = User::factory()->create();
        $user->assignRole('planeador');

        $this->assertTrue($user->hasPermissionTo('crear_programa'));
    }

    public function test_operador_no_tiene_permiso_crear_programa(): void
    {
        $user = User::factory()->create();
        $user->assignRole('operador');

        $this->assertFalse($user->hasPermissionTo('crear_programa'));
    }
}
```

Ejecutar los tests:

```bash
sail artisan test --filter RolesAndPermissionsTest
```

---

## Criterios de aceptación

- [ ] Paquete instalado y migraciones ejecutadas (tablas `roles`, `permissions`, `model_has_roles`, etc.)
- [ ] Seeder `RolesAndPermissionsSeeder` crea los 3 roles y 8 permisos correctamente
- [ ] Modelo `User` usa el trait `HasRoles`
- [ ] Test: usuario con rol `planeador` tiene permiso `crear_programa`
- [ ] Test: usuario con rol `operador` **NO** tiene permiso `crear_programa`

---

## Resumen de roles y permisos

| Permiso               | admin | planeador | operador |
|-----------------------|:-----:|:---------:|:--------:|
| gestionar_catalogos   |  ✓   |    ✓     |          |
| crear_programa        |  ✓   |    ✓     |          |
| editar_mir            |  ✓   |    ✓     |          |
| capturar_avance       |  ✓   |           |    ✓    |
| revisar_avance        |  ✓   |    ✓     |          |
| aprobar_avance        |  ✓   |    ✓     |          |
| exportar_reportes     |  ✓   |    ✓     |    ✓    |
| administrar_usuarios  |  ✓   |           |          |

---

## Notas

- Spatie usa guards de Laravel. El guard por defecto (`web`) es correcto para esta aplicación
- Los permisos se cachean automáticamente — al modificar roles/permisos en producción, ejecutar `php artisan permission:cache-reset`
- No usar el feature de "teams" de Spatie (`'teams' => false` en `config/permission.php`) — el aislamiento multi-UR se maneja con el middleware de S1-T5, no con el sistema de teams de Spatie
- Para proteger rutas se usará middleware: `->middleware('permission:crear_programa')` o `$this->authorize('crear_programa')` en controladores
