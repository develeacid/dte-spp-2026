# S0-T1 Configuración de Docker Implementation Plan

> **For Antigravity:** REQUIRED SUB-SKILL: Load executing-plans to implement this plan task-by-task.

**Goal:** Levantar la infraestructura base en Docker (Nginx, PHP 8.3, PostgreSQL 16 y Redis) para el Sistema PbR-SED.

**Architecture:** Una red local en Docker con 4 contenedores. Nginx servirá estáticos y proxyará el tráfico PHP hacia el contenedor PHP-FPM. PostgreSQL (con extensión pgvector preparada) almacena los datos transaccionales, y Redis maneja sesiones, colas y cachés.

**Tech Stack:** Docker Compose, Nginx, PHP 8.3-FPM, PostgreSQL 16, Redis 7.

---

### Task 1: Crear archivo .env.example base

**Files:**
- Create: `.env.example`

**Step 1: Crear configuración de variables de entorno**

```text
APP_NAME="Sistema PbR-SED"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8080

DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=secret

REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379
```

**Step 2: Verificar la creación del archivo**

Run: `cat .env.example`
Expected: Mostrar el contenido del archivo con las variables DB y REDIS correctas.

**Step 3: Crear el archivo .env real para local**

Run: `cp .env.example .env`
Expected: Comando ejecutado sin errores.

---

### Task 2: Configurar Nginx

**Files:**
- Create: `docker/nginx/default.conf`

**Step 1: Crear server block para Laravel**

```nginx
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

**Step 2: Verificar la creación del archivo**

Run: `cat docker/nginx/default.conf`
Expected: Mostrar la configuración de nginx.

---

### Task 3: Configurar imagen de PHP

**Files:**
- Create: `docker/php/Dockerfile`
- Create: `docker/php/php.ini`

**Step 1: Crear archivo php.ini base para desarrollo**

En `docker/php/php.ini`:
```ini
[PHP]
memory_limit = 512M
upload_max_filesize = 50M
post_max_size = 50M
max_execution_time = 300
date.timezone = America/Mexico_City
```

**Step 2: Crear Dockerfile optimizado para PHP 8.3 y Laravel 11**

En `docker/php/Dockerfile`:
```dockerfile
FROM php:8.3-fpm

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libpq-dev \
    zip \
    unzip

# Limpiar cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Instalar extensiones PHP necesarias (PostgreSQL, bcmath, pdo, etc)
RUN docker-php-ext-install pdo_pgsql mbstring exif pcntl bcmath gd intl

# Instalar y habilitar Redis
RUN pecl install redis && docker-php-ext-enable redis

# Instalar Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Configurar directorio de trabajo
WORKDIR /var/www/html
```

---

### Task 4: Crear el archivo Docker Compose

**Files:**
- Create: `docker-compose.yml`

**Step 1: Integrar todos los servicios en un docker-compose.yml**

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
    networks:
      - spp-network

  php:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    volumes:
      - ./:/var/www/html
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/custom.ini
    depends_on:
      - postgres
      - redis
    networks:
      - spp-network

  postgres:
    image: postgres:16-alpine # En S0-T2 configuraremos pgvector, por ahora usamos base
    environment:
      POSTGRES_DB: ${DB_DATABASE:-app}
      POSTGRES_USER: ${DB_USERNAME:-app}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - spp-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - spp-network

networks:
  spp-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

**Step 2: Levantar el entorno**

Run: `docker compose up -d`
Expected: Todos los servicios inician correctamente (web, php, postgres, redis).

**Step 3: Verificar que PostgreSQL acepta conexiones**

Run: `docker compose exec postgres psql -U app -d app -c "SELECT 1"`
Expected: Salida de query mostrando `1`.

**Step 4: Verificar que Redis responde**

Run: `docker compose exec redis redis-cli ping`
Expected: `PONG`

---

### Task 5: Crear página de prueba temporal y verificar stack

**Files:**
- Create: `public/index.php`

**Step 1: Crear archivo PHP para verificar Nginx <-> PHP-FPM**

En `public/index.php`:
```php
<?php

echo "Servicios Docker levantados existosamente. PHP-FPM funciona.";
```

**Step 2: Realizar petición HTTP**

Run: `curl http://localhost:8080`
Expected: `Servicios Docker levantados existosamente. PHP-FPM funciona.`

**Step 3: Commit de la infraestructura base**

```bash
git add docker-compose.yml docker/ .env.example
git commit -m "chore(infra): inicializar entorno base docker con nginx, php-fpm, postgres y redis (Fixes PRO-68)"
```
