# S2-T11 — Servicio de Busqueda Semantica por Similitud

**Tipo:** feat
**Rama:** `feat/S2-T11-busqueda-semantica`
**Depende de:** S2-T10 (EmbeddingService disponible), S0-T2 (pgvector con indices HNSW)

---

## Contexto

La busqueda semantica permite al sistema sugerir alineaciones automaticamente (S2-T8) y en el futuro ofrecer busqueda por significado en toda la plataforma. En lugar de buscar por palabras clave exactas, se vectoriza la consulta del usuario y se buscan los registros cuyos embeddings esten mas cerca en el espacio vectorial.

**Operadores pgvector:**

| Operador | Significado | Rango |
|----------|-------------|-------|
| `<=>` | Distancia coseno | 0 = identicos, 2 = opuestos |
| `<#>` | Producto interno negativo | menor = mas similar |
| `<->` | Distancia L2 | 0 = identicos |

Para este ticket se usa **distancia coseno (`<=>`)**. La similitud se calcula como:

```
similitud = 1 - (embedding <=> query_vector)
```

Para un umbral de 0.7: `WHERE 1 - (embedding <=> query_vector) >= 0.7`

**Indices HNSW** (Hierarchical Navigable Small World): mas rapidos que IVFFlat para consultas ANN (Approximate Nearest Neighbor). Se configuran con `m` (conectividad) y `ef_construction` (precision de construccion).

---

## Pre-requisitos

- S2-T10 completado: `EmbeddingService` disponible, columnas `embedding vector(1536)` en todas las tablas, jobs corriendo.
- Datos con embedding en al menos una tabla (para probar queries).
- pgvector >= 0.5.0 (soporte HNSW).

Verificar version de pgvector:

```bash
sail psql -U "${DB_USERNAME}" -d "${DB_DATABASE}" -c "SELECT extversion FROM pg_extension WHERE extname = 'vector';"
```

Verificar que hay registros con embedding no nulo:

```bash
sail psql -U "${DB_USERNAME}" -d "${DB_DATABASE}" -c "
    SELECT table_name,
           COUNT(*) FILTER (WHERE embedding IS NOT NULL) AS con_embedding,
           COUNT(*) AS total
    FROM (
        SELECT 'ods_metas' AS table_name, embedding FROM ods_metas
        UNION ALL
        SELECT 'pnd_objetivos', embedding FROM pnd_objetivos
        UNION ALL
        SELECT 'ped_objetivos_estrategicos', embedding FROM ped_objetivos_estrategicos
        UNION ALL
        SELECT 'ped_lineas_accion', embedding FROM ped_lineas_accion
        UNION ALL
        SELECT 'programa_derivado_objetivos', embedding FROM programa_derivado_objetivos
    ) t
    GROUP BY table_name;
"
```

---

## Pasos

### 1. Migracion para indices HNSW

```bash
sail artisan make:migration create_hnsw_embedding_indexes
```

```php
<?php
// database/migrations/xxxx_xx_xx_xxxxxx_create_hnsw_embedding_indexes.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    /**
     * Los indices HNSW de pgvector usan cosine distance para coincidir
     * con el operador <=> que usamos en las queries.
     *
     * Parametros:
     *   m              = 16  -> numero de conexiones por nodo (mas alto = mas preciso, mas RAM)
     *   ef_construction = 64 -> tamano de la lista de candidatos en construccion
     */
    public function up(): void
    {
        // Se deshabilita el mantenimiento de indices durante la creacion para mayor velocidad
        DB::statement('SET maintenance_work_mem = "512MB"');

        $tablas = [
            'ods_metas'                    => 'idx_hnsw_ods_metas_embedding',
            'pnd_objetivos'                => 'idx_hnsw_pnd_objetivos_embedding',
            'ped_objetivos_estrategicos'   => 'idx_hnsw_ped_objetivos_embedding',
            'ped_lineas_accion'            => 'idx_hnsw_ped_lineas_embedding',
            'programa_derivado_objetivos'  => 'idx_hnsw_prg_der_obj_embedding',
        ];

        foreach ($tablas as $tabla => $nombreIndice) {
            DB::statement("
                CREATE INDEX IF NOT EXISTS {$nombreIndice}
                ON {$tabla}
                USING hnsw (embedding vector_cosine_ops)
                WITH (m = 16, ef_construction = 64)
            ");
        }
    }

    public function down(): void
    {
        $indices = [
            'idx_hnsw_ods_metas_embedding',
            'idx_hnsw_pnd_objetivos_embedding',
            'idx_hnsw_ped_objetivos_embedding',
            'idx_hnsw_ped_lineas_embedding',
            'idx_hnsw_prg_der_obj_embedding',
        ];

        foreach ($indices as $indice) {
            DB::statement("DROP INDEX IF EXISTS {$indice}");
        }
    }
};
```

