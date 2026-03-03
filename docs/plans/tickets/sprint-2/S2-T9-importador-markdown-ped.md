# S2-T9 — Importador de PED desde Markdown

**Tipo:** feat
**Rama:** `feat/S2-T9-importador-markdown-ped`
**Depende de:** S2-T3 (modelos y migraciones del PED)

---

## Contexto

Los documentos del Plan Estatal de Desarrollo son entregados frecuentemente como archivos de texto o Word. Este ticket implementa un importador que acepta un archivo Markdown con una jerarquia de headings bien definida, lo parsea, muestra una previsualizacion editable en arbol y —al confirmar— crea todos los registros en cascada dentro de una transaccion de base de datos.

La jerarquia esperada es:

```
H1  → PedPlan (plan raiz)
H2  → PedEje
H3  → PedTema
H4  → PedObjetivoEstrategico
H5  → PedEstrategia
Li  → PedLineaAccion (lineas con guion "-")
```

---

## Pre-requisitos

- S2-T3 completado: modelos `PedPlan`, `PedEje`, `PedTema`, `PedObjetivoEstrategico`, `PedEstrategia`, `PedLineaAccion` existentes con sus migraciones.
- Permiso `gestionar_catalogos` de S1-T3.
- PHP `fileinfo` extension habilitada (viene por defecto en Sail).

Verificar modelos:

```bash
sail artisan tinker --execute="
    \$clases = [
        App\Models\PedPlan::class,
        App\Models\PedEje::class,
        App\Models\PedTema::class,
        App\Models\PedObjetivoEstrategico::class,
        App\Models\PedEstrategia::class,
        App\Models\PedLineaAccion::class,
    ];
    foreach (\$clases as \$c) { echo \$c . ': ' . \$c::count() . PHP_EOL; }
"
```

---

## Pasos

### 1. Crear la clase Parser

```bash
sail artisan make:class Services/PedMarkdownParser
```

