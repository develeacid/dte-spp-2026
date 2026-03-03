# S2-T6 — CRUD de PED con interfaz Livewire

**Tipo:** feat
**Rama:** `feat/S2-T6-crud-ped-livewire`
**Depende de:** S2-T3 (modelos PED completos), S1-T3 (permiso `gestionar_catalogos`)

---

## Contexto

Este ticket implementa la interfaz de gestion del Plan Estatal de Desarrollo (PED) usando Livewire v3 dentro del stack TALL (Tailwind, Alpine.js, Livewire, Laravel). La interfaz presenta la jerarquia completa del PED como un arbol colapsable con operaciones CRUD inline en cada nivel, sin modales, con validacion en tiempo real.

La jerarquia del PED es:

```
PedPlan
  └── PedEje
        └── PedTema
              └── PedObjetivoEstrategico
                    └── PedEstrategia
                          └── PedLineaAccion
```

Solo usuarios con el permiso `gestionar_catalogos` (configurado en S1-T3 con Spatie Permissions) pueden acceder a esta interfaz.

---

## Pre-requisitos

- S2-T3 completado: modelos `PedPlan`, `PedEje`, `PedTema`, `PedObjetivoEstrategico`, `PedEstrategia`, `PedLineaAccion` existen con sus relaciones.
- S1-T3 completado: permiso `gestionar_catalogos` creado con Spatie y asignado al rol `admin` y `planeador`.
- Livewire v3 instalado (verifica con `composer show livewire/livewire`).
- Sail corriendo: `sail up -d`

Verificar Livewire:

```bash
sail composer show livewire/livewire | grep versions
```

Verificar que el permiso existe:

```bash
sail artisan tinker --execute="echo \Spatie\Permission\Models\Permission::where('name','gestionar_catalogos')->exists() ? 'OK' : 'FALTA';"
```

---

## Estructura de Archivos

Despues de completar este ticket, los archivos creados o modificados seran:

```
app/
  Livewire/
    Ped/
      GestionPed.php                  <- componente Livewire principal

resources/
  views/
    livewire/
      ped/
        gestion-ped.blade.php         <- vista del componente

routes/
  web.php                             <- ruta protegida con middleware de permiso
```

---

## Pasos

### Paso 1: Crear el componente Livewire

```bash
sail artisan make:livewire Ped/GestionPed
```

Esto genera:
- `app/Livewire/Ped/GestionPed.php`
- `resources/views/livewire/ped/gestion-ped.blade.php`

---

### Paso 2: Implementar el componente PHP

Editar `app/Livewire/Ped/GestionPed.php`:

