# Plan: S2-T3 — Migraciones y modelos para PED

**Ticket:** S2-T3
**Tipo:** feat
**Rama:** `feat/S2-T3-modelos-ped`
**Sprint:** 2 — Catálogos Normativos
**Depende de:** S0-T2 (pgvector habilitado)

---

## Contexto

El Plan Estatal de Desarrollo (PED) es el instrumento rector de la política pública a nivel estatal. A diferencia de los catálogos ODS y PND (que son normativos y estáticos), el PED es propio de cada gobierno estatal, cambia cada sexenio y puede tener múltiples versiones (estatal, municipal).

La estructura jerárquica del PED es de seis niveles:

```
PedPlan
  └── PedEje
        └── PedTema
              └── PedObjetivoEstrategico
                    └── PedEstrategia
                          └── PedLineaAccion
```

Reglas de negocio clave:
- Solo un `ped_planes` puede estar activo a la vez (`activo = true`). Se implementa mediante un índice parcial único en PostgreSQL.
- Las FK en cascada permiten eliminar un plan completo sin dejar huérfanos.
- Los embeddings siguen el mismo patrón que S2-T1/T2: columna `vector(1536)` agregada con DDL raw, `NULL` hasta Sprint 5.

---

## Pre-requisitos

- S0-T2 completado (extensión `vector` activa en PostgreSQL)
- `sail up -d` ejecutado

---

## Pasos

### 1. Crear las migraciones de tablas base

```bash
sail artisan make:migration create_ped_planes_table
sail artisan make:migration create_ped_ejes_table
sail artisan make:migration create_ped_temas_table
sail artisan make:migration create_ped_objetivos_estrategicos_table
sail artisan make:migration create_ped_estrategias_table
sail artisan make:migration create_ped_lineas_accion_table
```

Crear también las migraciones de embeddings y el índice parcial:

```bash
sail artisan make:migration add_embeddings_to_ped_tables
sail artisan make:migration add_unique_activo_index_to_ped_planes_table
```

---

### 2. Migración: `create_ped_planes_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ped_planes', function (Blueprint $table) {
            $table->id();
            $table->string('nombre', 300);
            $table->enum('nivel_gobierno', ['estatal', 'municipal']);
            $table->unsignedSmallInteger('periodo_inicio');  // año, e.g. 2022
            $table->unsignedSmallInteger('periodo_fin');     // año, e.g. 2028
            $table->boolean('activo')->default(false);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ped_planes');
    }
};
```

---

### 3. Migración: `create_ped_ejes_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ped_ejes', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_plan_id')
                  ->constrained('ped_planes')
                  ->onDelete('cascade');
            $table->unsignedSmallInteger('numero');
            $table->string('nombre', 200);
            $table->text('descripcion')->nullable();
            $table->timestamps();

            $table->unique(['ped_plan_id', 'numero']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ped_ejes');
    }
};
```

---

### 4. Migración: `create_ped_temas_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ped_temas', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_eje_id')
                  ->constrained('ped_ejes')
                  ->onDelete('cascade');
            $table->unsignedSmallInteger('numero');
            $table->string('nombre', 200);
            $table->text('descripcion')->nullable();
            $table->timestamps();

            $table->unique(['ped_eje_id', 'numero']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ped_temas');
    }
};
```

---

### 5. Migración: `create_ped_objetivos_estrategicos_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ped_objetivos_estrategicos', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_tema_id')
                  ->constrained('ped_temas')
                  ->onDelete('cascade');
            $table->string('clave', 20);   // e.g. "OE1.1", "OE2.3"
            $table->text('descripcion');
            $table->timestamps();

            $table->unique(['ped_tema_id', 'clave']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ped_objetivos_estrategicos');
    }
};
```

---

### 6. Migración: `create_ped_estrategias_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ped_estrategias', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_objetivo_estrategico_id')
                  ->constrained('ped_objetivos_estrategicos')
                  ->onDelete('cascade');
            $table->string('clave', 20);   // e.g. "E1.1.1", "E2.3.2"
            $table->text('descripcion');
            $table->timestamps();

            $table->unique(['ped_objetivo_estrategico_id', 'clave']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ped_estrategias');
    }
};
```

---

### 7. Migración: `create_ped_lineas_accion_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ped_lineas_accion', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_estrategia_id')
                  ->constrained('ped_estrategias')
                  ->onDelete('cascade');
            $table->string('clave', 20);   // e.g. "LA1.1.1.a", "LA2.3.2.b"
            $table->text('descripcion');
            $table->timestamps();

            $table->unique(['ped_estrategia_id', 'clave']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ped_lineas_accion');
    }
};
```

---

### 8. Migración: `add_embeddings_to_ped_tables`

Se agrupan los seis `ALTER TABLE` en una sola migración para mantener el número de archivos manejable.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // ped_ejes
        DB::statement('ALTER TABLE ped_ejes ADD COLUMN embedding vector(1536)');

        // ped_temas
        DB::statement('ALTER TABLE ped_temas ADD COLUMN embedding vector(1536)');

        // ped_objetivos_estrategicos
        DB::statement('ALTER TABLE ped_objetivos_estrategicos ADD COLUMN embedding vector(1536)');

        // ped_estrategias
        DB::statement('ALTER TABLE ped_estrategias ADD COLUMN embedding vector(1536)');

        // ped_lineas_accion
        DB::statement('ALTER TABLE ped_lineas_accion ADD COLUMN embedding vector(1536)');

        // Nota: ped_planes no tiene embedding propio; su descripción
        // se representa por los embeddings de sus ejes.
    }

    public function down(): void
    {
        $tables = [
            'ped_ejes',
            'ped_temas',
            'ped_objetivos_estrategicos',
            'ped_estrategias',
            'ped_lineas_accion',
        ];

        foreach ($tables as $table) {
            if (Schema::hasColumn($table, 'embedding')) {
                DB::statement("ALTER TABLE {$table} DROP COLUMN embedding");
            }
        }
    }
};
```

