# S2-T8 — Interfaz de Matriz de Alineacion

**Tipo:** feat
**Rama:** `feat/S2-T8-interfaz-matriz-alineacion`
**Depende de:** S2-T5 (tablas pivote de alineacion), S2-T11 (busqueda semantica para sugerencias)

---

## Contexto

La Matriz de Alineacion establece la coherencia entre los distintos instrumentos de planeacion. Se compone de tres niveles jerarquicos:

| Nivel | Relacion |
|-------|----------|
| 1 | PED (PedObjetivoEstrategico) ↔ PND (PndObjetivo) |
| 2 | PND (PndObjetivo) ↔ ODS (OdsMeta) |
| 3 | PED (PedLineaAccion) ↔ Programa Derivado (ProgramaDerivadoObjetivo) |

La herencia funciona de arriba a abajo: al vincular una `PedLineaAccion` con un `ProgramaDerivadoObjetivo`, el sistema debe poder mostrar la cadena completa hasta el ODS correspondiente si existe la alineacion en los niveles superiores.

Las sugerencias de S2-T11 se integran como recomendaciones semanticas al seleccionar un elemento fuente.

---

## Pre-requisitos

- S2-T5 completado: tablas pivote `ped_objetivo_pnd_objetivo`, `pnd_objetivo_ods_meta`, `ped_linea_accion_programa_derivado_objetivo` existentes con sus modelos.
- S2-T11 completado (o al menos `SemanticSearchService` disponible con metodo `findSimilar`).
- Permiso `gestionar_catalogos` de S1-T3.
- Relaciones Eloquent definidas en los modelos correspondientes.

Verificar tablas pivote:

```bash
sail artisan tinker --execute="
    \DB::statement('SELECT COUNT(*) FROM ped_objetivo_pnd_objetivo');
    \DB::statement('SELECT COUNT(*) FROM pnd_objetivo_ods_meta');
    \DB::statement('SELECT COUNT(*) FROM ped_linea_accion_programa_derivado_objetivo');
    echo 'Tablas pivote OK';
"
```

---

## Pasos

### 1. Crear el componente Livewire principal

```bash
sail artisan make:livewire Alineacion/MatrizAlineacion
```

### 2. Implementar `MatrizAlineacion`

