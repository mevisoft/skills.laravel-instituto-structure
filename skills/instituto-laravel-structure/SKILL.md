---
name: instituto-laravel-structure
description: Institute management conventions for Laravel + Inertia + React. Activates when creating domain modules (admissions, maintenances, payments, portals), permissions, navigation, tables, controllers, or React pages for the Metropolitano virtual campus.
---

# Instituto — Convenciones

Skill del sistema de gestión del instituto. Complementa `laravel-best-practices`, `inertia-react-development`, `inertia-table-development`, `wayfinder-development`, `pest-testing`.

## Naming

| Capa | Convención | Ejemplo |
|------|------------|---------|
| Módulo (URL, rutas, permisos) | **plural**, lowercase | `maintenances`, `admissions`, `portals` |
| Controller namespace | PascalCase, plural | `App\Http\Controllers\Maintenances\UserController` |
| Table namespace | PascalCase, plural | `App\Tables\Maintenances\UsersIndexTable` |
| Permiso auto (HasApplyPermissions) | `{modulo}.{recurso}` | `maintenances.users.read` |
| Inertia page + archivo React | **todo lowercase** | `'maintenances/users/index'` → `resources/js/pages/maintenances/users/index.tsx` |

En macOS (disco case-insensitive) **nunca** uses `Maintenances/` en paths de `resources/js/pages` — solo lowercase.
| Código PHP/TS | inglés | clases, métodos, variables |
| UI visible al usuario | español | títulos, columnas, labels |

## Rutas

- Archivos modulares: `routes/maintenances.php`, `routes/admissions.php`, etc.
- Prefijo URL = nombre del módulo en plural: `/maintenances/users`
- Nombre de ruta: `maintenances.users.index`
- Middleware en rutas: solo `auth` + `verified` — **sin** `role:` middleware
- Autorización: `App\Abstracts\Http\Controller` + `HasApplyPermissions`

## Listados (index)

Usar **Inertia Table**, no `->paginate()` manual en el controller.

Referencia canónica: `references/example-users-index.md`

## Layout

Usar `AppLayout` (default en `app.tsx`). No crear `AdminLayout` hasta que los portales diverjan.

## Menú

`config/navigation.php` + `NavigationService`. Items con `route`, `permission`, `icon` (string → `nav-icons.ts`).

## Tests

- Feature HTTP: `tests/Feature/{ModulePlural}/`
- Table aislada: `tests/Feature/Tables/{ModulePlural}/`
- `$this->withoutVite()` en tests Inertia que renderizan páginas nuevas
- Seed: `RolesAndPermissionsSeeder` en `beforeEach`

## Checklist nuevo módulo

1. Permisos en `RolesAndPermissionsSeeder` (`{modulo}.{recurso}` → c,r,u,d)
2. Rutas en `routes/{modulo}.php`, require en `web.php`
3. Controller en `App\Http\Controllers\{ModulePlural}\`
4. `{Model}IndexTable` en `App\Tables\{ModulePlural}\`
5. Page `resources/js/pages/{modulo}/{recurso}/index.tsx` (lowercase)
6. `Inertia::render('{modulo}/{recurso}/index', …)`
7. Entrada en `config/navigation.php`
8. `php artisan wayfinder:generate`
9. Tests feature + table