---

### 9. Migración: `add_unique_activo_index_to_ped_planes_table`

El constraint "solo un plan activo" se implementa con un **índice parcial único** en PostgreSQL: solo aplica el constraint a las filas donde `activo = true`. Esto permite que múltiples planes tengan `activo = false` (histórico) pero solo uno tenga `activo = true`.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // Índice parcial único: solo puede haber un registro con activo = true.
        // No afecta los registros con activo = false (histórico).
        DB::statement(
            "CREATE UNIQUE INDEX ped_planes_unique_activo
             ON ped_planes (activo)
             WHERE activo = true"
        );
    }

    public function down(): void
    {
        DB::statement('DROP INDEX IF EXISTS ped_planes_unique_activo');
    }
};
```

> **Cómo funciona:** PostgreSQL solo aplica el constraint de unicidad a las filas que cumplen la condición `WHERE activo = true`. Como solo puede haber un valor `true` en un índice único, esto garantiza que máximo un plan esté activo. Los planes con `activo = false` no participan en el índice y pueden ser muchos.

---

### 10. Crear los modelos

```bash
sail artisan make:model PedPlan
sail artisan make:model PedEje
sail artisan make:model PedTema
sail artisan make:model PedObjetivoEstrategico
sail artisan make:model PedEstrategia
sail artisan make:model PedLineaAccion
```

---

### 11. Modelo: `app/Models/PedPlan.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasManyThrough;

class PedPlan extends Model
{
    protected $table = 'ped_planes';

    protected $fillable = [
        'nombre',
        'nivel_gobierno',
        'periodo_inicio',
        'periodo_fin',
        'activo',
    ];

    protected $casts = [
        'periodo_inicio' => 'integer',
        'periodo_fin'    => 'integer',
        'activo'         => 'boolean',
    ];

    // ── Relaciones ────────────────────────────────────────────

    public function ejes(): HasMany
    {
        return $this->hasMany(PedEje::class, 'ped_plan_id');
    }

    // ── Scopes ────────────────────────────────────────────────

    public function scopeActivo(Builder $query): Builder
    {
        return $query->where('activo', true);
    }

    // ── Métodos de negocio ────────────────────────────────────