Ejecutar:

```bash
sail artisan migrate
```

Verificar indices creados:

```bash
sail psql -U "${DB_USERNAME}" -d "${DB_DATABASE}" -c "
    SELECT indexname, tablename, indexdef
    FROM pg_indexes
    WHERE indexname LIKE 'idx_hnsw_%'
    ORDER BY tablename;
"
```

### 2. DTO para resultado de busqueda

```bash
sail artisan make:class DTOs/SemanticSearchResult
```

```php
<?php
// app/DTOs/SemanticSearchResult.php

namespace App\DTOs;

use Illuminate\Database\Eloquent\Model;

readonly class SemanticSearchResult
{
    public function __construct(
        public Model  $model,
        public float  $score,       // similitud coseno: 0.0 a 1.0
        public string $modelClass,  // nombre completo de la clase
    ) {}

    /**
     * Retorna el score como porcentaje entero (0-100).
     */
    public function scorePercent(): int
    {
        return (int) round($this->score * 100);
    }

    /**
     * Texto representativo del resultado para mostrar en UI.
     */
    public function label(): string
    {
        return $this->model->getAttribute('nombre')
            ?? $this->model->getAttribute('descripcion')
            ?? $this->model->getAttribute('titulo')
            ?? '—';
    }
}
```

### 3. Crear SemanticSearchService

```bash
sail artisan make:class Services/SemanticSearchService
```