```php
<?php
// app/Services/PedMarkdownParser.php

namespace App\Services;

use Illuminate\Support\Collection;

/**
 * Parsea un archivo Markdown con jerarquia PED y retorna
 * una estructura de arbol lista para previsualizar y confirmar.
 *
 * Formato esperado:
 *   H1  = Plan
 *   H2  = Eje
 *   H3  = Tema
 *   H4  = Objetivo Estrategico
 *   H5  = Estrategia
 *   Li  = Linea de Accion (- texto)
 */
class PedMarkdownParser
{
    /**
     * Errores encontrados durante el parseo.
     * Cada entrada: ['linea' => int, 'mensaje' => string]
     */
    private array $errores = [];

    /**
     * @param  string $contenido Contenido completo del archivo Markdown.
     * @return array  ['plan' => [...], 'errores' => [...]]
     */
    public function parse(string $contenido): array
    {
        $this->errores = [];
        $lineas        = explode("\n", str_replace("\r\n", "\n", $contenido));

        $plan       = null;
        $ejeActual  = null;
        $temaActual = null;
        $objActual  = null;
        $estActual  = null;

        foreach ($lineas as $numeroLinea => $lineaTexto) {
            $numero  = $numeroLinea + 1;
            $trimmed = trim($lineaTexto);

            if ($trimmed === '' || str_starts_with($trimmed, '<!--')) {
                continue;
            }

            // H1 — Plan
            if (preg_match('/^#\s+(.+)$/', $trimmed, $m)) {
                if ($plan !== null) {
                    $this->errores[] = [
                        'linea'   => $numero,
                        'mensaje' => 'Se encontro un segundo H1. El archivo debe tener exactamente un plan (H1).',
                    ];
                    continue;
                }
                $plan = $this->nodoPlan($m[1], $numero);
                $ejeActual = $temaActual = $objActual = $estActual = null;
                continue;
            }

            // H2 — Eje
            if (preg_match('/^##\s+(.+)$/', $trimmed, $m)) {
                if (! $plan) {
                    $this->errores[] = ['linea' => $numero, 'mensaje' => "Eje encontrado antes de definir el Plan (H1): '{$m[1]}'"];
                    continue;
                }
                $ejeActual = $this->nodoEje($m[1], $numero, count($plan['ejes']));
                $plan['ejes'][] = &$ejeActual;
                $temaActual = $objActual = $estActual = null;
                continue;
            }

            // H3 — Tema
            if (preg_match('/^###\s+(.+)$/', $trimmed, $m)) {
                if (! $ejeActual) {
                    $this->errores[] = ['linea' => $numero, 'mensaje' => "Tema encontrado sin Eje previo: '{$m[1]}'"];
                    continue;
                }
                $temaActual = $this->nodoTema($m[1], $numero, count($ejeActual['temas']));
                $ejeActual['temas'][] = &$temaActual;
                $objActual = $estActual = null;
                continue;
            }

            // H4 — Objetivo Estrategico
            if (preg_match('/^####\s+(.+)$/', $trimmed, $m)) {
                if (! $temaActual) {
                    $this->errores[] = ['linea' => $numero, 'mensaje' => "Objetivo encontrado sin Tema previo: '{$m[1]}'"];
                    continue;
                }
                $objActual = $this->nodoObjetivo($m[1], $numero, count($temaActual['objetivos']));
                $temaActual['objetivos'][] = &$objActual;
                $estActual = null;
                continue;
            }

            // H5 — Estrategia
            if (preg_match('/^#####\s+(.+)$/', $trimmed, $m)) {
                if (! $objActual) {
                    $this->errores[] = ['linea' => $numero, 'mensaje' => "Estrategia encontrada sin Objetivo previo: '{$m[1]}'"];
                    continue;
                }
                $estActual = $this->nodoEstrategia($m[1], $numero, count($objActual['estrategias']));
                $objActual['estrategias'][] = &$estActual;
                continue;
            }

            // Linea de Accion (lista con guion)
            if (preg_match('/^-\s+(.+)$/', $trimmed, $m)) {
                if (! $estActual) {
                    $this->errores[] = ['linea' => $numero, 'mensaje' => "Linea de accion encontrada sin Estrategia previa: '{$m[1]}'"];
                    continue;
                }
                $estActual['lineas'][] = $this->nodoLinea($m[1], $numero, count($estActual['lineas']));
                continue;
            }

            // Headings H6 u otros: advertencia
            if (preg_match('/^#{1,6}\s/', $trimmed)) {
                $this->errores[] = ['linea' => $numero, 'mensaje' => "Heading no reconocido (se esperan H1 a H5): '{$trimmed}'"];
            }
        }

        if (! $plan) {
            $this->errores[] = ['linea' => 0, 'mensaje' => 'No se encontro el titulo del plan (H1). El archivo parece estar vacio o tiene formato incorrecto.'];
        }

        return [
            'plan'    => $plan,
            'errores' => $this->errores,
        ];
    }

    public function tieneErrores(): bool
    {
        return count($this->errores) > 0;
    }

    public function getErrores(): array
    {
        return $this->errores;
    }

    // -------------------------------------------------------
    // Helpers de construccion de nodos
    // -------------------------------------------------------

    private function nodoPlan(string $titulo, int $linea): array
    {
        [$clave, $nombre] = $this->extraerClaveNombre($titulo);
        return [
            '_linea'  => $linea,
            'clave'   => $clave,
            'nombre'  => $nombre,
            'activo'  => false,
            'ejes'    => [],
        ];
    }

    private function nodoEje(string $titulo, int $linea, int $posicion): array
    {
        [$clave, $nombre] = $this->extraerClaveNombre($titulo);
        return [
            '_linea'  => $linea,
            'clave'   => $clave ?: 'Eje ' . ($posicion + 1),
            'nombre'  => $nombre,
            'temas'   => [],
        ];
    }

    private function nodoTema(string $titulo, int $linea, int $posicion): array
    {
        [$clave, $nombre] = $this->extraerClaveNombre($titulo);
        return [
            '_linea'    => $linea,
            'clave'     => $clave ?: ($posicion + 1),
            'nombre'    => $nombre,
            'objetivos' => [],
        ];
    }

    private function nodoObjetivo(string $titulo, int $linea, int $posicion): array
    {
        [$clave, $nombre] = $this->extraerClaveNombre($titulo);
        return [
            '_linea'      => $linea,
            'clave'       => $clave ?: ($posicion + 1),
            'nombre'      => $nombre,
            'descripcion' => '',
            'estrategias' => [],
        ];
    }

    private function nodoEstrategia(string $titulo, int $linea, int $posicion): array
    {
        [$clave, $nombre] = $this->extraerClaveNombre($titulo);
        return [
            '_linea'  => $linea,
            'clave'   => $clave ?: ($posicion + 1),
            'nombre'  => $nombre,
            'lineas'  => [],
        ];
    }

    private function nodoLinea(string $titulo, int $linea, int $posicion): array
    {
        [$clave, $nombre] = $this->extraerClaveNombre($titulo);
        return [
            '_linea'  => $linea,
            'clave'   => $clave ?: ($posicion + 1),
            'nombre'  => $nombre,
        ];
    }

    /**
     * Intenta extraer "Clave: Nombre" del titulo.
     * Ejemplo: "Objetivo 1.1.1: Reducir la incidencia..." -> ['1.1.1', 'Reducir...']
     * Si no hay separador, retorna ['', titulo completo].
     */
    private function extraerClaveNombre(string $titulo): array
    {
        // Patron: "Palabra Clave_numerica: resto"
        if (preg_match('/^[^\d]*(\d[\d\.]*)\s*[:\-]\s*(.+)$/', $titulo, $m)) {
            return [trim($m[1]), trim($m[2])];
        }
        // Patron: "Texto: Descripcion" sin numero (Eje 1: Seguridad)
        if (preg_match('/^(.+?)\s*:\s*(.+)$/', $titulo, $m)) {
            return [trim($m[1]), trim($m[2])];
        }
        return ['', trim($titulo)];
    }
}
```

