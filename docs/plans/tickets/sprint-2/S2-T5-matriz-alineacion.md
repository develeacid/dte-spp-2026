# S2-T5 — Tablas pivote para Matriz de Alineacion

**Tipo:** feat
**Rama:** `feat/S2-T5-matriz-alineacion`
**Depende de:** S2-T1 (ODS), S2-T2 (PND), S2-T3 (PED), S2-T4 (Programas Derivados)

---

## Contexto

La Matriz de Alineacion es el corazon del sistema de planeacion: permite trazar la cadena de herencia normativa desde una Linea de Accion del PED hasta los Objetivos del Desarrollo Sostenible (ODS) de la ONU, pasando por el Plan Nacional de Desarrollo (PND).

La jerarquia de alineacion es:

```
ODS Meta
  ^
  | alineacion_pnd_ods
PND Objetivo
  ^
  | alineacion_ped_pnd
PED Objetivo Estrategico
  |
  v (via jerarquia PED interna)
PED Linea de Accion
  |
  v alineacion_linea_programa_derivado
Programa Derivado Objetivo
```

Este ticket crea las tres tablas pivote, agrega las relaciones `belongsToMany` en los modelos existentes, un seeder de ejemplo, y tests que verifican la integridad de la cadena.

---

## Pre-requisitos

- S2-T1 completado: tabla `ods_metas` y modelo `OdsMeta` existen.
- S2-T2 completado: tabla `pnd_objetivos` y modelo `PndObjetivo` existen.
- S2-T3 completado: tablas `ped_objetivos_estrategicos` y `ped_lineas_accion`, modelos `PedObjetivoEstrategico` y `PedLineaAccion` existen.
- S2-T4 completado: tabla `programas_derivados_objetivos` y modelo `ProgramaDerivadoObjetivo` existen.
- Sail corriendo: `sail up -d`

Verificar que las tablas base existen:

```bash
sail psql -c "\dt ods_metas pnd_objetivos ped_objetivos_estrategicos ped_lineas_accion programas_derivados_objetivos"
```

---

## Pasos

### Paso 1: Crear las 3 migraciones

```bash
sail artisan make:migration create_alineacion_ped_pnd_table
sail artisan make:migration create_alineacion_pnd_ods_table
sail artisan make:migration create_alineacion_linea_programa_derivado_table
```

---

### Paso 2: Implementar migración `alineacion_ped_pnd`

Editar `database/migrations/YYYY_MM_DD_HHMMSS_create_alineacion_ped_pnd_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('alineacion_ped_pnd', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_objetivo_estrategico_id')
                  ->constrained('ped_objetivos_estrategicos')
                  ->cascadeOnDelete();
            $table->foreignId('pnd_objetivo_id')
                  ->constrained('pnd_objetivos')
                  ->cascadeOnDelete();
            $table->timestamps();

            // Indice unico compuesto: no se puede duplicar la misma alineacion
            $table->unique(
                ['ped_objetivo_estrategico_id', 'pnd_objetivo_id'],
                'uq_alineacion_ped_pnd'
            );
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('alineacion_ped_pnd');
    }
};
```

---

### Paso 3: Implementar migración `alineacion_pnd_ods`

Editar `database/migrations/YYYY_MM_DD_HHMMSS_create_alineacion_pnd_ods_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('alineacion_pnd_ods', function (Blueprint $table) {
            $table->id();
            $table->foreignId('pnd_objetivo_id')
                  ->constrained('pnd_objetivos')
                  ->cascadeOnDelete();
            $table->foreignId('ods_meta_id')
                  ->constrained('ods_metas')
                  ->cascadeOnDelete();
            $table->timestamps();

            $table->unique(
                ['pnd_objetivo_id', 'ods_meta_id'],
                'uq_alineacion_pnd_ods'
            );
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('alineacion_pnd_ods');
    }
};
```

---

### Paso 4: Implementar migración `alineacion_linea_programa_derivado`