```php
<?php
// app/Services/SemanticSearchService.php

namespace App\Services;

use App\DTOs\SemanticSearchResult;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class SemanticSearchService
{
    /**
     * Modelos soportados para busqueda semantica.
     * Mapea alias (corto) a clase completa.
     */
    public const MODELOS_SOPORTADOS = [
        'OdsMeta'                  => \App\Models\OdsMeta::class,
        'PndObjetivo'              => \App\Models\PndObjetivo::class,
        'PedObjetivoEstrategico'   => \App\Models\PedObjetivoEstrategico::class,
        'PedLineaAccion'           => \App\Models\PedLineaAccion::class,
        'ProgramaDerivadoObjetivo' => \App\Models\ProgramaDerivadoObjetivo::class,
    ];

    public function __construct(
        private readonly EmbeddingService $embeddingService,
    ) {}

    /**
     * Busca los registros mas similares semanticamente al texto dado.
     *
     * @param  string  $texto      Consulta en lenguaje natural.
     * @param  string  $model      Clase del modelo Eloquent (nombre corto o FQCN).
     * @param  int     $limit      Numero maximo de resultados.
     * @param  float   $threshold  Similitud minima (0.0 a 1.0). Por defecto 0.7.
     *
     * @return Collection<SemanticSearchResult>  Ordenada por score DESC.
     *
     * @throws \InvalidArgumentException Si el modelo no esta en la lista de soportados.
     * @throws \RuntimeException         Si la API de embeddings falla.
     */
    public function findSimilar(
        string $texto,
        string $model,
        int    $limit     = 5,
        float  $threshold = 0.7,
    ): Collection {
        $modelClass = $this->resolverClase($model);

        // Generar embedding del texto de consulta
        $queryVector    = $this->embeddingService->generate($texto);
        $queryVectorStr = $this->embeddingService->toVectorString($queryVector);

        /** @var \Illuminate\Database\Eloquent\Builder $instance */
        $instance = new $modelClass();
        $tabla    = $instance->getTable();
        $pk       = $instance->getKeyName();

        /*
         * Query pgvector:
         *   - Calcula distancia coseno con <=>
         *   - Convierte a similitud: 1 - distancia
         *   - Filtra por umbral minimo
         *   - Ordena por similitud DESC
         *   - Limita resultados
         *
         * Nota: pgvector retorna distancias, no similitudes.
         * Para cosine: distancia 0 = perfectamente identicos, 2 = opuestos.
         * similitud = 1 - distancia (rango: -1 a 1, en la practica 0 a 1)
         */
        $sql = "
            SELECT
                {$pk},
                1 - (embedding <=> ?::vector) AS similitud_coseno
            FROM {$tabla}
            WHERE
                embedding IS NOT NULL
                AND 1 - (embedding <=> ?::vector) >= ?
            ORDER BY embedding <=> ?::vector ASC
            LIMIT ?
        ";

        $rows = DB::select($sql, [
            $queryVectorStr, // para el SELECT
            $queryVectorStr, // para el WHERE
            $threshold,
            $queryVectorStr, // para el ORDER BY
            $limit,
        ]);

        if (empty($rows)) {
            return collect();
        }

        // Cargar los modelos en batch para eficiencia
        $ids       = collect($rows)->pluck($pk)->toArray();
        $modelos   = $modelClass::whereIn($pk, $ids)->get()->keyBy($pk);

        // Construir coleccion de resultados ordenada por similitud
        return collect($rows)
            ->map(function ($row) use ($modelos, $modelClass, $pk) {
                $id    = $row->$pk;
                $model = $modelos->get($id);

                if (! $model) {
                    return null; // El registro fue eliminado entre la query y el fetch
                }

                return new SemanticSearchResult(
                    model:      $model,
                    score:      (float) $row->similitud_coseno,
                    modelClass: $modelClass,
                );
            })
            ->filter()                             // quitar nulos
            ->sortByDesc(fn ($r) => $r->score)     // garantizar orden DESC
            ->values();
    }

    /**
     * Busca en multiples modelos y retorna resultados mezclados ordenados por score.
     *
     * @param  string   $texto
     * @param  string[] $modelos  Array de nombres de clase (cortos o FQCN).
     * @param  int      $limitPorModelo
     * @param  float    $threshold
     *
     * @return Collection<SemanticSearchResult>
     */
    public function findSimilarMultiple(
        string $texto,
        array  $modelos,
        int    $limitPorModelo = 5,
        float  $threshold      = 0.7,
    ): Collection {
        // Generar embedding una sola vez para todas las queries
        $queryVector    = $this->embeddingService->generate($texto);
        $queryVectorStr = $this->embeddingService->toVectorString($queryVector);

        $todos = collect();

        foreach ($modelos as $modelo) {
            try {
                $modelClass = $this->resolverClase($modelo);
                $resultados = $this->findSimilarConVector(
                    $queryVectorStr,
                    $modelClass,
                    $limitPorModelo,
                    $threshold,
                );
                $todos = $todos->concat($resultados);
            } catch (\Throwable $e) {
                Log::warning("SemanticSearchService: fallo al buscar en {$modelo}: " . $e->getMessage());
            }
        }

        return $todos->sortByDesc(fn ($r) => $r->score)->values();
    }

    /**
     * Version interna que recibe el vector ya generado (evita llamadas duplicadas a la API).
     */
    private function findSimilarConVector(
        string $queryVectorStr,
        string $modelClass,
        int    $limit,
        float  $threshold,
    ): Collection {
        $instance = new $modelClass();
        $tabla    = $instance->getTable();
        $pk       = $instance->getKeyName();

        $sql = "
            SELECT {$pk}, 1 - (embedding <=> ?::vector) AS similitud_coseno
            FROM {$tabla}
            WHERE embedding IS NOT NULL AND 1 - (embedding <=> ?::vector) >= ?
            ORDER BY embedding <=> ?::vector ASC
            LIMIT ?
        ";

        $rows    = DB::select($sql, [$queryVectorStr, $queryVectorStr, $threshold, $queryVectorStr, $limit]);
        $ids     = collect($rows)->pluck($pk)->toArray();
        $modelos = $modelClass::whereIn($pk, $ids)->get()->keyBy($pk);

        return collect($rows)->map(function ($row) use ($modelos, $modelClass, $pk) {
            $model = $modelos->get($row->$pk);
            if (! $model) return null;
            return new SemanticSearchResult(
                model:      $model,
                score:      (float) $row->similitud_coseno,
                modelClass: $modelClass,
            );
        })->filter()->values();
    }

    /**
     * Resuelve el nombre de clase (corto o FQCN) a un FQCN valido.
     *
     * @throws \InvalidArgumentException
     */
    private function resolverClase(string $model): string
    {
        // Si ya es un FQCN valido y esta en los soportados
        if (class_exists($model) && in_array($model, self::MODELOS_SOPORTADOS, true)) {
            return $model;
        }

        // Buscar por nombre corto
        if (isset(self::MODELOS_SOPORTADOS[$model])) {
            return self::MODELOS_SOPORTADOS[$model];
        }

        // Buscar por class_basename
        foreach (self::MODELOS_SOPORTADOS as $clase) {
            if (class_basename($clase) === $model) {
                return $clase;
            }
        }

        $disponibles = implode(', ', array_keys(self::MODELOS_SOPORTADOS));
        throw new \InvalidArgumentException(
            "Modelo '{$model}' no soportado para busqueda semantica. Modelos disponibles: {$disponibles}"
        );
    }
}
```

