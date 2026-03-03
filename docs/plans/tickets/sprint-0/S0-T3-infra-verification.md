# Plan: S0-T3 — Verificación de Base de Datos e Infraestructura

**Ticket:** S0-T3
**Tipo:** chore
**Rama:** `chore/S0-T3-infra-verification`
**Sprint:** 0 — Infraestructura y Entorno
**Depende de:** S0-T1, S0-T2

---

## Contexto

Antes de proceder con la instalación de Jetstream y el desarrollo de módulos de negocio, es crítico verificar que el "puente" entre Laravel y los servicios de infraestructura (Docker Sail) es estable. 

Este ticket se centra en la verificación de salud (healthcheck) de los servicios y en la validación de tipos de datos no estándar (vectores).

---

## Pre-requisitos

- S0-T1 completado (Sail up)
- S0-T2 completado (Extensión vector habilitada)

---

## Pasos

### 1. Verificar estado de los contenedores

```bash
sail ps
```
Todos los servicios (`pgsql`, `redis`, `laravel.test`) deben indicar "Up" o "Up (healthy)".

### 2. Prueba de conectividad PostgreSQL

Ejecutar un comando SQL simple desde Artisan para confirmar que el driver `pgsql` está bien configurado:

```bash
sail artisan db:show
```

### 3. Validación de tipos vectoriales (pgvector)

Confirmar que podemos realizar operaciones vectoriales básicas desde el entorno de Laravel:

```bash
sail artisan tinker
```

Dentro de Tinker:
```php
// Crear tabla temporal
DB::statement('CREATE TABLE IF NOT EXISTS migration_test (embedding vector(3))');

// Insertar datos
DB::table('migration_test')->insert(['embedding' => '[1,2,3]']);

// Consultar con operación de distancia (operador <->)
$result = DB::select('SELECT embedding <-> "[1,2,4]" as distance FROM migration_test');

echo "Distancia: " . $result[0]->distance; // Debe ser 1.0

// Limpiar
DB::statement('DROP TABLE migration_test');
exit
```

### 4. Prueba de latencia Redis

```bash
sail artisan tinker
```

Dentro de Tinker:
```php
$start = microtime(true);
Cache::store('redis')->put('ping', 'pong', 10);
$val = Cache::store('redis')->get('ping');
$end = microtime(true);

echo "Valor: " . $val . "\n";
echo "Tiempo: " . (($end - $start) * 1000) . "ms\n";
exit
```

---

## Criterios de aceptación

- [ ] Prueba exitosa de CRUD temporal con columnas `vector` desde Laravel.
- [ ] Redis responde a operaciones de caché en menos de 50ms (local).
- [ ] Logs de Sail no muestran errores de "connection refused" o "database not ready".
