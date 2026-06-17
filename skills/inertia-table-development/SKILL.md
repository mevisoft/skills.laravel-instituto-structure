---
name: inertia-table-development
description: >-
  Implementa tablas interactivas con el paquete local Modules\InertiaTable (PHP) y @inertia-table (React).
  Usa esta skill siempre que construyas o modifiques listados/índices con búsqueda, filtros, orden,
  paginación, columnas tipadas, acciones por fila o selección masiva en Laravel + Inertia v3 + React 19 —
  aunque el usuario no mencione "inertia-table" ni "DataTable". Cubre subclases de Table, Table::build,
  columnas, filtros, acciones server-side, EmptyState, exports y el componente <DataTable>.
metadata:
  package: packages/inertia-table
  language: es
---

# Inertia Table — Guía de implementación

Skill del paquete local `packages/inertia-table` (no es el producto comercial Inertia UI). Implementación PHP/Laravel + React/Inertia con API inspirada en [Inertia Table docs](https://inertiaui.com/inertia-table/docs/introduction).

Complementa `inertia-react-development`, `laravel-best-practices` y `instituto-laravel-structure` cuando el listado vive en el campus Metropolitano.

## Cuándo usar esta skill

Activar siempre que:

- Crees o modifiques una pantalla tipo índice/listado en Inertia + React con búsqueda, orden, filtros o paginación.
- Importes desde `@inertia-table` o `Modules\InertiaTable\…`.
- Definas columnas tipadas (`TextColumn`, `BadgeColumn`, `DateColumn`, `NumericColumn`, `BooleanColumn`, `ImageColumn`, `ActionColumn`).
- Configures filtros (`TextFilter`, `NumericFilter`, `DateFilter`, `BooleanFilter`, `SetFilter`) o acciones (`Action`).
- Necesites varias tablas en una misma página (anidado por `as()` o `$table_name`).

NO usar para tablas estáticas sin estado (un simple `<table>` JSX con datos enviados de una sola vez cabe en Inertia/React puros).

## Arquitectura en 30 segundos

| Capa | Ubicación | Rol |
|------|-----------|-----|
| PHP | `packages/inertia-table/src/` → `Modules\InertiaTable\` | Construye el query Eloquent, lee estado del Request, serializa columnas/filtros/filas para Inertia. |
| React | `packages/inertia-table/resources/js/` (alias `@inertia-table`) | Renderiza `<DataTable>` y gestiona la UI (toolbar, filtros activos, paginación, selección). |

Flujo: el controller construye una `Table` (subclase) o `Table::build(...)` → `toInertiaProp($request)` devuelve la prop → el componente `<DataTable resource={...} />` la renderiza y al cambiar estado dispara `router.get(...)` con `preserveState`/`preserveScroll`.

## Setup ya realizado en este repo (no tocar salvo necesidad)

Estas piezas ya están enlazadas; sólo verifica si algo falla:

- Composer autoload: `"Modules\\InertiaTable\\": "packages/inertia-table/src/"`
- Service provider: `Modules\InertiaTable\InertiaTableServiceProvider` en `bootstrap/providers.php`
- Vite alias: `@inertia-table` → `packages/inertia-table/resources/js` en `vite.config.ts`
- TS path: `@inertia-table` y `@inertia-table/*` en `tsconfig.json`
- Tailwind v4: `@source '../../packages/inertia-table/resources/js/**/*.{js,ts,tsx}';` en `resources/css/app.css`

Si Tailwind no genera clases del módulo, falta el `@source`. Si TS no resuelve, falta el path en `tsconfig.json`.

## Ejemplo canónico del repo

Lee `references/example-index-table.md` antes de crear una tabla nueva. Implementación real:

- Table: `app/Tables/Maintenances/UsersIndexTable.php`
- Controller: `app/Http/Controllers/Maintenances/UserController.php`
- Page: `resources/js/pages/maintenances/users/index.tsx`
- Tests: `tests/Feature/Tables/Maintenances/UsersIndexTableTest.php`

## Patrón recomendado: subclase de `Table`

Cuando la pantalla tenga lógica real (filtros, transformaciones, autorización), crea una subclase final bajo `app/Tables/{ModulePlural}/`, con namespace `App\Tables\{ModulePlural}\…` (p. ej. `App\Tables\Maintenances\UsersIndexTable`).

```php
<?php

declare(strict_types=1);

namespace App\Tables\Maintenances;

use App\Models\User;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\Relation;
use Modules\InertiaTable\Columns\DateColumn;
use Modules\InertiaTable\Columns\TextColumn;
use Modules\InertiaTable\Filters\SetFilter;
use Modules\InertiaTable\Filters\TextFilter;
use Modules\InertiaTable\Table;

final class UsersIndexTable extends Table
{
    protected string $table_name = 'users';

    /** @var list<int> */
    protected array $per_page_options = [15, 30, 50];

    protected ?string $default_sort = '-created_at';

    public function resource(): Builder|Relation
    {
        return User::query()->with('roles');
    }

    /** @return array<int, \Modules\InertiaTable\Columns\Column> */
    public function columns(): array
    {
        return [
            TextColumn::make('name', 'Nombre', sortable: true, searchable: true),
            TextColumn::make('email', 'Correo', sortable: true, searchable: true),
            TextColumn::make('roles_label', 'Roles')
                ->mapAs(fn (mixed $value, User $user): string => $user->getRoleNames()->implode(', ') ?: '—'),
            DateColumn::make('created_at', 'Registrado', sortable: true),
        ];
    }

    /** @return array<int, \Modules\InertiaTable\Filters\Filter> */
    public function filters(): array
    {
        return [
            TextFilter::make('name', 'Nombre'),
            SetFilter::make('status')->options(['active' => 'Activo', 'inactive' => 'Inactivo']),
        ];
    }
}
```

Y en el controller:

```php
return Inertia::render('maintenances/users/index', [
    'users_table' => UsersIndexTable::make()->toInertiaProp($request),
]);
```

### Propiedades protegidas relevantes en la subclase

| Propiedad | Default | Notas |
|-----------|---------|-------|
| `$table_name` | `'default'` | `'default'` → estado plano (`?search=…`). Otro valor → anidado (`?users.search=…`). Imprescindible si hay >1 tabla en una misma página. |
| `$per_page_options` | `[15, 30, 50, 100]` | Lista blanca de `per_page`. Valores fuera de la lista se reemplazan por el primero. |
| `$default_per_page` | `null` | Si es `null`, usa `per_page_options[0]`. |
| `$default_sort` | `null` | Token: `attr` (asc) o `-attr` (desc). |
| `$pagination` | `true` | `false` desactiva paginación y devuelve todas las filas. |
| `$pagination_type` | `PaginationType::Full` | **Sólo `Full` está implementado**. `Simple`/`Cursor` lanzan `LogicException`. |
| `$bulk_selection_type` | `BulkSelectionType::All` | `All` o `Page`. |

### Hooks útiles

- `transformModel(Model $model, array $data): array` — enriquece celdas ya mapeadas (p. ej. traducir el `value` de un badge).
- `isSelectable(Model $model): bool` — controla `_selectable` por fila.
- `authorizeInertiaTableAction(Request $request, Action $action, array $ids): bool` — autorización de acciones server-side.
- `emptyState(): ?EmptyState` — `EmptyState::make()->title(...)->message(...)`.

## Patrón rápido: `Table::build()`

Cuando es trivial (lista pequeña, sin filtros) y no merece subclase:

```php
return Inertia::render('Example', [
    'users_table' => Table::build(
        resource: User::class,
        columns: [
            TextColumn::make('name', sortable: true, searchable: true),
            TextColumn::make('email', searchable: true),
        ],
        name: 'users',
    )->toInertiaProp($request),
]);
```

Para varias tablas en la misma página, encadena `->as('users')`. El último argumento posicional `name` de `build()` es equivalente.

## Recurso Eloquent

`resource()` y `Table::build(resource: …)` aceptan:

| Tipo | Comportamiento |
|------|----------------|
| `class-string<Model>` | Internamente `Model::query()`. |
| `Builder` | Se **clona** por petición (estado base preservado). |
| `Relation` | Se usa el query subyacente (también clonado). |

Reglas:

- Aplica `with([...])` para evitar N+1 en columnas que leen relaciones.
- La búsqueda global hace `OR` de `LIKE %term%` sobre columnas con `searchable: true`. Si una columna usa **dot notation** (relación), debes envolver tu propio search vía `sortUsing` o un filtro `TextFilter` con `Clause::Contains` en una columna de la **tabla principal**; el search global SÓLO opera sobre `attribute` directos del modelo.

## Columnas (`Modules\InertiaTable\Columns\*`)

Todas extienden `Column` y comparten fluent API:

`->header(?string)`, `->sortable(bool)` / `->notSortable()`, `->searchable(bool)` / `->notSearchable()`, `->toggleable(bool)` / `->notToggleable()`, `->visible(bool)` / `->hidden()`, `->align(ColumnAlignment)`, `->wrap(bool)`, `->truncate(?int)`, `->meta(array)`, `->sortUsing(callable)`.

| Clase | `type` en JSON | Uso típico |
|-------|----------------|------------|
| `TextColumn::make($attr, $header?, $sortable?, $searchable?, …)` | `text` | Texto normal. `->mapAs(callable\|array)` para formatear o mapear (enum→label). |
| `NumericColumn::make(…)` | `numeric` | Alineación derecha por defecto. |
| `BooleanColumn::make(…)` | `boolean` | `->trueLabel('Sí')->falseLabel('No')`. |
| `DateColumn::make(…)` | `date` | `->format('Y-m-d')`, `->translate(true)`. |
| `DateTimeColumn::make(…)` | `datetime` | Igual que `DateColumn` con default distinto. |
| `BadgeColumn::make(…)` | `badge` | `->variant(array\|callable)` → `Modules\InertiaTable\Variant` (`Default/Info/Success/Warning/Danger`). |
| `ImageColumn::make(…)` | `image` | No searchable. |
| `ActionColumn::new($header = 'Actions')` | `action` | **Columna especial**: si NO la incluyes, las `Action` definidas no se exponen (`actions: []`). Atributo interno `_actions`. |

### Cuándo usar `mapAs`

Para celdas tipo enlace usa la forma `{ href, label }` (el front la reconoce con `isLinkCellValue`):

```php
TextColumn::make('view', __('common.actions'))
    ->align(ColumnAlignment::Right)
    ->notSortable()->notSearchable()->notToggleable()
    ->mapAs(fn (mixed $v, Model $model): array => [
        'href' => route('maintenances.users.show', $model),
        'label' => __('common.view'),
    ]),
```

### Orden personalizado

```php
use Illuminate\Database\Eloquent\Builder;
use Modules\InertiaTable\SortDirection;

TextColumn::make('full_name', sortable: true)
    ->sortUsing(function (Builder $query, SortDirection $direction): void {
        $query->orderBy('last_name', $direction === SortDirection::Asc ? 'asc' : 'desc')
              ->orderBy('first_name', $direction === SortDirection::Asc ? 'asc' : 'desc');
    });
```

Token de orden en el query string: `attr` (asc) o `-attr` (desc). El backend valida que la columna sea sortable; si no, lo ignora silenciosamente.

## Filtros (`Modules\InertiaTable\Filters\*`)

Métodos comunes: `->clauses(array<Clause>)`, `->nullable(bool)`, `->meta(array)`.

| Clase | Tipo | Default clauses |
|-------|------|-----------------|
| `TextFilter::make($attr, $label?, $nullable?)` | `text` | `Contains`, `NotContains`, `StartsWith`, `EndsWith`, `Equals`, `NotEquals`, … |
| `NumericFilter::make(…)` | `numeric` | `Equals`, `GreaterThan`, `LessThan`, `Between`, … |
| `DateFilter::make(…)` | `date` | `Before`, `After`, `Between`, `EqualOrBefore`, … |
| `BooleanFilter::make(…)` | `boolean` | `IsTrue`, `IsFalse`. |
| `SetFilter::make(…)` | `set` | `Equals`, `NotEquals`, `In`, `NotIn`. Métodos extra: `->options(array<string,string>)`, `->multiple()`, `->pluckOptionsFromModel(class, label, key='id')`. |

`->nullable()` añade automáticamente `Clause::IsSet` y `Clause::IsNotSet`.

### Cláusulas (`Modules\InertiaTable\Filters\Clause`)

Strings: `equals`, `not_equals`, `contains`, `not_contains`, `starts_with`, `not_starts_with`, `ends_with`, `not_ends_with`, `greater_than`, `greater_than_or_equal`, `less_than`, `less_than_or_equal`, `between`, `not_between`, `before`, `after`, `equal_or_before`, `equal_or_after`, `in`, `not_in`, `is_set`, `is_not_set`, `is_true`, `is_false`.

El backend ignora filas incompletas (valor vacío, `between` sin dos extremos, `in` sin elementos), así que no temas pasar filtros “medio rellenos” desde el front.

### Formato de filtros en la URL

Acepta DOS formatos (ambos normalizados por `Table::readState`):

1. Mapa por atributo (estilo Inertia Table): `?users.filters[name][clause]=contains&users.filters[name][value]=Ada`
2. Filas: `?users.filters[0][attribute]=name&users.filters[0][clause]=contains&users.filters[0][value]=Ada&users.filters[0][enabled]=1`

## Acciones, Exports, EmptyState

```php
use Modules\InertiaTable\Action;
use Modules\InertiaTable\Columns\ActionColumn;
use Modules\InertiaTable\EmptyState;
use Modules\InertiaTable\Export;

public function columns(): array
{
    return [
        // …
        ActionColumn::new('Acciones'),
    ];
}

public function actions(): array
{
    return [
        Action::make('Eliminar')
            ->id('delete')
            ->onlyAsBulkAction()
            ->handle(fn (Model $model) => $model->delete()),
        Action::make('Ver', url: fn (Model $m) => route('maintenances.users.show', $m)),
    ];
}

public function exports(): array
{
    return [Export::make('Exportar CSV', 'users.csv', route('maintenances.users.export'))];
}

public function emptyState(): ?EmptyState
{
    return EmptyState::make(
        title: 'Sin usuarios',
        message: 'No hay usuarios que coincidan con los filtros.',
    );
}
```

Importante:

- Si **no** existe una `ActionColumn` entre `columns()`, la prop `actions` se serializa vacía aunque definas `actions()`.
- Acciones con `->handle()` se ejecutan en el servidor vía `POST /inertia-table/actions/{base64_table_class}` (`InertiaTableActionController`). Implementa `authorizeInertiaTableAction()` para permisos.
- `Export::make($label, $filename, $href)` expone `label`, `filename`, `href` y `download` al front: con `href` el menú **Actions → Exports** renderiza un enlace descargable; sin `href` el ítem aparece deshabilitado hasta que cablees la URL.

## Front: usar `<DataTable>`

```tsx
import { DataTable, type InertiaTableResource } from '@inertia-table';

type Props = {
    users_table: InertiaTableResource;
};

export default function Index({ users_table }: Props) {
    return (
        <DataTable
            resource={users_table}
            only={['users_table']}
            emptyResultsLabel="No hay usuarios"
        />
    );
}
```

Props de `<DataTable>`:

| Prop | Tipo | Notas |
|------|------|-------|
| `resource` | `InertiaTableResource` | La prop devuelta por `toInertiaProp`. |
| `only` | `string[]` | Pasa SIEMPRE el nombre de la prop Inertia para visitas parciales. Sin esto, cada cambio de estado recarga toda la página. |
| `emptyResultsLabel` | `string` | Texto cuando `results` está vacío. |
| `onBulkSelectionIdsChange` | `(ids: number[]) => void` | Callback opcional cuando cambia la selección masiva. |

Estilos: clases prefijo `it-` (`it-wrapper`, `it-table`, `it-topbar`). Tailwind las descubre vía el `@source` declarado en `resources/css/app.css`.

Traducciones del paquete: `useInertiaTableTranslations()` o la prop compartida `inertia_table` registrada por el service provider.

## Varias tablas en la misma página

Cada una con `as('xxx')` (o `$table_name` distinto). El query string queda anidado:

```
?orders.search=Ada&orders.page=2&customers.search=acme
```

En el controller:

```php
return Inertia::render('Dashboard', [
    'orders_table' => OrdersIndexTable::make()->toInertiaProp($request),
    'customers_table' => CustomersIndexTable::make()->toInertiaProp($request),
]);
```

En el front, cada `<DataTable>` con su `only={['orders_table']}` / `only={['customers_table']}` para no recargar la otra.

## Pruebas

PHP (Pest) — prueba la tabla aislada sin HTTP cuando baste:

```php
use Illuminate\Http\Request;

test('users index table search filters by name or email', function () {
    User::factory()->create(['name' => 'Alice Admin', 'email' => 'alice@example.com']);
    User::factory()->create(['name' => 'Bob Baker', 'email' => 'bob@example.com']);

    $prop = UsersIndexTable::make()->toInertiaProp(
        Request::create('/maintenances/users', 'GET', ['users' => ['search' => 'alice']])
    );

    expect($prop['pagination']['total'])->toBe(1)
        ->and($prop['results'][0]['email'])->toBe('alice@example.com');
});
```

Ejecutar:

```bash
php artisan test --compact --filter=UsersIndexTableTest
```

JS (Vitest):

```bash
pnpm exec vitest run packages/inertia-table/resources/js
```

## Errores frecuentes y solución

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| La tabla recarga toda la página al filtrar/ordenar. | Falta `only={['nombre_prop']}` en `<DataTable>`. | Pásalo siempre. |
| `actions` llega vacío al front. | No incluiste `ActionColumn::new(...)` en `columns()`. | Añádela. |
| Search global no encuentra registros de relación. | El search hace `LIKE` sobre el atributo literal, no resuelve relaciones. | Usa `mapAs` + columna calculada, un `TextFilter`, o scopes. |
| `LogicException: Only full pagination is implemented`. | Pusiste `$pagination_type` distinto a `Full`. | Sólo `PaginationType::Full` está soportado. |
| Tailwind no aplica clases `it-…`. | Falta `@source` apuntando al paquete. | Verifica `resources/css/app.css`. |
| `ViteException: Unable to locate file in Vite manifest`. | No corriste `pnpm run dev` / `pnpm run build`. | Ejecuta uno de los dos. |
| Estado de varias tablas se mezcla. | Ambas usan `$table_name = 'default'`. | Asigna `as('orders')`, `as('customers')`. |
| Acción bulk devuelve 403. | `authorizeInertiaTableAction()` devuelve false. | Implementa autorización con permisos Spatie. |

## Límites actuales del paquete

- Sólo paginación **full** (`LengthAwarePaginator`). `Simple` y `Cursor` están definidos en el enum pero NO soportados.
- `Views` expone contrato pero NO implementa persistencia de vistas guardadas.
- `Export` serializa metadatos; la descarga real la cableas en la app (ruta + `href`).
- El search global SÓLO toca atributos directos del modelo principal.

## Convenciones del proyecto al crear una nueva tabla

1. Subclase final en `app/Tables/{ModulePlural}/{Model}IndexTable.php` (`App\Tables\{ModulePlural}\…`).
2. `declare(strict_types=1);` y todas las firmas tipadas (return types incluidos).
3. Variables locales en `snake_case`; métodos en `camelCase`; clases en `PascalCase`.
4. UI visible al usuario en español; código PHP/TS en inglés.
5. Mismo patrón en page React: `resources/js/pages/{modulo}/{recurso}/index.tsx` (paths lowercase) con `<DataTable resource={…} only={[…]} />`.
6. Test en `tests/Feature/Tables/{ModulePlural}/{Model}IndexTableTest.php` que cubra al menos: render base, un filtro o búsqueda, orden por defecto.
7. `vendor/bin/pint --dirty --format agent` antes de cerrar si tocaste PHP.

Para permisos, rutas, navegación y checklist de módulo completo, activa `instituto-laravel-structure`.

## Recursos del paquete

| Recurso | Path |
|---------|------|
| Clase base | `packages/inertia-table/src/Table.php` |
| Componente React | `packages/inertia-table/resources/js/DataTable.tsx` |
| Tipos TS | `packages/inertia-table/resources/js/types.ts` |
| Config | `packages/inertia-table/config/inertia-table.php` |
| Ejemplo index | `references/example-index-table.md` |
| Tests JS | `packages/inertia-table/resources/js/*.test.ts` |