### 4. Registrar el servicio en el contenedor

```php
// app/Providers/AppServiceProvider.php — dentro del metodo register()

use App\Services\EmbeddingService;
use App\Services\SemanticSearchService;

public function register(): void
{
    $this->app->singleton(EmbeddingService::class);
    $this->app->singleton(SemanticSearchService::class);
}
```

### 5. Probar manualmente con Tinker

```bash
sail artisan tinker
```

```php
// En Tinker:
$service = app(App\Services\SemanticSearchService::class);

// Buscar metas ODS similares a "reducir la pobreza"
$resultados = $service->findSimilar(
    texto:     'reducir la pobreza extrema en zonas rurales',
    model:     'OdsMeta',
    limit:     5,
    threshold: 0.5,
);

foreach ($resultados as $r) {
    echo sprintf(
        "[%.1f%%] %s - %s\n",
        $r->scorePercent(),
        class_basename($r->modelClass),
        $r->label()
    );
}

// Busqueda multi-modelo
$multiResultados = $service->findSimilarMultiple(
    texto:           'seguridad publica y reduccion de violencia',
    modelos:         ['OdsMeta', 'PndObjetivo', 'PedObjetivoEstrategico'],
    limitPorModelo:  3,
    threshold:       0.6,
);

foreach ($multiResultados as $r) {
    echo sprintf("[%.1f%%] (%s) %s\n", $r->scorePercent(), class_basename($r->modelClass), $r->label());
}
```

### 6. Tests

```bash
sail artisan make:test Services/SemanticSearchServiceTest
```