```php
<?php
// app/Livewire/Alineacion/MatrizAlineacion.php

namespace App\Livewire\Alineacion;

use App\Models\OdsMeta;
use App\Models\PedLineaAccion;
use App\Models\PedObjetivoEstrategico;
use App\Models\PndObjetivo;
use App\Models\ProgramaDerivadoObjetivo;
use App\Services\SemanticSearchService;
use Illuminate\Support\Collection;
use Livewire\Component;

class MatrizAlineacion extends Component
{
    // --- Nivel 1: PED <-> PND ---
    public ?int $pedObjetivoSeleccionado  = null;
    public ?int $pndObjetivoSeleccionado  = null;

    // --- Nivel 2: PND <-> ODS ---
    public ?int $pndObjetivoOdsSeleccionado = null;
    public ?int $odsMetaSeleccionada        = null;

    // --- Nivel 3: Linea de Accion <-> Programa Derivado ---
    public ?int $pedLineaAccionSeleccionada      = null;
    public ?int $programaDerivadoObjetivoSeleccionado = null;

    // Sugerencias semanticas
    public array $sugerenciasNivel1 = [];
    public array $sugerenciasNivel2 = [];
    public array $sugerenciasNivel3 = [];

    // Vista de cadena completa
    public array $cadenaCompleta = [];

    protected SemanticSearchService $semanticSearch;

    public function boot(SemanticSearchService $semanticSearch): void
    {
        $this->semanticSearch = $semanticSearch;
    }

    public function mount(): void
    {
        abort_unless(auth()->user()->can('gestionar_catalogos'), 403);
    }

    // =========================================================
    // NIVEL 1: PED Objetivo Estrategico <-> PND Objetivo
    // =========================================================

    public function updatedPedObjetivoSeleccionado(?int $value): void
    {
        $this->sugerenciasNivel1 = [];
        $this->cadenaCompleta    = [];

        if ($value) {
            $objetivo = PedObjetivoEstrategico::find($value);
            if ($objetivo) {
                $this->sugerenciasNivel1 = $this->buscarSugerencias(
                    $objetivo->descripcion ?? $objetivo->nombre,
                    PndObjetivo::class
                );
            }
        }
    }

    public function vincularNivel1(): void
    {
        $this->validate([
            'pedObjetivoSeleccionado' => 'required|exists:ped_objetivos_estrategicos,id',
            'pndObjetivoSeleccionado' => 'required|exists:pnd_objetivos,id',
        ]);

        $pedObj = PedObjetivoEstrategico::findOrFail($this->pedObjetivoSeleccionado);

        if (! $pedObj->pndObjetivos()->where('pnd_objetivo_id', $this->pndObjetivoSeleccionado)->exists()) {
            $pedObj->pndObjetivos()->attach($this->pndObjetivoSeleccionado);
            $this->dispatch('notify', ['message' => 'Vinculo PED-PND creado.', 'type' => 'success']);
        } else {
            $this->dispatch('notify', ['message' => 'Este vinculo ya existe.', 'type' => 'warning']);
        }

        $this->actualizarCadena();
    }

    public function quitarVinculoNivel1(int $pedObjetivoId, int $pndObjetivoId): void
    {
        PedObjetivoEstrategico::findOrFail($pedObjetivoId)
            ->pndObjetivos()
            ->detach($pndObjetivoId);

        $this->dispatch('notify', ['message' => 'Vinculo PED-PND eliminado.', 'type' => 'info']);
        $this->actualizarCadena();
    }

    public function aplicarSugerenciaNivel1(int $pndObjetivoId): void
    {
        $this->pndObjetivoSeleccionado = $pndObjetivoId;
    }

    // =========================================================
    // NIVEL 2: PND Objetivo <-> ODS Meta
    // =========================================================

    public function updatedPndObjetivoOdsSeleccionado(?int $value): void
    {
        $this->sugerenciasNivel2 = [];

        if ($value) {
            $objetivo = PndObjetivo::find($value);
            if ($objetivo) {
                $this->sugerenciasNivel2 = $this->buscarSugerencias(
                    $objetivo->descripcion ?? $objetivo->nombre,
                    OdsMeta::class
                );
            }
        }
    }

    public function vincularNivel2(): void
    {
        $this->validate([
            'pndObjetivoOdsSeleccionado' => 'required|exists:pnd_objetivos,id',
            'odsMetaSeleccionada'        => 'required|exists:ods_metas,id',
        ]);

        $pndObj = PndObjetivo::findOrFail($this->pndObjetivoOdsSeleccionado);

        if (! $pndObj->odsMetas()->where('ods_meta_id', $this->odsMetaSeleccionada)->exists()) {
            $pndObj->odsMetas()->attach($this->odsMetaSeleccionada);
            $this->dispatch('notify', ['message' => 'Vinculo PND-ODS creado.', 'type' => 'success']);
        } else {
            $this->dispatch('notify', ['message' => 'Este vinculo ya existe.', 'type' => 'warning']);
        }
    }

    public function quitarVinculoNivel2(int $pndObjetivoId, int $odsMetaId): void
    {
        PndObjetivo::findOrFail($pndObjetivoId)
            ->odsMetas()
            ->detach($odsMetaId);

        $this->dispatch('notify', ['message' => 'Vinculo PND-ODS eliminado.', 'type' => 'info']);
    }

    public function aplicarSugerenciaNivel2(int $odsMetaId): void
    {
        $this->odsMetaSeleccionada = $odsMetaId;
    }

    // =========================================================
    // NIVEL 3: Linea de Accion <-> Programa Derivado Objetivo
    // =========================================================

    public function updatedPedLineaAccionSeleccionada(?int $value): void
    {
        $this->sugerenciasNivel3 = [];
        $this->cadenaCompleta    = [];

        if ($value) {
            $linea = PedLineaAccion::find($value);
            if ($linea) {
                $this->sugerenciasNivel3 = $this->buscarSugerencias(
                    $linea->descripcion ?? $linea->nombre,
                    ProgramaDerivadoObjetivo::class
                );
                $this->actualizarCadena();
            }
        }
    }

    public function vincularNivel3(): void
    {
        $this->validate([
            'pedLineaAccionSeleccionada'           => 'required|exists:ped_lineas_accion,id',
            'programaDerivadoObjetivoSeleccionado' => 'required|exists:programa_derivado_objetivos,id',
        ]);

        $linea = PedLineaAccion::findOrFail($this->pedLineaAccionSeleccionada);

        if (! $linea->programaDerivadoObjetivos()
            ->where('programa_derivado_objetivo_id', $this->programaDerivadoObjetivoSeleccionado)
            ->exists()) {
            $linea->programaDerivadoObjetivos()->attach($this->programaDerivadoObjetivoSeleccionado);
            $this->dispatch('notify', ['message' => 'Vinculo Linea-Programa creado.', 'type' => 'success']);
        } else {
            $this->dispatch('notify', ['message' => 'Este vinculo ya existe.', 'type' => 'warning']);
        }

        $this->actualizarCadena();
    }

    public function quitarVinculoNivel3(int $lineaId, int $programaObjetivoId): void
    {
        PedLineaAccion::findOrFail($lineaId)
            ->programaDerivadoObjetivos()
            ->detach($programaObjetivoId);

        $this->dispatch('notify', ['message' => 'Vinculo Linea-Programa eliminado.', 'type' => 'info']);
        $this->actualizarCadena();
    }

    public function aplicarSugerenciaNivel3(int $programaObjetivoId): void
    {
        $this->programaDerivadoObjetivoSeleccionado = $programaObjetivoId;
    }

    // =========================================================
    // CADENA COMPLETA (herencia)
    // =========================================================

    public function actualizarCadena(): void
    {
        $this->cadenaCompleta = [];

        if (! $this->pedLineaAccionSeleccionada) {
            return;
        }

        $linea = PedLineaAccion::with([
            'pedEstrategia.pedObjetivoEstrategico.pndObjetivos.odsMetas',
        ])->find($this->pedLineaAccionSeleccionada);

        if (! $linea) {
            return;
        }

        $estrategia = $linea->pedEstrategia;
        $objetivo   = $estrategia?->pedObjetivoEstrategico;

        $this->cadenaCompleta = [
            'linea'    => $linea->nombre,
            'objetivo' => $objetivo?->nombre,
            'pnd'      => $objetivo?->pndObjetivos->pluck('nombre')->toArray() ?? [],
            'ods'      => $objetivo?->pndObjetivos
                ->flatMap(fn ($pnd) => $pnd->odsMetas->pluck('codigo'))
                ->unique()
                ->values()
                ->toArray() ?? [],
        ];
    }

    // =========================================================
    // HELPER: busqueda semantica
    // =========================================================

    private function buscarSugerencias(string $texto, string $modelClass): array
    {
        try {
            return $this->semanticSearch
                ->findSimilar($texto, $modelClass, limit: 5, threshold: 0.65)
                ->map(fn ($item) => [
                    'id'     => $item->model->id,
                    'nombre' => $item->model->nombre ?? $item->model->descripcion ?? '—',
                    'score'  => round($item->score * 100, 1),
                ])
                ->toArray();
        } catch (\Throwable $e) {
            \Log::warning("SemanticSearch fallo en MatrizAlineacion: " . $e->getMessage());
            return [];
        }
    }

    // =========================================================
    // RENDER
    // =========================================================

    public function render()
    {
        return view('livewire.alineacion.matriz-alineacion', [
            'pedObjetivos'              => PedObjetivoEstrategico::orderBy('clave')->get(),
            'pndObjetivos'              => PndObjetivo::orderBy('clave')->get(),
            'odsMetas'                  => OdsMeta::orderBy('codigo')->get(),
            'pedLineasAccion'           => PedLineaAccion::orderBy('clave')->get(),
            'programaDerivadoObjetivos' => ProgramaDerivadoObjetivo::with('programaDerivado')->orderBy('clave')->get(),

            // Vinculos existentes para mostrar listas
            'vinculosNivel1' => PedObjetivoEstrategico::with('pndObjetivos')
                ->whereHas('pndObjetivos')
                ->orderBy('clave')
                ->get(),
            'vinculosNivel2' => PndObjetivo::with('odsMetas')
                ->whereHas('odsMetas')
                ->orderBy('clave')
                ->get(),
            'vinculosNivel3' => PedLineaAccion::with('programaDerivadoObjetivos.programaDerivado')
                ->whereHas('programaDerivadoObjetivos')
                ->orderBy('clave')
                ->get(),
        ]);
    }
}
```

