# S2-T4 — Migraciones y modelos para Programas Derivados

**Tipo:** feat
**Rama:** `feat/S2-T4-programas-derivados`
**Depende de:** S2-T3 (tabla `ped_planes` existe)

---

## Contexto

Los Programas Derivados son instrumentos de planeación que se desprenden de un Plan Estatal de Desarrollo (PED). Cada programa puede ser de tipo sectorial, especial, institucional o regional. A su vez, cada programa contiene objetivos con descripción textual y un embedding vectorial (1536 dimensiones, compatible con OpenAI `text-embedding-3-small`) para búsqueda semántica futura.

Este ticket crea las dos tablas necesarias, sus migraciones, los modelos Eloquent con relaciones, y un seeder de ejemplo.

---

## Pre-requisitos

- Sprint 0 completado: pgvector habilitado en PostgreSQL 16, extensión `vector` activa en la base de datos.
- Sprint 2 T3 completado: tabla `ped_planes` existe y modelo `PedPlan` está disponible.
- Sail corriendo: `sail up -d`
- Verificar que la extensión vector está activa:

```bash
sail psql -c "\dx vector"
```

Debe mostrar la extensión `vector` en el listado.

---

## Pasos

### Paso 1: Crear la migración para `programas_derivados`

```bash
sail artisan make:migration create_programas_derivados_table
```

Editar el archivo generado en `database/migrations/YYYY_MM_DD_HHMMSS_create_programas_derivados_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        // Crear el tipo ENUM en PostgreSQL si no existe
        DB::statement("
            DO $$ BEGIN
                CREATE TYPE tipo_programa_derivado AS ENUM (
                    'sectorial',
                    'especial',
                    'institucional',
                    'regional'
                );
            EXCEPTION
                WHEN duplicate_object THEN null;
            END $$;
        ");

        Schema::create('programas_derivados', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_plan_id')
                  ->constrained('ped_planes')
                  ->cascadeOnDelete();
            $table->string('tipo'); // se validará en capa de aplicación; el ENUM nativo queda en DB
            $table->string('nombre');
            $table->text('descripcion')->nullable();
            $table->timestamps();
        });

        // Aplicar el tipo ENUM nativo de PostgreSQL a la columna
        DB::statement("
            ALTER TABLE programas_derivados
            ALTER COLUMN tipo TYPE tipo_programa_derivado
            USING tipo::tipo_programa_derivado;
        ");
    }

    public function down(): void
    {
        Schema::dropIfExists('programas_derivados');
        DB::statement("DROP TYPE IF EXISTS tipo_programa_derivado;");
    }
};
```

> **Nota sobre ENUMs en PostgreSQL con Laravel:** Laravel no tiene soporte nativo para ENUMs de PostgreSQL a través de Blueprint. La estrategia anterior crea el tipo nativo (`CREATE TYPE`) y lo asigna via `ALTER TABLE`. Esto garantiza integridad en la base de datos. En capa de aplicación se usa una regla `Rule::in([...])`.

---

### Paso 2: Crear la migración para `programas_derivados_objetivos`

```bash
sail artisan make:migration create_programas_derivados_objetivos_table
```

Editar el archivo generado en `database/migrations/YYYY_MM_DD_HHMMSS_create_programas_derivados_objetivos_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('programas_derivados_objetivos', function (Blueprint $table) {
            $table->id();
            $table->foreignId('programa_derivado_id')
                  ->constrained('programas_derivados')
                  ->cascadeOnDelete();
            $table->string('clave', 50);
            $table->text('descripcion');
            $table->timestamps();
        });

        // Agregar columna vector (1536 dims) usando SQL nativo de pgvector
        DB::statement("
            ALTER TABLE programas_derivados_objetivos
            ADD COLUMN embedding vector(1536);
        ");

        // Índice HNSW para búsqueda aproximada por similitud coseno
        DB::statement("
            CREATE INDEX idx_pdo_embedding_hnsw
            ON programas_derivados_objetivos
            USING hnsw (embedding vector_cosine_ops);
        ");
    }

    public function down(): void
    {
        Schema::dropIfExists('programas_derivados_objetivos');
    }
};
```

---

### Paso 3: Crear el modelo `ProgramaDerivado`

```bash
sail artisan make:model ProgramaDerivado
```

Editar `app/Models/ProgramaDerivado.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class ProgramaDerivado extends Model
{
    protected $table = 'programas_derivados';

    protected $fillable = [
        'ped_plan_id',
        'tipo',
        'nombre',
        'descripcion',
    ];

    /**
     * Valores válidos para el campo tipo.
     * Deben coincidir con el ENUM tipo_programa_derivado en PostgreSQL.
     */
    public const TIPOS = [
        'sectorial',
        'especial',
        'institucional',
        'regional',
    ];

    /**
     * El programa deriva de un Plan Estatal de Desarrollo.
     */
    public function pedPlan(): BelongsTo
    {
        return $this->belongsTo(PedPlan::class, 'ped_plan_id');
    }

    /**
     * Un programa tiene múltiples objetivos.
     */
    public function objetivos(): HasMany
    {
        return $this->hasMany(ProgramaDerivadoObjetivo::class, 'programa_derivado_id');
    }
}
```