```php
<?php

namespace App\Livewire\Ped;

use Livewire\Component;
use Livewire\Attributes\Computed;
use Livewire\Attributes\Validate;
use Illuminate\Support\Collection;
use App\Models\PedPlan;
use App\Models\PedEje;
use App\Models\PedTema;
use App\Models\PedObjetivoEstrategico;
use App\Models\PedEstrategia;
use App\Models\PedLineaAccion;

class GestionPed extends Component
{
    // -------------------------------------------------------------------------
    // Estado del formulario activo
    // -------------------------------------------------------------------------

    /** Nivel del nodo en edicion/creacion: 'plan'|'eje'|'tema'|'objetivo'|'estrategia'|'linea' */
    public ?string $formularioActivo = null;

    /** ID del nodo padre para el formulario de creacion */
    public ?int $parentId = null;

    /** ID del nodo en edicion (null si es creacion) */
    public ?int $editandoId = null;

    // -------------------------------------------------------------------------
    // Campos del formulario (compartidos entre todos los niveles)
    // -------------------------------------------------------------------------

    #[Validate('required|string|max:255', as: 'nombre')]
    public string $nombre = '';

    #[Validate('nullable|string|max:500', as: 'descripcion')]
    public string $descripcion = '';

    // -------------------------------------------------------------------------
    // Confirmacion de eliminacion
    // -------------------------------------------------------------------------

    /** ID del nodo pendiente de eliminacion */
    public ?int $eliminandoId = null;

    /** Nivel del nodo pendiente de eliminacion */
    public ?string $eliminandoNivel = null;

    /** Si el nodo a eliminar tiene hijos */
    public bool $tieneHijos = false;

    // -------------------------------------------------------------------------
    // Computed properties
    // -------------------------------------------------------------------------

    /**
     * Carga la jerarquia completa del PED con eager loading.
     * Se recalcula cada vez que el componente se re-renderiza.
     */
    #[Computed]
    public function planes(): Collection
    {
        return PedPlan::with([
            'ejes.temas.objetivosEstrategicos.estrategias.lineasAccion',
        ])->orderBy('nombre')->get();
    }

    // -------------------------------------------------------------------------
    // Acciones del formulario
    // -------------------------------------------------------------------------

    /**
     * Abre el formulario de creacion para un nivel e ID de padre dados.
     */
    public function abrirCrear(string $nivel, ?int $parentId = null): void
    {
        $this->resetFormulario();
        $this->formularioActivo = $nivel;
        $this->parentId = $parentId;
        $this->editandoId = null;
    }

    /**
     * Abre el formulario de edicion para un nodo especifico.
     */
    public function abrirEditar(string $nivel, int $id): void
    {
        $this->resetFormulario();
        $model = $this->resolverModelo($nivel, $id);

        if ($model) {
            $this->formularioActivo = $nivel;
            $this->editandoId = $id;
            $this->nombre = $model->nombre;
            $this->descripcion = $model->descripcion ?? '';
        }
    }

    /**
     * Guarda el formulario (crea o actualiza segun si hay editandoId).
     */
    public function guardar(): void
    {
        $this->validate();

        if ($this->editandoId) {
            $this->actualizar();
        } else {
            $this->crear();
        }

        $this->resetFormulario();
    }

    /**
     * Crea un nuevo nodo en el nivel y padre dados.
     */
    protected function crear(): void
    {
        $datos = [
            'nombre'      => $this->nombre,
            'descripcion' => $this->descripcion ?: null,
        ];

        match ($this->formularioActivo) {
            'plan'      => PedPlan::create($datos),
            'eje'       => PedEje::create(array_merge($datos, ['ped_plan_id' => $this->parentId])),
            'tema'      => PedTema::create(array_merge($datos, ['ped_eje_id' => $this->parentId])),
            'objetivo'  => PedObjetivoEstrategico::create(array_merge($datos, ['ped_tema_id' => $this->parentId])),
            'estrategia'=> PedEstrategia::create(array_merge($datos, ['ped_objetivo_estrategico_id' => $this->parentId])),
            'linea'     => PedLineaAccion::create(array_merge($datos, ['ped_estrategia_id' => $this->parentId])),
            default     => null,
        };

        $this->dispatch('ped-actualizado');
        session()->flash('exito', 'Registro creado correctamente.');
    }

    /**
     * Actualiza un nodo existente.
     */
    protected function actualizar(): void
    {
        $model = $this->resolverModelo($this->formularioActivo, $this->editandoId);

        if ($model) {
            $model->update([
                'nombre'      => $this->nombre,
                'descripcion' => $this->descripcion ?: null,
            ]);
        }

        $this->dispatch('ped-actualizado');
        session()->flash('exito', 'Registro actualizado correctamente.');
    }

    /**
     * Inicia el flujo de confirmacion de eliminacion.
     * Verifica si el nodo tiene hijos para mostrar advertencia.
     */
    public function confirmarEliminar(string $nivel, int $id): void
    {
        $this->eliminandoId    = $id;
        $this->eliminandoNivel = $nivel;
        $this->tieneHijos      = $this->verificarHijos($nivel, $id);
    }

    /**
     * Cancela el flujo de eliminacion.
     */
    public function cancelarEliminar(): void
    {
        $this->eliminandoId    = null;
        $this->eliminandoNivel = null;
        $this->tieneHijos      = false;
    }

    /**
     * Elimina el nodo si no tiene hijos (o si el usuario confirmo conocer la advertencia).
     * No se elimina en cascada silenciosamente: si tiene hijos, no se procede.
     */
    public function eliminar(): void
    {
        if ($this->tieneHijos) {
            session()->flash('error', 'No se puede eliminar: el registro tiene elementos hijo. Eliminalos primero.');
            $this->cancelarEliminar();
            return;
        }

        $model = $this->resolverModelo($this->eliminandoNivel, $this->eliminandoId);
        $model?->delete();

        $this->cancelarEliminar();
        $this->dispatch('ped-actualizado');
        session()->flash('exito', 'Registro eliminado correctamente.');
    }

    /**
     * Cancela el formulario activo y resetea el estado.
     */
    public function cancelar(): void
    {
        $this->resetFormulario();
    }

    // -------------------------------------------------------------------------
    // Helpers privados
    // -------------------------------------------------------------------------

    /**
     * Resetea todos los campos del formulario.
     */
    protected function resetFormulario(): void
    {
        $this->formularioActivo = null;
        $this->parentId         = null;
        $this->editandoId       = null;
        $this->nombre           = '';
        $this->descripcion      = '';
        $this->resetValidation();
    }

    /**
     * Resuelve el modelo Eloquent segun el nivel y el ID.
     */
    protected function resolverModelo(string $nivel, int $id): ?object
    {
        return match ($nivel) {
            'plan'       => PedPlan::find($id),
            'eje'        => PedEje::find($id),
            'tema'       => PedTema::find($id),
            'objetivo'   => PedObjetivoEstrategico::find($id),
            'estrategia' => PedEstrategia::find($id),
            'linea'      => PedLineaAccion::find($id),
            default      => null,
        };
    }

    /**
     * Verifica si un nodo tiene hijos directos.
     */
    protected function verificarHijos(string $nivel, int $id): bool
    {
        return match ($nivel) {
            'plan'       => PedEje::where('ped_plan_id', $id)->exists(),
            'eje'        => PedTema::where('ped_eje_id', $id)->exists(),
            'tema'       => PedObjetivoEstrategico::where('ped_tema_id', $id)->exists(),
            'objetivo'   => PedEstrategia::where('ped_objetivo_estrategico_id', $id)->exists(),
            'estrategia' => PedLineaAccion::where('ped_estrategia_id', $id)->exists(),
            'linea'      => false, // nodo hoja, nunca tiene hijos
            default      => false,
        };
    }

    // -------------------------------------------------------------------------
    // Render
    // -------------------------------------------------------------------------

    public function render()
    {
        return view('livewire.ped.gestion-ped')
            ->layout('layouts.app');
    }
}
```

