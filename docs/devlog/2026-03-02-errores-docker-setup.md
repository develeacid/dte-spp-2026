# Devlog — Errores lógicos durante setup Docker (Sprint 0)

**Fecha:** 2026-03-02  
**Sprint:** S0 — Infraestructura y entorno  
**Tickets afectados:** S0-T1, S0-T2

---

## Error 1 — Dockerfile PHP: faltan dependencias de compilación para PECL

**Ticket:** S0-T1 — Configurar Docker  
**Síntoma:** `docker compose up --build` falla con:
```
Cannot find autoconf. Please check your autoconf installation
```
**Causa:** La extensión Redis se instala vía PECL, que compila desde fuente. La imagen `php:8.3-fpm-alpine` no incluye las herramientas de compilación necesarias.

**Fix:** Agregar al `apk add` del Dockerfile:
```dockerfile
autoconf \
g++ \
make \
```

**Lección para futuros tickets:** Al especificar extensiones PHP instaladas vía PECL (vs. `docker-php-ext-install`), incluir explícitamente en el ticket que se requiere el grupo de buildtools de Alpine: `autoconf g++ make`. Alternativa: usar imagen `php:8.3-fpm` (Debian) donde ya existen.

---

## Error 2 — Dependencia circular: S0-T2 depende de S0-T3

**Tickets:** S0-T2 (pgvector) y S0-T3 (inicialización Laravel)  
**Síntoma:** El plan de S0-T2 pide ejecutar `php artisan migrate` para habilitar la extensión pgvector, pero Laravel no está instalado hasta S0-T3.

**Causa:** El ticket S0-T2 asumió que Laravel ya existía como precondición, pero su única dependencia declarada era S0-T1 (Docker). Faltó declarar dependencia sobre S0-T3.

**Workaround aplicado:** Verificar pgvector directamente vía SQL en el contenedor (`psql -c "CREATE EXTENSION..."`). Dejar la migración como stub para que S0-T3 la ejecute con `php artisan migrate`.

**Lección para futuros tickets:** Cuando un ticket de infraestructura de DB requiere una migración de Laravel, agregar **S0-T3 como dependencia explícita**, o dividir el ticket en dos:
- S0-T2a: verificar que la extensión existe en el contenedor (SQL puro)
- S0-T2b: crear y ejecutar migración Laravel (depende de S0-T3)

---

## Error 3 — Comandos `psql` multi-línea en shell se cuelgan

**Ticket:** S0-T2  
**Síntoma:** `docker compose exec postgres psql -U app -d app -c "..."` con saltos de línea dentro del string se queda esperando input sin ejecutar.

**Causa:** El heredoc mal interpretado por la shell del contenedor Alpine (ash vs bash).

**Fix:** Mantener toda la sentencia SQL en una sola línea dentro del `-c "..."`, o usar `;` como separador.

**Lección para futuros tickets:** En verificaciones manuales con `psql -c`, especificar las sentencias en una sola línea. Si se necesitan múltiples sentencias, usar un archivo `.sql` montado en el contenedor.

---

## Error 4 — Comandos `git log` / `git commit` se cuelgan en background

**Síntoma:** `git commit` y `git log` lanzados como background commands nunca retornan output.  
**Causa probable:** El pager de git (`less`) espera un TTY que no existe en el contexto de ejecución del agente. `git log` activa el pager por default.

**Fix:** Usar `PAGER=cat git log` o la flag `--no-pager`. Para commits, el problema era el background runner — en realidad el commit sí se ejecutó (verificado con `git status` limpio).

**Lección para futuros tickets:** No aplica directamente a tickets, sino al agente: verificar el estado del árbol con `git status --short` en lugar de `git log` para confirmar commits en contextos sin TTY.

---

## Resumen de mejoras para plantillas de tickets

| Área | Mejora propuesta |
|------|-----------------|
| PHP Dockerfile con PECL | Siempre listar `autoconf g++ make` como dependencias de sistema |
| Tickets con migraciones Laravel | Declarar S0-T3 (Laravel instalado) como dependencia explícita |
| Verificaciones SQL | Especificar que los comandos deben ir en una sola línea con `-c "..."` |
| Dependencias entre tickets | Revisar el grafo completo antes de ordenar el sprint: un ticket de DB que usa `artisan` depende implícitamente del ticket de instalación de Laravel |