    /**
     * Activa este plan y desactiva todos los demás del mismo nivel de gobierno.
     * Llama a este método en lugar de asignar activo = true directamente,
     * para mantener la integridad del constraint de un solo plan activo.
     */
    public function activar(): void
    {
        // Desactivar planes activos del mismo nivel
        static::where('nivel_gobierno', $this->nivel_gobierno)
              ->where('activo', true)
              ->where('id', '!=', $this->id)
              ->update(['activo' => false]);

        $this->update(['activo' => true]);
    }

    /**
     * Retorna el plan activo del nivel de gobierno indicado, o null si no hay.
     */
    public static function planActivo(string $nivelGobierno = 'estatal'): ?static
    {
        return static::where('nivel_gobierno', $nivelGobierno)
                     ->where('activo', true)
                     ->first();
    }
}
```

---

### 12. Modelo: `app/Models/PedEje.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PedEje extends Model
{
    protected $table = 'ped_ejes';

    protected $fillable = [
        'ped_plan_id',
        'numero',
        'nombre',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'numero'    => 'integer',
        'embedding' => 'array',
    ];

    public function plan(): BelongsTo
    {
        return $this->belongsTo(PedPlan::class, 'ped_plan_id');
    }

    public function temas(): HasMany
    {
        return $this->hasMany(PedTema::class, 'ped_eje_id');
    }
}
```

---

### 13. Modelo: `app/Models/PedTema.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PedTema extends Model
{
    protected $table = 'ped_temas';

    protected $fillable = [
        'ped_eje_id',
        'numero',
        'nombre',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'numero'    => 'integer',
        'embedding' => 'array',
    ];

    public function eje(): BelongsTo
    {
        return $this->belongsTo(PedEje::class, 'ped_eje_id');
    }

    public function objetivosEstrategicos(): HasMany
    {
        return $this->hasMany(PedObjetivoEstrategico::class, 'ped_tema_id');
    }
}
```

---

### 14. Modelo: `app/Models/PedObjetivoEstrategico.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PedObjetivoEstrategico extends Model
{
    protected $table = 'ped_objetivos_estrategicos';

    protected $fillable = [
        'ped_tema_id',
        'clave',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'embedding' => 'array',
    ];

    public function tema(): BelongsTo
    {
        return $this->belongsTo(PedTema::class, 'ped_tema_id');
    }

    public function estrategias(): HasMany
    {
        return $this->hasMany(PedEstrategia::class, 'ped_objetivo_estrategico_id');
    }
}
```

---

### 15. Modelo: `app/Models/PedEstrategia.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PedEstrategia extends Model
{
    protected $table = 'ped_estrategias';

    protected $fillable = [
        'ped_objetivo_estrategico_id',
        'clave',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'embedding' => 'array',
    ];

    public function objetivoEstrategico(): BelongsTo
    {
        return $this->belongsTo(PedObjetivoEstrategico::class, 'ped_objetivo_estrategico_id');
    }

    public function lineasAccion(): HasMany
    {
        return $this->hasMany(PedLineaAccion::class, 'ped_estrategia_id');
    }
}
```

---

### 16. Modelo: `app/Models/PedLineaAccion.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class PedLineaAccion extends Model
{
    protected $table = 'ped_lineas_accion';

    protected $fillable = [
        'ped_estrategia_id',
        'clave',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'embedding' => 'array',
    ];

    public function estrategia(): BelongsTo
    {
        return $this->belongsTo(PedEstrategia::class, 'ped_estrategia_id');
    }
}
```

---

### 17. Crear el seeder

```bash
sail artisan make:seeder PedFicticioSeeder
```

Editar `database/seeders/PedFicticioSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\PedEje;
use App\Models\PedEstrategia;
use App\Models\PedLineaAccion;
use App\Models\PedObjetivoEstrategico;
use App\Models\PedPlan;
use App\Models\PedTema;
use Illuminate\Database\Seeder;

