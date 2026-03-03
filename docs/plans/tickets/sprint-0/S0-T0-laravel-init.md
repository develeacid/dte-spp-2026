# Plan: S0-T0 — Inicializar proyecto Laravel 12 en laravel/dte-spp-2026

**Ticket:** S0-T0
**Tipo:** chore
**Rama:** `chore/S0-T0-laravel-init`
**Sprint:** 0 — Infraestructura y Entorno

---

## Contexto

El objetivo inicial era instalar Laravel 12 en la raíz. Tras revisar que PHP/Composer no estaban instalados localmente, se procedió a crear el proyecto dentro de la carpeta `laravel/dte-spp-2026` utilizando Docker.

---

## Status Actual

- [x] PHP y Composer no instalados localmente.
- [x] Proyecto creado utilizando la imagen de Composer en Docker.
- [x] Permisos de archivos corregidos (chown a 1000:1000).
- [x] Dependencias de frontend instaladas y compiladas (npm install && npm run build).

---

## Pasos Realizados

### 1. Crear el proyecto Laravel vía Docker
```bash
mkdir laravel
docker run --rm -v $(pwd)/laravel:/app -w /app composer create-project laravel/laravel dte-spp-2026
```

### 2. Corregir permisos (Linux)
```bash
docker run --rm -v $(pwd)/laravel/dte-spp-2026:/app alpine chown -R 1000:1000 /app
```

### 3. Generar APP_KEY y Compilar Frontend
```bash
cd laravel/dte-spp-2026
npm install
npm run build
```

---

## Estructura Final

```
.
├── docs/             ← Documentación de planificación
└── laravel/
    └── dte-spp-2026/ ← Raíz del proyecto Laravel
        ├── app/
        ├── artisan
        ├── bootstrap/
        ├── composer.json
        ├── .env
        ├── public/
        ├── resources/
        ├── routes/
        ├── storage/
        └── tests/
```

---

## Criterios de aceptación (Verificados)

- [x] Proyecto instalado en `laravel/dte-spp-2026/`
- [x] `.env` generado con `APP_KEY` válida
- [x] `php artisan --version` muestra `Laravel Framework 12.x.x` (verificado vía Docker)
- [x] `npm run build` compila sin errores
- [x] Carpeta `docs/` intacta en la raíz del repo
