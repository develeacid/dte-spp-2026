# S2-T10 — Pipeline de Generacion de Embeddings

**Tipo:** feat
**Rama:** `feat/S2-T10-pipeline-embeddings`
**Depende de:** S0-T2 (pgvector instalado), S0-T4 (Redis queues configurado)

---

## Contexto

Todos los modelos de planes (ODS, PND, PED, Programas Derivados) necesitan vectores de embedding para poder realizar busqueda semantica (S2-T11). El pipeline consiste en:

1. **EmbeddingService** — encapsula la llamada HTTP a la API de embeddings (OpenAI o cualquier compatible con el mismo contrato).
2. **GenerateEmbedding** — Job despachado a cola Redis que llama al servicio y persiste el vector.
3. **Observers** — Escuchan `created` y `updated` en los modelos y despachan el Job si `descripcion` o `nombre` cambiaron.

El vector se guarda con raw SQL para aprovechar el cast de pgvector: `UPDATE tabla SET embedding = ?::vector WHERE id = ?`.

---

## Pre-requisitos

Verificar Redis y colas:

```bash
sail artisan tinker --execute="
    \Redis::ping();
    echo 'Redis OK';
"

# Verificar que QUEUE_CONNECTION=redis en .env
grep QUEUE_CONNECTION .env
```

Verificar extension pgvector activa:

```bash
sail psql -U "${DB_USERNAME}" -d "${DB_DATABASE}" -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
```

Verificar que las columnas `embedding vector(1536)` existen en las tablas target:

```bash
sail psql -U "${DB_USERNAME}" -d "${DB_DATABASE}" -c "
    SELECT table_name, column_name, data_type
    FROM information_schema.columns
    WHERE column_name = 'embedding'
    ORDER BY table_name;
"
```

---

## Pasos

### 1. Variables de entorno

Agregar a `.env` y a `.env.example`:

```env
EMBEDDING_API_KEY=sk-...
EMBEDDING_API_URL=https://api.openai.com/v1/embeddings
EMBEDDING_MODEL=text-embedding-ada-002
EMBEDDING_DIMENSIONS=1536
EMBEDDING_QUEUE=embeddings
```

Agregar a `config/services.php`:

```php
'embeddings' => [
    'api_key'    => env('EMBEDDING_API_KEY'),
    'api_url'    => env('EMBEDDING_API_URL', 'https://api.openai.com/v1/embeddings'),
    'model'      => env('EMBEDDING_MODEL', 'text-embedding-ada-002'),
    'dimensions' => (int) env('EMBEDDING_DIMENSIONS', 1536),
    'queue'      => env('EMBEDDING_QUEUE', 'embeddings'),
],
```

### 2. Crear EmbeddingService

```bash
sail artisan make:class Services/EmbeddingService
```