### 3. Vista Blade

```blade
{{-- resources/views/livewire/alineacion/matriz-alineacion.blade.php --}}
<div class="space-y-8">
    <div class="flex items-center justify-between">
        <h1 class="text-2xl font-bold text-gray-900">Matriz de Alineacion</h1>
    </div>

    {{-- ============================================================
         NIVEL 1: PED Objetivo Estrategico <-> PND Objetivo
         ============================================================ --}}
    <section class="bg-white rounded-xl shadow p-6">
        <h2 class="text-lg font-semibold text-gray-800 mb-4 pb-2 border-b">
            Nivel 1 — Objetivo Estrategico PED ↔ Objetivo PND
        </h2>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
            {{-- Fuente: PED Objetivo --}}
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Objetivo Estrategico PED (fuente)</label>
                <select wire:model.live="pedObjetivoSeleccionado"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">-- Seleccionar --</option>
                    @foreach($pedObjetivos as $obj)
                        <option value="{{ $obj->id }}">{{ $obj->clave }} — {{ Str::limit($obj->nombre, 60) }}</option>
                    @endforeach
                </select>
            </div>

            {{-- Destino: PND Objetivo --}}
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Objetivo PND (destino)</label>
                <select wire:model="pndObjetivoSeleccionado"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">-- Seleccionar --</option>
                    @foreach($pndObjetivos as $obj)
                        <option value="{{ $obj->id }}">{{ $obj->clave }} — {{ Str::limit($obj->nombre, 60) }}</option>
                    @endforeach
                </select>

                {{-- Sugerencias semanticas --}}
                @if(count($sugerenciasNivel1) > 0)
                    <div class="mt-2 p-3 bg-indigo-50 rounded-lg border border-indigo-100">
                        <p class="text-xs font-semibold text-indigo-700 mb-2">Sugerencias semanticas:</p>
                        <ul class="space-y-1">
                            @foreach($sugerenciasNivel1 as $sug)
                                <li>
                                    <button type="button"
                                            wire:click="aplicarSugerenciaNivel1({{ $sug['id'] }})"
                                            class="text-xs text-indigo-600 hover:text-indigo-800 hover:underline text-left">
                                        {{ $sug['nombre'] }}
                                        <span class="text-gray-400">({{ $sug['score'] }}%)</span>
                                    </button>
                                </li>
                            @endforeach
                        </ul>
                    </div>
                @endif
            </div>
        </div>

        <button wire:click="vincularNivel1"
                class="px-4 py-2 bg-indigo-600 text-white text-sm font-medium rounded-lg hover:bg-indigo-700 mb-6">
            Vincular
        </button>

        {{-- Lista de vinculos existentes --}}
        @foreach($vinculosNivel1 as $pedObj)
            <div class="mb-3">
                <p class="text-sm font-semibold text-gray-800">{{ $pedObj->clave }} — {{ $pedObj->nombre }}</p>
                <ul class="mt-1 ml-4 space-y-1">
                    @foreach($pedObj->pndObjetivos as $pndObj)
                        <li class="flex items-center justify-between text-sm text-gray-600 py-1 border-b border-gray-100">
                            <span>↳ {{ $pndObj->clave }} — {{ $pndObj->nombre }}</span>
                            <button wire:click="quitarVinculoNivel1({{ $pedObj->id }}, {{ $pndObj->id }})"
                                    wire:confirm="¿Quitar este vinculo?"
                                    class="text-xs text-red-500 hover:text-red-700 ml-4 flex-shrink-0">
                                Quitar
                            </button>
                        </li>
                    @endforeach
                </ul>
            </div>
        @endforeach
    </section>

    {{-- ============================================================
         NIVEL 2: PND Objetivo <-> ODS Meta
         ============================================================ --}}
    <section class="bg-white rounded-xl shadow p-6">
        <h2 class="text-lg font-semibold text-gray-800 mb-4 pb-2 border-b">
            Nivel 2 — Objetivo PND ↔ Meta ODS
        </h2>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Objetivo PND (fuente)</label>
                <select wire:model.live="pndObjetivoOdsSeleccionado"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">-- Seleccionar --</option>
                    @foreach($pndObjetivos as $obj)
                        <option value="{{ $obj->id }}">{{ $obj->clave }} — {{ Str::limit($obj->nombre, 60) }}</option>
                    @endforeach
                </select>
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Meta ODS (destino)</label>
                <select wire:model="odsMetaSeleccionada"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">-- Seleccionar --</option>
                    @foreach($odsMetas as $meta)
                        <option value="{{ $meta->id }}">{{ $meta->codigo }} — {{ Str::limit($meta->descripcion, 60) }}</option>
                    @endforeach
                </select>
                @if(count($sugerenciasNivel2) > 0)
                    <div class="mt-2 p-3 bg-green-50 rounded-lg border border-green-100">
                        <p class="text-xs font-semibold text-green-700 mb-2">Sugerencias semanticas:</p>
                        <ul class="space-y-1">
                            @foreach($sugerenciasNivel2 as $sug)
                                <li>
                                    <button type="button"
                                            wire:click="aplicarSugerenciaNivel2({{ $sug['id'] }})"
                                            class="text-xs text-green-600 hover:text-green-800 hover:underline text-left">
                                        {{ $sug['nombre'] }}
                                        <span class="text-gray-400">({{ $sug['score'] }}%)</span>
                                    </button>
                                </li>
                            @endforeach
                        </ul>
                    </div>
                @endif
            </div>
        </div>

        <button wire:click="vincularNivel2"
                class="px-4 py-2 bg-green-600 text-white text-sm font-medium rounded-lg hover:bg-green-700 mb-6">
            Vincular
        </button>

        @foreach($vinculosNivel2 as $pndObj)
            <div class="mb-3">
                <p class="text-sm font-semibold text-gray-800">{{ $pndObj->clave }} — {{ $pndObj->nombre }}</p>
                <ul class="mt-1 ml-4 space-y-1">
                    @foreach($pndObj->odsMetas as $meta)
                        <li class="flex items-center justify-between text-sm text-gray-600 py-1 border-b border-gray-100">
                            <span>↳ ODS {{ $meta->codigo }} — {{ Str::limit($meta->descripcion, 70) }}</span>
                            <button wire:click="quitarVinculoNivel2({{ $pndObj->id }}, {{ $meta->id }})"
                                    wire:confirm="¿Quitar este vinculo?"
                                    class="text-xs text-red-500 hover:text-red-700 ml-4 flex-shrink-0">
                                Quitar
                            </button>
                        </li>
                    @endforeach
                </ul>
            </div>
        @endforeach
    </section>

    {{-- ============================================================
         NIVEL 3: Linea de Accion <-> Programa Derivado Objetivo
         ============================================================ --}}
    <section class="bg-white rounded-xl shadow p-6">
        <h2 class="text-lg font-semibold text-gray-800 mb-4 pb-2 border-b">
            Nivel 3 — Linea de Accion PED ↔ Objetivo de Programa Derivado
        </h2>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-4">
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Linea de Accion PED (fuente)</label>
                <select wire:model.live="pedLineaAccionSeleccionada"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">-- Seleccionar --</option>
                    @foreach($pedLineasAccion as $linea)
                        <option value="{{ $linea->id }}">{{ $linea->clave }} — {{ Str::limit($linea->nombre, 55) }}</option>
                    @endforeach
                </select>
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Objetivo de Programa Derivado (destino)</label>
                <select wire:model="programaDerivadoObjetivoSeleccionado"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">-- Seleccionar --</option>
                    @foreach($programaDerivadoObjetivos as $obj)
                        <option value="{{ $obj->id }}">
                            [{{ $obj->programaDerivado->tipo ?? '?' }}] {{ $obj->clave }} — {{ Str::limit($obj->nombre, 50) }}
                        </option>
                    @endforeach
                </select>
                @if(count($sugerenciasNivel3) > 0)
                    <div class="mt-2 p-3 bg-amber-50 rounded-lg border border-amber-100">
                        <p class="text-xs font-semibold text-amber-700 mb-2">Sugerencias semanticas:</p>
                        <ul class="space-y-1">
                            @foreach($sugerenciasNivel3 as $sug)
                                <li>
                                    <button type="button"
                                            wire:click="aplicarSugerenciaNivel3({{ $sug['id'] }})"
                                            class="text-xs text-amber-600 hover:text-amber-800 hover:underline text-left">
                                        {{ $sug['nombre'] }}
                                        <span class="text-gray-400">({{ $sug['score'] }}%)</span>
                                    </button>
                                </li>
                            @endforeach
                        </ul>
                    </div>
                @endif
            </div>
        </div>

        <button wire:click="vincularNivel3"
                class="px-4 py-2 bg-amber-600 text-white text-sm font-medium rounded-lg hover:bg-amber-700 mb-4">
            Vincular
        </button>

        {{-- Cadena completa --}}
        @if(count($cadenaCompleta) > 0)
            <div class="mb-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
                <p class="text-xs font-semibold text-gray-600 mb-2">Cadena de alineacion para la linea seleccionada:</p>
                <div class="text-sm text-gray-700 space-y-1">
                    <p><span class="font-medium">Linea:</span> {{ $cadenaCompleta['linea'] }}</p>
                    @if($cadenaCompleta['objetivo'])
                        <p><span class="font-medium">Objetivo PED:</span> {{ $cadenaCompleta['objetivo'] }}</p>
                    @endif
                    @if(count($cadenaCompleta['pnd']) > 0)
                        <p><span class="font-medium">Objetivos PND:</span> {{ implode(', ', $cadenaCompleta['pnd']) }}</p>
                    @endif
                    @if(count($cadenaCompleta['ods']) > 0)
                        <p>
                            <span class="font-medium">ODS heredados:</span>
                            @foreach($cadenaCompleta['ods'] as $ods)
                                <span class="inline-block px-2 py-0.5 bg-blue-100 text-blue-800 rounded text-xs mr-1">{{ $ods }}</span>
                            @endforeach
                        </p>
                    @else
                        <p class="text-gray-400 text-xs italic">Sin ODS heredados (el objetivo PED no tiene alineacion con PND-ODS).</p>
                    @endif
                </div>
            </div>
        @endif

        @foreach($vinculosNivel3 as $linea)
            <div class="mb-3">
                <p class="text-sm font-semibold text-gray-800">{{ $linea->clave }} — {{ $linea->nombre }}</p>
                <ul class="mt-1 ml-4 space-y-1">
                    @foreach($linea->programaDerivadoObjetivos as $obj)
                        <li class="flex items-center justify-between text-sm text-gray-600 py-1 border-b border-gray-100">
                            <span>
                                ↳ [{{ $obj->programaDerivado->tipo }}] {{ $obj->clave }} — {{ $obj->nombre }}
                            </span>
                            <button wire:click="quitarVinculoNivel3({{ $linea->id }}, {{ $obj->id }})"
                                    wire:confirm="¿Quitar este vinculo?"
                                    class="text-xs text-red-500 hover:text-red-700 ml-4 flex-shrink-0">
                                Quitar
                            </button>
                        </li>
                    @endforeach
                </ul>
            </div>
        @endforeach
    </section>
</div>
```

