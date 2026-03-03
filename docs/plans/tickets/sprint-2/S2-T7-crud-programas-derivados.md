# S2-T7 — CRUD de Programas Derivados con interfaz Livewire

**Tipo:** feat
**Rama:** `feat/S2-T7-crud-programas-derivados`
**Depende de:** S2-T4 (modelos de planes), S1-T3 (permiso gestionar_catalogos)

---

## Contexto

Los Programas Derivados son instrumentos de planeacion que se desprenden del Plan Estatal de Desarrollo (PED). Existen varios tipos (PRODES, PRESUPUESTO, SECTORIAL, etc.). Cada programa derivado contiene objetivos propios que posteriormente se alinean con el PED, PND y ODS a traves de la Matriz de Alineacion.

Este ticket implementa el CRUD completo para la entidad `ProgramaDerivado` y sus `ProgramaDerivadoObjetivo`, con una interfaz Livewire que permite filtrar por tipo mediante tabs o un selector.

---

## Pre-requisitos

- S2-T4 completado: migracion y modelo `ProgramaDerivado`, `ProgramaDerivadoObjetivo` existentes.
- S1-T3 completado: permiso `gestionar_catalogos` registrado en Spatie y asignado al rol correspondiente.
- Columna `activo` en la tabla `ped_planes` para identificar el PED vigente.
- Sail corriendo: `sail up -d`

Verificar que los modelos existen:

```bash
sail artisan tinker --execute="echo App\Models\ProgramaDerivado::count();"
sail artisan tinker --execute="echo App\Models\ProgramaDerivadoObjetivo::count();"
```

---

## Pasos

### 1. Crear el componente Livewire principal

```bash
sail artisan make:livewire Catalogos/ProgramasDerivados/ProgramasDerivadosIndex
sail artisan make:livewire Catalogos/ProgramasDerivados/ProgramaDerivadoForm
```

### 2. Implementar `ProgramasDerivadosIndex`

```php
<?php
// app/Livewire/Catalogos/ProgramasDerivados/ProgramasDerivadosIndex.php

namespace App\Livewire\Catalogos\ProgramasDerivados;

use App\Models\PedPlan;
use App\Models\ProgramaDerivado;
use Livewire\Component;
use Livewire\WithPagination;

class ProgramasDerivadosIndex extends Component
{
    use WithPagination;

    public string $tipoFiltro = 'todos';
    public string $busqueda = '';
    public bool $mostrarFormulario = false;
    public ?int $programaEditandoId = null;

    // Tipos disponibles de programas derivados
    public array $tipos = [
        'PRODES'       => 'Programa de Desarrollo Sectorial',
        'PRESUPUESTO'  => 'Programa Presupuestario',
        'REGIONAL'     => 'Programa Regional',
        'ESPECIAL'     => 'Programa Especial',
        'INSTITUCIONAL' => 'Programa Institucional',
    ];

    protected $listeners = [
        'programaGuardado'   => 'onProgramaGuardado',
        'cancelarFormulario' => 'cerrarFormulario',
    ];

    public function mount(): void
    {
        abort_unless(
            auth()->user()->can('gestionar_catalogos'),
            403,
            'No tienes permiso para gestionar catalogos.'
        );
    }

    public function updatingBusqueda(): void
    {
        $this->resetPage();
    }

    public function updatingTipoFiltro(): void
    {
        $this->resetPage();
    }

    public function filtrarPorTipo(string $tipo): void
    {
        $this->tipoFiltro = $tipo;
        $this->resetPage();
    }

    public function crear(): void
    {
        $this->programaEditandoId = null;
        $this->mostrarFormulario   = true;
    }

    public function editar(int $id): void
    {
        $this->programaEditandoId = $id;
        $this->mostrarFormulario   = true;
    }

    public function eliminar(int $id): void
    {
        $programa = ProgramaDerivado::findOrFail($id);
        $programa->delete();
        $this->dispatch('notify', ['message' => 'Programa eliminado correctamente.', 'type' => 'success']);
    }

    public function onProgramaGuardado(): void
    {
        $this->cerrarFormulario();
        $this->dispatch('notify', ['message' => 'Programa guardado correctamente.', 'type' => 'success']);
    }

    public function cerrarFormulario(): void
    {
        $this->mostrarFormulario   = false;
        $this->programaEditandoId = null;
    }

    public function getPedActivoProperty(): ?PedPlan
    {
        return PedPlan::where('activo', true)->first();
    }

    public function render()
    {
        $query = ProgramaDerivado::query()
            ->with(['pedPlan', 'objetivos'])
            ->when($this->tipoFiltro !== 'todos', fn ($q) => $q->where('tipo', $this->tipoFiltro))
            ->when($this->busqueda, fn ($q) => $q->where('nombre', 'ilike', "%{$this->busqueda}%"))
            ->orderBy('tipo')
            ->orderBy('nombre');

        $programas = $query->paginate(15);

        return view('livewire.catalogos.programas-derivados.programas-derivados-index', [
            'programas'   => $programas,
            'pedActivo'   => $this->pedActivo,
        ]);
    }
}
```