```php
<?php
// app/Services/EmbeddingService.php

namespace App\Services;

use Illuminate\Http\Client\ConnectionException;
use Illuminate\Http\Client\RequestException;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class EmbeddingService
{
    private string $apiKey;
    private string $apiUrl;
    private string $model;
    private int    $dimensions;

    public function __construct()
    {
        $this->apiKey     = config('services.embeddings.api_key');
        $this->apiUrl     = config('services.embeddings.api_url');
        $this->model      = config('services.embeddings.model');
        $this->dimensions = config('services.embeddings.dimensions');
    }

    /**
     * Genera un embedding para el texto dado.
     *
     * @param  string $texto  Texto a vectorizar.
     * @return float[]        Array de floats con dimension $this->dimensions (1536 por defecto).
     *
     * @throws \RuntimeException Si la API falla o retorna datos invalidos.
     */
    public function generate(string $texto): array
    {
        if (blank($texto)) {
            throw new \RuntimeException('El texto para generar embedding no puede estar vacio.');
        }

        // Truncar a ~8000 tokens (aproximadamente 32000 caracteres) para evitar errores 400
        $texto = mb_substr($texto, 0, 32000);

        $response = Http::withToken($this->apiKey)
            ->timeout(30)
            ->retry(2, 1000, function (\Throwable $e) {
                // Solo reintentar en errores de conexion o 429 (rate limit)
                return $e instanceof ConnectionException
                    || ($e instanceof RequestException && $e->response->status() === 429);
            })
            ->post($this->apiUrl, [
                'input'           => $texto,
                'model'           => $this->model,
                'encoding_format' => 'float',
            ]);

        if ($response->failed()) {
            $status = $response->status();
            $body   = $response->body();
            Log::error("EmbeddingService: API respondio con HTTP {$status}", ['body' => $body]);
            throw new \RuntimeException("API de embeddings respondio con HTTP {$status}: {$body}");
        }

        $data = $response->json();

        if (! isset($data['data'][0]['embedding']) || ! is_array($data['data'][0]['embedding'])) {
            Log::error('EmbeddingService: respuesta inesperada de la API', ['response' => $data]);
            throw new \RuntimeException('La API de embeddings devolvio una respuesta con formato inesperado.');
        }

        $vector = $data['data'][0]['embedding'];

        if (count($vector) !== $this->dimensions) {
            throw new \RuntimeException(
                "Dimensiones inesperadas: se esperaban {$this->dimensions}, se recibieron " . count($vector)
            );
        }

        return $vector;
    }

    /**
     * Convierte un array de floats al formato string de pgvector.
     * Ejemplo: [0.1, 0.2, 0.3] -> "[0.1,0.2,0.3]"
     */
    public function toVectorString(array $embedding): string
    {
        return '[' . implode(',', $embedding) . ']';
    }
}
```

### 3. Crear el Job GenerateEmbedding

```bash
sail artisan make:job GenerateEmbedding
```

```php
<?php
// app/Jobs/GenerateEmbedding.php

namespace App\Jobs;

use App\Services\EmbeddingService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class GenerateEmbedding implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Numero de intentos antes de marcar como fallido.
     */
    public int $tries = 3;

    /**
     * Segundos de espera entre intentos (backoff exponencial).
     * Intento 1 -> 60s, Intento 2 -> 300s
     */
    public array $backoff = [60, 300];

    /**
     * Timeout por intento en segundos.
     */
    public int $timeout = 60;

    /**
     * @param  Model  $model     La instancia del modelo Eloquent.
     * @param  string $campo     Campo cuyo texto se usara para generar el embedding.
     */
    public function __construct(
        public readonly Model $model,
        public readonly string $campo = 'descripcion',
    ) {}

    public function handle(EmbeddingService $embeddingService): void
    {
        // Re-fetch fresco del modelo para evitar datos stale de la serializacion
        $modelo = $this->model->fresh();

        if (! $modelo) {
            Log::warning('GenerateEmbedding: modelo ya no existe en DB, saltando.', [
                'class' => get_class($this->model),
                'id'    => $this->model->getKey(),
            ]);
            return;
        }

        $texto = $this->extraerTexto($modelo);

        if (blank($texto)) {
            Log::info('GenerateEmbedding: campo vacio, no se genera embedding.', [
                'class'  => get_class($modelo),
                'id'     => $modelo->getKey(),
                'campo'  => $this->campo,
            ]);
            return;
        }

        try {
            $vector          = $embeddingService->generate($texto);
            $vectorString    = $embeddingService->toVectorString($vector);
            $tabla           = $modelo->getTable();
            $primaryKeyName  = $modelo->getKeyName();

            DB::statement(
                "UPDATE {$tabla} SET embedding = ?::vector WHERE {$primaryKeyName} = ?",
                [$vectorString, $modelo->getKey()]
            );

            Log::info('GenerateEmbedding: embedding guardado.', [
                'class' => get_class($modelo),
                'id'    => $modelo->getKey(),
            ]);
        } catch (\Throwable $e) {
            Log::error('GenerateEmbedding: fallo al generar o guardar embedding.', [
                'class'     => get_class($modelo),
                'id'        => $modelo->getKey(),
                'error'     => $e->getMessage(),
                'intento'   => $this->attempts(),
            ]);

            // Relanzar para que la cola reintente segun $backoff
            throw $e;
        }
    }

    /**
     * Callback cuando el job agota todos los intentos.
     */
    public function failed(\Throwable $exception): void
    {
        Log::error('GenerateEmbedding: agoto todos los intentos, embedding NO guardado.', [
            'class' => get_class($this->model),
            'id'    => $this->model->getKey(),
            'error' => $exception->getMessage(),
        ]);
    }

    /**
     * Construye el texto a vectorizar combinando campos relevantes del modelo.
     */
    private function extraerTexto(Model $modelo): string
    {
        $partes = [];

        // Siempre incluir nombre/titulo si existe
        foreach (['nombre', 'titulo', 'title'] as $campoNombre) {
            if ($modelo->getAttribute($campoNombre)) {
                $partes[] = $modelo->getAttribute($campoNombre);
                break;
            }
        }

        // Incluir el campo principal configurado
        if ($modelo->getAttribute($this->campo)) {
            $partes[] = $modelo->getAttribute($this->campo);
        }

        return implode('. ', array_filter($partes));
    }
}
```

