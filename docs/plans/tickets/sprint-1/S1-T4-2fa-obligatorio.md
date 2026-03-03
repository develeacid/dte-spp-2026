# Plan: S1-T4 — Activar 2FA Obligatorio

**Ticket:** S1-T4
**Tipo:** feat
**Rama:** `feat/S1-T4-2fa-obligatorio`
**Sprint:** 1 — Identidad y Aislamiento
**Depende de:** S1-T1 (Jetstream instalado)

---

## Contexto

Jetstream incluye soporte nativo de 2FA (TOTP) mediante el feature `TwoFactorAuthentication`. Por defecto, 2FA es **opcional**. Este ticket lo hace **obligatorio**: cualquier usuario que no tenga 2FA configurado es redirigido a la pantalla de configuración antes de acceder a cualquier ruta protegida.

Jetstream genera el middleware `EnsureUserHasTwoFactorEnabled` pero no lo activa por defecto — es necesario crearlo y registrarlo.

---

## Pre-requisitos

- S1-T1 completado (Jetstream instalado con `--teams`)

---

## Pasos

### 1. Verificar que el feature TwoFactor está habilitado en Jetstream

Abrir `config/jetstream.php` y confirmar que `TwoFactorAuthentication` está en el array `features`:

```php
'features' => [
    Features::profilePhotos(),
    Features::api(),
    Features::teams(['invitations' => false]),
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
],
```

Si `twoFactorAuthentication` no está, agregarlo.

### 2. Crear el middleware de 2FA obligatorio

```bash
sail artisan make:middleware RequireTwoFactorAuthentication
```

Editar `app/Http/Middleware/RequireTwoFactorAuthentication.php`:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Laravel\Fortify\Features;
use Symfony\Component\HttpFoundation\Response;

class RequireTwoFactorAuthentication
{
    public function handle(Request $request, Closure $next): Response
    {
        $user = $request->user();

        if (! $user) {
            return $next($request);
        }

        // Si 2FA no está habilitado para este usuario, redirigir a configuración
        if (Features::enabled(Features::twoFactorAuthentication())
            && ! $user->hasEnabledTwoFactorAuthentication()
        ) {
            // Evitar loop de redirección: permitir acceso a las rutas de perfil/2FA
            if ($request->routeIs('profile.*') || $request->routeIs('two-factor.*')) {
                return $next($request);
            }

            return redirect()->route('profile.show')
                ->with('flash.banner', 'Debes activar la autenticación de dos factores para continuar.')
                ->with('flash.bannerStyle', 'warning');
        }

        return $next($request);
    }
}
```

### 3. Registrar el middleware en el kernel de la aplicación

En Laravel 12, los middlewares se registran en `bootstrap/app.php`. Abrir ese archivo y agregar el middleware al grupo `web`:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        \App\Http\Middleware\RequireTwoFactorAuthentication::class,
    ]);
})
```

> **Alternativa:** Si se prefiere aplicarlo solo a rutas específicas, asignarlo como middleware de grupo en `routes/web.php`:
> ```php
> Route::middleware(['auth:sanctum', 'require.2fa'])->group(function () { ... });
> ```

### 4. Registrar el alias del middleware (opcional)

En `bootstrap/app.php`, dentro de `withMiddleware`:

```php
$middleware->alias([
    'require.2fa' => \App\Http\Middleware\RequireTwoFactorAuthentication::class,
]);
```

### 5. Verificar el flujo completo

1. Registrar un usuario nuevo en `http://localhost/register`
2. Intentar acceder al dashboard — debe redirigir a `/user/profile` con mensaje de advertencia
3. En el perfil, activar 2FA:
   - Click en "Enable Two Factor Authentication"
   - Escanear el código QR con Google Authenticator o similar
   - Confirmar con un código válido
4. Después de activar 2FA, el acceso al dashboard debe funcionar normalmente

### 6. Verificar que las rutas de perfil no crean loop de redirección

Acceder directamente a `http://localhost/user/profile` sin 2FA configurado — debe cargar sin redirecciones infinitas.

---

## Criterios de aceptación

- [ ] Middleware `RequireTwoFactorAuthentication` creado y registrado
- [ ] Usuario sin 2FA configurado es redirigido a `/user/profile` con mensaje de aviso
- [ ] Usuario no puede acceder a ninguna ruta protegida sin 2FA activo
- [ ] Flujo de activación de 2FA funcional (QR + confirmación + códigos de respaldo)
- [ ] Rutas de perfil y configuración de 2FA no generan redirecciones en loop

---

## Notas

- Jetstream almacena el secreto 2FA en `users.two_factor_secret` (encrypted) y los códigos de respaldo en `users.two_factor_recovery_codes`
- El método `$user->hasEnabledTwoFactorAuthentication()` es provisto por el trait `TwoFactorAuthenticatable` de Fortify — disponible en el modelo User después de S1-T1
- Para desarrollo local, se puede usar la app **Google Authenticator**, **Authy** o **1Password** para escanear el QR
- En entornos de prueba (`APP_ENV=testing`), evaluar si se quiere omitir este middleware para facilitar los tests automatizados — agregar condición `if (app()->environment('testing')) return $next($request);`
