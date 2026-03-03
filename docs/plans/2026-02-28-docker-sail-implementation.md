# Docker + Laravel Sail + TALL Stack Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Initialize git repo, create Laravel project with Sail (PHP 8.4, PostgreSQL 16 + pgvector, Redis, Mailpit, pgAdmin) and TALL stack.

**Architecture:** Laravel Sail provides Docker-based dev environment. We customize it to swap PostgreSQL for pgvector image and add Mailpit + pgAdmin services. The TALL stack (Tailwind, Alpine.js, Livewire) is installed on top.

**Tech Stack:** Laravel 12, PHP 8.4, PostgreSQL 16 + pgvector, Redis, Livewire 3, Tailwind CSS 4, Alpine.js

**Important:** No PHP/Composer installed locally. All commands run via Docker containers.

**Existing files to preserve:** `docs/`, `.github/`, `.claude/`, `.vscode/`, `contexto.txt` — these must NOT be overwritten during Laravel project creation.

---

### Task 1: Initialize git repository

**Files:**
- Create: `.gitignore` (will be replaced by Laravel's later, but we need git first)

**Step 1: Initialize git**

```bash
cd /home/eleacid/code/planeacion-seguimiento-evaluacion
git init
```

Expected: `Initialized empty Git repository`

**Step 2: Create initial .gitignore**

```gitignore
/vendor
/node_modules
.env
.env.backup
.phpunit.result.cache
storage/*.key
```

**Step 3: Stage existing files and commit**

```bash
git add docs/ .github/ .claude/ .vscode/ contexto.txt .gitignore
git commit -m "chore: initial commit with project docs and plans"
```

Expected: Commit created with existing project files.

**Step 4: Link remote repository and push**

```bash
git remote add origin https://github.com/develeacid/dte-spp-2026.git
git branch -M main
git push -u origin main
```

Expected: Branch `main` pushed to remote. Output includes `Branch 'main' set up to track remote branch 'main'`.

---

### Task 2: Create Laravel project via Docker

Since there's no local PHP/Composer, we use Laravel's official Docker installer. The challenge is that the project directory already has files — we need to install Laravel in a temp directory and merge.

**Step 1: Create Laravel project in temp directory**

```bash
docker run --rm -v /home/eleacid/code:/opt \
  -w /opt \
  laravelsail/php84-composer:latest \
  composer create-project laravel/laravel planeacion-temp --prefer-dist
```

Expected: Laravel project created in `/home/eleacid/code/planeacion-temp/`

**Step 2: Move Laravel files into project directory**

```bash
# Move all Laravel files (excluding our existing dirs) into the project
cd /home/eleacid/code
# Copy Laravel files, skip existing directories
rsync -av --exclude='.git' planeacion-temp/ planeacion-seguimiento-evaluacion/ --ignore-existing
# Move files that rsync skipped because they need overwriting (Laravel core files)
cp planeacion-temp/.gitignore planeacion-seguimiento-evaluacion/.gitignore
cp planeacion-temp/.env.example planeacion-seguimiento-evaluacion/.env.example
cp planeacion-temp/.env.example planeacion-seguimiento-evaluacion/.env
```

Expected: Laravel project files merged into existing directory.

**Step 3: Clean up temp directory**

```bash
rm -rf /home/eleacid/code/planeacion-temp
```

**Step 4: Verify Laravel structure**

```bash
ls /home/eleacid/code/planeacion-seguimiento-evaluacion/artisan
ls /home/eleacid/code/planeacion-seguimiento-evaluacion/composer.json
ls /home/eleacid/code/planeacion-seguimiento-evaluacion/app/
```

Expected: `artisan`, `composer.json`, and `app/` directory exist.

**Step 5: Commit**

```bash
cd /home/eleacid/code/planeacion-seguimiento-evaluacion
git add -A
git commit -m "chore(S0-T3): initialize Laravel 12 project"
```

---

### Task 3: Install Sail with PostgreSQL and Redis

**Files:**
- Modify: `docker-compose.yml` (created by Sail)
- Modify: `.env`

**Step 1: Install Sail and configure**

```bash
cd /home/eleacid/code/planeacion-seguimiento-evaluacion
docker run --rm -v $(pwd):/opt -w /opt \
  laravelsail/php84-composer:latest \
  bash -c "composer require laravel/sail --dev && php artisan sail:install --with=pgsql,redis"
```

Expected: Sail installed, `docker-compose.yml` created with pgsql and redis services.

**Step 2: Publish Sail's Dockerfile for customization**

```bash
docker run --rm -v $(pwd):/opt -w /opt \
  laravelsail/php84-composer:latest \
  php artisan sail:publish
```

Expected: `docker/8.4/` directory created with editable Dockerfile.

**Step 3: Generate app key**

```bash
docker run --rm -v $(pwd):/opt -w /opt \
  laravelsail/php84-composer:latest \
  php artisan key:generate
```

Expected: `APP_KEY` set in `.env`.

**Step 4: Commit**

```bash
git add -A
git commit -m "chore(S0-T1): install Laravel Sail with pgsql and redis"
```

---

### Task 4: Customize docker-compose.yml

**Files:**
- Modify: `docker-compose.yml`

**Step 1: Replace PostgreSQL image with pgvector**

In `docker-compose.yml`, find the `pgsql` service and change:

```yaml
# FROM:
image: 'postgres:17'
# TO:
image: 'pgvector/pgvector:pg16'
```

**Step 2: Add Mailpit service**

Add to `services:` section in `docker-compose.yml`:

```yaml
    mailpit:
        image: 'axllent/mailpit:latest'
        ports:
            - '${FORWARD_MAILPIT_PORT:-1025}:1025'
            - '${FORWARD_MAILPIT_DASHBOARD_PORT:-8025}:8025'
        networks:
            - sail
```

**Step 3: Add pgAdmin service**

Add to `services:` section in `docker-compose.yml`:

```yaml
    pgadmin:
        image: 'dpage/pgadmin4:latest'
        environment:
            PGADMIN_DEFAULT_EMAIL: '${PGADMIN_EMAIL:-admin@admin.com}'
            PGADMIN_DEFAULT_PASSWORD: '${PGADMIN_PASSWORD:-admin}'
            PGADMIN_LISTEN_PORT: 5050
        ports:
            - '${FORWARD_PGADMIN_PORT:-5050}:5050'
        volumes:
            - 'sail-pgadmin:/var/lib/pgadmin'
        networks:
            - sail
        depends_on:
            - pgsql
```

**Step 4: Add pgadmin volume to volumes section**

```yaml
volumes:
    sail-pgsql:
        driver: local
    sail-redis:
        driver: local
    sail-pgadmin:
        driver: local
```

**Step 5: Commit**

```bash
git add docker-compose.yml
git commit -m "chore(S0-T1): add pgvector, mailpit, and pgadmin to docker-compose"
```

---

### Task 5: Configure .env and .env.example

**Files:**
- Modify: `.env`
- Modify: `.env.example`

**Step 1: Update .env with correct values**

Ensure these values are set in `.env`:

```env
APP_NAME="Programas Presupuestales"
APP_URL=http://localhost

DB_CONNECTION=pgsql
DB_HOST=pgsql
DB_PORT=5432
DB_DATABASE=planeacion_seguimiento
DB_USERNAME=sail
DB_PASSWORD=password

REDIS_HOST=redis

CACHE_STORE=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_FROM_ADDRESS="noreply@programas.gob.mx"
MAIL_FROM_NAME="${APP_NAME}"

# pgAdmin
PGADMIN_EMAIL=admin@admin.com
PGADMIN_PASSWORD=admin
```

**Step 2: Mirror the same keys in .env.example**

Copy the same keys with placeholder values to `.env.example`, replacing sensitive values with descriptive placeholders.

**Step 3: Commit**

```bash
git add .env.example
git commit -m "chore(S0-T4): configure redis for cache/queue and mailpit for email"
```

Note: `.env` is in `.gitignore`, only `.env.example` is committed.

---

### Task 6: Install TALL stack (Livewire + Tailwind + Alpine)

**Files:**
- Modify: `composer.json` (Livewire added)
- Modify: `package.json` (Tailwind already included in Laravel 12)

**Step 1: Start Sail**

```bash
cd /home/eleacid/code/planeacion-seguimiento-evaluacion
./vendor/bin/sail up -d
```

Expected: All containers start. Wait ~30s for first build.

**Step 2: Install Livewire**

```bash
./vendor/bin/sail composer require livewire/livewire
```

Expected: Livewire 3.x installed.

**Step 3: Install npm dependencies and build**

```bash
./vendor/bin/sail npm install
./vendor/bin/sail npm run build
```

Expected: Tailwind CSS and Alpine.js (bundled with Laravel 12) compile successfully.

**Step 4: Commit**

```bash
git add -A
git commit -m "feat(S0-T3): install TALL stack (Tailwind, Alpine, Livewire)"
```

---

### Task 7: Verify all services and acceptance criteria

**Step 1: Verify containers are running**

```bash
./vendor/bin/sail ps
```

Expected: 5 services running (laravel.test, pgsql, redis, mailpit, pgadmin).

**Step 2: Verify PostgreSQL**

```bash
./vendor/bin/sail exec pgsql psql -U sail -d planeacion_seguimiento -c "SELECT version();"
```

Expected: PostgreSQL 16.x output.

**Step 3: Verify Redis**

```bash
./vendor/bin/sail exec redis redis-cli ping
```

Expected: `PONG`

**Step 4: Verify Laravel responds**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost
```

Expected: `200`

**Step 5: Run migrations to verify DB connectivity**

```bash
./vendor/bin/sail artisan migrate
```

Expected: Migrations run successfully.

**Step 6: Verify Redis cache**

```bash
./vendor/bin/sail artisan tinker --execute="Cache::put('test', 'works', 60); echo Cache::get('test');"
```

Expected: `works`

**Step 7: Verify Livewire is available**

```bash
./vendor/bin/sail artisan livewire:list 2>&1 | head -3
```

Expected: No errors (empty list or Livewire header).

**Step 8: Verify npm build**

```bash
./vendor/bin/sail npm run build
```

Expected: Build completes without errors.

**Step 9: Check Mailpit and pgAdmin are accessible**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8025
curl -s -o /dev/null -w "%{http_code}" http://localhost:5050
```

Expected: `200` for both.

**Step 10: Final commit if any changes**

```bash
git status
# If there are changes from migrations or config:
git add -A
git commit -m "chore(S0-T1): verify all services running correctly"
```

---

## Summary

| Task | Tickets | Description |
|------|---------|-------------|
| 1 | — | Git init + commit existing docs |
| 2 | S0-T3 | Create Laravel project via Docker |
| 3 | S0-T1 | Install Sail with pgsql + redis |
| 4 | S0-T1 | Customize compose (pgvector, mailpit, pgadmin) |
| 5 | S0-T4 | Configure .env (redis cache/queue, mailpit) |
| 6 | S0-T3 | Install TALL stack |
| 7 | S0-T1/T3/T4 | Verify all acceptance criteria |
