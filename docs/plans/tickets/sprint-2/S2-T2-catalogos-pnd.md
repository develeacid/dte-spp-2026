# Plan: S2-T2 — Migraciones y modelos para catálogos PND

**Ticket:** S2-T2
**Tipo:** feat
**Rama:** `feat/S2-T2-catalogos-pnd`
**Sprint:** 2 — Catálogos Normativos
**Depende de:** S0-T2 (pgvector habilitado)

---

## Contexto

El Plan Nacional de Desarrollo (PND) es el documento rector de la política pública federal en México para un sexenio. Está estructurado jerárquicamente en tres niveles:

1. **Ejes** (`pnd_ejes`): Las grandes orientaciones estratégicas del plan (e.g. "Justicia y Estado de Derecho", "Bienestar").
2. **Objetivos** (`pnd_objetivos`): Los objetivos específicos dentro de cada eje, identificados por número dentro del eje.
3. **Estrategias** (`pnd_estrategias`): Las líneas de acción concretas que instrumentan cada objetivo, identificadas por clave alfanumérica (e.g. "1.1", "2.3").

Las tres tablas incluyen columna `embedding vector(1536)` para búsqueda semántica y alineación automática contra programas presupuestarios (Sprint 5). Los embeddings se dejan en `NULL` hasta entonces.

Se usan tres migraciones separadas para las tablas base y tres adicionales para las columnas `embedding`, siguiendo el mismo patrón que S2-T1 (DDL raw con `DB::statement`).

---

## Pre-requisitos

- S0-T2 completado (extensión `vector` activa en PostgreSQL)
- `sail up -d` ejecutado

---

## Pasos

### 1. Crear las migraciones

```bash
sail artisan make:migration create_pnd_ejes_table
sail artisan make:migration create_pnd_objetivos_table
sail artisan make:migration create_pnd_estrategias_table
sail artisan make:migration add_embedding_to_pnd_ejes_table
sail artisan make:migration add_embedding_to_pnd_objetivos_table
sail artisan make:migration add_embedding_to_pnd_estrategias_table
```

---