### 4. Crear el Observer base

```bash
sail artisan make:observer EmbeddingObserver --model=OdsMeta
```

```php
<?php
// app/Observers/EmbeddingObserver.php

namespace App\Observers;

use App\Jobs\GenerateEmbedding;
use Illuminate\Database\Eloquent\Model;

/**
 * Observer reutilizable para todos los modelos con embedding.
 * Despacha GenerateEmbedding cuando se crea o actualiza
 * un campo semanticamente relevante.
 */
class EmbeddingObserver
{
    /**
     * Campos que, al cambiar, deben disparar una regeneracion de embedding.
     */
    private array $camposRelevantes = ['nombre', 'descripcion', 'titulo', 'texto'];

    public function created(Model $model): void
    {
        $this->despacharSiProcede($model);
    }

    public function updated(Model $model): void
    {
        // Solo despachar si cambio un campo semanticamente relevante
        $camposCambiados = array_keys($model->getDirty());
        $relevantes      = array_intersect($camposCambiados, $this->camposRelevantes);

        if (count($relevantes) > 0) {
            $this->despacharSiProcede($model);
        }
    }

    private function despacharSiProcede(Model $model): void
    {
        $cola = config('services.embeddings.queue', 'embeddings');

        // Campo preferido para texto de embedding
        $campo = 'descripcion';
        if (! $model->getAttribute('descripcion') && $model->getAttribute('nombre')) {
            $campo = 'nombre';
        }

        GenerateEmbedding::dispatch($model, $campo)
            ->onQueue($cola)
            ->delay(now()->addSeconds(5)); // Pequena demora para evitar race conditions
    }
}
```

### 5. Registrar Observer para todos los modelos target

```php
<?php
// app/Providers/AppServiceProvider.php

namespace App\Providers;

use App\Models\OdsMeta;
use App\Models\OdsObjetivo;
use App\Models\PedEje;
use App\Models\PedEstrategia;
use App\Models\PedLineaAccion;
use App\Models\PedObjetivoEstrategico;
use App\Models\PedTema;
use App\Models\PndEje;
use App\Models\PndEstrategia;
use App\Models\PndObjetivo;
use App\Models\ProgramaDerivadoObjetivo;
use App\Observers\EmbeddingObserver;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Modelos ODS
        OdsObjetivo::observe(EmbeddingObserver::class);
        OdsMeta::observe(EmbeddingObserver::class);

        // Modelos PND
        PndEje::observe(EmbeddingObserver::class);
        PndObjetivo::observe(EmbeddingObserver::class);
        PndEstrategia::observe(EmbeddingObserver::class);

        // Modelos PED
        PedEje::observe(EmbeddingObserver::class);
        PedTema::observe(EmbeddingObserver::class);
        PedObjetivoEstrategico::observe(EmbeddingObserver::class);
        PedEstrategia::observe(EmbeddingObserver::class);
        PedLineaAccion::observe(EmbeddingObserver::class);

        // Programas Derivados
        ProgramaDerivadoObjetivo::observe(EmbeddingObserver::class);
    }
}
```