### 4. Registrar ruta

```php
// routes/web.php

use App\Livewire\Alineacion\MatrizAlineacion;

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/alineacion/matriz', MatrizAlineacion::class)
        ->name('alineacion.matriz');
});
```

### 5. Verificar relaciones Eloquent necesarias

Los modelos deben tener estas relaciones (verificar o agregar en S2-T4/S2-T5):

```php
// En PedObjetivoEstrategico:
public function pndObjetivos(): BelongsToMany
{
    return $this->belongsToMany(PndObjetivo::class, 'ped_objetivo_pnd_objetivo');
}

// En PndObjetivo:
public function odsMetas(): BelongsToMany
{
    return $this->belongsToMany(OdsMeta::class, 'pnd_objetivo_ods_meta');
}

// En PedLineaAccion:
public function programaDerivadoObjetivos(): BelongsToMany
{
    return $this->belongsToMany(
        ProgramaDerivadoObjetivo::class,
        'ped_linea_accion_programa_derivado_objetivo'
    );
}

// En PedLineaAccion (para herencia hacia arriba):
public function pedEstrategia(): BelongsTo
{
    return $this->belongsTo(PedEstrategia::class);
}

// En PedEstrategia:
public function pedObjetivoEstrategico(): BelongsTo
{
    return $this->belongsTo(PedObjetivoEstrategico::class);
}
```