---

### Paso 3: Implementar la vista Blade

Editar `resources/views/livewire/ped/gestion-ped.blade.php`:

```blade
<div class="max-w-6xl mx-auto py-8 px-4">

    {{-- Encabezado --}}
    <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold text-gray-800">Gestion del Plan Estatal de Desarrollo</h1>
        <button
            wire:click="abrirCrear('plan')"
            class="inline-flex items-center px-4 py-2 bg-indigo-600 text-white text-sm font-medium rounded-md hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-indigo-500"
        >
            + Nuevo Plan PED
        </button>
    </div>

    {{-- Mensajes flash --}}
    @if (session()->has('exito'))
        <div class="mb-4 p-3 bg-green-100 border border-green-300 text-green-800 rounded-md text-sm">
            {{ session('exito') }}
        </div>
    @endif
    @if (session()->has('error'))
        <div class="mb-4 p-3 bg-red-100 border border-red-300 text-red-800 rounded-md text-sm">
            {{ session('error') }}
        </div>
    @endif

    {{-- Confirmacion de eliminacion --}}
    @if ($eliminandoId)
        <div class="mb-4 p-4 bg-yellow-50 border border-yellow-300 rounded-md">
            @if ($tieneHijos)
                <p class="text-yellow-800 font-medium">
                    Advertencia: Este registro tiene elementos hijo. No se puede eliminar hasta eliminar primero todos sus hijos.
                </p>
            @else
                <p class="text-yellow-800 font-medium">
                    Esta accion no se puede deshacer. Confirma la eliminacion del registro.
                </p>
            @endif
            <div class="mt-3 flex gap-2">
                @if (!$tieneHijos)
                    <button
                        wire:click="eliminar"
                        class="px-3 py-1.5 bg-red-600 text-white text-sm rounded-md hover:bg-red-700"
                    >
                        Confirmar eliminacion
                    </button>
                @endif
                <button
                    wire:click="cancelarEliminar"
                    class="px-3 py-1.5 bg-gray-200 text-gray-700 text-sm rounded-md hover:bg-gray-300"
                >
                    Cancelar
                </button>
            </div>
        </div>
    @endif

    {{-- Formulario inline para nuevo Plan --}}
    @if ($formularioActivo === 'plan' && !$editandoId)
        @include('livewire.ped._formulario-inline', ['titulo' => 'Nuevo Plan PED', 'nivel' => 'plan'])
    @endif

    {{-- Arbol de planes --}}
    @forelse ($this->planes as $plan)
        {{-- NIVEL 1: Plan --}}
        <div
            x-data="{ abierto: true }"
            class="mb-4 border border-gray-200 rounded-lg overflow-hidden"
        >
            {{-- Cabecera del Plan --}}
            <div class="flex items-center gap-2 p-3 bg-indigo-50 border-b border-gray-200">
                <button @click="abierto = !abierto" class="text-indigo-600 hover:text-indigo-800 focus:outline-none">
                    <svg x-show="!abierto" class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
                    </svg>
                    <svg x-show="abierto" class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
                    </svg>
                </button>
                <span class="font-semibold text-indigo-800 flex-1">{{ $plan->nombre }}</span>
                @include('livewire.ped._acciones-nodo', [
                    'nivel'    => 'plan',
                    'id'       => $plan->id,
                    'hijoNivel'=> 'eje',
                    'hijoLabel'=> 'Eje',
                    'color'    => 'indigo',
                ])
            </div>

            {{-- Formulario edicion Plan --}}
            @if ($formularioActivo === 'plan' && $editandoId === $plan->id)
                <div class="p-3 bg-indigo-25 border-b border-gray-200">
                    @include('livewire.ped._formulario-inline', ['titulo' => 'Editar Plan PED', 'nivel' => 'plan'])
                </div>
            @endif

            {{-- Formulario nuevo Eje dentro de este Plan --}}
            @if ($formularioActivo === 'eje' && $parentId === $plan->id && !$editandoId)
                <div class="p-3 bg-blue-25 border-b border-gray-200">
                    @include('livewire.ped._formulario-inline', ['titulo' => 'Nuevo Eje', 'nivel' => 'eje'])
                </div>
            @endif

            {{-- NIVEL 2: Ejes --}}
            <div x-show="abierto" x-transition class="divide-y divide-gray-100">
                @foreach ($plan->ejes->sortBy('nombre') as $eje)
                    <div x-data="{ abierto: false }" class="pl-4">

                        {{-- Cabecera del Eje --}}
                        <div class="flex items-center gap-2 p-2 bg-blue-50 hover:bg-blue-100">
                            <button @click="abierto = !abierto" class="text-blue-500 hover:text-blue-700 focus:outline-none">
                                <svg x-show="!abierto" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
                                </svg>
                                <svg x-show="abierto" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
                                </svg>
                            </button>
                            <span class="text-blue-800 font-medium flex-1 text-sm">{{ $eje->nombre }}</span>
                            @include('livewire.ped._acciones-nodo', [
                                'nivel'    => 'eje',
                                'id'       => $eje->id,
                                'hijoNivel'=> 'tema',
                                'hijoLabel'=> 'Tema',
                                'color'    => 'blue',
                            ])
                        </div>

                        {{-- Formulario edicion Eje --}}
                        @if ($formularioActivo === 'eje' && $editandoId === $eje->id)
                            <div class="p-3 bg-blue-25 border-b border-gray-200 ml-4">
                                @include('livewire.ped._formulario-inline', ['titulo' => 'Editar Eje', 'nivel' => 'eje'])
                            </div>
                        @endif

                        {{-- Formulario nuevo Tema dentro de este Eje --}}
                        @if ($formularioActivo === 'tema' && $parentId === $eje->id && !$editandoId)
                            <div class="p-3 bg-green-25 border-b border-gray-200 ml-4">
                                @include('livewire.ped._formulario-inline', ['titulo' => 'Nuevo Tema', 'nivel' => 'tema'])
                            </div>
                        @endif

                        {{-- NIVEL 3: Temas --}}
                        <div x-show="abierto" x-transition class="ml-4 divide-y divide-gray-100">
                            @foreach ($eje->temas->sortBy('nombre') as $tema)
                                <div x-data="{ abierto: false }" class="pl-4">

                                    {{-- Cabecera del Tema --}}
                                    <div class="flex items-center gap-2 p-2 bg-green-50 hover:bg-green-100">
                                        <button @click="abierto = !abierto" class="text-green-500 hover:text-green-700 focus:outline-none">
                                            <svg x-show="!abierto" class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
                                            </svg>
                                            <svg x-show="abierto" class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
                                            </svg>
                                        </button>
                                        <span class="text-green-800 flex-1 text-sm">{{ $tema->nombre }}</span>
                                        @include('livewire.ped._acciones-nodo', [
                                            'nivel'    => 'tema',
                                            'id'       => $tema->id,
                                            'hijoNivel'=> 'objetivo',
                                            'hijoLabel'=> 'Objetivo',
                                            'color'    => 'green',
                                        ])
                                    </div>

                                    {{-- Formulario edicion Tema --}}
                                    @if ($formularioActivo === 'tema' && $editandoId === $tema->id)
                                        <div class="p-3 ml-4">
                                            @include('livewire.ped._formulario-inline', ['titulo' => 'Editar Tema', 'nivel' => 'tema'])
                                        </div>
                                    @endif

                                    {{-- Formulario nuevo Objetivo dentro de este Tema --}}
                                    @if ($formularioActivo === 'objetivo' && $parentId === $tema->id && !$editandoId)
                                        <div class="p-3 ml-4">
                                            @include('livewire.ped._formulario-inline', ['titulo' => 'Nuevo Objetivo Estrategico', 'nivel' => 'objetivo'])
                                        </div>
                                    @endif

                                    {{-- NIVEL 4: Objetivos Estrategicos --}}
                                    <div x-show="abierto" x-transition class="ml-4">
                                        @foreach ($tema->objetivosEstrategicos->sortBy('nombre') as $objetivo)
                                            <div x-data="{ abierto: false }" class="pl-4">

                                                <div class="flex items-center gap-2 p-2 bg-yellow-50 hover:bg-yellow-100">
                                                    <button @click="abierto = !abierto" class="text-yellow-500 hover:text-yellow-700 focus:outline-none">
                                                        <svg x-show="!abierto" class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
                                                        </svg>
                                                        <svg x-show="abierto" class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
                                                        </svg>
                                                    </button>
                                                    <span class="text-yellow-800 flex-1 text-sm">{{ $objetivo->nombre }}</span>
                                                    @include('livewire.ped._acciones-nodo', [
                                                        'nivel'    => 'objetivo',
                                                        'id'       => $objetivo->id,
                                                        'hijoNivel'=> 'estrategia',
                                                        'hijoLabel'=> 'Estrategia',
                                                        'color'    => 'yellow',
                                                    ])
                                                </div>

                                                @if ($formularioActivo === 'objetivo' && $editandoId === $objetivo->id)
                                                    <div class="p-3 ml-4">
                                                        @include('livewire.ped._formulario-inline', ['titulo' => 'Editar Objetivo Estrategico', 'nivel' => 'objetivo'])
                                                    </div>
                                                @endif

                                                @if ($formularioActivo === 'estrategia' && $parentId === $objetivo->id && !$editandoId)
                                                    <div class="p-3 ml-4">
                                                        @include('livewire.ped._formulario-inline', ['titulo' => 'Nueva Estrategia', 'nivel' => 'estrategia'])
                                                    </div>
                                                @endif

                                                {{-- NIVEL 5: Estrategias --}}
                                                <div x-show="abierto" x-transition class="ml-4">
                                                    @foreach ($objetivo->estrategias->sortBy('nombre') as $estrategia)
                                                        <div x-data="{ abierto: false }" class="pl-4">

                                                            <div class="flex items-center gap-2 p-2 bg-orange-50 hover:bg-orange-100">
                                                                <button @click="abierto = !abierto" class="text-orange-400 hover:text-orange-600 focus:outline-none">
                                                                    <svg x-show="!abierto" class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
                                                                    </svg>
                                                                    <svg x-show="abierto" class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
                                                                    </svg>
                                                                </button>
                                                                <span class="text-orange-800 flex-1 text-xs">{{ $estrategia->nombre }}</span>
                                                                @include('livewire.ped._acciones-nodo', [
                                                                    'nivel'    => 'estrategia',
                                                                    'id'       => $estrategia->id,
                                                                    'hijoNivel'=> 'linea',
                                                                    'hijoLabel'=> 'Linea de Accion',
                                                                    'color'    => 'orange',
                                                                ])
                                                            </div>

                                                            @if ($formularioActivo === 'estrategia' && $editandoId === $estrategia->id)
                                                                <div class="p-3 ml-4">
                                                                    @include('livewire.ped._formulario-inline', ['titulo' => 'Editar Estrategia', 'nivel' => 'estrategia'])
                                                                </div>
                                                            @endif

                                                            @if ($formularioActivo === 'linea' && $parentId === $estrategia->id && !$editandoId)
                                                                <div class="p-3 ml-4">
                                                                    @include('livewire.ped._formulario-inline', ['titulo' => 'Nueva Linea de Accion', 'nivel' => 'linea'])
                                                                </div>
                                                            @endif

                                                            {{-- NIVEL 6: Lineas de Accion (nodo hoja) --}}
                                                            <div x-show="abierto" x-transition class="ml-4">
                                                                @foreach ($estrategia->lineasAccion->sortBy('nombre') as $linea)
                                                                    <div class="flex items-center gap-2 p-2 bg-gray-50 hover:bg-gray-100 pl-6">
                                                                        <span class="text-gray-700 flex-1 text-xs">{{ $linea->nombre }}</span>
                                                                        @include('livewire.ped._acciones-nodo', [
                                                                            'nivel'    => 'linea',
                                                                            'id'       => $linea->id,
                                                                            'hijoNivel'=> null,
                                                                            'hijoLabel'=> null,
                                                                            'color'    => 'gray',
                                                                        ])
                                                                    </div>

                                                                    @if ($formularioActivo === 'linea' && $editandoId === $linea->id)
                                                                        <div class="p-3 ml-6">
                                                                            @include('livewire.ped._formulario-inline', ['titulo' => 'Editar Linea de Accion', 'nivel' => 'linea'])
                                                                        </div>
                                                                    @endif
                                                                @endforeach
                                                            </div>

                                                        </div>
                                                    @endforeach
                                                </div>

                                            </div>
                                        @endforeach
                                    </div>

                                </div>
                            @endforeach
                        </div>

                    </div>
                @endforeach
            </div>
        </div>
    @empty
        <div class="text-center py-12 text-gray-500">
            <p class="text-lg">No hay planes PED registrados.</p>
            <p class="text-sm mt-1">Crea el primer plan usando el boton "Nuevo Plan PED".</p>
        </div>
    @endforelse

</div>
```