### 6. Arrancar el worker de colas en Sail

Para desarrollo:

```bash
sail artisan queue:work redis --queue=embeddings --tries=3 --backoff=60 --timeout=60
```

Para produccion (en `Supervisor` dentro del contenedor o en `docker-compose`):

```ini
# /etc/supervisor/conf.d/queue-embeddings.conf
[program:queue-embeddings]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work redis --queue=embeddings --tries=3 --backoff=60 --timeout=60 --sleep=3 --max-jobs=500
autostart=true
autorestart=true
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/queue-embeddings.log
```

Verificar que el job se envia correctamente:

```bash
sail artisan tinker --execute="
    \$modelo = App\Models\OdsMeta::first();
    if (\$modelo) {
        App\Jobs\GenerateEmbedding::dispatch(\$modelo, 'descripcion')->onQueue('embeddings');
        echo 'Job despachado para OdsMeta ID: ' . \$modelo->id . PHP_EOL;
    } else {
        echo 'No hay registros de OdsMeta para probar.';
    }
"
```

Monitorear la cola:

```bash
sail artisan queue:monitor redis:embeddings
```

### 7. Comando Artisan para re-generar embeddings en batch

```bash
sail artisan make:command GenerateAllEmbeddings
```

```php
<?php
// app/Console/Commands/GenerateAllEmbeddings.php

namespace App\Console\Commands;

use App\Jobs\GenerateEmbedding;
use Illuminate\Console\Command;
use Illuminate\Database\Eloquent\Model;

class GenerateAllEmbeddings extends Command
{
    protected $signature   = 'embeddings:generate-all {--model= : Clase del modelo (ej: OdsMeta)} {--chunk=100}';
    protected $description = 'Despacha jobs de embedding para todos los registros sin vector o para un modelo especifico.';

    private array $modelos = [
        'OdsObjetivo'            => \App\Models\OdsObjetivo::class,
        'OdsMeta'                => \App\Models\OdsMeta::class,
        'PndEje'                 => \App\Models\PndEje::class,
        'PndObjetivo'            => \App\Models\PndObjetivo::class,
        'PndEstrategia'          => \App\Models\PndEstrategia::class,
        'PedEje'                 => \App\Models\PedEje::class,
        'PedTema'                => \App\Models\PedTema::class,
        'PedObjetivoEstrategico' => \App\Models\PedObjetivoEstrategico::class,
        'PedEstrategia'          => \App\Models\PedEstrategia::class,
        'PedLineaAccion'         => \App\Models\PedLineaAccion::class,
        'ProgramaDerivadoObjetivo' => \App\Models\ProgramaDerivadoObjetivo::class,
    ];

    public function handle(): int
    {
        $soloModelo = $this->option('model');
        $chunk      = (int) $this->option('chunk');
        $cola       = config('services.embeddings.queue', 'embeddings');

        $modelosAProcesar = $soloModelo
            ? array_filter($this->modelos, fn ($c) => class_basename($c) === $soloModelo)
            : $this->modelos;

        if (empty($modelosAProcesar)) {
            $this->error("Modelo '{$soloModelo}' no reconocido. Modelos disponibles: " . implode(', ', array_keys($this->modelos)));
            return self::FAILURE;
        }

        foreach ($modelosAProcesar as $nombre => $clase) {
            $this->info("Procesando {$nombre}...");

            /** @var \Illuminate\Database\Eloquent\Builder $query */
            $query = $clase::query()->whereNull('embedding');

            $total = $query->count();
            $this->line("  {$total} registros sin embedding.");

            $bar = $this->output->createProgressBar($total);
            $bar->start();

            $query->chunkById($chunk, function ($registros) use ($cola, $bar) {
                foreach ($registros as $registro) {
                    GenerateEmbedding::dispatch($registro, 'descripcion')
                        ->onQueue($cola);
                    $bar->advance();
                }
            });

            $bar->finish();
            $this->newLine();
            $this->info("  Jobs despachados para {$nombre}.");
        }

        $this->info('Todos los jobs han sido despachados a la cola "' . $cola . '".');
        return self::SUCCESS;
    }
}
```