class PedFicticioSeeder extends Seeder
{
    public function run(): void
    {
        // Plan activo del ejercicio de gobierno actual (datos ficticios para desarrollo)
        $plan = PedPlan::create([
            'nombre'          => 'Plan Estatal de Desarrollo 2022-2028',
            'nivel_gobierno'  => 'estatal',
            'periodo_inicio'  => 2022,
            'periodo_fin'     => 2028,
            'activo'          => true,
        ]);

        // ── EJE 1: Bienestar Social ────────────────────────────────────────
        $eje1 = PedEje::create([
            'ped_plan_id' => $plan->id,
            'numero'      => 1,
            'nombre'      => 'Bienestar Social',
            'descripcion' => 'Garantizar el bienestar integral de la población mediante el fortalecimiento de los servicios de salud, educación, vivienda y desarrollo social.',
        ]);

        // Eje 1 > Tema 1.1
        $tema1_1 = PedTema::create([
            'ped_eje_id'  => $eje1->id,
            'numero'      => 1,
            'nombre'      => 'Salud para Todos',
            'descripcion' => 'Ampliar la cobertura y calidad de los servicios de salud en zonas urbanas y rurales.',
        ]);

        $oe1_1_1 = PedObjetivoEstrategico::create([
            'ped_tema_id' => $tema1_1->id,
            'clave'       => 'OE1.1.1',
            'descripcion' => 'Fortalecer la infraestructura y el equipamiento de las unidades de salud del primer nivel de atención.',
        ]);

        $e1_1_1_1 = PedEstrategia::create([
            'ped_objetivo_estrategico_id' => $oe1_1_1->id,
            'clave'                       => 'E1.1.1.1',
            'descripcion'                 => 'Construir y rehabilitar unidades médicas rurales en los municipios con mayor rezago sanitario.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e1_1_1_1->id,
            'clave'             => 'LA1.1.1.1.a',
            'descripcion'       => 'Identificar y priorizar los municipios con déficit de infraestructura de salud mediante diagnóstico georreferenciado.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e1_1_1_1->id,
            'clave'             => 'LA1.1.1.1.b',
            'descripcion'       => 'Gestionar recursos federales y estatales para la construcción de 30 nuevas unidades médicas en el periodo.',
        ]);

        // Eje 1 > Tema 1.2
        $tema1_2 = PedTema::create([
            'ped_eje_id'  => $eje1->id,
            'numero'      => 2,
            'nombre'      => 'Educación de Calidad',
            'descripcion' => 'Mejorar los indicadores de cobertura, eficiencia terminal y calidad educativa en todos los niveles.',
        ]);

        $oe1_2_1 = PedObjetivoEstrategico::create([
            'ped_tema_id' => $tema1_2->id,
            'clave'       => 'OE1.2.1',
            'descripcion' => 'Reducir la deserción escolar en educación básica y media superior.',
        ]);

        $e1_2_1_1 = PedEstrategia::create([
            'ped_objetivo_estrategico_id' => $oe1_2_1->id,
            'clave'                       => 'E1.2.1.1',
            'descripcion'                 => 'Implementar programas de becas y apoyos socioeconómicos para estudiantes en situación de vulnerabilidad.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e1_2_1_1->id,
            'clave'             => 'LA1.2.1.1.a',
            'descripcion'       => 'Otorgar becas mensuales a estudiantes de familias con ingreso inferior a la línea de bienestar.',
        ]);

        // ── EJE 2: Desarrollo Económico Sustentable ────────────────────────
        $eje2 = PedEje::create([
            'ped_plan_id' => $plan->id,
            'numero'      => 2,
            'nombre'      => 'Desarrollo Económico Sustentable',
            'descripcion' => 'Impulsar el crecimiento económico incluyente, la generación de empleo de calidad y el aprovechamiento sostenible de los recursos naturales del estado.',
        ]);

        // Eje 2 > Tema 2.1
        $tema2_1 = PedTema::create([
            'ped_eje_id'  => $eje2->id,
            'numero'      => 1,
            'nombre'      => 'Fomento Productivo y Empleo',
            'descripcion' => 'Fortalecer las cadenas productivas locales y promover la creación de empleos formales y bien remunerados.',
        ]);

        $oe2_1_1 = PedObjetivoEstrategico::create([
            'ped_tema_id' => $tema2_1->id,
            'clave'       => 'OE2.1.1',
            'descripcion' => 'Incrementar la inversión privada nacional y extranjera en sectores estratégicos del estado.',
        ]);

        $e2_1_1_1 = PedEstrategia::create([
            'ped_objetivo_estrategico_id' => $oe2_1_1->id,
            'clave'                       => 'E2.1.1.1',
            'descripcion'                 => 'Simplificar los trámites de apertura de empresas y mejorar el clima de negocios estatal.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e2_1_1_1->id,
            'clave'             => 'LA2.1.1.1.a',
            'descripcion'       => 'Implementar la ventanilla única de trámites empresariales digital con resolución máxima en 72 horas.',
        ]);

        // Eje 2 > Tema 2.2
        $tema2_2 = PedTema::create([
            'ped_eje_id'  => $eje2->id,
            'numero'      => 2,
            'nombre'      => 'Turismo y Economía Rural',
            'descripcion' => 'Aprovechar el potencial turístico y agropecuario del estado para diversificar la economía y beneficiar a las comunidades locales.',
        ]);

        $oe2_2_1 = PedObjetivoEstrategico::create([
            'ped_tema_id' => $tema2_2->id,
            'clave'       => 'OE2.2.1',
            'descripcion' => 'Desarrollar productos turísticos sustentables que integren la cultura y naturaleza locales.',
        ]);

        $e2_2_1_1 = PedEstrategia::create([
            'ped_objetivo_estrategico_id' => $oe2_2_1->id,
            'clave'                       => 'E2.2.1.1',
            'descripcion'                 => 'Fortalecer los circuitos de turismo de naturaleza y cultural en las regiones con mayor potencial.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e2_2_1_1->id,
            'clave'             => 'LA2.2.1.1.a',
            'descripcion'       => 'Certificar guías de turismo de naturaleza y fortalecer la oferta de servicios turísticos comunitarios.',
        ]);

        // ── EJE 3: Gobernanza y Seguridad ─────────────────────────────────
        $eje3 = PedEje::create([
            'ped_plan_id' => $plan->id,
            'numero'      => 3,
            'nombre'      => 'Gobernanza y Seguridad',
            'descripcion' => 'Fortalecer la institucionalidad democrática, la transparencia gubernamental y la seguridad pública con enfoque de derechos humanos.',
        ]);

        // Eje 3 > Tema 3.1
        $tema3_1 = PedTema::create([
            'ped_eje_id'  => $eje3->id,
            'numero'      => 1,
            'nombre'      => 'Seguridad Ciudadana',
            'descripcion' => 'Reducir los índices de violencia y delincuencia mediante estrategias integrales de prevención, atención y sanción.',
        ]);

        $oe3_1_1 = PedObjetivoEstrategico::create([
            'ped_tema_id' => $tema3_1->id,
            'clave'       => 'OE3.1.1',
            'descripcion' => 'Fortalecer las capacidades operativas y de inteligencia de las corporaciones de seguridad pública estatal.',
        ]);

        $e3_1_1_1 = PedEstrategia::create([
            'ped_objetivo_estrategico_id' => $oe3_1_1->id,
            'clave'                       => 'E3.1.1.1',
            'descripcion'                 => 'Profesionalizar al personal de seguridad pública mediante programas de formación continua.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e3_1_1_1->id,
            'clave'             => 'LA3.1.1.1.a',
            'descripcion'       => 'Implementar un programa de certificación policial con estándares nacionales para el 100% del personal activo.',
        ]);

        // Eje 3 > Tema 3.2
        $tema3_2 = PedTema::create([
            'ped_eje_id'  => $eje3->id,
            'numero'      => 2,
            'nombre'      => 'Transparencia y Rendición de Cuentas',
            'descripcion' => 'Garantizar el acceso a la información pública y la rendición de cuentas de todos los entes gubernamentales.',
        ]);

        $oe3_2_1 = PedObjetivoEstrategico::create([
            'ped_tema_id' => $tema3_2->id,
            'clave'       => 'OE3.2.1',
            'descripcion' => 'Fortalecer los mecanismos de acceso a la información y participación ciudadana en la gestión pública.',
        ]);

        $e3_2_1_1 = PedEstrategia::create([
            'ped_objetivo_estrategico_id' => $oe3_2_1->id,
            'clave'                       => 'E3.2.1.1',
            'descripcion'                 => 'Modernizar los sistemas de transparencia proactiva y gobierno abierto del estado.',
        ]);

        PedLineaAccion::create([
            'ped_estrategia_id' => $e3_2_1_1->id,
            'clave'             => 'LA3.2.1.1.a',
            'descripcion'       => 'Publicar en formato de datos abiertos la totalidad del gasto público estatal en el portal de transparencia.',
        ]);

        $this->command->info(sprintf(
            'PED cargado: 1 plan, %d ejes, %d temas, %d OE, %d estrategias, %d líneas de acción.',
            PedEje::count(),
            PedTema::count(),
            PedObjetivoEstrategico::count(),
            PedEstrategia::count(),
            PedLineaAccion::count()
        ));
    }
}
```

---

### 18. Registrar el seeder en DatabaseSeeder

```php
public function run(): void
{
    $this->call([
        // ... seeders existentes ...
        OdsCatalogoSeeder::class,
        PndCatalogoSeeder::class,
        PedFicticioSeeder::class,
    ]);
}
```

---

### 19. Ejecutar y verificar

```bash
sail artisan migrate:fresh --seed
```

Verificar en Tinker:

```bash
sail artisan tinker
```

```php
use App\Models\PedPlan;
use App\Models\PedEje;
use App\Models\PedTema;
use App\Models\PedObjetivoEstrategico;
use App\Models\PedEstrategia;
use App\Models\PedLineaAccion;