---

### Paso 4: Crear la partial `_formulario-inline.blade.php`

Crear `resources/views/livewire/ped/_formulario-inline.blade.php`:

```blade
<div class="bg-white border border-gray-300 rounded-md p-4 shadow-sm">
    <h3 class="text-sm font-semibold text-gray-700 mb-3">{{ $titulo }}</h3>
    <form wire:submit="guardar" class="space-y-3">

        {{-- Campo Nombre --}}
        <div>
            <label class="block text-xs font-medium text-gray-600 mb-1">
                Nombre <span class="text-red-500">*</span>
            </label>
            <input
                type="text"
                wire:model.live="nombre"
                maxlength="255"
                class="w-full border border-gray-300 rounded-md px-3 py-1.5 text-sm focus:ring-2 focus:ring-indigo-400 focus:border-indigo-400 @error('nombre') border-red-400 @enderror"
                placeholder="Nombre del {{ strtolower($titulo) }}"
            />
            @error('nombre')
                <p class="text-red-500 text-xs mt-1">{{ $message }}</p>
            @enderror
        </div>

        {{-- Campo Descripcion --}}
        <div>
            <label class="block text-xs font-medium text-gray-600 mb-1">Descripcion</label>
            <textarea
                wire:model.live="descripcion"
                maxlength="500"
                rows="2"
                class="w-full border border-gray-300 rounded-md px-3 py-1.5 text-sm focus:ring-2 focus:ring-indigo-400 focus:border-indigo-400 @error('descripcion') border-red-400 @enderror"
                placeholder="Descripcion (opcional, max 500 caracteres)"
            ></textarea>
            @error('descripcion')
                <p class="text-red-500 text-xs mt-1">{{ $message }}</p>
            @enderror
            <p class="text-xs text-gray-400 mt-1">{{ strlen($descripcion) }}/500</p>
        </div>

        {{-- Botones --}}
        <div class="flex items-center gap-2 pt-1">
            <button
                type="submit"
                class="px-3 py-1.5 bg-indigo-600 text-white text-xs font-medium rounded-md hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-indigo-400"
            >
                Guardar
            </button>
            <button
                type="button"
                wire:click="cancelar"
                class="px-3 py-1.5 bg-gray-200 text-gray-700 text-xs rounded-md hover:bg-gray-300"
            >
                Cancelar
            </button>
        </div>

    </form>
</div>
```