Editar `database/migrations/YYYY_MM_DD_HHMMSS_create_alineacion_linea_programa_derivado_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('alineacion_linea_programa_derivado', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ped_linea_accion_id')
                  ->constrained('ped_lineas_accion')
                  ->cascadeOnDelete();
            $table->foreignId('programa_derivado_objetivo_id')
                  ->constrained('programas_derivados_objetivos')
                  ->cascadeOnDelete();
            $table->timestamps();

            $table->unique(
                ['ped_linea_accion_id', 'programa_derivado_objetivo_id'],
                'uq_alineacion_linea_programa_derivado'
            );
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('alineacion_linea_programa_derivado');
    }
};
```

---

### Paso 5: Agregar relaciones `belongsToMany` en los modelos existentes

#### En `app/Models/PedObjetivoEstrategico.php`

Agregar el import y el metodo:

```php
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use App\Models\PndObjetivo;

// Dentro de la clase:

/**
 * Objetivos del PND con los que este objetivo estrategico del PED se alinea.
 */
public function pndObjetivos(): BelongsToMany
{
    return $this->belongsToMany(
        PndObjetivo::class,
        'alineacion_ped_pnd',
        'ped_objetivo_estrategico_id',
        'pnd_objetivo_id'
    )->withTimestamps();
}
```

#### En `app/Models/PndObjetivo.php`

```php
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use App\Models\OdsMeta;
use App\Models\PedObjetivoEstrategico;

// Dentro de la clase:

/**
 * Metas ODS con las que este objetivo del PND se alinea.
 */
public function odsMetas(): BelongsToMany
{
    return $this->belongsToMany(
        OdsMeta::class,
        'alineacion_pnd_ods',
        'pnd_objetivo_id',
        'ods_meta_id'
    )->withTimestamps();
}

/**
 * Objetivos estrategicos del PED que apuntan a este objetivo del PND.
 */
public function pedObjetivosEstrategicos(): BelongsToMany
{
    return $this->belongsToMany(
        PedObjetivoEstrategico::class,
        'alineacion_ped_pnd',
        'pnd_objetivo_id',
        'ped_objetivo_estrategico_id'
    )->withTimestamps();
}
```

#### En `app/Models/OdsMeta.php`

```php
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use App\Models\PndObjetivo;

// Dentro de la clase:

/**
 * Objetivos del PND que estan alineados a esta meta ODS.
 */
public function pndObjetivos(): BelongsToMany
{
    return $this->belongsToMany(
        PndObjetivo::class,
        'alineacion_pnd_ods',
        'ods_meta_id',
        'pnd_objetivo_id'
    )->withTimestamps();
}
```

#### En `app/Models/PedLineaAccion.php`

```php
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use App\Models\ProgramaDerivadoObjetivo;

// Dentro de la clase:

/**
 * Objetivos de programas derivados a los que contribuye esta linea de accion.
 */
public function programaDerivadoObjetivos(): BelongsToMany
{
    return $this->belongsToMany(
        ProgramaDerivadoObjetivo::class,
        'alineacion_linea_programa_derivado',
        'ped_linea_accion_id',
        'programa_derivado_objetivo_id'
    )->withTimestamps();
}
```

#### En `app/Models/ProgramaDerivadoObjetivo.php`

```php
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use App\Models\PedLineaAccion;

// Dentro de la clase:

/**
 * Lineas de accion del PED que contribuyen a este objetivo de programa derivado.
 */
public function pedLineasAccion(): BelongsToMany
{
    return $this->belongsToMany(
        PedLineaAccion::class,
        'alineacion_linea_programa_derivado',
        'programa_derivado_objetivo_id',
        'ped_linea_accion_id'
    )->withTimestamps();
}
```

---

### Paso 6: Crear el Seeder de alineaciones

```bash
sail artisan make:seeder AlineacionesSeeder
```