### 2. Crear el servicio de persistencia en cascada

```bash
sail artisan make:class Services/PedImportService
```

```php
<?php
// app/Services/PedImportService.php

namespace App\Services;

use App\Models\PedEje;
use App\Models\PedEstrategia;
use App\Models\PedLineaAccion;
use App\Models\PedObjetivoEstrategico;
use App\Models\PedPlan;
use App\Models\PedTema;
use Illuminate\Support\Facades\DB;

class PedImportService
{
    /**
     * Persiste el arbol parseado en una sola transaccion.
     *
     * @param  array $arbol  Estructura retornada por PedMarkdownParser::parse()['plan']
     * @return PedPlan El plan creado.
     * @throws \Throwable Si ocurre cualquier error, la transaccion hace rollback.
     */
    public function importar(array $arbol): PedPlan
    {
        return DB::transaction(function () use ($arbol) {
            $plan = PedPlan::create([
                'nombre' => $arbol['nombre'],
                'clave'  => $arbol['clave'],
                'activo' => $arbol['activo'] ?? false,
            ]);

            foreach ($arbol['ejes'] as $ejeData) {
                $eje = PedEje::create([
                    'ped_plan_id' => $plan->id,
                    'clave'       => $ejeData['clave'],
                    'nombre'      => $ejeData['nombre'],
                ]);

                foreach ($ejeData['temas'] as $temaData) {
                    $tema = PedTema::create([
                        'ped_eje_id' => $eje->id,
                        'clave'      => $temaData['clave'],
                        'nombre'     => $temaData['nombre'],
                    ]);

                    foreach ($temaData['objetivos'] as $objData) {
                        $objetivo = PedObjetivoEstrategico::create([
                            'ped_tema_id'  => $tema->id,
                            'clave'        => $objData['clave'],
                            'nombre'       => $objData['nombre'],
                            'descripcion'  => $objData['descripcion'] ?? null,
                        ]);

                        foreach ($objData['estrategias'] as $estData) {
                            $estrategia = PedEstrategia::create([
                                'ped_objetivo_estrategico_id' => $objetivo->id,
                                'clave'                        => $estData['clave'],
                                'nombre'                       => $estData['nombre'],
                            ]);

                            foreach ($estData['lineas'] as $lineaData) {
                                PedLineaAccion::create([
                                    'ped_estrategia_id' => $estrategia->id,
                                    'clave'             => $lineaData['clave'],
                                    'nombre'            => $lineaData['nombre'],
                                ]);
                            }
                        }
                    }
                }
            }

            return $plan;
        });
    }

    /**
     * Retorna estadisticas del arbol antes de importar.
     */
    public function estadisticas(array $arbol): array
    {
        $ejes       = count($arbol['ejes'] ?? []);
        $temas      = 0;
        $objetivos  = 0;
        $estrategias = 0;
        $lineas     = 0;

        foreach ($arbol['ejes'] as $eje) {
            $temas += count($eje['temas']);
            foreach ($eje['temas'] as $tema) {
                $objetivos += count($tema['objetivos']);
                foreach ($tema['objetivos'] as $obj) {
                    $estrategias += count($obj['estrategias']);
                    foreach ($obj['estrategias'] as $est) {
                        $lineas += count($est['lineas']);
                    }
                }
            }
        }

        return compact('ejes', 'temas', 'objetivos', 'estrategias', 'lineas');
    }
}
```