---

### Paso 4: Crear el modelo `ProgramaDerivadoObjetivo`

```bash
sail artisan make:model ProgramaDerivadoObjetivo
```

Editar `app/Models/ProgramaDerivadoObjetivo.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class ProgramaDerivadoObjetivo extends Model
{
    protected $table = 'programas_derivados_objetivos';

    protected $fillable = [
        'programa_derivado_id',
        'clave',
        'descripcion',
        'embedding',
    ];

    /**
     * Casting personalizado para el campo vector.
     * pgvector devuelve el vector como string "[0.1,0.2,...]".
     * Se convierte a array PHP para facilitar el uso.
     */
    protected $casts = [
        // No usar cast nativo de Laravel para vector; usar accessor/mutator
    ];

    /**
     * Obtener el embedding como array PHP.
     */
    public function getEmbeddingAttribute(?string $value): ?array
    {
        if ($value === null) {
            return null;
        }
        // pgvector devuelve "[0.1,0.2,...]" — limpiar corchetes y convertir
        $clean = trim($value, '[]');
        return array_map('floatval', explode(',', $clean));
    }

    /**
     * Guardar embedding desde array PHP como string de pgvector.
     */
    public function setEmbeddingAttribute(?array $value): void
    {
        if ($value === null) {
            $this->attributes['embedding'] = null;
            return;
        }
        $this->attributes['embedding'] = '[' . implode(',', $value) . ']';
    }

    /**
     * El objetivo pertenece a un Programa Derivado.
     */
    public function programaDerivado(): BelongsTo
    {
        return $this->belongsTo(ProgramaDerivado::class, 'programa_derivado_id');
    }

    /**
     * Búsqueda semántica por similitud coseno usando pgvector.
     * Uso: ProgramaDerivadoObjetivo::nearestTo($embedding, 5)->get()
     *
     * @param  \Illuminate\Database\Eloquent\Builder  $query
     * @param  array  $embedding  Vector de 1536 dimensiones
     * @param  int    $limit
     */
    public function scopeNearestTo($query, array $embedding, int $limit = 10)
    {
        $vector = '[' . implode(',', $embedding) . ']';
        return $query
            ->orderByRaw("embedding <=> ?::vector", [$vector])
            ->limit($limit);
    }
}
```

---

### Paso 5: Agregar la relación inversa en `PedPlan`

Abrir `app/Models/PedPlan.php` y agregar:

```php
use Illuminate\Database\Eloquent\Relations\HasMany;

// Dentro de la clase PedPlan:

/**
 * Un PED puede originar múltiples programas derivados.
 */
public function programasDerivados(): HasMany
{
    return $this->hasMany(ProgramaDerivado::class, 'ped_plan_id');
}
```

---

### Paso 6: Crear el Seeder

```bash
sail artisan make:seeder ProgramasDerivadosSeeder
```

Editar `database/seeders/ProgramasDerivadosSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\PedPlan;
use App\Models\ProgramaDerivado;
use App\Models\ProgramaDerivadoObjetivo;

class ProgramasDerivadosSeeder extends Seeder
{
    public function run(): void
    {
        // Obtener el primer PED plan disponible
        $pedPlan = PedPlan::firstOrFail();

        $programas = [
            [
                'tipo'        => 'sectorial',
                'nombre'      => 'Programa Sectorial de Educación 2022-2028',
                'descripcion' => 'Instrumento rector de la política educativa del estado.',
                'objetivo'    => [
                    'clave'      => 'PSE-OBJ-01',
                    'descripcion' => 'Mejorar la cobertura y calidad educativa en todos los niveles.',
                ],
            ],
            [
                'tipo'        => 'especial',
                'nombre'      => 'Programa Especial de Combate a la Pobreza 2022-2028',
                'descripcion' => 'Programa transversal para reducir la pobreza extrema.',
                'objetivo'    => [
                    'clave'      => 'PECP-OBJ-01',
                    'descripcion' => 'Reducir en 20% la pobreza extrema mediante transferencias focalizadas.',
                ],
            ],
            [
                'tipo'        => 'institucional',
                'nombre'      => 'Programa Institucional de Salud 2022-2028',
                'descripcion' => 'Programa de la Secretaría de Salud estatal.',
                'objetivo'    => [
                    'clave'      => 'PIS-OBJ-01',
                    'descripcion' => 'Incrementar el acceso a servicios de salud preventivos en zonas rurales.',
                ],
            ],
            [
                'tipo'        => 'regional',
                'nombre'      => 'Programa Regional de Desarrollo del Norte 2022-2028',
                'descripcion' => 'Programa orientado al desarrollo de los municipios del norte del estado.',
                'objetivo'    => [
                    'clave'      => 'PRDN-OBJ-01',
                    'descripcion' => 'Impulsar la infraestructura productiva en los 12 municipios del norte.',
                ],
            ],
        ];

        foreach ($programas as $data) {
            $programa = ProgramaDerivado::create([
                'ped_plan_id' => $pedPlan->id,
                'tipo'        => $data['tipo'],
                'nombre'      => $data['nombre'],
                'descripcion' => $data['descripcion'],
            ]);

            ProgramaDerivadoObjetivo::create([
                'programa_derivado_id' => $programa->id,
                'clave'                => $data['objetivo']['clave'],
                'descripcion'          => $data['objetivo']['descripcion'],
                // embedding se deja null hasta la integración con OpenAI en sprints posteriores
            ]);
        }

        $this->command->info('ProgramasDerivadosSeeder: 4 programas (uno por tipo) y 4 objetivos creados.');
    }
}
```