Editar `database/seeders/AlineacionesSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\PedObjetivoEstrategico;
use App\Models\PndObjetivo;
use App\Models\OdsMeta;
use App\Models\PedLineaAccion;
use App\Models\ProgramaDerivadoObjetivo;

class AlineacionesSeeder extends Seeder
{
    public function run(): void
    {
        // Obtener registros existentes (creados por seeders anteriores)
        $pedObj   = PedObjetivoEstrategico::firstOrFail();
        $pndObj   = PndObjetivo::firstOrFail();
        $odsMeta  = OdsMeta::firstOrFail();
        $lineaAccion = PedLineaAccion::firstOrFail();
        $programaObj = ProgramaDerivadoObjetivo::firstOrFail();

        // Alineacion PED -> PND
        $pedObj->pndObjetivos()->syncWithoutDetaching([$pndObj->id]);
        $this->command->info("Alineacion creada: PED Objetivo [{$pedObj->id}] -> PND Objetivo [{$pndObj->id}]");

        // Alineacion PND -> ODS
        $pndObj->odsMetas()->syncWithoutDetaching([$odsMeta->id]);
        $this->command->info("Alineacion creada: PND Objetivo [{$pndObj->id}] -> ODS Meta [{$odsMeta->id}]");

        // Alineacion Linea de Accion -> Programa Derivado Objetivo
        $lineaAccion->programaDerivadoObjetivos()->syncWithoutDetaching([$programaObj->id]);
        $this->command->info("Alineacion creada: Linea Accion [{$lineaAccion->id}] -> Programa Derivado Obj [{$programaObj->id}]");

        $this->command->info('AlineacionesSeeder: cadena de alineacion de ejemplo creada.');
    }
}
```

Registrar en `database/seeders/DatabaseSeeder.php`:

```php
// Despues de ProgramasDerivadosSeeder
$this->call(AlineacionesSeeder::class);
```

---

### Paso 7: Ejecutar migraciones y seeder

```bash
sail artisan migrate:fresh --seed
```

---

### Paso 8: Navegacion de la cadena completa de herencia

La cadena completa es:

```
PedLineaAccion -> ProgramaDerivadoObjetivo -> ProgramaDerivado -> PedPlan
PedLineaAccion -> PedEstrategia -> PedObjetivoEstrategico -> PndObjetivo -> OdsMeta
```

#### Opcion A: Eager Loading manual (recomendado para consultas puntuales)

```php
// En Tinker o en un controlador/componente Livewire:
$linea = PedLineaAccion::with([
    // Hacia Programas Derivados
    'programaDerivadoObjetivos.programaDerivado.pedPlan',
    // Hacia la cadena PED -> PND -> ODS
    // (asumiendo que PedLineaAccion belongsTo PedEstrategia belongsTo PedObjetivoEstrategico)
    'pedEstrategia.pedObjetivoEstrategico.pndObjetivos.odsMetas',
])->findOrFail($id);

// Navegar la cadena:
foreach ($linea->programaDerivadoObjetivos as $pdObj) {
    echo $pdObj->programaDerivado->nombre; // nombre del programa
}

foreach ($linea->pedEstrategia->pedObjetivoEstrategico->pndObjetivos as $pndObj) {
    foreach ($pndObj->odsMetas as $meta) {
        echo $meta->descripcion; // meta ODS
    }
}
```

#### Opcion B: Metodo en el modelo `PedLineaAccion` para obtener la cadena completa

```php
// En app/Models/PedLineaAccion.php — agregar este metodo:

/**
 * Obtiene la cadena de alineacion completa de esta linea de accion.
 * Retorna un array con todos los niveles relacionados.
 */
public function getCadenaAlineacion(): array
{
    $this->loadMissing([
        'programaDerivadoObjetivos.programaDerivado',
        'pedEstrategia.pedObjetivoEstrategico.pndObjetivos.odsMetas',
    ]);

    $pndObjetivos = $this->pedEstrategia?->pedObjetivoEstrategico?->pndObjetivos ?? collect();
    $odsMetas = $pndObjetivos->flatMap(fn($p) => $p->odsMetas);

    return [
        'linea_accion'             => $this,
        'estrategia'               => $this->pedEstrategia,
        'objetivo_estrategico'     => $this->pedEstrategia?->pedObjetivoEstrategico,
        'pnd_objetivos'            => $pndObjetivos,
        'ods_metas'                => $odsMetas,
        'programas_derivados_objs' => $this->programaDerivadoObjetivos,
    ];
}
```

#### Opcion C: Query directa SQL para reportes (maxima performance)