### 3. Implementar `ProgramaDerivadoForm`

```php
<?php
// app/Livewire/Catalogos/ProgramasDerivados/ProgramaDerivadoForm.php

namespace App\Livewire\Catalogos\ProgramasDerivados;

use App\Models\PedPlan;
use App\Models\ProgramaDerivado;
use App\Models\ProgramaDerivadoObjetivo;
use Livewire\Component;

class ProgramaDerivadoForm extends Component
{
    public ?int $programaId = null;

    // Campos del programa
    public string $nombre      = '';
    public string $tipo        = 'PRODES';
    public string $descripcion = '';
    public string $vigencia    = '';
    public ?int   $pedPlanId   = null;

    // Objetivos del programa (array para edicion inline)
    public array $objetivos = [];

    public array $tipos = [
        'PRODES'        => 'Programa de Desarrollo Sectorial',
        'PRESUPUESTO'   => 'Programa Presupuestario',
        'REGIONAL'      => 'Programa Regional',
        'ESPECIAL'      => 'Programa Especial',
        'INSTITUCIONAL' => 'Programa Institucional',
    ];

    protected function rules(): array
    {
        return [
            'nombre'               => 'required|string|max:255',
            'tipo'                 => 'required|string|in:' . implode(',', array_keys($this->tipos)),
            'descripcion'          => 'nullable|string',
            'vigencia'             => 'nullable|string|max:50',
            'pedPlanId'            => 'nullable|exists:ped_planes,id',
            'objetivos'            => 'array',
            'objetivos.*.clave'    => 'required|string|max:50',
            'objetivos.*.nombre'   => 'required|string|max:500',
            'objetivos.*.descripcion' => 'nullable|string',
        ];
    }

    public function mount(?int $programaId = null): void
    {
        $this->pedPlanId = PedPlan::where('activo', true)->value('id');

        if ($programaId) {
            $this->programaId = $programaId;
            $programa = ProgramaDerivado::with('objetivos')->findOrFail($programaId);

            $this->nombre      = $programa->nombre;
            $this->tipo        = $programa->tipo;
            $this->descripcion = $programa->descripcion ?? '';
            $this->vigencia    = $programa->vigencia ?? '';
            $this->pedPlanId   = $programa->ped_plan_id;

            $this->objetivos = $programa->objetivos
                ->map(fn ($obj) => [
                    'id'          => $obj->id,
                    'clave'       => $obj->clave,
                    'nombre'      => $obj->nombre,
                    'descripcion' => $obj->descripcion ?? '',
                ])
                ->toArray();
        }
    }

    public function agregarObjetivo(): void
    {
        $this->objetivos[] = [
            'id'          => null,
            'clave'       => '',
            'nombre'      => '',
            'descripcion' => '',
        ];
    }

    public function eliminarObjetivo(int $index): void
    {
        array_splice($this->objetivos, $index, 1);
        $this->objetivos = array_values($this->objetivos);
    }

    public function guardar(): void
    {
        $validated = $this->validate();

        \DB::transaction(function () use ($validated) {
            $programa = ProgramaDerivado::updateOrCreate(
                ['id' => $this->programaId],
                [
                    'nombre'      => $validated['nombre'],
                    'tipo'        => $validated['tipo'],
                    'descripcion' => $validated['descripcion'],
                    'vigencia'    => $validated['vigencia'],
                    'ped_plan_id' => $validated['pedPlanId'],
                ]
            );

            // Sincronizar objetivos
            $idsExistentes = [];
            foreach ($validated['objetivos'] as $objetivoData) {
                $objetivo = ProgramaDerivadoObjetivo::updateOrCreate(
                    [
                        'id'                  => $objetivoData['id'] ?? null,
                        'programa_derivado_id' => $programa->id,
                    ],
                    [
                        'clave'               => $objetivoData['clave'],
                        'nombre'              => $objetivoData['nombre'],
                        'descripcion'         => $objetivoData['descripcion'],
                        'programa_derivado_id' => $programa->id,
                    ]
                );
                $idsExistentes[] = $objetivo->id;
            }

            // Eliminar objetivos que se quitaron del formulario
            if ($this->programaId) {
                ProgramaDerivadoObjetivo::where('programa_derivado_id', $programa->id)
                    ->whereNotIn('id', $idsExistentes)
                    ->delete();
            }
        });

        $this->dispatch('programaGuardado');
    }

    public function cancelar(): void
    {
        $this->dispatch('cancelarFormulario');
    }

    public function render()
    {
        $pedPlanes = PedPlan::orderByDesc('activo')->orderBy('nombre')->get();

        return view('livewire.catalogos.programas-derivados.programa-derivado-form', [
            'pedPlanes' => $pedPlanes,
        ]);
    }
}
```

