# Plan: S0-T1 — Configurar Laravel Sail con PostgreSQL y pgvector

**Ticket:** S0-T1
**Tipo:** chore
**Rama:** `chore/S0-T1-sail-config`
**Sprint:** 0 — Infraestructura y Entorno
**Depende de:** S0-T0

---

## Contexto

Laravel Sail es la interfaz para el entorno Docker. Dado que el sistema local no tiene PHP, Composer ni Node de forma global (o se prefiere el aislamiento), utilizaremos Docker para la configuración inicial y Sail para el desarrollo diario.

El proyecto está en `laravel/dte-spp-2026`.

---

## Pasos

### 1. Entrar al directorio
```bash
cd laravel/dte-spp-2026
```

### 2. Instalar Laravel Sail (vía Docker)
```bash
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v "$(pwd):/var/www/html" \
    -w /var/www/html \
    laravelsail/php84-composer:latest \
    composer require laravel/sail --dev
```

### 3. Publicar el archivo compose.yaml
```bash
# Usamos un contenedor temporal para ejecutar artisan sail:install
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v "$(pwd):/var/www/html" \
    -w /var/www/html \
    laravelsail/php84-composer:latest \
    php artisan sail:install --with=pgsql,redis
```

### 4. Configurar pgvector
En `compose.yaml`, localizar el servicio `pgsql` y cambiar la imagen:
```yaml
pgsql:
    image: 'pgvector/pgvector:pg16'
    # ... resto de la config (ports, environment, etc.)
```

### 5. Configurar Alias de Sail
Añadir a `~/.bashrc` o `~/.zshrc`:
```bash
alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'
```
Luego: `source ~/.bashrc`

### 6. Levantar el Entorno
```bash
sail up -d
```

### 7. Comandos de Frontend (vía Docker/Sail)
Dado que no hay Node local, usar Sail o un contenedor temporal:
```bash
# Vía Sail (una vez levantado):
sail npm install
sail npm run dev

# O vía Docker directo (si el entorno está abajo):
docker run --rm -v $(pwd):/app -w /app node:22 npm install && npm run build
```

---

## Verificación

### 8. Verificar PostreSQL y Migraciones
```bash
sail artisan db:show
sail artisan migrate
```

---

## Criterios de aceptación

- [x] Alias `sail` configurado y funcional.
- [x] `compose.yaml` usa `pgvector/pgvector:pg16`.
- [x] Contenedores `laravel.test`, `pgsql` y `redis` corriendo.
- [x] Migraciones ejecutadas exitosamente en PostgreSQL.
- [x] Frontend compilado correctamente (vía Sail npm).