### 2. Migración: `create_pnd_ejes_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('pnd_ejes', function (Blueprint $table) {
            $table->id();
            $table->unsignedTinyInteger('numero')->unique();
            $table->string('nombre', 200);
            $table->text('descripcion');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('pnd_ejes');
    }
};
```

---

### 3. Migración: `create_pnd_objetivos_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('pnd_objetivos', function (Blueprint $table) {
            $table->id();
            $table->foreignId('pnd_eje_id')
                  ->constrained('pnd_ejes')
                  ->onDelete('cascade');
            $table->unsignedSmallInteger('numero');
            $table->string('nombre', 300);
            $table->text('descripcion');
            $table->timestamps();

            // Un número de objetivo es único dentro de su eje
            $table->unique(['pnd_eje_id', 'numero']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('pnd_objetivos');
    }
};
```

---

### 4. Migración: `create_pnd_estrategias_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('pnd_estrategias', function (Blueprint $table) {
            $table->id();
            $table->foreignId('pnd_objetivo_id')
                  ->constrained('pnd_objetivos')
                  ->onDelete('cascade');
            $table->string('clave', 20);   // e.g. "1.1", "2.3", "3.10"
            $table->text('descripcion');
            $table->timestamps();

            // La clave es única dentro del objetivo
            $table->unique(['pnd_objetivo_id', 'clave']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('pnd_estrategias');
    }
};
```

---

### 5. Migración: `add_embedding_to_pnd_ejes_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        DB::statement('ALTER TABLE pnd_ejes ADD COLUMN embedding vector(1536)');
    }

    public function down(): void
    {
        if (Schema::hasColumn('pnd_ejes', 'embedding')) {
            DB::statement('ALTER TABLE pnd_ejes DROP COLUMN embedding');
        }
    }
};
```

---

### 6. Migración: `add_embedding_to_pnd_objetivos_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        DB::statement('ALTER TABLE pnd_objetivos ADD COLUMN embedding vector(1536)');
    }

    public function down(): void
    {
        if (Schema::hasColumn('pnd_objetivos', 'embedding')) {
            DB::statement('ALTER TABLE pnd_objetivos DROP COLUMN embedding');
        }
    }
};
```

---

### 7. Migración: `add_embedding_to_pnd_estrategias_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        DB::statement('ALTER TABLE pnd_estrategias ADD COLUMN embedding vector(1536)');
    }

    public function down(): void
    {
        if (Schema::hasColumn('pnd_estrategias', 'embedding')) {
            DB::statement('ALTER TABLE pnd_estrategias DROP COLUMN embedding');
        }
    }
};
```

---

### 8. Crear los modelos

```bash
sail artisan make:model PndEje
sail artisan make:model PndObjetivo
sail artisan make:model PndEstrategia
```

---

### 9. Modelo: `app/Models/PndEje.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PndEje extends Model
{
    protected $table = 'pnd_ejes';

    protected $fillable = [
        'numero',
        'nombre',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'numero'    => 'integer',
        'embedding' => 'array',
    ];

    public function objetivos(): HasMany
    {
        return $this->hasMany(PndObjetivo::class, 'pnd_eje_id');
    }
}
```

---

### 10. Modelo: `app/Models/PndObjetivo.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PndObjetivo extends Model
{
    protected $table = 'pnd_objetivos';

    protected $fillable = [
        'pnd_eje_id',
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
        return $this->belongsTo(PndEje::class, 'pnd_eje_id');
    }

    public function estrategias(): HasMany
    {
        return $this->hasMany(PndEstrategia::class, 'pnd_objetivo_id');
    }
}
```

---

### 11. Modelo: `app/Models/PndEstrategia.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class PndEstrategia extends Model
{
    protected $table = 'pnd_estrategias';

    protected $fillable = [
        'pnd_objetivo_id',
        'clave',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'embedding' => 'array',
    ];

    public function objetivo(): BelongsTo
    {
        return $this->belongsTo(PndObjetivo::class, 'pnd_objetivo_id');
    }
}
```

---

### 12. Crear el seeder

```bash
sail artisan make:seeder PndCatalogoSeeder
```

Editar `database/seeders/PndCatalogoSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\PndEje;
use App\Models\PndEstrategia;
use App\Models\PndObjetivo;
use Illuminate\Database\Seeder;