---

### Paso 5: Crear la partial `_acciones-nodo.blade.php`

Crear `resources/views/livewire/ped/_acciones-nodo.blade.php`:

```blade
<div class="flex items-center gap-1 shrink-0">
    {{-- Boton crear hijo (solo si hay nivel hijo) --}}
    @if ($hijoNivel)
        <button
            wire:click="abrirCrear('{{ $hijoNivel }}', {{ $id }})"
            title="Agregar {{ $hijoLabel }}"
            class="px-2 py-0.5 text-xs bg-{{ $color }}-100 text-{{ $color }}-700 rounded hover:bg-{{ $color }}-200 focus:outline-none"
        >
            + {{ $hijoLabel }}
        </button>
    @endif

    {{-- Boton editar --}}
    <button
        wire:click="abrirEditar('{{ $nivel }}', {{ $id }})"
        title="Editar"
        class="px-2 py-0.5 text-xs bg-gray-100 text-gray-600 rounded hover:bg-gray-200 focus:outline-none"
    >
        Editar
    </button>

    {{-- Boton eliminar --}}
    <button
        wire:click="confirmarEliminar('{{ $nivel }}', {{ $id }})"
        title="Eliminar"
        class="px-2 py-0.5 text-xs bg-red-50 text-red-600 rounded hover:bg-red-100 focus:outline-none"
    >
        Eliminar
    </button>
</div>
```