### 3. Crear el componente Livewire

```bash
sail artisan make:livewire Catalogos/ImportadorPed
```

```php
<?php
// app/Livewire/Catalogos/ImportadorPed.php

namespace App\Livewire\Catalogos;

use App\Services\PedImportService;
use App\Services\PedMarkdownParser;
use Livewire\Component;
use Livewire\WithFileUploads;

class ImportadorPed extends Component
{
    use WithFileUploads;

    public $archivo = null;

    // Pasos: 'upload' | 'preview' | 'confirmado' | 'error'
    public string $paso = 'upload';

    // Arbol parseado (serializable para session de Livewire)
    public ?array $arbol = null;

    // Estadisticas del arbol
    public array $estadisticas = [];

    // Errores del parser
    public array $erroresParseo = [];

    // ID del plan importado
    public ?int $planImportadoId = null;

    protected array $rules = [
        'archivo' => 'required|file|mimes:md,txt|max:2048',
    ];

    public function mount(): void
    {
        abort_unless(auth()->user()->can('gestionar_catalogos'), 403);
    }

    public function procesar(): void
    {
        $this->validate();

        $contenido = file_get_contents($this->archivo->getRealPath());

        $parser  = new PedMarkdownParser();
        $result  = $parser->parse($contenido);

        $this->erroresParseo = $result['errores'];

        if (count($this->erroresParseo) > 0 && ! $result['plan']) {
            // Errores criticos: no hay plan
            $this->paso = 'error';
            return;
        }

        $this->arbol = $result['plan'];

        $importService       = new PedImportService();
        $this->estadisticas  = $importService->estadisticas($this->arbol);

        $this->paso = 'preview';
    }

    /**
     * Permite al usuario editar el nombre del plan directamente en la preview.
     */
    public function actualizarNombrePlan(string $nombre): void
    {
        $this->arbol['nombre'] = $nombre;
    }

    /**
     * Actualizar nombre de un eje en el arbol.
     */
    public function actualizarNombreEje(int $ejeIndex, string $nombre): void
    {
        if (isset($this->arbol['ejes'][$ejeIndex])) {
            $this->arbol['ejes'][$ejeIndex]['nombre'] = $nombre;
        }
    }

    /**
     * Confirmar e importar.
     */
    public function confirmar(): void
    {
        if (! $this->arbol) {
            $this->dispatch('notify', ['message' => 'No hay datos para importar.', 'type' => 'error']);
            return;
        }

        try {
            $importService        = app(PedImportService::class);
            $plan                 = $importService->importar($this->arbol);
            $this->planImportadoId = $plan->id;
            $this->paso           = 'confirmado';
            $this->dispatch('notify', ['message' => "Plan '{$plan->nombre}' importado correctamente.", 'type' => 'success']);
        } catch (\Throwable $e) {
            \Log::error('PedImportService fallo: ' . $e->getMessage(), ['exception' => $e]);
            $this->erroresParseo[] = ['linea' => 0, 'mensaje' => 'Error al guardar en la base de datos: ' . $e->getMessage()];
            $this->paso = 'error';
        }
    }

    public function reiniciar(): void
    {
        $this->reset(['archivo', 'arbol', 'estadisticas', 'erroresParseo', 'planImportadoId']);
        $this->paso = 'upload';
    }

    public function render()
    {
        return view('livewire.catalogos.importador-ped');
    }
}
```