```php
<?php
// tests/Feature/Services/SemanticSearchServiceTest.php

namespace Tests\Feature\Services;

use App\DTOs\SemanticSearchResult;
use App\Models\OdsMeta;
use App\Services\EmbeddingService;
use App\Services\SemanticSearchService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\DB;
use Mockery;
use Tests\TestCase;

class SemanticSearchServiceTest extends TestCase
{
    use RefreshDatabase;

    private SemanticSearchService $service;
    private $embeddingMock;

    protected function setUp(): void
    {
        parent::setUp();

        $this->embeddingMock = Mockery::mock(EmbeddingService::class);
        $this->service       = new SemanticSearchService($this->embeddingMock);
    }

    /**
     * Test principal: buscar "reducir pobreza" retorna ODS 1 como primer resultado.
     * Simula la situacion real donde ODS 1 tiene embedding cercano al texto de consulta.
     */
    public function test_find_similar_retorna_ods_1_para_consulta_de_pobreza(): void
    {
        // Crear dos metas ODS con diferentes embeddings en la DB
        // ODS 1.1: "Erradicar la pobreza extrema" - deberia tener alta similitud
        $odsPobrezaMeta = OdsMeta::factory()->create([
            'codigo'      => '1.1',
            'descripcion' => 'Erradicar la pobreza extrema para todas las personas',
        ]);

        // ODS 13.1: "Medidas de adaptacion climatica" - baja similitud con pobreza
        $odsClimaMeta = OdsMeta::factory()->create([
            'codigo'      => '13.1',
            'descripcion' => 'Fortalecer la resiliencia y capacidad de adaptacion al cambio climatico',
        ]);

        // Vector de alta similitud (cercano al query)
        $vectorPobrezaStr = '[' . implode(',', array_fill(0, 1536, 0.9)) . ']';
        // Vector de baja similitud (alejado del query)
        $vectorClimaStr = '[' . implode(',', array_fill(0, 1536, 0.1)) . ']';

        // Guardar embeddings directamente en DB
        DB::statement(
            "UPDATE ods_metas SET embedding = ?::vector WHERE id = ?",
            [$vectorPobrezaStr, $odsPobrezaMeta->id]
        );
        DB::statement(
            "UPDATE ods_metas SET embedding = ?::vector WHERE id = ?",
            [$vectorClimaStr, $odsClimaMeta->id]
        );

        // Vector de query: muy cercano al vector de pobreza
        $queryVector = array_fill(0, 1536, 0.89);

        $this->embeddingMock
            ->shouldReceive('generate')
            ->once()
            ->with('reducir pobreza')
            ->andReturn($queryVector);

        $this->embeddingMock
            ->shouldReceive('toVectorString')
            ->once()
            ->andReturn('[' . implode(',', $queryVector) . ']');

        $resultados = $this->service->findSimilar(
            texto:     'reducir pobreza',
            model:     'OdsMeta',
            limit:     5,
            threshold: 0.3, // umbral bajo para capturar ambos en el test
        );

        $this->assertNotEmpty($resultados);
        $this->assertInstanceOf(SemanticSearchResult::class, $resultados->first());

        // El primer resultado debe ser ODS 1.1 (pobreza)
        $primero = $resultados->first();
        $this->assertEquals($odsPobrezaMeta->id, $primero->model->id);
        $this->assertGreaterThan(0.5, $primero->score);

        // Verificar orden: score del primero >= score del segundo
        if ($resultados->count() > 1) {
            $this->assertGreaterThanOrEqual(
                $resultados->get(1)->score,
                $resultados->first()->score
            );
        }
    }

    public function test_find_similar_retorna_collection_vacia_sin_resultados_sobre_umbral(): void
    {
        OdsMeta::factory()->create(['descripcion' => 'Meta con embedding bajo']);

        $queryVector = array_fill(0, 1536, 0.5);
        $vectorBajo  = array_fill(0, 1536, -0.5); // opuesto: similitud negativa

        DB::statement(
            "UPDATE ods_metas SET embedding = ?::vector",
            ['[' . implode(',', $vectorBajo) . ']']
        );

        $this->embeddingMock->shouldReceive('generate')->once()->andReturn($queryVector);
        $this->embeddingMock->shouldReceive('toVectorString')->once()
            ->andReturn('[' . implode(',', $queryVector) . ']');

        $resultados = $this->service->findSimilar(
            texto:     'seguridad publica',
            model:     'OdsMeta',
            limit:     5,
            threshold: 0.9, // umbral muy alto
        );

        $this->assertTrue($resultados->isEmpty());
    }

    public function test_find_similar_lanza_excepcion_con_modelo_no_soportado(): void
    {
        $this->embeddingMock->shouldReceive('generate')->andReturn(array_fill(0, 1536, 0.1));
        $this->embeddingMock->shouldReceive('toVectorString')->andReturn('[...]');

        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessageMatches('/no soportado/');

        $this->service->findSimilar('texto', 'ModeloInventado');
    }

    public function test_find_similar_acepta_fqcn_como_nombre_de_modelo(): void
    {
        OdsMeta::factory()->create(['descripcion' => 'Meta ODS']);
        $queryVector    = array_fill(0, 1536, 0.5);
        $vectorStr      = '[' . implode(',', $queryVector) . ']';

        DB::statement("UPDATE ods_metas SET embedding = ?::vector", [$vectorStr]);

        $this->embeddingMock->shouldReceive('generate')->once()->andReturn($queryVector);
        $this->embeddingMock->shouldReceive('toVectorString')->once()->andReturn($vectorStr);

        // Debe funcionar con FQCN
        $resultados = $this->service->findSimilar(
            texto:     'pobreza',
            model:     \App\Models\OdsMeta::class,
            threshold: 0.0,
        );

        $this->assertInstanceOf(\Illuminate\Support\Collection::class, $resultados);
    }

    public function test_resultado_tiene_score_entre_0_y_1(): void
    {
        $meta = OdsMeta::factory()->create(['descripcion' => 'Meta de prueba']);
        $vec  = array_fill(0, 1536, 0.7);
        $vecStr = '[' . implode(',', $vec) . ']';

        DB::statement("UPDATE ods_metas SET embedding = ?::vector WHERE id = ?", [$vecStr, $meta->id]);

        $this->embeddingMock->shouldReceive('generate')->once()->andReturn($vec);
        $this->embeddingMock->shouldReceive('toVectorString')->once()->andReturn($vecStr);

        $resultados = $this->service->findSimilar('texto', 'OdsMeta', threshold: 0.0);

        foreach ($resultados as $resultado) {
            $this->assertGreaterThanOrEqual(0.0, $resultado->score);
            $this->assertLessThanOrEqual(1.0, $resultado->score);
        }
    }

    public function test_find_similar_multiple_llama_api_una_sola_vez(): void
    {
        // La API solo debe llamarse UNA vez aunque se busque en multiples modelos
        $this->embeddingMock
            ->shouldReceive('generate')
            ->once() // <-- solo una vez
            ->andReturn(array_fill(0, 1536, 0.5));

        $this->embeddingMock
            ->shouldReceive('toVectorString')
            ->once()
            ->andReturn('[' . implode(',', array_fill(0, 1536, 0.5)) . ']');

        $this->service->findSimilarMultiple(
            texto:          'pobreza',
            modelos:        ['OdsMeta', 'PndObjetivo', 'PedLineaAccion'],
            limitPorModelo: 3,
            threshold:      0.9,
        );
    }

    public function test_score_percent_retorna_entero_entre_0_y_100(): void
    {
        $meta     = OdsMeta::factory()->create();
        $resultado = new SemanticSearchResult($meta, 0.856, OdsMeta::class);

        $this->assertEquals(86, $resultado->scorePercent());
    }

    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }
}
```

