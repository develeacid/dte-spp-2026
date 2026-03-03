# Diseño: Docker con Laravel Sail (S0-T1 + S0-T3 fusionados)

**Fecha:** 2026-02-28
**Decisión:** Laravel Sail customizado sobre Dockerfile manual
**PHP:** 8.4
**Tickets:** S0-T1 (Docker), S0-T3 (Laravel TALL), S0-T4 (Redis colas/cache)

---

## Servicios

| Servicio | Imagen | Puerto | Propósito |
|----------|--------|--------|-----------|
| laravel.test | sail-8.4/app (custom) | 80 | PHP-FPM + Nginx (Sail built-in) |
| pgsql | pgvector/pgvector:pg16 | 5432 | Base de datos con soporte vectorial |
| redis | redis:alpine | 6379 | Colas y cache |
| mailpit | axllent/mailpit | 8025 (web), 1025 (SMTP) | Testing de emails |
| pgadmin | dpage/pgadmin4 | 5050 | Administración visual de PostgreSQL |

## Persistencia

Named volumes para datos que sobreviven a `docker compose down`:
- `sail-pgsql` → datos de PostgreSQL
- `sail-redis` → datos de Redis

Para limpiar: `./vendor/bin/sail down -v`

## Pasos de implementación

### 1. Inicializar git y proyecto Laravel
- `git init` + `.gitignore`
- `composer create-project laravel/laravel .`
- Instalar stack TALL: Livewire, Tailwind CSS, Alpine.js

### 2. Instalar y publicar Sail
- `composer require laravel/sail --dev`
- `php artisan sail:install` (seleccionar pgsql + redis)
- `php artisan sail:publish` (Dockerfile editable en `docker/`)

### 3. Customizar docker-compose.yml
- Reemplazar imagen PostgreSQL: `pgvector/pgvector:pg16`
- Agregar servicio Mailpit
- Agregar servicio pgAdmin
- Configurar named volumes

### 4. Configurar .env
- `DB_CONNECTION=pgsql`
- `CACHE_STORE=redis`
- `QUEUE_CONNECTION=redis`
- `MAIL_MAILER=smtp` → Mailpit (localhost:1025)
- Documentar todo en `.env.example`

### 5. Verificación
- `./vendor/bin/sail up -d` levanta todos los servicios
- PostgreSQL acepta conexiones en puerto 5432
- Redis responde a `ping`
- Mailpit accesible en http://localhost:8025
- pgAdmin accesible en http://localhost:5050
- `sail npm run build` compila frontend sin errores
- `sail artisan queue:work` procesa jobs desde Redis

## Criterios de aceptación combinados (S0-T1 + S0-T3 + S0-T4)

- [ ] `./vendor/bin/sail up -d` levanta todos los servicios sin errores
- [ ] PostgreSQL acepta conexiones en el puerto configurado
- [ ] Redis responde a `ping`
- [ ] Archivo `.env.example` documenta las variables de conexión
- [ ] `composer create-project laravel/laravel` ejecutado exitosamente
- [ ] Livewire instalado y funcional
- [ ] Tailwind CSS configurado y `npm run build` compila sin errores
- [ ] Alpine.js disponible en el frontend
- [ ] `.gitignore` configurado correctamente
- [ ] `.env` configurado con `QUEUE_CONNECTION=redis` y `CACHE_STORE=redis`
- [ ] `sail artisan queue:work` procesa un job de prueba
- [ ] Cache de Laravel lee y escribe en Redis