### 4. Vista Blade del importador

```blade
{{-- resources/views/livewire/catalogos/importador-ped.blade.php --}}
<div class="max-w-4xl mx-auto space-y-6">
    <div class="flex items-center justify-between">
        <h1 class="text-2xl font-bold text-gray-900">Importar PED desde Markdown</h1>
    </div>

    {{-- Indicador de pasos --}}
    <div class="flex items-center space-x-2 text-sm text-gray-500 mb-4">
        <span class="{{ $paso === 'upload' ? 'font-semibold text-indigo-600' : '' }}">1. Subir archivo</span>
        <span>→</span>
        <span class="{{ $paso === 'preview' ? 'font-semibold text-indigo-600' : '' }}">2. Previsualizar</span>
        <span>→</span>
        <span class="{{ $paso === 'confirmado' ? 'font-semibold text-green-600' : '' }}">3. Confirmar</span>
    </div>

    {{-- PASO 1: Upload --}}
    @if($paso === 'upload')
        <div class="bg-white rounded-xl shadow p-6">
            <h2 class="text-base font-semibold text-gray-800 mb-4">Seleccionar archivo Markdown</h2>

            <div class="mb-4 p-4 bg-gray-50 rounded-lg border border-gray-200 text-sm text-gray-600">
                <p class="font-medium mb-2">Formato esperado:</p>
                <pre class="text-xs text-gray-700 leading-5"># Plan Estatal de Desarrollo 2022-2027
## Eje 1: Seguridad y Justicia
### Tema 1.1: Prevencion del delito
#### Objetivo 1.1.1: Reducir la incidencia delictiva
##### Estrategia 1.1.1.1: Programas de intervencion
- Linea 1.1.1.1.1: Implementar talleres en zonas de riesgo</pre>
            </div>

            <div wire:loading.remove>
                <label class="block w-full border-2 border-dashed border-gray-300 rounded-lg p-8 text-center cursor-pointer hover:border-indigo-400 hover:bg-indigo-50 transition-colors">
                    <svg class="mx-auto h-10 w-10 text-gray-400 mb-3" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 13h6m-3-3v6m5 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
                    </svg>
                    <span class="text-sm text-gray-600">Haz clic o arrastra tu archivo <strong>.md</strong> aqui</span>
                    <input type="file" wire:model="archivo" accept=".md,.txt" class="hidden">
                </label>
                @error('archivo') <p class="text-red-500 text-sm mt-2">{{ $message }}</p> @enderror
            </div>

            <div wire:loading class="text-center py-6 text-sm text-gray-500">
                <svg class="animate-spin h-6 w-6 mx-auto mb-2 text-indigo-500" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                </svg>
                Procesando archivo...
            </div>

            @if($archivo)
                <div class="mt-4 flex justify-end">
                    <button wire:click="procesar"
                            class="px-6 py-2 bg-indigo-600 text-white text-sm font-medium rounded-lg hover:bg-indigo-700">
                        Parsear y previsualizar
                    </button>
                </div>
            @endif
        </div>
    @endif

    {{-- PASO 2: Preview --}}
    @if($paso === 'preview' && $arbol)
        <div class="bg-white rounded-xl shadow p-6">
            <div class="flex items-start justify-between mb-4">
                <h2 class="text-base font-semibold text-gray-800">Previsualizar estructura</h2>
                <button wire:click="reiniciar" class="text-sm text-gray-400 hover:text-gray-600">Volver a subir</button>
            </div>

            {{-- Advertencias (errores no criticos) --}}
            @if(count($erroresParseo) > 0)
                <div class="mb-4 p-4 bg-yellow-50 border border-yellow-200 rounded-lg">
                    <p class="text-sm font-semibold text-yellow-800 mb-2">Advertencias durante el parseo:</p>
                    <ul class="text-xs text-yellow-700 space-y-1">
                        @foreach($erroresParseo as $err)
                            <li>Linea {{ $err['linea'] }}: {{ $err['mensaje'] }}</li>
                        @endforeach
                    </ul>
                </div>
            @endif

            {{-- Estadisticas --}}
            <div class="grid grid-cols-5 gap-3 mb-6">
                @foreach(['ejes' => 'Ejes', 'temas' => 'Temas', 'objetivos' => 'Objetivos', 'estrategias' => 'Estrategias', 'lineas' => 'Lineas'] as $key => $label)
                    <div class="bg-indigo-50 rounded-lg p-3 text-center">
                        <p class="text-2xl font-bold text-indigo-700">{{ $estadisticas[$key] ?? 0 }}</p>
                        <p class="text-xs text-indigo-500 mt-1">{{ $label }}</p>
                    </div>
                @endforeach
            </div>

            {{-- Arbol editable --}}
            <div class="mb-4">
                <label class="block text-sm font-medium text-gray-700 mb-1">Nombre del Plan</label>
                <input type="text"
                       wire:change="actualizarNombrePlan($event.target.value)"
                       value="{{ $arbol['nombre'] }}"
                       class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
            </div>

            <div class="max-h-96 overflow-y-auto border border-gray-200 rounded-lg p-4">
                @foreach($arbol['ejes'] as $ejeIndex => $eje)
                    <details class="mb-2" open>
                        <summary class="cursor-pointer font-semibold text-gray-800 text-sm py-1">
                            Eje: {{ $eje['clave'] }} — {{ $eje['nombre'] }}
                        </summary>
                        <div class="ml-4 mt-1 space-y-1">
                            @foreach($eje['temas'] as $tema)
                                <details>
                                    <summary class="cursor-pointer text-sm text-gray-700 py-1">
                                        Tema: {{ $tema['clave'] }} — {{ $tema['nombre'] }}
                                    </summary>
                                    <div class="ml-4 text-xs text-gray-600 space-y-1 mt-1">
                                        @foreach($tema['objetivos'] as $obj)
                                            <p class="py-0.5">
                                                Obj: {{ $obj['clave'] }} — {{ $obj['nombre'] }}
                                                ({{ count($obj['estrategias']) }} estrategias)
                                            </p>
                                        @endforeach
                                    </div>
                                </details>
                            @endforeach
                        </div>
                    </details>
                @endforeach
            </div>

            <div class="flex justify-end gap-3 mt-6">
                <button wire:click="reiniciar"
                        class="px-4 py-2 text-sm text-gray-700 bg-white border border-gray-300 rounded-lg hover:bg-gray-50">
                    Cancelar
                </button>
                <button wire:click="confirmar"
                        wire:loading.attr="disabled"
                        class="px-6 py-2 bg-green-600 text-white text-sm font-medium rounded-lg hover:bg-green-700 disabled:opacity-50">
                    <span wire:loading.remove wire:target="confirmar">Confirmar e importar</span>
                    <span wire:loading wire:target="confirmar">Importando...</span>
                </button>
            </div>
        </div>
    @endif

    {{-- PASO 3: Confirmado --}}
    @if($paso === 'confirmado')
        <div class="bg-white rounded-xl shadow p-12 text-center">
            <svg class="mx-auto h-16 w-16 text-green-500 mb-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
            </svg>
            <h2 class="text-xl font-bold text-gray-900 mb-2">Plan importado correctamente</h2>
            <p class="text-gray-500 text-sm mb-6">
                Se crearon {{ $estadisticas['ejes'] }} ejes, {{ $estadisticas['objetivos'] }} objetivos y {{ $estadisticas['lineas'] }} lineas de accion.
            </p>
            <div class="flex justify-center gap-4">
                <a href="{{ route('catalogos.ped.show', $planImportadoId) }}"
                   class="px-6 py-2 bg-indigo-600 text-white text-sm rounded-lg hover:bg-indigo-700">
                    Ver plan importado
                </a>
                <button wire:click="reiniciar"
                        class="px-6 py-2 bg-white text-sm text-gray-700 border border-gray-300 rounded-lg hover:bg-gray-50">
                    Importar otro
                </button>
            </div>
        </div>
    @endif

    {{-- ERROR --}}
    @if($paso === 'error')
        <div class="bg-white rounded-xl shadow p-6">
            <h2 class="text-base font-semibold text-red-700 mb-4">Errores en el archivo</h2>
            <ul class="space-y-2">
                @foreach($erroresParseo as $err)
                    <li class="flex gap-3 text-sm text-red-600">
                        <span class="font-medium w-20 flex-shrink-0">Linea {{ $err['linea'] }}:</span>
                        <span>{{ $err['mensaje'] }}</span>
                    </li>
                @endforeach
            </ul>
            <div class="mt-6">
                <button wire:click="reiniciar"
                        class="px-4 py-2 bg-indigo-600 text-white text-sm rounded-lg hover:bg-indigo-700">
                    Intentar de nuevo
                </button>
            </div>
        </div>
    @endif
</div>
```