```bash
sail artisan test --filter=SemanticSearchServiceTest
```

### 7. Ruta y controlador de busqueda (para busqueda publica desde la UI)

```bash
sail artisan make:livewire Busqueda/BusquedaSemantica
```

```php
<?php
// app/Livewire/Busqueda/BusquedaSemantica.php

namespace App\Livewire\Busqueda;

use App\Services\SemanticSearchService;
use Illuminate\Support\Collection;
use Livewire\Component;

class BusquedaSemantica extends Component
{
    public string $consulta  = '';
    public string $modeloFiltro = 'todos';
    public float  $umbral    = 0.7;
    public int    $limite    = 10;

    public Collection $resultados;
    public bool $buscando  = false;
    public bool $buscado   = false;
    public ?string $error  = null;

    public array $modelosDisponibles = [
        'todos'                    => 'Todos los instrumentos',
        'OdsMeta'                  => 'Metas ODS',
        'PndObjetivo'              => 'Objetivos PND',
        'PedObjetivoEstrategico'   => 'Objetivos PED',
        'PedLineaAccion'           => 'Lineas de Accion PED',
        'ProgramaDerivadoObjetivo' => 'Objetivos de Programas Derivados',
    ];

    public function mount(): void
    {
        $this->resultados = collect();
    }

    public function buscar(SemanticSearchService $searchService): void
    {
        $this->validate([
            'consulta' => 'required|string|min:3|max:500',
            'umbral'   => 'numeric|min:0|max:1',
            'limite'   => 'integer|min:1|max:50',
        ]);

        $this->buscando  = true;
        $this->error     = null;

        try {
            if ($this->modeloFiltro === 'todos') {
                $modelos = array_keys(SemanticSearchService::MODELOS_SOPORTADOS);
                $this->resultados = $searchService->findSimilarMultiple(
                    texto:          $this->consulta,
                    modelos:        $modelos,
                    limitPorModelo: (int) ceil($this->limite / count($modelos)),
                    threshold:      $this->umbral,
                );
            } else {
                $this->resultados = $searchService->findSimilar(
                    texto:     $this->consulta,
                    model:     $this->modeloFiltro,
                    limit:     $this->limite,
                    threshold: $this->umbral,
                );
            }
        } catch (\InvalidArgumentException $e) {
            $this->error = 'Modelo de busqueda no valido: ' . $e->getMessage();
        } catch (\RuntimeException $e) {
            $this->error = 'Error al procesar la busqueda. Por favor intenta de nuevo.';
            \Log::error('BusquedaSemantica Livewire error: ' . $e->getMessage());
        } finally {
            $this->buscando = false;
            $this->buscado  = true;
        }
    }

    public function render()
    {
        return view('livewire.busqueda.busqueda-semantica');
    }
}
```