class PndCatalogoSeeder extends Seeder
{
    public function run(): void
    {
        // Datos de ejemplo basados en la estructura del PND vigente.
        // Los textos son representativos para desarrollo; los textos oficiales
        // completos se importan desde el documento fuente en Sprint 5.
        // Los embeddings se calculan en Sprint 5 y quedan NULL por ahora.

        $ejes = [
            [
                'numero'     => 1,
                'nombre'     => 'Justicia y Estado de Derecho',
                'descripcion' => 'Garantizar el acceso a la justicia, el fortalecimiento de las instituciones del Estado y el pleno respeto a los derechos humanos de todas las personas.',
                'objetivos' => [
                    [
                        'numero'     => 1,
                        'nombre'     => 'Garantizar el acceso a la justicia',
                        'descripcion' => 'Fortalecer el sistema de justicia para garantizar el acceso efectivo a la justicia de toda la población, con especial atención a grupos vulnerables.',
                        'estrategias' => [
                            ['clave' => '1.1', 'descripcion' => 'Ampliar la cobertura territorial de los servicios de defensoría pública y asistencia jurídica gratuita.'],
                            ['clave' => '1.2', 'descripcion' => 'Promover mecanismos alternativos de solución de controversias para descongestionar el sistema judicial.'],
                            ['clave' => '1.3', 'descripcion' => 'Fortalecer la carrera judicial y los mecanismos de rendición de cuentas en el Poder Judicial.'],
                        ],
                    ],
                    [
                        'numero'     => 2,
                        'nombre'     => 'Combate a la corrupción e impunidad',
                        'descripcion' => 'Erradicar la corrupción en todas las instancias gubernamentales mediante mecanismos de transparencia, control interno y sanción efectiva.',
                        'estrategias' => [
                            ['clave' => '2.1', 'descripcion' => 'Consolidar el Sistema Nacional Anticorrupción y sus instancias de coordinación interinstitucional.'],
                            ['clave' => '2.2', 'descripcion' => 'Fortalecer las capacidades de investigación y sanción de los órganos internos de control.'],
                        ],
                    ],
                ],
            ],
            [
                'numero'     => 2,
                'nombre'     => 'Bienestar',
                'descripcion' => 'Garantizar el ejercicio pleno de los derechos sociales de toda la población, con énfasis en los grupos históricamente marginados, a través de políticas públicas integrales de desarrollo social.',
                'objetivos' => [
                    [
                        'numero'     => 1,
                        'nombre'     => 'Garantizar el derecho a la educación',
                        'descripcion' => 'Asegurar el acceso universal, la permanencia y el logro educativo en todos los niveles, con pertinencia cultural y enfoque de derechos.',
                        'estrategias' => [
                            ['clave' => '1.1', 'descripcion' => 'Universalizar la educación inicial y preescolar con calidad y pertinencia territorial.'],
                            ['clave' => '1.2', 'descripcion' => 'Reducir el abandono escolar en educación básica y media superior mediante becas y apoyos socioemocionales.'],
                            ['clave' => '1.3', 'descripcion' => 'Fortalecer la formación docente continua con enfoque en competencias pedagógicas y uso de tecnología.'],
                        ],
                    ],
                    [
                        'numero'     => 2,
                        'nombre'     => 'Garantizar el derecho a la salud',
                        'descripcion' => 'Consolidar un sistema de salud universal, gratuito y de calidad que atienda las necesidades de toda la población, especialmente la que carece de seguridad social.',
                        'estrategias' => [
                            ['clave' => '2.1', 'descripcion' => 'Fortalecer la red de unidades de salud del primer nivel de atención en zonas rurales y marginadas.'],
                            ['clave' => '2.2', 'descripcion' => 'Garantizar el abasto suficiente y oportuno de medicamentos en el sistema público de salud.'],
                        ],
                    ],
                ],
            ],
            [
                'numero'     => 3,
                'nombre'     => 'Seguridad',
                'descripcion' => 'Garantizar la paz y la seguridad pública a través de la estrategia de seguridad con enfoque humanista, la atención a las causas de la violencia y la coordinación entre los tres órdenes de gobierno.',
                'objetivos' => [],
            ],
            [
                'numero'     => 4,
                'nombre'     => 'Economía',
                'descripcion' => 'Impulsar el desarrollo económico sostenible e incluyente que genere empleos de calidad y mejore las condiciones de vida de la población, con atención prioritaria a las regiones y sectores rezagados.',
                'objetivos' => [],
            ],
        ];

        foreach ($ejes as $ejeData) {
            $objetivosData = $ejeData['objetivos'];
            unset($ejeData['objetivos']);

            $eje = PndEje::create($ejeData);

            foreach ($objetivosData as $objetivoData) {
                $estrategiasData = $objetivoData['estrategias'];
                unset($objetivoData['estrategias']);

                $objetivo = PndObjetivo::create(array_merge(
                    $objetivoData,
                    ['pnd_eje_id' => $eje->id]
                ));

                foreach ($estrategiasData as $estrategiaData) {
                    PndEstrategia::create(array_merge(
                        $estrategiaData,
                        ['pnd_objetivo_id' => $objetivo->id]
                    ));
                }
            }
        }

        $this->command->info(sprintf(
            'PND cargado: %d ejes, %d objetivos, %d estrategias.',
            PndEje::count(),
            PndObjetivo::count(),
            PndEstrategia::count()
        ));
    }
}
```

---

### 13. Registrar el seeder en DatabaseSeeder

```php
public function run(): void
{
    $this->call([
        // ... seeders existentes ...
        OdsCatalogoSeeder::class,
        PndCatalogoSeeder::class,
    ]);
}
```

---

### 14. Ejecutar y verificar

```bash
sail artisan migrate:fresh --seed
```

Verificar en Tinker:

```bash
sail artisan tinker
```

```php
use App\Models\PndEje;
use App\Models\PndObjetivo;
use App\Models\PndEstrategia;