### 5. Registrar ruta

```php
// routes/web.php

use App\Livewire\Catalogos\ImportadorPed;

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/catalogos/importar-ped', ImportadorPed::class)
        ->name('catalogos.importar-ped');
});
```

### 6. Configurar almacenamiento temporal de uploads

```bash
# Livewire guarda temporales en storage/app/livewire-tmp
sail artisan storage:link
```

En `.env` verificar:

```env
LIVEWIRE_TEMPORARY_FILE_UPLOAD_DISK=local
```

### 7. Crear tests

```bash
sail artisan make:test Services/PedMarkdownParserTest --unit
sail artisan make:test Livewire/ImportadorPedTest
```

```php
<?php
// tests/Unit/Services/PedMarkdownParserTest.php

namespace Tests\Unit\Services;

use App\Services\PedMarkdownParser;
use Tests\TestCase;

class PedMarkdownParserTest extends TestCase
{
    private PedMarkdownParser $parser;

    protected function setUp(): void
    {
        parent::setUp();
        $this->parser = new PedMarkdownParser();
    }

    public function test_parsea_estructura_completa(): void
    {
        $md = <<<MD
        # Plan Estatal 2022-2027
        ## Eje 1: Seguridad y Justicia
        ### Tema 1.1: Prevencion del delito
        #### Objetivo 1.1.1: Reducir incidencia delictiva
        ##### Estrategia 1.1.1.1: Programas de intervencion
        - Linea 1.1.1.1.1: Implementar talleres en zonas de riesgo
        MD;

        $result = $this->parser->parse($md);

        $this->assertNotNull($result['plan']);
        $this->assertEmpty($result['errores']);
        $this->assertEquals('Plan Estatal 2022-2027', $result['plan']['nombre']);
        $this->assertCount(1, $result['plan']['ejes']);
        $this->assertCount(1, $result['plan']['ejes'][0]['temas']);
        $this->assertCount(1, $result['plan']['ejes'][0]['temas'][0]['objetivos']);
        $this->assertCount(1, $result['plan']['ejes'][0]['temas'][0]['objetivos'][0]['estrategias']);
        $this->assertCount(1, $result['plan']['ejes'][0]['temas'][0]['objetivos'][0]['estrategias'][0]['lineas']);
    }

    public function test_reporta_error_si_falta_h1(): void
    {
        $md = "## Eje sin plan\n### Tema sin plan";
        $result = $this->parser->parse($md);

        $this->assertNotEmpty($result['errores']);
        $this->assertNull($result['plan']);
    }

    public function test_reporta_error_linea_sin_estrategia(): void
    {
        $md = <<<MD
        # Plan
        ## Eje 1: Test
        ### Tema 1.1: Test
        #### Objetivo 1.1.1: Test
        - Linea sin estrategia previa
        MD;

        $result = $this->parser->parse($md);

        $errores = collect($result['errores'])->pluck('mensaje');
        $this->assertTrue($errores->contains(fn ($m) => str_contains($m, 'sin Estrategia')));
    }

    public function test_extrae_clave_y_nombre_correctamente(): void
    {
        $md = <<<MD
        # Plan 2022-2027
        ## Eje 1: Seguridad
        ### Tema 1.1: Prevencion
        #### Objetivo 1.1.1: Reducir incidencia
        ##### Estrategia 1.1.1.1: Intervencion
        - Linea 1.1.1.1.1: Talleres
        MD;

        $result = $this->parser->parse($md);

        $this->assertEquals('1', $result['plan']['ejes'][0]['clave']);
        $this->assertEquals('Seguridad', $result['plan']['ejes'][0]['nombre']);
    }
}
```

