# Plan: S0-T2 — Habilitar extensión pgvector en PostgreSQL

**Ticket:** S0-T2
**Tipo:** chore
**Rama:** `chore/S0-T2-pgvector-setup`
**Sprint:** 0 — Infraestructura y Entorno
**Depende de:** S0-T1

---

## Contexto

La imagen `pgvector/pgvector:pg16` incluye la extensión `vector` compilada, pero **no la activa automáticamente**. Se debe ejecutar `CREATE EXTENSION vector` en la base de datos.

En Laravel, la práctica correcta es hacer esto a través de una migración, de forma que sea reproducible en cualquier entorno (desarrollo, staging, producción).

Esta extensión es el fundamento de los embeddings vectoriales que se usarán en Sprints 2+ para búsqueda semántica y alineación automática de planes.

---

## Pre-requisitos

- S0-T1 completado (Sail corriendo con `pgvector/pgvector:pg16`)
- `sail up -d` ejecutado

---

## Pasos

### 1. Crear la migración para habilitar pgvector

```bash
sail artisan make:migration enable_pgvector_extension
```

Esto genera un archivo en `database/migrations/` con timestamp. Ejemplo:
`2026_03_02_000000_enable_pgvector_extension.php`

### 2. Editar la migración

Abrir el archivo generado y reemplazar su contenido con:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        DB::statement('CREATE EXTENSION IF NOT EXISTS vector');
    }

    public function down(): void
    {
        DB::statement('DROP EXTENSION IF EXISTS vector');
    }
};
```

> **Nota:** `IF NOT EXISTS` hace la migración idempotente — puede ejecutarse múltiples veces sin error.

### 3. Ejecutar la migración

```bash
sail artisan migrate
```

Debe mostrar algo como:
```
Running migrations.
  2026_03_02_000000_enable_pgvector_extension ............... 24ms DONE
```

### 4. Verificar que la extensión está activa

```bash
sail shell
psql -U sail -d laravel -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
exit
```

Resultado esperado:
```
 extname | extversion
---------+------------
 vector  | 0.8.x
(1 row)
```

### 5. Prueba de columna vectorial

Verificar que PostgreSQL acepta columnas de tipo `vector(1536)`:

```bash
sail artisan tinker
```

Dentro de Tinker:
```php
DB::statement('CREATE TABLE IF NOT EXISTS test_vector (id serial, embedding vector(1536))');
// No debe arrojar error

DB::statement('DROP TABLE IF EXISTS test_vector');
// Limpiar
exit
```

---

## Criterios de aceptación

- [ ] Migración generada con `CREATE EXTENSION IF NOT EXISTS vector`
- [ ] `sail artisan migrate` ejecuta la migración sin errores
- [ ] La extensión `vector` aparece en `pg_extension`
- [ ] Prueba manual: se permite crear una tabla con columna tipo `vector(1536)`

---

## Notas

- Esta migración debe ser la **primera** de todas las demás — debe ejecutarse antes de cualquier migración que use columnas `vector(n)`
- Nombrar el archivo con timestamp `000000` o asegurarse de que quede ordenado primero en `database/migrations/`
- Para futuros sprints: los modelos que requieran columnas vectoriales usarán `$table->vector('embedding', 1536)` si se instala el paquete `tpetry/laravel-postgresql-enhanced`, o `$table->addColumn('vector', 'embedding', ['dimensions' => 1536])` con raw DDL
- La dimensión `1536` corresponde al modelo `text-embedding-ada-002` de OpenAI. Si se usa otro modelo de embeddings, ajustar la dimensión en consecuencia