```blade
{{-- resources/views/livewire/busqueda/busqueda-semantica.blade.php --}}
<div class="max-w-4xl mx-auto">
    <h1 class="text-2xl font-bold text-gray-900 mb-6">Busqueda Semantica</h1>

    <form wire:submit="buscar" class="bg-white rounded-xl shadow p-6 mb-6">
        <div class="flex gap-3">
            <input wire:model="consulta"
                   type="text"
                   placeholder="Describe lo que buscas... ej: 'reducir la desigualdad economica'"
                   class="flex-1 border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
            <button type="submit"
                    wire:loading.attr="disabled"
                    class="px-5 py-2 bg-indigo-600 text-white text-sm font-medium rounded-lg hover:bg-indigo-700 disabled:opacity-50">
                <span wire:loading.remove>Buscar</span>
                <span wire:loading>Buscando...</span>
            </button>
        </div>
        @error('consulta') <p class="text-red-500 text-xs mt-1">{{ $message }}</p> @enderror

        <div class="flex gap-4 mt-3">
            <div>
                <label class="text-xs text-gray-500">Instrumento</label>
                <select wire:model="modeloFiltro"
                        class="block mt-1 border-gray-300 rounded text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    @foreach($modelosDisponibles as $valor => $etiqueta)
                        <option value="{{ $valor }}">{{ $etiqueta }}</option>
                    @endforeach
                </select>
            </div>
            <div>
                <label class="text-xs text-gray-500">Similitud minima</label>
                <input wire:model="umbral" type="number" step="0.05" min="0" max="1"
                       class="block mt-1 w-20 border-gray-300 rounded text-sm focus:ring-indigo-500 focus:border-indigo-500">
            </div>
            <div>
                <label class="text-xs text-gray-500">Max resultados</label>
                <input wire:model="limite" type="number" min="1" max="50"
                       class="block mt-1 w-16 border-gray-300 rounded text-sm focus:ring-indigo-500 focus:border-indigo-500">
            </div>
        </div>
    </form>

    @if($error)
        <div class="bg-red-50 border border-red-200 text-red-700 rounded-lg p-4 mb-4 text-sm">
            {{ $error }}
        </div>
    @endif

    @if($buscado)
        <div class="mb-2 text-sm text-gray-500">
            {{ $resultados->count() }} resultado(s) para "<strong>{{ $consulta }}</strong>"
            con similitud >= {{ number_format($umbral * 100, 0) }}%
        </div>

        @if($resultados->isEmpty())
            <div class="bg-white rounded-xl shadow p-12 text-center text-gray-400">
                <p class="text-base font-medium">Sin resultados</p>
                <p class="text-sm mt-1">Prueba bajando el umbral de similitud o usando otras palabras.</p>
            </div>
        @else
            <div class="space-y-3">
                @foreach($resultados as $resultado)
                    <div class="bg-white rounded-xl shadow p-4 flex items-start gap-4">
                        {{-- Score badge --}}
                        <div class="flex-shrink-0 w-14 h-14 rounded-full flex items-center justify-center
                                    {{ $resultado->score >= 0.85
                                        ? 'bg-green-100 text-green-700'
                                        : ($resultado->score >= 0.70
                                            ? 'bg-yellow-100 text-yellow-700'
                                            : 'bg-gray-100 text-gray-600') }}">
                            <span class="text-sm font-bold">{{ $resultado->scorePercent() }}%</span>
                        </div>

                        {{-- Contenido --}}
                        <div class="flex-1 min-w-0">
                            <p class="text-xs font-semibold text-indigo-600 uppercase tracking-wide mb-1">
                                {{ class_basename($resultado->modelClass) }}
                            </p>
                            <p class="text-sm font-medium text-gray-900">{{ $resultado->label() }}</p>
                            @if($resultado->model->getAttribute('descripcion') && $resultado->model->getAttribute('nombre'))
                                <p class="text-xs text-gray-500 mt-1 line-clamp-2">
                                    {{ $resultado->model->getAttribute('descripcion') }}
                                </p>
                            @endif
                        </div>
                    </div>
                @endforeach
            </div>
        @endif
    @endif
</div>
```