### 4. Vista del listado

```blade
{{-- resources/views/livewire/catalogos/programas-derivados/programas-derivados-index.blade.php --}}
<div>
    {{-- Encabezado --}}
    <div class="flex items-center justify-between mb-6">
        <div>
            <h1 class="text-2xl font-bold text-gray-900">Programas Derivados</h1>
            @if($pedActivo)
                <p class="text-sm text-gray-500 mt-1">
                    Vinculados al PED activo: <span class="font-medium">{{ $pedActivo->nombre }}</span>
                </p>
            @else
                <p class="text-sm text-red-500 mt-1">No hay PED activo. Los nuevos programas no podran vincularse.</p>
            @endif
        </div>
        <button wire:click="crear"
                class="inline-flex items-center px-4 py-2 bg-indigo-600 text-white text-sm font-medium rounded-lg hover:bg-indigo-700">
            + Nuevo Programa
        </button>
    </div>

    {{-- Formulario modal --}}
    @if($mostrarFormulario)
        <div class="fixed inset-0 z-50 overflow-y-auto bg-black bg-opacity-40 flex items-start justify-center pt-10">
            <div class="bg-white rounded-xl shadow-2xl w-full max-w-3xl mx-4">
                <livewire:catalogos.programas-derivados.programa-derivado-form
                    :programa-id="$programaEditandoId"
                    :key="'form-' . ($programaEditandoId ?? 'nuevo')" />
            </div>
        </div>
    @endif

    {{-- Tabs de tipo --}}
    <div class="border-b border-gray-200 mb-4">
        <nav class="-mb-px flex space-x-6 overflow-x-auto">
            <button wire:click="filtrarPorTipo('todos')"
                    class="whitespace-nowrap pb-3 px-1 border-b-2 text-sm font-medium
                           {{ $tipoFiltro === 'todos'
                               ? 'border-indigo-500 text-indigo-600'
                               : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300' }}">
                Todos
            </button>
            @foreach($tipos as $clave => $etiqueta)
                <button wire:click="filtrarPorTipo('{{ $clave }}')"
                        class="whitespace-nowrap pb-3 px-1 border-b-2 text-sm font-medium
                               {{ $tipoFiltro === $clave
                                   ? 'border-indigo-500 text-indigo-600'
                                   : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300' }}">
                    {{ $clave }}
                </button>
            @endforeach
        </nav>
    </div>

    {{-- Busqueda --}}
    <div class="mb-4">
        <input wire:model.live.debounce.300ms="busqueda"
               type="text"
               placeholder="Buscar programa por nombre..."
               class="w-full max-w-sm border-gray-300 rounded-lg shadow-sm text-sm focus:ring-indigo-500 focus:border-indigo-500">
    </div>

    {{-- Tabla --}}
    <div class="bg-white shadow rounded-lg overflow-hidden">
        <table class="min-w-full divide-y divide-gray-200">
            <thead class="bg-gray-50">
                <tr>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Tipo</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nombre</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Vigencia</th>
                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Objetivos</th>
                    <th class="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Acciones</th>
                </tr>
            </thead>
            <tbody class="bg-white divide-y divide-gray-200">
                @forelse($programas as $programa)
                    <tr>
                        <td class="px-6 py-4 whitespace-nowrap">
                            <span class="px-2 py-1 text-xs font-semibold rounded-full bg-indigo-100 text-indigo-800">
                                {{ $programa->tipo }}
                            </span>
                        </td>
                        <td class="px-6 py-4">
                            <div class="text-sm font-medium text-gray-900">{{ $programa->nombre }}</div>
                            @if($programa->descripcion)
                                <div class="text-xs text-gray-500 mt-1 line-clamp-2">{{ $programa->descripcion }}</div>
                            @endif
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                            {{ $programa->vigencia ?? '—' }}
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                            {{ $programa->objetivos_count ?? $programa->objetivos->count() }}
                        </td>
                        <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium space-x-2">
                            <button wire:click="editar({{ $programa->id }})"
                                    class="text-indigo-600 hover:text-indigo-900">Editar</button>
                            <button wire:click="eliminar({{ $programa->id }})"
                                    wire:confirm="¿Eliminar este programa y todos sus objetivos?"
                                    class="text-red-600 hover:text-red-900">Eliminar</button>
                        </td>
                    </tr>
                @empty
                    <tr>
                        <td colspan="5" class="px-6 py-12 text-center text-gray-400 text-sm">
                            No hay programas derivados registrados con los filtros actuales.
                        </td>
                    </tr>
                @endforelse
            </tbody>
        </table>

        <div class="px-6 py-4 border-t border-gray-200">
            {{ $programas->links() }}
        </div>
    </div>
</div>
```