// Verificar conteos
PedPlan::count();                    // 1
PedEje::count();                     // 3
PedTema::count();                    // 6  (2 por eje)
PedObjetivoEstrategico::count();     // >= 6
PedEstrategia::count();              // >= 6
PedLineaAccion::count();             // >= 7

// Verificar plan activo
$plan = PedPlan::planActivo('estatal');
$plan->nombre;                       // "Plan Estatal de Desarrollo 2022-2028"
$plan->activo;                       // true

// Verificar jerarquía completa (eager loading)
$eje = PedEje::with([
    'temas.objetivosEstrategicos.estrategias.lineasAccion'
])->where('numero', 1)->first();

$eje->nombre;                                                   // "Bienestar Social"
$eje->temas->count();                                           // 2
$eje->temas->first()->nombre;                                   // "Salud para Todos"
$eje->temas->first()->objetivosEstrategicos->first()->clave;    // "OE1.1.1"

// Verificar constraint de un solo plan activo
// Intentar activar un segundo plan lanzaría una excepción de DB por el índice único parcial.
// El método activar() maneja esto correctamente:
// $otroPlan->activar(); // Desactiva el primero y activa el nuevo.

// Verificar que embeddings son NULL
PedEje::whereNotNull('embedding')->count();  // 0

exit
```

---

### 20. Verificar el índice parcial en PostgreSQL

```bash
sail shell
psql -U sail -d laravel -c "\d ped_planes"
exit
```

Debe aparecer en la sección de índices:

```
Indexes:
    "ped_planes_pkey" PRIMARY KEY, btree (id)
    "ped_planes_unique_activo" UNIQUE, btree (activo) WHERE activo = true
