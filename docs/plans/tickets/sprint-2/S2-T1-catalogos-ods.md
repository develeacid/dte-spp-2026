# Plan: S2-T1 — Migraciones y modelos para catálogos ODS

**Ticket:** S2-T1
**Tipo:** feat
**Rama:** `feat/S2-T1-catalogos-ods`
**Sprint:** 2 — Catálogos Normativos
**Depende de:** S0-T2 (pgvector habilitado)

---

## Contexto

Los Objetivos de Desarrollo Sostenible (ODS) de la ONU son 17 objetivos globales con 169 metas asociadas. En este sistema se usan como catálogo normativo de referencia para la alineación de programas presupuestarios y planes estatales de desarrollo.

Se crean dos tablas:

- `ods_objetivos`: Los 17 objetivos globales, cada uno con su número (1-17), nombre y descripción. La columna `embedding` almacena la representación vectorial de la descripción para búsqueda semántica (generada en Sprint 5).
- `ods_metas`: Las metas de cada objetivo, identificadas por clave compuesta (e.g. "1.1", "1.2"). También incluyen embedding para búsqueda semántica.

Los embeddings se dejan en `NULL` hasta que el módulo de generación de embeddings (Sprint 5) los calcule. Las columnas se crean mediante `DB::statement` con DDL raw porque Laravel Eloquent no soporta el tipo `vector(n)` de pgvector de forma nativa.

---

## Pre-requisitos

- S0-T2 completado (extensión `vector` activa en PostgreSQL)
- `sail up -d` ejecutado

---

## Pasos

### 1. Crear las migraciones

```bash
sail artisan make:migration create_ods_objetivos_table
sail artisan make:migration create_ods_metas_table
sail artisan make:migration add_embedding_to_ods_objetivos_table
sail artisan make:migration add_embedding_to_ods_metas_table
```

> **Estrategia:** Se crean las tablas base primero con Blueprint normal. Las columnas `embedding vector(1536)` se agregan en migraciones separadas usando `DB::statement`, ya que el tipo `vector` no está soportado por el Blueprint estándar. Esto permite que `down()` use `dropColumn` limpiamente y mantiene la separación de responsabilidades.

---

### 2. Migración: `create_ods_objetivos_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ods_objetivos', function (Blueprint $table) {
            $table->id();
            $table->unsignedTinyInteger('numero')->unique(); // 1 al 17
            $table->string('nombre', 200);
            $table->text('descripcion');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ods_objetivos');
    }
};
```

---

### 3. Migración: `create_ods_metas_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ods_metas', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ods_objetivo_id')
                  ->constrained('ods_objetivos')
                  ->onDelete('cascade');
            $table->string('clave', 10);   // e.g. "1.1", "1.2", "17.19"
            $table->text('descripcion');
            $table->timestamps();

            $table->unique(['ods_objetivo_id', 'clave']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('ods_metas');
    }
};
```

---

### 4. Migración: `add_embedding_to_ods_objetivos_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // El tipo vector(1536) no está soportado por Blueprint — se usa DDL raw.
        // 1536 dimensiones corresponden al modelo text-embedding-ada-002 de OpenAI.
        DB::statement('ALTER TABLE ods_objetivos ADD COLUMN embedding vector(1536)');
    }

    public function down(): void
    {
        if (Schema::hasColumn('ods_objetivos', 'embedding')) {
            DB::statement('ALTER TABLE ods_objetivos DROP COLUMN embedding');
        }
    }
};
```

---

### 5. Migración: `add_embedding_to_ods_metas_table`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        DB::statement('ALTER TABLE ods_metas ADD COLUMN embedding vector(1536)');
    }

    public function down(): void
    {
        if (Schema::hasColumn('ods_metas', 'embedding')) {
            DB::statement('ALTER TABLE ods_metas DROP COLUMN embedding');
        }
    }
};
```

---

### 6. Crear los modelos

```bash
sail artisan make:model OdsObjetivo
sail artisan make:model OdsMeta
```

---