### 5. Vista del formulario

```blade
{{-- resources/views/livewire/catalogos/programas-derivados/programa-derivado-form.blade.php --}}
<div class="p-6">
    <h2 class="text-lg font-semibold text-gray-900 mb-6">
        {{ $programaId ? 'Editar Programa Derivado' : 'Nuevo Programa Derivado' }}
    </h2>

    <form wire:submit="guardar" class="space-y-6">

        {{-- Campos principales --}}
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Nombre *</label>
                <input wire:model="nombre" type="text"
                       class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                @error('nombre') <p class="text-red-500 text-xs mt-1">{{ $message }}</p> @enderror
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Tipo *</label>
                <select wire:model="tipo"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    @foreach($tipos as $clave => $etiqueta)
                        <option value="{{ $clave }}">{{ $clave }} — {{ $etiqueta }}</option>
                    @endforeach
                </select>
                @error('tipo') <p class="text-red-500 text-xs mt-1">{{ $message }}</p> @enderror
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Vigencia</label>
                <input wire:model="vigencia" type="text" placeholder="Ej: 2022-2027"
                       class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Vincular al PED</label>
                <select wire:model="pedPlanId"
                        class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500">
                    <option value="">— Sin vincular —</option>
                    @foreach($pedPlanes as $ped)
                        <option value="{{ $ped->id }}">
                            {{ $ped->nombre }}{{ $ped->activo ? ' (activo)' : '' }}
                        </option>
                    @endforeach
                </select>
            </div>
        </div>

        <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Descripcion</label>
            <textarea wire:model="descripcion" rows="3"
                      class="w-full border-gray-300 rounded-lg text-sm focus:ring-indigo-500 focus:border-indigo-500"></textarea>
        </div>

        {{-- Objetivos --}}
        <div>
            <div class="flex items-center justify-between mb-3">
                <h3 class="text-sm font-semibold text-gray-700">Objetivos del Programa</h3>
                <button type="button" wire:click="agregarObjetivo"
                        class="text-sm text-indigo-600 hover:text-indigo-800 font-medium">
                    + Agregar objetivo
                </button>
            </div>

            @if(count($objetivos) === 0)
                <p class="text-sm text-gray-400 italic">Sin objetivos definidos. Agrega al menos uno.</p>
            @endif

            <div class="space-y-3">
                @foreach($objetivos as $i => $objetivo)
                    <div class="border border-gray-200 rounded-lg p-4 bg-gray-50">
                        <div class="flex items-start gap-3">
                            <div class="w-24 flex-shrink-0">
                                <label class="text-xs text-gray-500 mb-1 block">Clave</label>
                                <input wire:model="objetivos.{{ $i }}.clave" type="text"
                                       placeholder="1.1"
                                       class="w-full border-gray-300 rounded text-sm focus:ring-indigo-500 focus:border-indigo-500">
                                @error("objetivos.{$i}.clave") <p class="text-red-500 text-xs mt-1">{{ $message }}</p> @enderror
                            </div>
                            <div class="flex-1">
                                <label class="text-xs text-gray-500 mb-1 block">Nombre del objetivo</label>
                                <input wire:model="objetivos.{{ $i }}.nombre" type="text"
                                       class="w-full border-gray-300 rounded text-sm focus:ring-indigo-500 focus:border-indigo-500">
                                @error("objetivos.{$i}.nombre") <p class="text-red-500 text-xs mt-1">{{ $message }}</p> @enderror
                            </div>
                            <button type="button" wire:click="eliminarObjetivo({{ $i }})"
                                    class="mt-5 text-red-400 hover:text-red-600">
                                <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                                    <path d="M6 2l2-2h4l2 2h4v2H2V2h4zM3 6h14l-1 12H4L3 6zm5 2v8h1V8H8zm3 0v8h1V8h-1z"/>
                                </svg>
                            </button>
                        </div>
                        <div class="mt-2">
                            <label class="text-xs text-gray-500 mb-1 block">Descripcion (opcional)</label>
                            <textarea wire:model="objetivos.{{ $i }}.descripcion" rows="2"
                                      class="w-full border-gray-300 rounded text-sm focus:ring-indigo-500 focus:border-indigo-500"></textarea>
                        </div>
                    </div>
                @endforeach
            </div>
        </div>

        {{-- Botones --}}
        <div class="flex justify-end gap-3 pt-4 border-t border-gray-100">
            <button type="button" wire:click="cancelar"
                    class="px-4 py-2 text-sm text-gray-700 bg-white border border-gray-300 rounded-lg hover:bg-gray-50">
                Cancelar
            </button>
            <button type="submit"
                    class="px-4 py-2 text-sm font-medium text-white bg-indigo-600 rounded-lg hover:bg-indigo-700">
                <span wire:loading.remove>Guardar</span>
                <span wire:loading>Guardando...</span>
            </button>
        </div>
    </form>
</div>
```

