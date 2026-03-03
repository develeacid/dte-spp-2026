## Sprint 0: Infraestructura y Entorno

---

### S0-T0: Inicializar proyecto Laravel 12 en raíz

**Tipo:** chore
**Rama:** `chore/S0-T0-laravel-init`

**Descripcion:**
Ejecutar `composer create-project laravel/laravel .` directamente en la raiz del repositorio para que la estructura del framework sea el proyecto principal.

**Criterios de aceptacion:**
- [ ] Directorio raiz contiene la estructura completa de Laravel 12 (app, artisan, composer.json, etc.)
- [ ] `.env` inicial generado correctamente
- [ ] Git detecta los nuevos archivos de Laravel

---

### S0-T1: Configurar Laravel Sail con PostgreSQL y pgvector

**Tipo:** chore
**Rama:** `chore/S0-T1-sail-config`

**Descripcion:**
Instalar Laravel Sail y configurar el archivo `docker-compose.yml` para usar la imagen de PostgreSQL compatible con `pgvector` y el servicio de Redis.

**Servicios requeridos:**
- Laravel Sail (PHP 8.3+)
- PostgreSQL 16 (Imagen: `pgvector/pgvector:pg16`)
- Redis

**Criterios de aceptacion:**
- [ ] `php artisan sail:install` configurado para pgsql y redis
- [ ] `docker-compose.yml` modificado para usar `pgvector/pgvector:pg16`
- [ ] `./vendor/bin/sail up -d` levanta los servicios correctamente
- [ ] Conexión a base de datos PostgreSQL exitosa desde Sail

---

### S0-T2: Habilitar extensión pgvector en PostgreSQL

**Tipo:** chore
**Rama:** `chore/S0-T2-pgvector-setup`

**Descripcion:**
Crear una migración inicial en Laravel para habilitar la extensión `pgvector` en la base de datos PostgreSQL.

**Criterios de aceptacion:**
- [ ] Migración generada con `CREATE EXTENSION IF NOT EXISTS vector`
- [ ] `sail artisan migrate` ejecuta la migración sin errores
- [ ] La extensión `vector` está activa en el esquema de PostgreSQL
- [ ] Prueba manual: se permite crear una tabla con columna tipo `vector(1536)`

---

### S0-T4: Configurar Redis como driver de colas y caché

**Tipo:** chore
**Rama:** `chore/S0-T4-redis-setup`

**Descripcion:**
Configurar las variables de entorno para que Laravel utilice Redis para manejar el sistema de colas, caché y sesiones.

**Criterios de aceptacion:**
- [ ] `.env` configurado con `QUEUE_CONNECTION=redis` y `CACHE_STORE=redis`
- [ ] `sail artisan queue:work` procesa jobs de prueba
- [ ] Cache de Laravel operativa sobre Redis
- [ ] Sesiones de usuario persistidas en Redis

---