> **Nota sobre clases Tailwind dinamicas**: El uso de `bg-{{ $color }}-100` puede no funcionar correctamente con el purging de Tailwind en produccion porque Tailwind no reconoce clases generadas dinamicamente. Para produccion, usar la lista `safelist` en `tailwind.config.js` o usar clases completas con `@switch`. El ejemplo anterior es funcional en desarrollo. Ver la nota al final del documento para la solucion de produccion.

---

### Paso 6: Agregar la ruta protegida

Editar `routes/web.php` y agregar dentro del grupo de rutas autenticadas de Jetstream:

```php
use App\Livewire\Ped\GestionPed;

// Dentro del grupo Route::middleware(['auth', 'verified']) o similar:
Route::middleware(['auth', 'verified', 'permission:gestionar_catalogos'])
    ->group(function () {
        Route::get('/catalogos/ped', GestionPed::class)->name('catalogos.ped');
    });
```

Si se usa Jetstream con el layout de app existente, verificar que el grupo de rutas principal ya incluye los middlewares de auth. En ese caso, agregar solo el middleware de permiso:

```php
Route::get('/catalogos/ped', GestionPed::class)
    ->middleware(['auth', 'verified', 'permission:gestionar_catalogos'])
    ->name('catalogos.ped');
```

El middleware `permission:gestionar_catalogos` es provisto por Spatie Laravel Permission y redirige automaticamente con 403 si el usuario no tiene el permiso.