```bash
sail artisan test --filter=PedMarkdownParserTest
sail artisan test --filter=ImportadorPedTest
```

---

## Criterios de Aceptacion

- [ ] La ruta `/catalogos/importar-ped` es accesible solo con permiso `gestionar_catalogos`.
- [ ] Solo se aceptan archivos `.md` o `.txt` (validacion de MIME).
- [ ] Un archivo bien formado pasa el parser sin errores y muestra la previsualización en arbol.
- [ ] La previsualización muestra estadisticas (conteo de ejes, temas, objetivos, estrategias, lineas).
- [ ] El usuario puede editar el nombre del plan en la preview antes de confirmar.
- [ ] Al confirmar, todos los registros se crean en una sola transaccion DB; si falla un INSERT, hace rollback completo.
- [ ] Archivos con errores de formato muestran mensajes indicando numero de linea y descripcion del problema.
- [ ] Los tests unitarios del parser pasan en verde.
- [ ] No se deja basura en la base de datos si el import falla a mitad.

---

## Notas

- `PedMarkdownParser` no depende de Laravel y puede testearse como clase PHP pura (`TestCase` sin `RefreshDatabase`).
- El componente `ImportadorPed` guarda `$arbol` en el estado de Livewire; para planes muy grandes (>500 nodos) esto puede exceder el limite de session. Considerar almacenar el arbol en `cache` con una clave UUID de sesion si el volumen lo requiere.
- Los embeddings de los nuevos registros se generaran automaticamente via el Observer de S2-T10 al crear cada modelo; no requiere accion adicional aqui.
- El boton de "Ver plan importado" apunta a `catalogos.ped.show` que debe definirse en el ticket de CRUD del PED (no en este ticket).