// Verificar conteos
PndEje::count();          // 4
PndObjetivo::count();     // >= 4 (2 objetivos en cada uno de los 2 primeros ejes)
PndEstrategia::count();   // >= 7

// Verificar jerarquía completa
$eje = PndEje::with('objetivos.estrategias')->where('numero', 1)->first();
$eje->nombre;                                         // "Justicia y Estado de Derecho"
$eje->objetivos->count();                             // 2
$eje->objetivos->first()->estrategias->count();       // 3

// Verificar relaciones inversas
$estrategia = PndEstrategia::where('clave', '1.1')
    ->whereHas('objetivo', fn($q) => $q->where('numero', 1))
    ->first();
$estrategia->objetivo->nombre;                        // "Garantizar el acceso a la justicia"
$estrategia->objetivo->eje->nombre;                   // "Justicia y Estado de Derecho"

// Verificar que embeddings son NULL (se llenan en Sprint 5)
PndEje::whereNotNull('embedding')->count();           // 0

exit
```

---

## Criterios de aceptación

- [ ] 3 migraciones de tablas base: `pnd_ejes`, `pnd_objetivos`, `pnd_estrategias` con `up()` y `down()` correctos
- [ ] 3 migraciones separadas agregan columna `embedding vector(1536)` a cada tabla vía `DB::statement`
- [ ] FK `pnd_objetivos.pnd_eje_id` → `pnd_ejes.id` con `CASCADE`
- [ ] FK `pnd_estrategias.pnd_objetivo_id` → `pnd_objetivos.id` con `CASCADE`
- [ ] Unique constraint en `(pnd_eje_id, numero)` en `pnd_objetivos`
- [ ] Unique constraint en `(pnd_objetivo_id, clave)` en `pnd_estrategias`
- [ ] Modelo `PndEje` con `hasMany PndObjetivo`
- [ ] Modelo `PndObjetivo` con `belongsTo PndEje` y `hasMany PndEstrategia`
- [ ] Modelo `PndEstrategia` con `belongsTo PndObjetivo`
- [ ] Los tres modelos tienen `$fillable` y cast `embedding => array`
- [ ] `PndCatalogoSeeder` carga mínimo 2 ejes con 2 objetivos cada uno y sus estrategias
- [ ] `sail artisan migrate:fresh --seed` ejecuta sin errores

---

## Esquema resultante

```
pnd_ejes
├── id                   BIGSERIAL PK
├── numero               SMALLINT UNIQUE
├── nombre               VARCHAR(200)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

pnd_objetivos
├── id                   BIGSERIAL PK
├── pnd_eje_id           BIGINT FK → pnd_ejes.id CASCADE
├── numero               SMALLINT
├── nombre               VARCHAR(300)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(pnd_eje_id, numero)

pnd_estrategias
├── id                   BIGSERIAL PK
├── pnd_objetivo_id      BIGINT FK → pnd_objetivos.id CASCADE
├── clave                VARCHAR(20)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(pnd_objetivo_id, clave)
```

---

## Notas

- **Sobre el PND vigente:** Los datos del seeder son representativos para desarrollo. El documento oficial del PND vigente se importará desde su fuente (PDF o Markdown) en Sprint 5 mediante un comando Artisan de carga masiva.
- **Número vs. clave en estrategias:** Se usa `clave` (string) en lugar de un entero porque las claves de estrategias en el PND a veces incluyen letras o siguen convenciones de numeración no secuenciales (e.g. "2.a", "3bis"). Si el PND vigente usa siempre numeración limpia, se puede refactorizar a `unsignedSmallInteger`.
- **Embeddings:** Mismo patrón que S2-T1: columna `NULL` hasta Sprint 5. La dimensión `1536` es para `text-embedding-ada-002`; ajustar si se cambia el modelo.
- **Índices vectoriales:** No se crean en este sprint. Los índices `ivfflat` o `hnsw` se agregan en Sprint 5 después de poblar los embeddings, ya que IVFFlat requiere datos previos para construir los centroides.
- **Relación con programas presupuestarios:** La tabla de alineación `programa_pnd` (M:M entre programas y estrategias PND) se crea en Sprint 3 cuando exista la tabla `programas_presupuestarios`.