### 6. Registrar la ruta

```php
// routes/web.php — dentro del grupo con middleware auth y verified

use App\Livewire\Catalogos\ProgramasDerivados\ProgramasDerivadosIndex;

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/catalogos/programas-derivados', ProgramasDerivadosIndex::class)
        ->name('catalogos.programas-derivados.index');
});
```

### 7. Verificar que el permiso esta disponible

```bash
sail artisan tinker --execute="
    \$perm = Spatie\Permission\Models\Permission::where('name', 'gestionar_catalogos')->first();
    echo \$perm ? 'Permiso OK: ' . \$perm->name : 'FALTA el permiso gestionar_catalogos';
"
```

### 8. Ejecutar tests

```bash
sail artisan make:test Livewire/ProgramasDerivadosTest

# Contenido de ejemplo para el test:
# tests/Feature/Livewire/ProgramasDerivadosTest.php

sail artisan test --filter=ProgramasDerivadosTest
```

Esqueleto del test:

```php
<?php
// tests/Feature/Livewire/ProgramasDerivadosTest.php

namespace Tests\Feature\Livewire;

use App\Livewire\Catalogos\ProgramasDerivados\ProgramasDerivadosIndex;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Livewire\Livewire;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;
use Tests\TestCase;

class ProgramasDerivadosTest extends TestCase
{
    use RefreshDatabase;

    private User $userConPermiso;
    private User $userSinPermiso;

    protected function setUp(): void
    {
        parent::setUp();

        Permission::create(['name' => 'gestionar_catalogos', 'guard_name' => 'web']);
        $rol = Role::create(['name' => 'planeador', 'guard_name' => 'web']);
        $rol->givePermissionTo('gestionar_catalogos');

        $this->userConPermiso = User::factory()->create();
        $this->userConPermiso->assignRole('planeador');

        $this->userSinPermiso = User::factory()->create();
    }

    public function test_usuario_sin_permiso_recibe_403(): void
    {
        Livewire::actingAs($this->userSinPermiso)
            ->test(ProgramasDerivadosIndex::class)
            ->assertForbidden();
    }

    public function test_usuario_con_permiso_ve_el_listado(): void
    {
        Livewire::actingAs($this->userConPermiso)
            ->test(ProgramasDerivadosIndex::class)
            ->assertOk();
    }

    public function test_filtro_por_tipo_actualiza_listado(): void
    {
        Livewire::actingAs($this->userConPermiso)
            ->test(ProgramasDerivadosIndex::class)
            ->call('filtrarPorTipo', 'PRODES')
            ->assertSet('tipoFiltro', 'PRODES');
    }

    public function test_se_puede_abrir_formulario_de_creacion(): void
    {
        Livewire::actingAs($this->userConPermiso)
            ->test(ProgramasDerivadosIndex::class)
            ->call('crear')
            ->assertSet('mostrarFormulario', true)
            ->assertSet('programaEditandoId', null);
    }
}
```

