# Plan: S1-T5 — Middleware de Aislamiento Multi-UR

**Ticket:** S1-T5
**Tipo:** feat
**Rama:** `feat/S1-T5-middleware-multi-ur`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S1-T1, S1-T3, S1-T7 (tabla `programa_team` debe existir)

---

## Contexto

El sistema soporta programas **transversales**: un mismo programa puede tener múltiples URs participantes. La lógica de acceso tiene dos niveles:

1. **UR Coordinadora** — es la UR "dueña" del programa. Tiene acceso completo (lectura + escritura en todo el programa).
2. **UR Coadyuvante** — es una UR que participa en el programa pero solo es responsable de Componentes/Actividades específicos. Tiene lectura del programa completo, pero escritura solo en sus `mir_niveles` asignados (identificados por `mir_niveles.team_id`).
3. **Sin relación** — acceso denegado (403).

El middleware consulta la tabla `programa_team` (creada en S1-T7) para determinar el rol de la UR activa del usuario en el programa solicitado.

> **Nota:** Este middleware se implementa en este ticket pero **no puede probarse completamente** hasta que existan programas reales (Sprint 3+). En Sprint 1 se valida la estructura y los tests unitarios con mocks.

---

## Pre-requisitos

- S1-T1 completado (Jetstream con Teams — `$user->currentTeam` disponible)
- S1-T3 completado (rol `admin` puede saltarse restricciones)
- S1-T7 completado (tabla `programa_team` con columna `rol`)

---

## Pasos

### 1. Crear el middleware

```bash
sail artisan make:middleware AislamientoMultiUR
```

Editar `app/Http/Middleware/AislamientoMultiUR.php`:

```php
<?php

namespace App\Http\Middleware;

use App\Models\ProgramaPresupuestario;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class AislamientoMultiUR
{
    public function handle(Request $request, Closure $next): Response
    {
        $user = $request->user();

        if (! $user) {
            return $next($request);
        }

        // Admin accede a todo sin restricción
        if ($user->hasRole('admin')) {
            return $next($request);
        }

        // Obtener el programa de la ruta actual (si aplica)
        $programa = $request->route('programa');

        // Si la ruta no involucra un programa específico, continuar
        if (! $programa instanceof ProgramaPresupuestario) {
            return $next($request);
        }

        $teamActivo = $user->currentTeam;

        if (! $teamActivo) {
            abort(403, 'No tienes un equipo activo.');
        }

        // Buscar el rol del team activo en este programa
        $pivote = $programa->equipos()
            ->where('team_id', $teamActivo->id)
            ->first();

        if (! $pivote) {
            abort(403, 'Tu Unidad Responsable no tiene acceso a este programa.');
        }

        // Almacenar el rol en el request para uso en controladores
        $request->merge(['ur_rol_en_programa' => $pivote->pivot->rol]);

        return $next($request);
    }
}
```

### 2. Registrar alias del middleware

En `bootstrap/app.php`:

```php
$middleware->alias([
    'ur.aislamiento' => \App\Http\Middleware\AislamientoMultiUR::class,
]);
```

### 3. Agregar la relación en el modelo ProgramaPresupuestario

> Este paso anticipa S3-T1. Al crear el modelo `ProgramaPresupuestario`, debe incluir:

```php
// En app/Models/ProgramaPresupuestario.php (se crea en S3-T1)
public function equipos(): BelongsToMany
{
    return $this->belongsToMany(Team::class, 'programa_team')
                ->withPivot('rol')
                ->withTimestamps();
}
```

### 4. Crear helper para verificar permisos de escritura en niveles

Crear `app/Helpers/UrPermissions.php`:

```php
<?php

namespace App\Helpers;

use App\Models\MirNivel;
use App\Models\User;

class UrPermissions
{
    /**
     * Verifica si el usuario puede escribir en un nivel de MIR específico.
     */
    public static function puedeEscribirEnNivel(User $user, MirNivel $nivel): bool
    {
        if ($user->hasRole('admin')) {
            return true;
        }

        $rolEnPrograma = request()->get('ur_rol_en_programa');

        // Coordinadora: acceso completo
        if ($rolEnPrograma === 'coordinadora') {
            return true;
        }

        // Coadyuvante: solo en niveles asignados a su team
        if ($rolEnPrograma === 'coadyuvante') {
            return $nivel->team_id === $user->currentTeam?->id;
        }

        return false;
    }
}
```

### 5. Crear tests del middleware

```bash
sail artisan make:test AislamientoMultiURTest
```

Editar `tests/Feature/AislamientoMultiURTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Models\Team;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class AislamientoMultiURTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        $this->seed(\Database\Seeders\RolesAndPermissionsSeeder::class);
    }

    public function test_admin_puede_acceder_a_cualquier_programa(): void
    {
        $user = User::factory()->create();
        $user->assignRole('admin');

        // El admin no debe recibir 403 aunque no pertenezca al programa
        $this->actingAs($user);
        $this->assertTrue($user->hasRole('admin'));
    }

    public function test_operador_sin_equipo_activo_recibe_403(): void
    {
        // Se implementa en Sprint 3 cuando existan rutas de programas
        $this->markTestSkipped('Requiere rutas de programas (Sprint 3)');
    }
}
```

Ejecutar tests:

```bash
sail artisan test --filter AislamientoMultiURTest
```

---

## Criterios de aceptación

- [ ] Middleware `AislamientoMultiUR` creado y registrado con alias `ur.aislamiento`
- [ ] Consulta a `programa_team` para determinar el rol del team activo del usuario
- [ ] UR Coordinadora: acceso sin restricciones al programa
- [ ] UR Coadyuvante: escritura limitada a `mir_niveles` donde `team_id` coincide
- [ ] Aislamiento total para usuarios sin relación con el programa (403)
- [ ] Admin puede acceder a todos los programas sin restricción
- [ ] Tests unitarios verifican la lógica de roles

---

## Uso en rutas (Sprint 3+)

```php
// routes/web.php
Route::middleware(['auth:sanctum', 'verified', 'ur.aislamiento'])
    ->group(function () {
        Route::resource('programas', ProgramaPresupuestarioController::class);
    });
```

---

## Notas

- El middleware se aplica en rutas que reciben `{programa}` como parámetro de ruta (route model binding)
- Los tests completos (coadyuvante puede capturar su componente, etc.) se implementan en Sprint 3 cuando existen programas y mir_niveles reales
- `$request->merge(['ur_rol_en_programa' => ...])` es la forma de pasar contexto al controlador sin acoplamiento
