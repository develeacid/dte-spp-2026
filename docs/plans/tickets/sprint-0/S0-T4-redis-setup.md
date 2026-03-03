# Plan: S0-T4 — Configurar Redis como driver de colas y caché

**Ticket:** S0-T4
**Tipo:** chore
**Rama:** `chore/S0-T4-redis-setup`
**Sprint:** 0 — Infraestructura y Entorno
**Depende de:** S0-T1

---

## Contexto

Laravel soporta múltiples drivers para colas, caché y sesiones. Por defecto usa drivers síncronos o basados en archivo/base de datos. Para producción y para el sistema de jobs asíncronos de IA (S3-T8, S2-T10), se requiere Redis.

Redis ya fue incluido en el `compose.yaml` de Sail en S0-T1. Este ticket configura Laravel para usarlo.

---

## Pre-requisitos

- S0-T1 completado (Sail corriendo con Redis activo)
- `sail up -d` ejecutado

---

## Pasos

### 1. Instalar el cliente PHP de Redis

Sail ya incluye la extensión `phpredis`, pero es necesario instalar el paquete de Laravel:

```bash
sail composer require predis/predis
```

> **Alternativa:** Se puede usar la extensión `phpredis` nativa (ya incluida en Sail). En ese caso, no se necesita `predis/predis`. Verificar con `sail php -m | grep redis`.

### 2. Actualizar `.env` — Colas

```ini
QUEUE_CONNECTION=redis
```

### 3. Actualizar `.env` — Caché

```ini
CACHE_STORE=redis
```

### 4. Actualizar `.env` — Sesiones (recomendado)

```ini
SESSION_DRIVER=redis
```

### 5. Verificar variables de Redis en `.env`

Confirmar que estas variables están presentes (Sail las genera en S0-T1):

```ini
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_CLIENT=predis
```

> Si se usa `phpredis` nativo en lugar de `predis`, omitir `REDIS_CLIENT=predis` o dejarlo como `phpredis`.

### 6. Verificar configuración de cache

```bash
sail artisan config:show cache
```

Debe mostrar `default: redis`.

### 7. Probar la caché de Redis

```bash
sail artisan tinker
```

Dentro de Tinker:
```php
Cache::put('test_key', 'hello_redis', 60);
Cache::get('test_key');
// Debe devolver: "hello_redis"
exit
```

### 8. Probar el sistema de colas

Crear un job de prueba:

```bash
sail artisan make:job TestRedisJob
```

Editar `app/Jobs/TestRedisJob.php`:
```php
public function handle(): void
{
    \Log::info('TestRedisJob ejecutado correctamente en Redis');
}
```

Despachar desde Tinker:
```bash
sail artisan tinker
```
```php
dispatch(new App\Jobs\TestRedisJob());
exit
```

Iniciar el worker para procesar el job:
```bash
sail artisan queue:work --once
```

Verificar en `storage/logs/laravel.log` que aparece:
```
TestRedisJob ejecutado correctamente en Redis
```

### 9. Limpiar el job de prueba

```bash
rm app/Jobs/TestRedisJob.php
```

---

## Criterios de aceptación

- [ ] `.env` configurado con `QUEUE_CONNECTION=redis` y `CACHE_STORE=redis`
- [ ] `Cache::put/get` funcionan correctamente sobre Redis
- [ ] `sail artisan queue:work --once` procesa un job de prueba exitosamente
- [ ] Log confirma ejecución del job
- [ ] Sesiones de usuario configuradas en Redis (`SESSION_DRIVER=redis`)

---

## Configuración de Horizon (opcional, para Sprint 3+)

Para monitoreo avanzado de colas, instalar Laravel Horizon en Sprint 3:
```bash
sail composer require laravel/horizon
sail artisan horizon:install
```

Por ahora, el worker básico con `queue:work` es suficiente para Sprint 0.

---

## Notas

- En desarrollo, `sail artisan queue:work` debe estar corriendo en una terminal separada para procesar jobs
- En producción, el worker se gestiona con Supervisor (incluido en las imágenes de Sail)
- Redis se usa también para broadcasting en tiempo real (futuro) — la configuración actual es compatible
- `SESSION_DRIVER=redis` es opcional para Sprint 0 pero recomendado para evitar problemas de sesión con múltiples workers en sprints posteriores