---

## Criterios de Aceptacion

- [ ] La ruta `/catalogos/programas-derivados` devuelve HTTP 403 para usuarios sin `gestionar_catalogos`.
- [ ] La ruta devuelve HTTP 200 para usuarios con `gestionar_catalogos`.
- [ ] Los tabs de tipo filtran correctamente la lista sin recargar la pagina.
- [ ] El buscador por nombre filtra con debounce de 300ms.
- [ ] El formulario de creacion valida campos requeridos (nombre, tipo).
- [ ] Al guardar, los objetivos se crean/actualizan/eliminan en una sola transaccion.
- [ ] Solo se muestran programas vinculados al PED activo cuando se aplica ese filtro.
- [ ] Al eliminar un programa se eliminan en cascada sus objetivos.
- [ ] Todos los tests del archivo `ProgramasDerivadosTest` pasan en verde.

---

## Notas

- El campo `ped_plan_id` en `programas_derivados` debe tener `onDelete('set null')` o `onDelete('restrict')` segun la decision de negocio; se recomienda `restrict` para no perder vinculaciones accidentalmente.
- El componente `ProgramaDerivadoForm` usa `key` dinamica para forzar re-montaje al cambiar entre crear/editar.
- Si el proyecto usa Jetstream Teams, agregar `scoped by team` al query en `ProgramasDerivadosIndex` usando `auth()->user()->currentTeam->id`.
- Los embeddings de `ProgramaDerivadoObjetivo` se generan automaticamente via Observer en S2-T10; este ticket no los toca.