Registrar el seeder en `database/seeders/DatabaseSeeder.php`:

```php
// Al final del método run(), después de los seeders de S2-T1 al T3:
$this->call(ProgramasDerivadosSeeder::class);
```

---

### Paso 7: Ejecutar migraciones y seeder

```bash
# Solo migraciones nuevas (ambiente de desarrollo con datos existentes)
sail artisan migrate

# O fresh con todos los seeders (ambiente limpio)
sail artisan migrate:fresh --seed
```

---

### Paso 8: Verificar en PostgreSQL

```bash
# Verificar estructura de tablas
sail psql -c "\d programas_derivados"
sail psql -c "\d programas_derivados_objetivos"

# Verificar que el ENUM fue creado
sail psql -c "\dT tipo_programa_derivado"

# Verificar datos del seeder
sail psql -c "SELECT tipo, nombre FROM programas_derivados;"
sail psql -c "SELECT clave, descripcion FROM programas_derivados_objetivos;"

# Verificar columna vector
sail psql -c "SELECT column_name, data_type, udt_name FROM information_schema.columns WHERE table_name = 'programas_derivados_objetivos' AND column_name = 'embedding';"
```

---

### Paso 9: Verificar relaciones con Tinker

```bash
sail artisan tinker
```

```php
// Cargar un programa con su plan y sus objetivos
$p = App\Models\ProgramaDerivado::with(['pedPlan', 'objetivos'])->first();
$p->tipo;          // 'sectorial'
$p->pedPlan->nombre; // nombre del plan PED
$p->objetivos->count(); // 1

// Navegar en sentido inverso
$obj = App\Models\ProgramaDerivadoObjetivo::with('programaDerivado.pedPlan')->first();
$obj->programaDerivado->tipo;      // 'sectorial'
$obj->programaDerivado->pedPlan->nombre; // nombre del plan

// Verificar que el ENUM rechaza valores inválidos (esto lanzará excepción de DB)
// App\Models\ProgramaDerivado::create(['ped_plan_id' => 1, 'tipo' => 'invalido', 'nombre' => 'X']);
```

---

## Criterios de Aceptacion

| # | Criterio | Verificacion |
|---|----------|--------------|
| 1 | Migración `programas_derivados` con ENUM nativo PostgreSQL (4 valores) | `sail psql -c "\dT tipo_programa_derivado"` muestra los 4 valores |
| 2 | Migración `programas_derivados_objetivos` con columna `vector(1536)` | `\d programas_derivados_objetivos` muestra tipo `vector` |
| 3 | `down()` limpio en ambas migraciones | `sail artisan migrate:rollback` sin errores, luego `migrate` nuevamente |
| 4 | Relación `ProgramaDerivado belongsTo PedPlan` funcional | Tinker: `$p->pedPlan` retorna instancia |
| 5 | Relación `ProgramaDerivado hasMany ProgramaDerivadoObjetivo` funcional | Tinker: `$p->objetivos` retorna colección |
| 6 | Seeder crea 1 programa de cada tipo (4 total) | `SELECT tipo, nombre FROM programas_derivados;` retorna 4 filas |
| 7 | `sail artisan migrate:fresh --seed` sin errores | Sin excepciones en consola |
| 8 | Accessor `getEmbeddingAttribute` devuelve array o null | Tinker: `$obj->embedding` es `null` (sin datos aún) |

---

## Notas

- El campo `embedding` se deja `null` en el seeder; se poblará en el sprint de integración con OpenAI.
- El índice HNSW (`hnsw`) es el recomendado por pgvector para búsqueda aproximada de alta performance. El índice IVFFlat requiere datos pre-existentes para construirse correctamente y es menos adecuado en esta etapa.
- Si en el futuro se necesita agregar o modificar valores del ENUM, usar `ALTER TYPE tipo_programa_derivado ADD VALUE 'nuevo_valor';` (PostgreSQL no permite eliminar valores de un ENUM sin recrearlo).
- El accessor/mutator para `embedding` en el modelo es suficiente para el uso básico. Para producción con búsqueda semántica, considerar el paquete `pgvector/pgvector` de PHP o manejar el casting directamente con `DB::raw`.
- La regla de validación recomendada en FormRequests: `'tipo' => ['required', Rule::in(ProgramaDerivado::TIPOS)]`.