Uso:

```bash
# Re-generar todos los que no tienen embedding
sail artisan embeddings:generate-all

# Solo un modelo especifico
sail artisan embeddings:generate-all --model=OdsMeta

# Procesar en bloques de 50
sail artisan embeddings:generate-all --chunk=50
```

### 8. Tests

```bash
sail artisan make:test Services/EmbeddingServiceTest --unit
sail artisan make:test Jobs/GenerateEmbeddingTest
```

```php
<?php
// tests/Unit/Services/EmbeddingServiceTest.php

namespace Tests\Unit\Services;

use App\Services\EmbeddingService;
use Illuminate\Http\Client\Request;
use Illuminate\Support\Facades\Http;
use Tests\TestCase;

class EmbeddingServiceTest extends TestCase
{
    public function test_generate_retorna_array_de_1536_floats(): void
    {
        $vectorFalso = array_fill(0, 1536, 0.001);

        Http::fake([
            'api.openai.com/v1/embeddings' => Http::response([
                'data' => [['embedding' => $vectorFalso]],
                'model' => 'text-embedding-ada-002',
            ], 200),
        ]);

        $service = new EmbeddingService();
        $result  = $service->generate('texto de prueba');

        $this->assertIsArray($result);
        $this->assertCount(1536, $result);
        $this->assertIsFloat($result[0]);
    }

    public function test_generate_lanza_excepcion_si_api_falla(): void
    {
        Http::fake([
            'api.openai.com/v1/embeddings' => Http::response(['error' => 'Unauthorized'], 401),
        ]);

        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessageMatches('/HTTP 401/');

        $service = new EmbeddingService();
        $service->generate('texto de prueba');
    }

    public function test_generate_lanza_excepcion_con_texto_vacio(): void
    {
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessageMatches('/vacio/');

        $service = new EmbeddingService();
        $service->generate('');
    }

    public function test_to_vector_string_formatea_correctamente(): void
    {
        $service = new EmbeddingService();
        $result  = $service->toVectorString([0.1, 0.2, 0.3]);

        $this->assertEquals('[0.1,0.2,0.3]', $result);
    }

    public function test_generate_envia_model_correcto_en_request(): void
    {
        config(['services.embeddings.model' => 'text-embedding-ada-002']);

        $vectorFalso = array_fill(0, 1536, 0.001);
        Http::fake([
            '*' => Http::response(['data' => [['embedding' => $vectorFalso]]], 200),
        ]);

        $service = new EmbeddingService();
        $service->generate('texto');

        Http::assertSent(function (Request $request) {
            return $request->data()['model'] === 'text-embedding-ada-002';
        });
    }
}
```