### 8. Registrar ruta

```php
// routes/web.php

use App\Livewire\Busqueda\BusquedaSemantica;

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/busqueda/semantica', BusquedaSemantica::class)
        ->name('busqueda.semantica');
});
```

---

## Criterios de Aceptacion

- [ ] Migracion de indices HNSW ejecuta sin errores con `sail artisan migrate`.
- [ ] Los 5 indices HNSW aparecen en `pg_indexes` con `indexname LIKE 'idx_hnsw_%'`.
- [ ] `SemanticSearchService::findSimilar` retorna una `Collection` ordenada por `score DESC`.
- [ ] Con `threshold: 0.7`, solo se retornan resultados con similitud >= 0.7.
- [ ] Funciona contra los 5 modelos soportados: `OdsMeta`, `PndObjetivo`, `PedObjetivoEstrategico`, `PedLineaAccion`, `ProgramaDerivadoObjetivo`.
- [ ] `findSimilarMultiple` llama a la API de embeddings exactamente una vez (no una por modelo).
- [ ] Test: buscar "reducir pobreza" retorna ODS 1 como primer resultado (con mocks).
- [ ] Un modelo no soportado lanza `InvalidArgumentException` con mensaje descriptivo.
- [ ] El umbral y el limite son configurables en cada llamada.
- [ ] Todos los tests del archivo `SemanticSearchServiceTest` pasan en verde sin llamadas reales a la API.

---

## Notas

- **`ef_search`:** Para ajustar la precision en tiempo de query (no de construccion), ejecutar `SET hnsw.ef_search = 100;` en la sesion antes del query. Valor por defecto de pgvector es 40. Aumentar si se necesitan resultados mas precisos a costo de velocidad.

  ```php
  // En SemanticSearchService::findSimilar, antes del DB::select:
  DB::statement('SET hnsw.ef_search = 100');
  ```

- **`findSimilarMultiple` y la API:** El metodo llama a `EmbeddingService::generate` una sola vez y reutiliza el vector serializado para todas las queries SQL. Esto es esencial para no multiplicar el costo de API.

- **Indices parciales:** Para tablas con muchos registros sin embedding (ej: durante la carga inicial), considerar indices parciales: `WHERE embedding IS NOT NULL`. Esto reduce el tamano del indice y acelera la construccion.

  ```sql
  CREATE INDEX idx_hnsw_ods_metas_embedding
  ON ods_metas USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64)
  WHERE embedding IS NOT NULL;
  ```

- **Similitud vs Distancia:** pgvector siempre retorna distancias. La conversion `1 - distancia_coseno` funciona correctamente para vectores normalizados (como los que produce OpenAI `text-embedding-ada-002`). Para modelos que producen vectores no normalizados, usar `<#>` (producto interno) puede ser mas apropiado.

- **Seguridad:** Las queries usan `DB::select` con parametros vinculados (`?`), no interpolacion de cadenas, por lo que no son vulnerables a SQL injection. El `$tabla` y `$pk` se obtienen del modelo Eloquent (confiables), no de input del usuario.

- **S2-T8 integracion:** `MatrizAlineacion` llama a `SemanticSearchService` capturando `\Throwable`, de modo que si el servicio no esta disponible (ej: API caida), la UI sigue funcionando sin sugerencias.