```

---

## Criterios de aceptación

- [ ] 6 migraciones de tablas base con `up()` y `down()` correctos
- [ ] 1 migración agrega `embedding vector(1536)` a las 5 tablas con embeddings (excluye `ped_planes`)
- [ ] 1 migración crea el índice parcial único `ped_planes_unique_activo WHERE activo = true`
- [ ] FK en cascada en toda la cadena: `ped_planes` → `ped_ejes` → `ped_temas` → `ped_objetivos_estrategicos` → `ped_estrategias` → `ped_lineas_accion`
- [ ] Modelo `PedPlan` con método `activar()` y scope `activo()` y método estático `planActivo()`
- [ ] Modelos `PedEje`, `PedTema`, `PedObjetivoEstrategico`, `PedEstrategia`, `PedLineaAccion` con `$fillable`, cast `embedding => array` y relaciones `hasMany`/`belongsTo`
- [ ] `PedFicticioSeeder` carga 1 plan, 3 ejes, 2 temas por eje, con OE, estrategias y líneas de acción
- [ ] `sail artisan migrate:fresh --seed` ejecuta sin errores
- [ ] Solo un plan puede tener `activo = true` (verificado por el índice parcial en PostgreSQL)
- [ ] `PedPlan::planActivo('estatal')` retorna el plan activo

---

## Esquema resultante

```
ped_planes
├── id                   BIGSERIAL PK
├── nombre               VARCHAR(300)
├── nivel_gobierno       ENUM('estatal','municipal')
├── periodo_inicio       SMALLINT
├── periodo_fin          SMALLINT
├── activo               BOOLEAN DEFAULT false
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