### 6. Test de herencia automatica

```bash
sail artisan make:test Livewire/MatrizAlineacionTest
```

```php
<?php
// tests/Feature/Livewire/MatrizAlineacionTest.php

namespace Tests\Feature\Livewire;

use App\Livewire\Alineacion\MatrizAlineacion;
use App\Models\OdsMeta;
use App\Models\PedEstrategia;
use App\Models\PedLineaAccion;
use App\Models\PedObjetivoEstrategico;
use App\Models\PndObjetivo;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Livewire\Livewire;
use Tests\TestCase;

class MatrizAlineacionTest extends TestCase
{
    use RefreshDatabase;

    public function test_seleccionar_linea_accion_muestra_ods_heredados(): void
    {
        // Arrange: cadena completa PED -> PND -> ODS
        $pedObj  = PedObjetivoEstrategico::factory()->create(['clave' => '1.1', 'nombre' => 'Obj PED']);
        $pndObj  = PndObjetivo::factory()->create(['clave' => 'P1', 'nombre' => 'Obj PND']);
        $odsMeta = OdsMeta::factory()->create(['codigo' => '1.1']);

        $pedObj->pndObjetivos()->attach($pndObj->id);
        $pndObj->odsMetas()->attach($odsMeta->id);

        $estrategia = PedEstrategia::factory()->create([
            'ped_objetivo_estrategico_id' => $pedObj->id,
        ]);
        $linea = PedLineaAccion::factory()->create([
            'ped_estrategia_id' => $estrategia->id,
            'nombre'            => 'Linea test',
        ]);

        $user = User::factory()->create();
        $user->givePermissionTo('gestionar_catalogos');

        // Act
        $component = Livewire::actingAs($user)
            ->test(MatrizAlineacion::class)
            ->set('pedLineaAccionSeleccionada', $linea->id);

        // Assert: la cadena completa incluye el ODS heredado
        $cadena = $component->get('cadenaCompleta');
        $this->assertContains('1.1', $cadena['ods']);
        $this->assertEquals('Linea test', $cadena['linea']);
    }

    public function test_vincular_nivel1_crea_registro_en_pivote(): void
    {
        $pedObj = PedObjetivoEstrategico::factory()->create();
        $pndObj = PndObjetivo::factory()->create();
        $user   = User::factory()->create();
        $user->givePermissionTo('gestionar_catalogos');

        Livewire::actingAs($user)
            ->test(MatrizAlineacion::class)
            ->set('pedObjetivoSeleccionado', $pedObj->id)
            ->set('pndObjetivoSeleccionado', $pndObj->id)
            ->call('vincularNivel1');

        $this->assertDatabaseHas('ped_objetivo_pnd_objetivo', [
            'ped_objetivo_estrategico_id' => $pedObj->id,
            'pnd_objetivo_id'             => $pndObj->id,
        ]);
    }
}
```