### 7. Modelo: `app/Models/OdsObjetivo.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class OdsObjetivo extends Model
{
    protected $table = 'ods_objetivos';

    protected $fillable = [
        'numero',
        'nombre',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'numero'    => 'integer',
        'embedding' => 'array',  // PostgreSQL devuelve el vector como string "[0.1,0.2,...]"
    ];

    public function metas(): HasMany
    {
        return $this->hasMany(OdsMeta::class, 'ods_objetivo_id');
    }
}
```

---

### 8. Modelo: `app/Models/OdsMeta.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class OdsMeta extends Model
{
    protected $table = 'ods_metas';

    protected $fillable = [
        'ods_objetivo_id',
        'clave',
        'descripcion',
        'embedding',
    ];

    protected $casts = [
        'embedding' => 'array',
    ];

    public function objetivo(): BelongsTo
    {
        return $this->belongsTo(OdsObjetivo::class, 'ods_objetivo_id');
    }
}
```

---

### 9. Crear el seeder

```bash
sail artisan make:seeder OdsCatalogoSeeder
```

Editar `database/seeders/OdsCatalogoSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\OdsMeta;
use App\Models\OdsObjetivo;
use Illuminate\Database\Seeder;

class OdsCatalogoSeeder extends Seeder
{
    public function run(): void
    {
        // Datos fuente: https://sdgs.un.org/goals
        // Los embeddings se calculan en Sprint 5 (módulo de embeddings).
        // Este seeder carga texto base; embedding queda NULL hasta entonces.

        $objetivos = [
            [
                'numero'     => 1,
                'nombre'     => 'Fin de la pobreza',
                'descripcion' => 'Poner fin a la pobreza en todas sus formas en todo el mundo.',
                'metas' => [
                    ['clave' => '1.1', 'descripcion' => 'Para 2030, erradicar la pobreza extrema para todas las personas en el mundo, actualmente medida por un ingreso por persona inferior a 1,25 dólares de los Estados Unidos al día.'],
                    ['clave' => '1.2', 'descripcion' => 'Para 2030, reducir al menos a la mitad la proporción de hombres, mujeres y niños de todas las edades que viven en la pobreza en todas sus dimensiones con arreglo a las definiciones nacionales.'],
                    ['clave' => '1.3', 'descripcion' => 'Poner en práctica a nivel nacional sistemas y medidas apropiados de protección social para todos, incluidos niveles mínimos, y, para 2030, lograr una amplia cobertura de las personas pobres y vulnerables.'],
                ],
            ],
            [
                'numero'     => 2,
                'nombre'     => 'Hambre cero',
                'descripcion' => 'Poner fin al hambre, lograr la seguridad alimentaria y la mejora de la nutrición y promover la agricultura sostenible.',
                'metas' => [
                    ['clave' => '2.1', 'descripcion' => 'Para 2030, poner fin al hambre y asegurar el acceso de todas las personas, en particular los pobres y las personas en situaciones vulnerables, incluidos los lactantes, a una alimentación sana, nutritiva y suficiente durante todo el año.'],
                    ['clave' => '2.2', 'descripcion' => 'Para 2030, poner fin a todas las formas de malnutrición, incluso logrando, a más tardar en 2025, las metas convenidas internacionalmente sobre el retraso del crecimiento y la emaciación de los niños menores de 5 años.'],
                ],
            ],
            [
                'numero'     => 3,
                'nombre'     => 'Salud y bienestar',
                'descripcion' => 'Garantizar una vida sana y promover el bienestar para todos en todas las edades.',
                'metas' => [],
            ],
            [
                'numero'     => 4,
                'nombre'     => 'Educación de calidad',
                'descripcion' => 'Garantizar una educación inclusiva, equitativa y de calidad y promover oportunidades de aprendizaje durante toda la vida para todos.',
                'metas' => [],
            ],
            [
                'numero'     => 5,
                'nombre'     => 'Igualdad de género',
                'descripcion' => 'Lograr la igualdad entre los géneros y empoderar a todas las mujeres y las niñas.',
                'metas' => [],
            ],
            [
                'numero'     => 6,
                'nombre'     => 'Agua limpia y saneamiento',
                'descripcion' => 'Garantizar la disponibilidad de agua y su gestión sostenible y el saneamiento para todos.',
                'metas' => [],
            ],
            [
                'numero'     => 7,
                'nombre'     => 'Energía asequible y no contaminante',
                'descripcion' => 'Garantizar el acceso a una energía asequible, segura, sostenible y moderna para todos.',
                'metas' => [],
            ],
            [
                'numero'     => 8,
                'nombre'     => 'Trabajo decente y crecimiento económico',
                'descripcion' => 'Promover el crecimiento económico inclusivo y sostenible, el empleo y el trabajo decente para todos.',
                'metas' => [],
            ],
            [
                'numero'     => 9,
                'nombre'     => 'Industria, innovación e infraestructura',
                'descripcion' => 'Construir infraestructuras resilientes, promover la industrialización inclusiva y sostenible, y fomentar la innovación.',
                'metas' => [],
            ],
            [
                'numero'     => 10,
                'nombre'     => 'Reducción de las desigualdades',
                'descripcion' => 'Reducir la desigualdad en y entre los países.',
                'metas' => [],
            ],
            [
                'numero'     => 11,
                'nombre'     => 'Ciudades y comunidades sostenibles',
                'descripcion' => 'Lograr que las ciudades y los asentamientos humanos sean inclusivos, seguros, resilientes y sostenibles.',
                'metas' => [],
            ],
            [
                'numero'     => 12,
                'nombre'     => 'Producción y consumo responsables',
                'descripcion' => 'Garantizar modalidades de consumo y producción sostenibles.',
                'metas' => [],
            ],
            [
                'numero'     => 13,
                'nombre'     => 'Acción por el clima',
                'descripcion' => 'Adoptar medidas urgentes para combatir el cambio climático y sus efectos.',
                'metas' => [],
            ],
            [
                'numero'     => 14,
                'nombre'     => 'Vida submarina',
                'descripcion' => 'Conservar y utilizar en forma sostenible los océanos, los mares y los recursos marinos para el desarrollo sostenible.',
                'metas' => [],
            ],
            [
                'numero'     => 15,
                'nombre'     => 'Vida de ecosistemas terrestres',
                'descripcion' => 'Gestionar sosteniblemente los bosques, luchar contra la desertificación, detener e invertir la degradación de las tierras y detener la pérdida de biodiversidad.',
                'metas' => [],
            ],
            [
                'numero'     => 16,
                'nombre'     => 'Paz, justicia e instituciones sólidas',
                'descripcion' => 'Promover sociedades justas, pacíficas e inclusivas.',
                'metas' => [],
            ],
            [
                'numero'     => 17,
                'nombre'     => 'Alianzas para lograr los objetivos',
                'descripcion' => 'Revitalizar la Alianza Mundial para el Desarrollo Sostenible.',
                'metas' => [],
            ],
        ];

        foreach ($objetivos as $data) {
            $metas = $data['metas'];
            unset($data['metas']);

            $objetivo = OdsObjetivo::create($data);

            foreach ($metas as $meta) {
                OdsMeta::create([
                    'ods_objetivo_id' => $objetivo->id,
                    'clave'           => $meta['clave'],
                    'descripcion'     => $meta['descripcion'],
                ]);
            }
        }

        $this->command->info('ODS cargados: ' . OdsObjetivo::count() . ' objetivos, ' . OdsMeta::count() . ' metas.');
    }
}
```

---

### 10. Registrar el seeder en DatabaseSeeder

Editar `database/seeders/DatabaseSeeder.php` para incluir el nuevo seeder:

```php
public function run(): void
{
    $this->call([
        // ... seeders existentes de Sprint 1 ...
        OdsCatalogoSeeder::class,
    ]);
}
```

---

### 11. Ejecutar y verificar

```bash
sail artisan migrate:fresh --seed
```

Verificar en Tinker:

```bash
sail artisan tinker
```

```php
use App\Models\OdsObjetivo;
use App\Models\OdsMeta;