---

### Paso 7: Agregar enlace en la navegacion (opcional)

En el layout de navegacion de Jetstream (`resources/views/navigation-menu.blade.php` o el equivalente), agregar:

```blade
@can('gestionar_catalogos')
    <x-nav-link href="{{ route('catalogos.ped') }}" :active="request()->routeIs('catalogos.ped')">
        PED — Plan Estatal de Desarrollo
    </x-nav-link>
@endcan
```

---

### Paso 8: Verificar la solucion de clases Tailwind dinamicas en produccion

Editar `tailwind.config.js` para agregar la lista de clases seguras:

```javascript
module.exports = {
    // ...
    safelist: [
        // Colores de fondo para niveles del arbol PED
        { pattern: /bg-(indigo|blue|green|yellow|orange|gray)-(50|100|200)/ },
        { pattern: /text-(indigo|blue|green|yellow|orange|gray)-(600|700|800)/ },
        { pattern: /border-(indigo|blue|green|yellow|orange|gray)-(300|400)/ },
        { pattern: /hover:bg-(indigo|blue|green|yellow|orange|gray)-(100|200)/ },
    ],
    // ...
}
```

Luego recompilar assets:

```bash
sail npm run build
```

---

### Paso 9: Verificar el componente

```bash
# Verificar que el componente fue creado correctamente
sail artisan livewire:list | grep GestionPed

# Limpiar cache de vistas
sail artisan view:clear
sail artisan cache:clear

# Iniciar servidor de desarrollo
sail npm run dev
```