```php
// Consulta que recorre toda la cadena en una sola query SQL:
$resultados = \DB::select("
    SELECT
        pla.id          AS linea_accion_id,
        pla.nombre      AS linea_accion,
        pe.nombre       AS estrategia,
        poe.nombre      AS objetivo_estrategico,
        po.nombre       AS pnd_objetivo,
        om.descripcion  AS ods_meta,
        pdo.clave       AS programa_derivado_objetivo,
        pd.nombre       AS programa_derivado,
        pd.tipo         AS tipo_programa
    FROM ped_lineas_accion pla
    JOIN ped_estrategias pe
        ON pe.id = pla.ped_estrategia_id
    JOIN ped_objetivos_estrategicos poe
        ON poe.id = pe.ped_objetivo_estrategico_id
    JOIN alineacion_ped_pnd app2
        ON app2.ped_objetivo_estrategico_id = poe.id
    JOIN pnd_objetivos po
        ON po.id = app2.pnd_objetivo_id
    JOIN alineacion_pnd_ods apo
        ON apo.pnd_objetivo_id = po.id
    JOIN ods_metas om
        ON om.id = apo.ods_meta_id
    LEFT JOIN alineacion_linea_programa_derivado alpd
        ON alpd.ped_linea_accion_id = pla.id
    LEFT JOIN programas_derivados_objetivos pdo
        ON pdo.id = alpd.programa_derivado_objetivo_id
    LEFT JOIN programas_derivados pd
        ON pd.id = pdo.programa_derivado_id
    ORDER BY pla.id, po.id, om.id
");
```

---

### Paso 9: Crear los Tests

```bash
sail artisan make:test AlineacionTest
```

Editar `tests/Feature/AlineacionTest.php`:

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Database\UniqueConstraintViolationException;
use App\Models\PedObjetivoEstrategico;
use App\Models\PndObjetivo;
use App\Models\OdsMeta;
use App\Models\PedLineaAccion;
use App\Models\ProgramaDerivadoObjetivo;
use App\Models\ProgramaDerivado;
use App\Models\PedPlan;

class AlineacionTest extends TestCase
{
    use RefreshDatabase;

    /**
     * Test 1: No se puede duplicar una alineacion PED -> PND.
     */
    public function test_no_se_puede_duplicar_alineacion_ped_pnd(): void
    {
        $pedObj = PedObjetivoEstrategico::factory()->create();
        $pndObj = PndObjetivo::factory()->create();

        // Primera alineacion: debe funcionar
        $pedObj->pndObjetivos()->attach($pndObj->id);

        // Segunda alineacion identica: debe lanzar excepcion de constraint unico
        $this->expectException(UniqueConstraintViolationException::class);

        // attach() sin syncWithoutDetaching lanza la excepcion del UNIQUE constraint
        $pedObj->pndObjetivos()->attach($pndObj->id);
    }

    /**
     * Test 2: No se puede duplicar una alineacion PND -> ODS.
     */
    public function test_no_se_puede_duplicar_alineacion_pnd_ods(): void
    {
        $pndObj  = PndObjetivo::factory()->create();
        $odsMeta = OdsMeta::factory()->create();

        $pndObj->odsMetas()->attach($odsMeta->id);

        $this->expectException(UniqueConstraintViolationException::class);

        $pndObj->odsMetas()->attach($odsMeta->id);
    }

    /**
     * Test 3: No se puede duplicar una alineacion Linea -> Programa Derivado Objetivo.
     */
    public function test_no_se_puede_duplicar_alineacion_linea_programa_derivado(): void
    {
        $linea   = PedLineaAccion::factory()->create();
        $pdObj   = ProgramaDerivadoObjetivo::factory()->create();

        $linea->programaDerivadoObjetivos()->attach($pdObj->id);

        $this->expectException(UniqueConstraintViolationException::class);

        $linea->programaDerivadoObjetivos()->attach($pdObj->id);
    }