// Verificar cantidad
OdsObjetivo::count();          // 17
OdsMeta::count();              // >= 5 (ODS 1 tiene 3, ODS 2 tiene 2)

// Verificar ODS 1 completo
$ods1 = OdsObjetivo::with('metas')->where('numero', 1)->first();
$ods1->nombre;                 // "Fin de la pobreza"
$ods1->metas->pluck('clave'); // ["1.1", "1.2", "1.3"]

// Verificar relación inversa
$meta = OdsMeta::where('clave', '1.1')->first();
$meta->objetivo->nombre;       // "Fin de la pobreza"

// Verificar que embedding es NULL (se llena en Sprint 5)
$ods1->embedding;              // null

exit
```

---

## Criterios de aceptación

- [ ] Migración `ods_objetivos` con columnas `id`, `numero` (unique), `nombre`, `descripcion`, `timestamps`
- [ ] Migración `ods_metas` con FK a `ods_objetivos` con cascade, columnas `clave`, `descripcion`, `timestamps`, unique en `(ods_objetivo_id, clave)`
- [ ] Migraciones separadas agregan columna `embedding vector(1536)` a ambas tablas vía `DB::statement`
- [ ] Modelo `OdsObjetivo` con `$fillable`, cast `embedding => array`, relación `hasMany OdsMeta`
- [ ] Modelo `OdsMeta` con `$fillable`, cast `embedding => array`, relación `belongsTo OdsObjetivo`
- [ ] `OdsCatalogoSeeder` carga los 17 ODS con al menos ODS 1 (3 metas) y ODS 2 (2 metas) completos
- [ ] `sail artisan migrate:fresh --seed` ejecuta sin errores
- [ ] `OdsObjetivo::count()` retorna 17 y `OdsMeta::count()` retorna >= 5

---

## Esquema resultante

```
ods_objetivos
├── id                   BIGSERIAL PK
├── numero               SMALLINT UNIQUE (1-17)
├── nombre               VARCHAR(200)
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