Navegar a `http://localhost/catalogos/ped` con un usuario que tenga el permiso `gestionar_catalogos`.

---

### Paso 10: Verificar control de acceso

```bash
sail artisan tinker
```

```php
// Verificar que el middleware de permiso funciona
// Con usuario sin permiso: debe retornar 403
// Con usuario admin: debe mostrar la interfaz

// Asignar permiso a un usuario de prueba:
$user = App\Models\User::find(1);
$user->givePermissionTo('gestionar_catalogos');

// Verificar:
$user->can('gestionar_catalogos'); // true
```

---

## Criterios de Aceptacion

| # | Criterio | Verificacion |
|---|----------|--------------|
| 1 | Solo accesible con permiso `gestionar_catalogos` | Acceso con usuario sin permiso retorna 403; con permiso carga la vista |
| 2 | Arbol colapsable renderiza los 6 niveles de la jerarquia PED | Navegar la vista y colapsar/expandir cada nivel |
| 3 | Boton "+ Eje/Tema/..." abre formulario inline sin recargar pagina | Al hacer clic, el formulario aparece sin navegacion |
| 4 | Formulario inline valida en tiempo real (nombre requerido, descripcion max 500) | Dejar campo vacio y escribir mas de 500 chars: aparecen mensajes de error |
| 5 | Crear registro: aparece en el arbol despues de guardar | Crear un nuevo eje y verificar que aparece bajo el plan correspondiente |
| 6 | Editar registro: formulario se prellena con valores actuales | Hacer clic en "Editar" y verificar que nombre y descripcion estan precargados |
| 7 | Eliminar nodo con hijos: muestra advertencia y NO elimina | Intentar eliminar un plan que tiene ejes: aparece la advertencia, no se elimina |
| 8 | Eliminar nodo hoja: pide confirmacion y elimina | Eliminar una linea de accion: pide confirmar y desaparece del arbol |
| 9 | Responsive con Tailwind en movil (sm) y escritorio (md+) | Abrir en viewport de 375px: la interfaz es usable |
| 10 | `sail artisan livewire:list` muestra `Ped\GestionPed` | Comando ejecuta sin errores y lista el componente |

---

## Notas

- **Componente vs. pagina completa**: El componente se monta como pagina completa (`->layout('layouts.app')`). Si se prefiere embeber dentro de otra vista Blade, usar `<livewire:ped.gestion-ped />`.

- **Numero N+1 con el arbol**: El eager loading `with(['ejes.temas.objetivosEstrategicos.estrategias.lineasAccion'])` en el `#[Computed]` previene el problema N+1. El atributo `#[Computed]` de Livewire v3 cachea el resultado durante el ciclo de vida del request; no hace una nueva query SQL en cada render mientras no haya cambios de estado.

- **Validacion en tiempo real**: `wire:model.live` (equivalente a `wire:model.debounce.300ms` en Livewire v3) activa la validacion mientras el usuario escribe. Para validacion solo al submit, usar `wire:model` sin `.live`.

- **Formularios inline vs. modales**: La decision de usar formularios inline (sin modales) simplifica el manejo del estado y mejora la accesibilidad. El costo es que un solo formulario puede estar abierto a la vez, lo cual se gestiona con `$formularioActivo`.

- **Performance con arboles grandes**: Si el PED tiene cientos de nodos, el arbol completo puede volverse lento. En ese caso, considerar lazy loading de nodos hijo via `wire:click` + `$this->dispatch('load-children', nivel: 'eje', parentId: $id)`, cargando solo el primer nivel y expandiendo bajo demanda.

- **Clases Tailwind dinamicas en produccion**: Como se indica en el Paso 8, las clases generadas dinamicamente (`bg-{{ $color }}-100`) requieren configuracion en `tailwind.config.js`. La alternativa mas simple es usar un array de clases en el componente PHP y pasarlo a la vista, o usar estilos inline para los colores de nivel.

- **Middleware de Spatie en rutas**: `permission:gestionar_catalogos` es el alias registrado por el ServiceProvider de Spatie. Si el alias no esta registrado, verificar en `config/permission.php` o registrar manualmente en `app/Http/Kernel.php` (Laravel 10) o en `bootstrap/app.php` (Laravel 11/12).
