# Plan: S1-T7 — Migración tabla pivote programa_team (Multi-UR)

**Ticket:** S1-T7
**Tipo:** feat
**Rama:** `feat/S1-T7-tabla-programa-team`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S1-T1 (tabla `teams` existe), S1-T2

---

## Contexto

La tabla `programa_team` es el pivote M:M entre programas presupuestarios y equipos/URs. Registra el **rol** de cada UR en cada programa (`coordinadora` o `coadyuvante`).

Adicionalmente, la tabla `mir_niveles` (que se crea en Sprint 3) necesita una columna `team_id` nullable para identificar qué UR Coadyuvante es responsable de cada Componente/Actividad específico. Este ticket crea solo la migración de `programa_team` — la columna en `mir_niveles` se agrega en S3 con una migración alter.

> **Nota:** La tabla `programas_presupuestarios` se crea en S3-T1. La migración de `programa_team` se crea ahora para documentar el esquema, pero la FK a `programas_presupuestarios` debe crearse **después** de que esa tabla exista. Se recomienda crear esta migración como un "stub" que se complete en S3-T1.

---

## Pre-requisitos

- S1-T1 completado (tabla `teams` existe)
- S1-T2 completado (campos de UR en `teams`)

---

## Estrategia

Dado que `programas_presupuestarios` aún no existe, se crean **dos migraciones en este ticket**:

1. **Migración stub** de `programa_team` (sin FK a programas) — se completa en S3-T1
2. La FK real se agrega en S3-T1 cuando `programas_presupuestarios` ya existe

**Alternativa (recomendada):** Crear la migración con la FK a `programas_presupuestarios` directamente en S3-T1, junto con la tabla padre. Este ticket solo crea el archivo de migración como placeholder.

---

## Pasos

### 1. Crear la migración de programa_team

```bash
sail artisan make:migration create_programa_team_table
```

Editar el archivo generado:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('programa_team', function (Blueprint $table) {
            $table->id();

            // FK a programas_presupuestarios (se crea en S3-T1)
            $table->unsignedBigInteger('programa_presupuestario_id');
            $table->foreign('programa_presupuestario_id')
                  ->references('id')
                  ->on('programas_presupuestarios')
                  ->onDelete('cascade');

            // FK a teams
            $table->foreignId('team_id')
                  ->constrained('teams')
                  ->onDelete('cascade');

            // Rol de la UR en el programa
            $table->enum('rol', ['coordinadora', 'coadyuvante']);

            $table->timestamps();

            // Constraint: un team no puede aparecer dos veces en el mismo programa
            $table->unique(['programa_presupuestario_id', 'team_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('programa_team');
    }
};
```

> **Importante:** Esta migración **fallará** si se ejecuta antes de que exista la tabla `programas_presupuestarios`. Debe ejecutarse después de la migración de S3-T1. Asegurarse de que el timestamp del archivo sea posterior al de S3-T1.

### 2. Preparar la migración de alter table mir_niveles

Crear el stub de la migración que se completará en Sprint 3:

```bash
sail artisan make:migration add_team_id_to_mir_niveles_table --table=mir_niveles
```

Editar el archivo:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('mir_niveles', function (Blueprint $table) {
            $table->foreignId('team_id')
                  ->nullable()
                  ->after('orden')
                  ->constrained('teams')
                  ->nullOnDelete();
        });
    }

    public function down(): void
    {
        Schema::table('mir_niveles', function (Blueprint $table) {
            $table->dropForeign(['team_id']);
            $table->dropColumn('team_id');
        });
    }
};
```

> **Importante:** Esta migración también requiere que `mir_niveles` exista (S3-T2). Su timestamp debe ser posterior a S3-T2.

### 3. Preparar el modelo ProgramaPresupuestario con la relación

En S3-T1, al crear el modelo `ProgramaPresupuestario`, agregar:

```php
// Documentar aquí para referencia en Sprint 3
public function equipos(): BelongsToMany
{
    return $this->belongsToMany(Team::class, 'programa_team')
                ->withPivot('rol')
                ->withTimestamps();
}

public function urCoordinadora(): BelongsToMany
{
    return $this->equipos()->wherePivot('rol', 'coordinadora');
}

public function ursCoadyuvantes(): BelongsToMany
{
    return $this->equipos()->wherePivot('rol', 'coadyuvante');
}
```

### 4. Crear seeder de prueba (para Sprint 3)

Documentar el seeder que se implementará en S1-T6 y S3-T1:

```php
// Ejemplo de uso esperado (para referencia):
$programa = ProgramaPresupuestario::find(1);

// Asignar UR Coordinadora
$programa->equipos()->attach($educacion->id, ['rol' => 'coordinadora']);

// Asignar UR Coadyuvante
$programa->equipos()->attach($salud->id, ['rol' => 'coadyuvante']);

// Verificar
$programa->equipos()->wherePivot('rol', 'coordinadora')->first()->name;
// => "Secretaría de Educación"
```

---

## Criterios de aceptación

- [ ] Migración `programa_team` con constraint UNIQUE en `(programa_presupuestario_id, team_id)`
- [ ] Migración alter table `mir_niveles` agrega `team_id` nullable con FK a `teams`
- [ ] ENUM `rol` con valores `coordinadora` y `coadyuvante`
- [ ] Relación `belongsToMany Team via programa_team` documentada para S3-T1
- [ ] Timestamps en `programa_team`

---

## Esquema resultante

```
programa_team
├── id
├── programa_presupuestario_id (FK → programas_presupuestarios.id, CASCADE)
├── team_id                    (FK → teams.id, CASCADE)
├── rol                        (ENUM: coordinadora | coadyuvante)
├── created_at
└── updated_at

UNIQUE(programa_presupuestario_id, team_id)

mir_niveles (alter, se agrega en S3-T2+)
└── team_id (FK nullable → teams.id, SET NULL)
```

---

## Notas

- El orden de ejecución de migraciones es crítico: `programa_team` debe ejecutarse **después** de `programas_presupuestarios`
- Se recomienda renombrar el archivo de migración en Sprint 3 para que su timestamp quede posterior al de `programas_presupuestarios`
- La columna `mir_niveles.team_id` es el mecanismo de asignación de responsabilidad por nivel — se activa desde la interfaz de MIR en S4-T12