INDEX: ped_planes_unique_activo UNIQUE (activo) WHERE activo = true

ped_ejes
├── id                   BIGSERIAL PK
├── ped_plan_id          BIGINT FK → ped_planes.id CASCADE
├── numero               SMALLINT
├── nombre               VARCHAR(200)
├── descripcion          TEXT NULL
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(ped_plan_id, numero)

ped_temas
├── id                   BIGSERIAL PK
├── ped_eje_id           BIGINT FK → ped_ejes.id CASCADE
├── numero               SMALLINT
├── nombre               VARCHAR(200)
├── descripcion          TEXT NULL
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(ped_eje_id, numero)

ped_objetivos_estrategicos
├── id                   BIGSERIAL PK
├── ped_tema_id          BIGINT FK → ped_temas.id CASCADE
├── clave                VARCHAR(20)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(ped_tema_id, clave)

ped_estrategias
├── id                   BIGSERIAL PK
├── ped_objetivo_estrategico_id  BIGINT FK → ped_objetivos_estrategicos.id CASCADE
├── clave                VARCHAR(20)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(ped_objetivo_estrategico_id, clave)

ped_lineas_accion
├── id                   BIGSERIAL PK
├── ped_estrategia_id    BIGINT FK → ped_estrategias.id CASCADE
├── clave                VARCHAR(20)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(ped_estrategia_id, clave)
```

---

## Notas

- **Constraint de plan activo — índice parcial vs. validación en modelo:** Se elige el índice parcial de PostgreSQL (`WHERE activo = true`) porque es un constraint a nivel de base de datos, garantizando integridad incluso si se accede a la BD directamente o desde otro proceso. La validación en el modelo (`activar()`) es complementaria para dar una experiencia de usuario controlada. No se recomienda depender solo de la validación del modelo.

- **Alternativa al índice parcial:** Una solución alternativa es una columna `activo` de tipo `unsignedBigInteger nullable` con un unique nullable:
  ```sql
  -- Enfoque alternativo: solo almacena el ID del plan activo en una tabla de configuración
  -- No recomendado aquí porque complica las consultas.
  ```
  El índice parcial es más elegante y directamente expresivo de la regla de negocio.

- **`ped_planes` sin embedding:** El nivel de plan no tiene embedding propio porque su representación semántica se infiere de los ejes que lo componen. Esto reduce el cómputo de embeddings en Sprint 5 y simplifica el modelo.

- **Migraciones de embeddings agrupadas:** A diferencia de S2-T1 y S2-T2 donde cada tabla tiene su propia migración de embedding, aquí se agrupan en una sola para reducir el número de archivos de migración dado que son 5 tablas. Ambos enfoques son válidos.

- **Orden de ejecución de migraciones:** El timestamp de los archivos determina el orden. Es crítico que `ped_planes` se cree antes que `ped_ejes`, `ped_ejes` antes que `ped_temas`, etc. Al crear con `sail artisan make:migration` en el orden listado en el Paso 1, los timestamps quedan correctamente ordenados.

- **Importación del PED real:** El seeder usa datos ficticios. La importación del PED oficial se realizará en Sprint 5 mediante un comando Artisan que parsea el documento PDF/Word fuente. El seeder ficticio es suficiente para desarrollar la UI y los flujos de alineación durante Sprints 3 y 4.

- **Relación con programas presupuestarios:** La tabla de alineación entre `ped_lineas_accion` y `programas_presupuestarios` se crea en Sprint 3 (ticket S3-T3 tentativo). Este sprint solo establece el catálogo PED.
