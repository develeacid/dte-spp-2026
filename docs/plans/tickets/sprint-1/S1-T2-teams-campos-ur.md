# Plan: S1-T2 — Extender tabla teams con campos de Unidad Responsable

**Ticket:** S1-T2
**Tipo:** feat
**Rama:** `feat/S1-T2-teams-campos-ur`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S1-T1

---

## Contexto

Jetstream crea la tabla `teams` con campos base (`id`, `user_id`, `name`, `personal_team`, `timestamps`). En este sistema, cada `Team` representa una **Unidad Responsable (UR)** del gobierno estatal. Se deben agregar campos específicos del dominio: clave oficial, titular, tipo de UR y estado activo.

El modelo `Team` generado por Jetstream está en `app/Models/Team.php` y usa `HasFactory`.

---

## Pre-requisitos

- S1-T1 completado (Jetstream con Teams instalado y migrado)

---

## Pasos

### 1. Crear la migración

```bash
sail artisan make:migration add_campos_ur_to_teams_table --table=teams
```

### 2. Editar la migración

Abrir el archivo generado en `database/migrations/` y escribir:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('teams', function (Blueprint $table) {
            $table->string('clave_ur')->nullable()->unique()->after('name');
            $table->string('titular')->nullable()->after('clave_ur');
            $table->enum('tipo_ur', ['sustantiva', 'apoyo'])->nullable()->after('titular');
            $table->boolean('activa')->default(true)->after('tipo_ur');
        });
    }

    public function down(): void
    {
        Schema::table('teams', function (Blueprint $table) {
            $table->dropColumn(['clave_ur', 'titular', 'tipo_ur', 'activa']);
        });
    }
};
```

### 3. Ejecutar la migración

```bash
sail artisan migrate
```

### 4. Actualizar el modelo Team

Abrir `app/Models/Team.php` y agregar los nuevos campos al `$fillable`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Laravel\Jetstream\Events\TeamCreated;
use Laravel\Jetstream\Events\TeamDeleted;
use Laravel\Jetstream\Events\TeamUpdated;
use Laravel\Jetstream\Team as JetstreamTeam;

class Team extends JetstreamTeam
{
    use HasFactory;

    protected $fillable = [
        'name',
        'personal_team',
        'clave_ur',
        'titular',
        'tipo_ur',
        'activa',
    ];

    protected $casts = [
        'personal_team' => 'boolean',
        'activa' => 'boolean',
    ];

    protected $dispatchesEvents = [
        'created' => TeamCreated::class,
        'updated' => TeamUpdated::class,
        'deleted' => TeamDeleted::class,
    ];
}
```

### 5. Crear el Seeder de URs de ejemplo

```bash
sail artisan make:seeder UnidadesResponsablesSeeder
```

Editar `database/seeders/UnidadesResponsablesSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\Team;
use Illuminate\Database\Seeder;

class UnidadesResponsablesSeeder extends Seeder
{
    public function run(): void
    {
        // UR Sustantiva
        Team::create([
            'user_id'      => 1, // Se sobreescribe en S1-T6
            'name'         => 'Secretaría de Educación',
            'clave_ur'     => 'SE-001',
            'titular'      => 'Dr. Juan Pérez',
            'tipo_ur'      => 'sustantiva',
            'activa'       => true,
            'personal_team' => false,
        ]);

        // UR de Apoyo
        Team::create([
            'user_id'      => 1,
            'name'         => 'Secretaría de Salud',
            'clave_ur'     => 'SS-002',
            'titular'      => 'Dra. María López',
            'tipo_ur'      => 'apoyo',
            'activa'       => true,
            'personal_team' => false,
        ]);

        // UR Inactiva
        Team::create([
            'user_id'      => 1,
            'name'         => 'Secretaría de Seguridad',
            'clave_ur'     => 'SEG-003',
            'titular'      => 'Lic. Roberto Sánchez',
            'tipo_ur'      => 'sustantiva',
            'activa'       => false,
            'personal_team' => false,
        ]);
    }
}
```

> **Nota:** Este seeder se integrará con los usuarios en S1-T6. Por ahora usa `user_id = 1` como placeholder.

### 6. Verificar con migrate:fresh

```bash
sail artisan migrate:fresh
```

Debe ejecutar todas las migraciones en orden (S0-T2 pgvector → S0-T1 base → S1-T1 Jetstream → S1-T2 campos UR) sin errores.

### 7. Confirmar los campos en la base de datos

```bash
sail artisan tinker
```

```php
Schema::getColumnListing('teams');
// Debe incluir: id, user_id, name, personal_team, clave_ur, titular, tipo_ur, activa, created_at, updated_at
exit
```

---

## Criterios de aceptación

- [ ] Migración con `up()` y `down()` completos
- [ ] `sail artisan migrate` y `migrate:fresh` ejecutan sin errores
- [ ] Modelo `Team` extendido con `$fillable` actualizado (`clave_ur`, `titular`, `tipo_ur`, `activa`)
- [ ] Seeder `UnidadesResponsablesSeeder` crea 3 URs de ejemplo (1 sustantiva, 1 apoyo, 1 inactiva)

---

## Notas

- `personal_team = true` identifica el equipo personal auto-creado por Jetstream al registrar un usuario — estos no son URs reales, se filtrarán en la UI
- `clave_ur` es `nullable` para permitir que los teams personales de Jetstream no requieran clave
- La documentación del esquema de BD se actualiza en el archivo `docs/schema/teams.md` (crear si no existe)