    /**
     * Test 4: Recorrer la cadena completa LineaAccion -> ProgramaDerivado -> PED -> PND -> ODS via Eloquent.
     *
     * Estructura del test:
     * OdsMeta <- PndObjetivo <- PedObjetivoEstrategico <- (jerarquia PED) <- PedLineaAccion
     *                                                                       -> ProgramaDerivadoObjetivo -> ProgramaDerivado
     */
    public function test_cadena_completa_alineacion_via_eloquent(): void
    {
        // Crear la cadena de datos
        $pedPlan        = PedPlan::factory()->create();
        $pedObjEst      = PedObjetivoEstrategico::factory()->create(['ped_plan_id' => $pedPlan->id]);
        $pndObj         = PndObjetivo::factory()->create();
        $odsMeta        = OdsMeta::factory()->create();
        $lineaAccion    = PedLineaAccion::factory()->create(); // debe pertenecer a la jerarquia del pedObjEst
        $programaDerivado = ProgramaDerivado::factory()->create(['ped_plan_id' => $pedPlan->id]);
        $pdObj          = ProgramaDerivadoObjetivo::factory()->create([
            'programa_derivado_id' => $programaDerivado->id,
        ]);

        // Crear alineaciones
        $pedObjEst->pndObjetivos()->attach($pndObj->id);
        $pndObj->odsMetas()->attach($odsMeta->id);
        $lineaAccion->programaDerivadoObjetivos()->attach($pdObj->id);

        // Verificar cadena PED -> PND
        $pndObjetivosDeObjEst = $pedObjEst->pndObjetivos()->get();
        $this->assertCount(1, $pndObjetivosDeObjEst);
        $this->assertEquals($pndObj->id, $pndObjetivosDeObjEst->first()->id);

        // Verificar cadena PND -> ODS
        $odsMetasDePnd = $pndObj->odsMetas()->get();
        $this->assertCount(1, $odsMetasDePnd);
        $this->assertEquals($odsMeta->id, $odsMetasDePnd->first()->id);

        // Verificar cadena Linea -> Programa Derivado Objetivo
        $pdObjsDeLinea = $lineaAccion->programaDerivadoObjetivos()->get();
        $this->assertCount(1, $pdObjsDeLinea);
        $this->assertEquals($pdObj->id, $pdObjsDeLinea->first()->id);

        // Verificar navegacion completa: Linea -> PD Objetivo -> Programa Derivado -> PedPlan
        $pdObjConRelaciones = $lineaAccion->programaDerivadoObjetivos()
            ->with('programaDerivado.pedPlan')
            ->first();
        $this->assertEquals($pedPlan->id, $pdObjConRelaciones->programaDerivado->pedPlan->id);

        // Verificar cadena inversa: OdsMeta -> PndObjetivo -> PedObjetivoEstrategico
        $pndObjsDeOdsMeta = $odsMeta->pndObjetivos()->with('pedObjetivosEstrategicos')->get();
        $this->assertCount(1, $pndObjsDeOdsMeta);
        $pedObjsDeOds = $pndObjsDeOdsMeta->first()->pedObjetivosEstrategicos;
        $this->assertCount(1, $pedObjsDeOds);
        $this->assertEquals($pedObjEst->id, $pedObjsDeOds->first()->id);
    }

    /**
     * Test 5: syncWithoutDetaching no lanza excepcion al intentar agregar relacion ya existente.
     */
    public function test_sync_without_detaching_es_idempotente(): void
    {
        $pedObj = PedObjetivoEstrategico::factory()->create();
        $pndObj = PndObjetivo::factory()->create();

        // Llamar dos veces con syncWithoutDetaching: no debe lanzar excepcion
        $pedObj->pndObjetivos()->syncWithoutDetaching([$pndObj->id]);
        $pedObj->pndObjetivos()->syncWithoutDetaching([$pndObj->id]);

        $this->assertCount(1, $pedObj->pndObjetivos()->get());
    }
}
```

Ejecutar los tests:

```bash
sail artisan test --filter AlineacionTest
```

---

### Paso 10: Verificar estructura en PostgreSQL

```bash
# Verificar las 3 tablas pivote
sail psql -c "\d alineacion_ped_pnd"
sail psql -c "\d alineacion_pnd_ods"
sail psql -c "\d alineacion_linea_programa_derivado"