ods_metas
├── id                   BIGSERIAL PK
├── ods_objetivo_id      BIGINT FK → ods_objetivos.id CASCADE
├── clave                VARCHAR(10)  (e.g. "1.1", "17.19")
├── descripcion          TEXT
├── embedding            VECTOR(1536) NULL
├── created_at           TIMESTAMP
└── updated_at           TIMESTAMP

UNIQUE(ods_objetivo_id, clave)
```

---

## Notas

- **Sobre columnas vector:** Laravel Blueprint no soporta el tipo `vector(n)` de pgvector. Las dos opciones disponibles son:
  1. **DDL raw con `DB::statement`** (opción elegida en este ticket): simple, sin dependencias adicionales, compatible con cualquier versión de Laravel.
  2. **Paquete `tpetry/laravel-postgresql-enhanced`**: permite usar `$table->vector('embedding', 1536)` en Blueprint. Considerar si se instala este paquete en el proyecto para unificar la sintaxis en sprints posteriores.
- **Dimensión 1536:** Corresponde al modelo `text-embedding-ada-002` de OpenAI. Si se cambia a otro modelo (e.g. `text-embedding-3-small` con 1536 dims o `text-embedding-3-large` con 3072 dims), ajustar la dimensión en todas las migraciones de embeddings.
- **Embeddings NULL:** Las columnas `embedding` se dejan en `NULL` en este sprint. El módulo de generación de embeddings (Sprint 5) las llenará procesando el texto de `descripcion` vía la API de OpenAI.
- **Cast `embedding => array`:** pgvector devuelve los vectores como string en formato `[0.1,0.2,...]`. El cast a `array` convierte ese string a un array PHP. Para operaciones de similitud coseno se usarán queries raw en Sprint 5.
- **Datos fuente:** El seeder usa datos hardcoded para desarrollo. Los textos oficiales completos de los ODS (con las 169 metas) se importarán desde el archivo Markdown fuente en Sprint 5 al implementar el comando de carga masiva.
- Este ticket no incluye índices vectoriales (`CREATE INDEX ... USING ivfflat`). Los índices de búsqueda aproximada se crean en Sprint 5 una vez que los embeddings están poblados.