```php
<?php
// tests/Feature/Jobs/GenerateEmbeddingTest.php

namespace Tests\Feature\Jobs;

use App\Jobs\GenerateEmbedding;
use App\Models\OdsMeta;
use App\Services\EmbeddingService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Bus;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Mockery;
use Tests\TestCase;

class GenerateEmbeddingTest extends TestCase
{
    use RefreshDatabase;

    public function test_job_guarda_embedding_en_db(): void
    {
        $meta = OdsMeta::factory()->create(['descripcion' => 'Erradicar la pobreza extrema']);

        $vectorFalso      = array_fill(0, 1536, 0.001);
        $embeddingMock    = Mockery::mock(EmbeddingService::class);
        $embeddingMock->shouldReceive('generate')->once()->andReturn($vectorFalso);
        $embeddingMock->shouldReceive('toVectorString')
            ->once()
            ->andReturn('[' . implode(',', $vectorFalso) . ']');

        app()->instance(EmbeddingService::class, $embeddingMock);

        (new GenerateEmbedding($meta, 'descripcion'))->handle($embeddingMock);

        // Verificar que el embedding fue guardado (no null)
        $raw = DB::selectOne("SELECT embedding IS NOT NULL as tiene FROM ods_metas WHERE id = ?", [$meta->id]);
        $this->assertTrue((bool) $raw->tiene);
    }

    public function test_job_es_despachado_al_crear_modelo(): void
    {
        Bus::fake();

        OdsMeta::factory()->create(['descripcion' => 'Meta de prueba']);

        Bus::assertDispatched(GenerateEmbedding::class);
    }

    public function test_job_no_bloquea_si_modelo_fue_eliminado(): void
    {
        $meta = OdsMeta::factory()->create();
        $job  = new GenerateEmbedding($meta, 'descripcion');

        $meta->delete();

        $embeddingMock = Mockery::mock(EmbeddingService::class);
        $embeddingMock->shouldNotReceive('generate');

        // No debe lanzar excepcion
        $job->handle($embeddingMock);

        $this->assertTrue(true); // Si llegamos aqui, no hubo excepcion
    }

    public function test_job_registra_en_log_y_relanza_si_api_falla(): void
    {
        Log::spy();

        $meta = OdsMeta::factory()->create(['descripcion' => 'Meta test']);

        $embeddingMock = Mockery::mock(EmbeddingService::class);
        $embeddingMock->shouldReceive('generate')
            ->andThrow(new \RuntimeException('API timeout'));

        $job = new GenerateEmbedding($meta, 'descripcion');

        $this->expectException(\RuntimeException::class);

        $job->handle($embeddingMock);

        Log::shouldHaveReceived('error')->once();
    }
}
```

```bash
sail artisan test --filter=EmbeddingServiceTest
sail artisan test --filter=GenerateEmbeddingTest
```

---

## Criterios de Aceptacion

- [ ] `EmbeddingService::generate(string $texto): array` retorna un array con exactamente 1536 floats cuando la API responde correctamente.
- [ ] `GenerateEmbedding` tiene `$tries = 3` y `$backoff = [60, 300]`.
- [ ] Al crear o actualizar `descripcion` en cualquiera de los 11 modelos target, se despacha un job a la cola `embeddings`.
- [ ] El job persiste el vector con `UPDATE tabla SET embedding = ?::vector WHERE id = ?`.
- [ ] Si la API falla: el embedding no se guarda, se registra en `Log::error`, el job se reintenta hasta 3 veces.
- [ ] Si el modelo fue eliminado antes de que el job corra, el job termina sin excepcion.
- [ ] El comando `embeddings:generate-all` despacha jobs solo para registros con `embedding IS NULL`.
- [ ] Todos los tests pasan con mocks (no se hacen llamadas reales a la API en el test suite).

---

## Notas

- **Texto vacio:** Si un modelo tiene `nombre` y `descripcion` ambos nulos o vacios, el job termina silenciosamente sin generar embedding. Esto es intencional para no bloquear la UI.
- **Dimensiones:** Si en el futuro se cambia a un modelo de 3072 dimensiones (text-embedding-3-large), cambiar `EMBEDDING_DIMENSIONS` en `.env` Y ejecutar una nueva migracion que cambie `vector(1536)` a `vector(3072)` en todas las tablas. Los indices HNSW deben reconstruirse.
- **Rate limiting:** La API de OpenAI tiene limites de tokens por minuto. El `$backoff = [60, 300]` asegura que los reintentos respetan el rate limit tipico (RPM reset cada 60s). Ajustar si se usan modelos con limites mas estrictos.
- **Cola dedicada:** Se usa la cola `embeddings` en lugar de `default` para poder escalar el worker independientemente y no bloquear otras operaciones del sistema.
- **`withoutObservers`:** En factories/seeders usar `Model::withoutObservers(fn() => ...)` para no generar embeddings en seed, o usar `GenerateAllEmbeddings` al final del seed.