```bash
sail artisan test --filter=MatrizAlineacionTest
```

---

## Criterios de Aceptacion

- [ ] La vista muestra 3 secciones independientes (Nivel 1, 2 y 3) sin recargar la pagina.
- [ ] Seleccionar un elemento fuente dispara la busqueda semantica y muestra sugerencias en el selector de destino.
- [ ] Al hacer clic en una sugerencia, el selector de destino se pre-selecciona con ese elemento.
- [ ] El boton "Vincular" crea el registro en la tabla pivote correspondiente.
- [ ] El boton "Quitar" elimina el registro pivote y actualiza la lista inmediatamente.
- [ ] Al seleccionar una Linea de Accion, el bloque "Cadena de alineacion" muestra los ODS heredados correctamente.
- [ ] Un usuario sin `gestionar_catalogos` recibe HTTP 403.
- [ ] El test de herencia automatica pasa en verde.
- [ ] No se generan duplicados en las tablas pivote al vincular el mismo par dos veces.

---

## Notas

- La carga de los selects puede ser lenta si hay muchos registros. Considerar usar `wire:model.live` solo en los selects fuente y aplicar lazy loading o busqueda por texto (Livewire `WithPagination`) en los selects destino si el volumen supera 200 registros.
- Las sugerencias semanticas requieren que S2-T11 este completado. Si no esta disponible, `buscarSugerencias` retorna array vacio sin romper la UI (hay manejo de `\Throwable`).
- La "cadena completa" asume que `PedLineaAccion -> PedEstrategia -> PedObjetivoEstrategico` es una relacion 1-a-muchos (many-to-one ascendiendo). Si el modelo tiene relaciones diferentes (ej: linea directamente a objetivo sin estrategia intermedia), ajustar `actualizarCadena`.