# Verificar indices unicos compuestos
sail psql -c "SELECT indexname, indexdef FROM pg_indexes WHERE tablename IN ('alineacion_ped_pnd', 'alineacion_pnd_ods', 'alineacion_linea_programa_derivado') AND indexname LIKE 'uq_%';"

# Verificar datos del seeder
sail psql -c "SELECT COUNT(*) FROM alineacion_ped_pnd;"
sail psql -c "SELECT COUNT(*) FROM alineacion_pnd_ods;"
sail psql -c "SELECT COUNT(*) FROM alineacion_linea_programa_derivado;"
```

---

## Criterios de Aceptacion

| # | Criterio | Verificacion |
|---|----------|--------------|
| 1 | 3 migraciones con `up()` y `down()` limpios | `sail artisan migrate:rollback` y `migrate` sin errores |
| 2 | Indices unicos compuestos en las 3 tablas | `pg_indexes` muestra los 3 indices `uq_*` |
| 3 | `PedObjetivoEstrategico belongsToMany PndObjetivo` via `alineacion_ped_pnd` | Tinker: `$pedObj->pndObjetivos` retorna coleccion |
| 4 | `PndObjetivo belongsToMany OdsMeta` via `alineacion_pnd_ods` | Tinker: `$pndObj->odsMetas` retorna coleccion |
| 5 | `PedLineaAccion belongsToMany ProgramaDerivadoObjetivo` via tabla pivote | Tinker: `$linea->programaDerivadoObjetivos` retorna coleccion |
| 6 | Seeder crea alineaciones de ejemplo sin errores | `sail artisan migrate:fresh --seed` exitoso |
| 7 | Test: `UniqueConstraintViolationException` al duplicar alineacion | `sail artisan test --filter AlineacionTest` - tests 1, 2, 3 pasan |
| 8 | Test: cadena completa LineaAccion -> ODS navegable via Eloquent | Test 4 pasa |
| 9 | `syncWithoutDetaching` es idempotente | Test 5 pasa |

---

## Notas

- **Por que usar `attach()` en los tests de unicidad en lugar de `syncWithoutDetaching()`**: `attach()` intenta insertar directamente y lanza `UniqueConstraintViolationException` si ya existe el registro. `syncWithoutDetaching()` verifica primero si la relacion existe y solo inserta si no existe, por lo que no lanza excepcion (comportamiento idempotente). Para los tests de integridad de constraint, usar `attach()`.

- **Cascade delete en tablas pivote**: Al usar `cascadeOnDelete()` en las FK, si se elimina un `PndObjetivo`, automaticamente se eliminan las filas en `alineacion_ped_pnd` y `alineacion_pnd_ods`. Esto es el comportamiento correcto para mantener integridad referencial.

- **Factories necesarias para los tests**: Los tests asumen que existen factories para todos los modelos involucrados. Si no existen, crearlas con:

```bash
sail artisan make:factory PedObjetivoEstrategicoFactory --model=PedObjetivoEstrategico
sail artisan make:factory PndObjetivoFactory --model=PndObjetivo
sail artisan make:factory OdsMetaFactory --model=OdsMeta
sail artisan make:factory PedLineaAccionFactory --model=PedLineaAccion
sail artisan make:factory ProgramaDerivadoObjetivoFactory --model=ProgramaDerivadoObjetivo
sail artisan make:factory ProgramaDerivadoFactory --model=ProgramaDerivado
```

- **Rendimiento de la cadena completa**: Para reportes que recorren la cadena completa de todos los registros, usar la Opcion C (query SQL directa) en lugar de Eloquent para evitar el problema N+1. Para vistas detalladas de un registro especifico, el eager loading de Eloquent (Opcion A) es suficiente.

- **Alineacion muchos a muchos vs. uno a muchos**: Se eligio `belongsToMany` (tabla pivote) porque en la realidad de la planeacion publica, un objetivo estrategico del PED puede alinearse a multiples objetivos del PND, y un objetivo del PND puede relacionarse con multiples metas ODS. Esto es mas flexible que una FK directa.
