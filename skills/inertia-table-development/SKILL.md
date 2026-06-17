---
name: inertia-table-development
description: Implementa tablas interactivas con el módulo local Modules\InertiaTable (PHP) y `@inertia-table` (React). Activar al construir índices/listados con búsqueda, filtros, orden, paginación, columnas tipadas (texto, badge, fecha, booleano, numérico, imagen, acción) o selección masiva sobre Inertia v2 + React 19. Cubre `Table::build`, subclases de `Table`, columnas, filtros, acciones, EmptyState, y el componente `<DataTable>`.
metadata:
  module: packages/inertia-table
  language: es
---

# Inertia Table — Guía de implementación

Skill del módulo local `packages/inertia-table` (no es el producto comercial Inertia UI). Implementación PHP/Laravel + React/Inertia con API inspirada en [Inertia Table docs](https://inertiaui.com/inertia-table/docs/introduction).

Documentación viva del módulo: `packages/inertia-table/README.md`. Esta skill resume y guía la implementación.

## Cuándo usar esta skill

Activar siempre que:

- Crees o modifiques una pantalla tipo índice/listado en Inertia + React con búsqueda/orden/filtros/paginación.
- Importes desde `@inertia-table` o `Modules\InertiaTable\…`.
- Definas columnas tipadas (`TextColumn`, `BadgeColumn`, `DateColumn`, `NumericColumn`, `BooleanColumn`, `ImageColumn`, `ActionColumn`).
- Configures filtros (`TextFilter`, `NumericFilter`, `DateFilter`, `BooleanFilter`, `SetFilter`) o acciones (`Action`).
- Necesites varias tablas en una misma página (anidado por `as()`).

NO usar para tablas estáticas sin estado (un simple `<table>` JSX con datos enviados de una sola vez ya cabe en Inertia/React puros).

## Arquitectura en 30 segundos

| Capa | Ubicación | Rol |
|------|-----------|-----|
| PHP | `packages/inertia-table/src/` → `Modules\InertiaTable\` | Construye el query Eloquent, lee estado del Request, serializa columnas/filtros/filas para Inertia. |
| React | `packages/inertia-table/resources/js/` (alias `@inertia-table`) | Renderiza `<DataTable>` y gestiona la UI (toolbar, filtros activos, paginación, selección). |

Flujo: el controller construye una `Table` (subclase) o `Table::build(...)` → `toInertiaProp($request)` devuelve la prop → el componente `<DataTable resource={...} />` la renderiza y al cambiar estado dispara `router.get(...)` con `preserveState`/`preserveScroll`.

## Setup ya realizado en este repo (no tocar salvo necesidad)

Estas piezas ya están enlazadas; sólo verifica si algo falla:

- Composer autoload: `"Modules\\InertiaTable\\": "packages/inertia-table/src/"`
- Service provider: `Modules\InertiaTable\InertiaTableServiceProvider` registrado en `bootstrap/providers.php`
- Vite alias: `@inertia-table` → `packages/inertia-table/resources/js` en `vite.config.ts`
- TS path: `@inertia-table` y `@inertia-table/*` en `tsconfig.json`
- Tailwind v4: `@source '../../packages/inertia-table/resources/js/**/*.{js,ts,tsx}';` en `resources/css/app.css`

Si Tailwind no genera clases del módulo, falta el `@source`. Si TS no resuelve, falta el path en `tsconfig.json`.

## Patrón recomendado: subclase de `Table`

Cuando la pantalla tenga lógica real (filtros, transformaciones, autorización), crea una subclase bajo `app/Tables/{Area}/` (p. ej. `App\Tables\Maintenances\UsersIndexTable`). Referencia canónica: `app/Tables/Maintenances/UsersIndexTable.php`.

```php
<?php

declare(strict_types=1);

namespace App\Tables;

use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\Relation;
use Modules\InertiaTable\Columns\BadgeColumn;
use Modules\InertiaTable\Columns\Column;
use Modules\InertiaTable\Columns\DateColumn;
use Modules\InertiaTable\Columns\TextColumn;
use Modules\InertiaTable\Filters\Filter;
use Modules\InertiaTable\Filters\SetFilter;
use Modules\InertiaTable\Filters\TextFilter;
use Modules\InertiaTable\Table;

final class ClientOrdersIndexTable extends Table
{
    protected string $table_name = 'orders';

    /** @var list<int> */
    protected array $per_page_options = [12, 15, 30, 50];

    protected ?int $default_per_page = 12;

    protected ?string $default_sort = '-created_at';

    public function resource(): Builder|Relation
    {
        return auth()->user()
            ->orders()
            ->with(['pack', 'state'])
            ->latest();
    }

    /** @return array<int, Column> */
    public function columns(): array
    {
        return [
            TextColumn::make('llc_name', __('client.col_llc'), sortable: true, searchable: true),
            BadgeColumn::make('status', __('client.col_status'), sortable: true),
            DateColumn::make('created_at', __('client.col_created_at'), sortable: true),
        ];
    }

    /** @return array<int, Filter> */
    public function filters(): array
    {
        return [
            TextFilter::make('llc_name'),
            SetFilter::make('status')->options($status_options),
        ];
    }
}
```

Y en el controller:

```php
return inertia('maintenances/users/index', [
    'orders_table' => (new ClientOrdersIndexTable)->toInertiaProp($request),
]);
```

### Propiedades protegidas relevantes en la subclase

| Propiedad | Default | Notas |
|-----------|---------|-------|
| `$table_name` | `'default'` | `'default'` → estado plano (`?search=…`). Otro valor → anidado (`?orders.search=…`). Imprescindible si hay >1 tabla en una misma página. |
| `$per_page_options` | `[15, 30, 50, 100]` | Lista blanca de `per_page`. Valores fuera de la lista se reemplazan por el primero. |
| `$default_per_page` | `null` | Si es `null`, usa `per_page_options[0]`. |
| `$default_sort` | `null` | Token: `attr` (asc) o `-attr` (desc). |
| `$pagination` | `true` | `false` desactiva paginación y devuelve todas las filas. |
| `$pagination_type` | `PaginationType::Full` | **Sólo `Full` está implementado**. `Simple`/`Cursor` lanzan `LogicException`. |
| `$bulk_selection_type` | `BulkSelectionType::All` | `All` o `Page`. |

### Hooks útiles

- `transformModel(Model $model, array $data): array` — enriquece celdas ya mapeadas (p. ej. traducir el `value` de un badge). Ver `ClientOrdersIndexTable::transformModel`.
- `isSelectable(Model $model): bool` — controla `_selectable` por fila.
- `emptyState(): ?EmptyState` — `EmptyState::make()->title(...)->message(...)`.

## Patrón rápido: `Table::build()`

Cuando es trivial (lista pequeña, sin filtros) y no merece subclase:

```php
return inertia('Example', [
    'users' => Table::build(
        resource: User::class,
        columns: [
            TextColumn::make('name', sortable: true, searchable: true),
            TextColumn::make('email', searchable: true),
        ],
    )->toInertiaProp(request()),
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

- Aplica `with([...])` para evitar N+1 en columnas que leen relaciones (`'state.abbreviation'`, `'pack.name'`).
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
        'href' => route('client.orders.show', $model),
        'label' => __('client.orders_index.view'),
    ]),
```

### Orden personalizado

```php
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

1. Mapa por atributo (estilo Inertia Table): `?filters[name][clause]=contains&filters[name][value]=Ada`
2. Filas: `?filters[0][attribute]=name&filters[0][clause]=contains&filters[0][value]=Ada&filters[0][enabled]=1`

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
        ActionColumn::new(__('common.actions')),
    ];
}

public function actions(): array
{
    return [
        Action::make(__('common.delete'))
            ->id('delete')
            ->onlyAsBulkAction(),
        Action::make(__('common.view'), url: fn (Model $m) => route('orders.show', $m)),
    ];
}

public function exports(): array
{
    return [Export::make(__('common.export_csv'), 'orders.csv')];
}

public function emptyState(): ?EmptyState
{
    return EmptyState::make(
        title: __('orders.empty_title'),
        message: __('orders.empty_message'),
    );
}
```

Importante: si **no** existe una `ActionColumn` entre `columns()`, la prop `actions` se serializa vacía aunque definas `actions()`. `Export::make($label, $filename, $href)` expone `label`, `filename`, `href` y `download` al front: con `href` el menú **Actions → Exports** renderiza un enlace descargable; sin `href` el ítem aparece deshabilitado hasta que cablees la URL (firmada o ruta propia).

## Front: usar `<DataTable>`

```tsx
import { DataTable } from '@inertia-table';
import type { InertiaTableResource } from '@inertia-table';

interface OrdersIndexProps {
    orders_table: InertiaTableResource;
}

export default function OrdersIndex({ orders_table }: OrdersIndexProps) {
    return (
        <DataTable
            resource={orders_table}
            only={['orders_table']}
            emptyResultsLabel="No hay órdenes"
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

Estilos: clases prefijo `it-` (`it-wrapper`, `it-table`, `it-topbar`). Tailwind las descubre vía el `@source` declarado en `resources/css/app.css`.

## Varias tablas en la misma página

Cada una con `as('xxx')` (o `$table_name` distinto). El query string queda anidado:

```
?orders.search=Ada&orders.page=2&customers.search=acme
```

En el controller:

```php
return inertia('Dashboard', [
    'orders' => (new OrdersTable)->toInertiaProp($request),
    'customers' => (new CustomersTable)->toInertiaProp($request),
]);
```

En el front, cada `<DataTable>` con su `only={['orders']}` / `only={['customers']}` para no recargar la otra.

## Pruebas

PHP (Pest, ya hay ejemplos en `tests/Feature/InertiaTable/GenericTableTest.php`):

```php
it('filtra órdenes por status', function (): void {
    Order::factory()->create(['status' => 'pending']);
    Order::factory()->create(['status' => 'completed']);

    $request = Request::create('/orders', 'GET', [
        'orders' => ['filters' => ['status' => ['clause' => 'equals', 'value' => 'pending']]],
    ]);

    $prop = (new ClientOrdersIndexTable)->toInertiaProp($request);

    expect($prop['results'])->toHaveCount(1);
});
```

Ejecutar:

```bash
php artisan test --compact --filter=GenericTableTest
```

JS (Vitest, p. ej. `data-table-helpers.test.ts`):

```bash
npx vitest run packages/inertia-table/resources/js
```

## Errores frecuentes y solución

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| La tabla recarga toda la página al filtrar/ordenar. | Falta `only={['nombre_prop']}` en `<DataTable>`. | Pásalo siempre. |
| `actions` llega vacío al front. | No incluiste `ActionColumn::new(...)` en `columns()`. | Añádela. |
| Search global no encuentra registros de relación. | El search hace `LIKE` sobre el atributo literal, no resuelve relaciones. | Define un `TextFilter` con `Contains` o usa `sortUsing`/scopes. |
| `LogicException: Only full pagination is implemented`. | Pusiste `$pagination_type` distinto a `Full`. | Sólo `PaginationType::Full` está soportado. |
| Tailwind no aplica clases `it-…`. | Falta `@source` apuntando al módulo. | Verifica `resources/css/app.css`. |
| `ViteException: Unable to locate file in Vite manifest`. | No corriste `npm run dev` / `npm run build`. | Ejecuta uno de los dos. |
| Estado de varias tablas se mezcla. | Ambas usan `$table_name = 'default'`. | Asigna `as('orders')`, `as('customers')`. |

## Límites actuales del módulo

- Sólo paginación **full** (`LengthAwarePaginator`). `Simple` y `Cursor` están definidos en el enum pero NO soportados.
- `Views` y `Export` exponen contratos pero NO implementan persistencia ni descarga (debes cablearlas en la app).
- El search global SÓLO toca atributos directos del modelo principal.

## Convenciones del proyecto al crear una nueva tabla

1. Subclase final en `app/Tables/{Area}/{Modelo}IndexTable.php` (p. ej. `App\Tables\Maintenances\UsersIndexTable`).
2. `declare(strict_types=1);` y todas las firmas tipadas (return types incluidos).
3. Variables locales en `snake_case` (regla del repo); métodos en `camelCase`; clases en `PascalCase`.
4. Tradúcelo todo con `__('namespace.key')`. No string literals en headers.
5. Page React en **lowercase**: `resources/js/pages/{modulo}/{recurso}/index.tsx` (p. ej. `resources/js/pages/maintenances/users/index.tsx`). El string de `Inertia::render()` debe coincidir: `'maintenances/users/index'`. Módulos en **plural** (`maintenances`, `admissions`, `portals`). `<DataTable resource={table} only={['table']} />`.
6. Test feature en `tests/Feature/{Contexto}/{Modelo}IndexTableTest.php` que cubra al menos: render base, un filtro, orden por defecto.
7. `vendor/bin/pint --dirty --format agent` antes de cerrar.

## Recursos del repo

- README del módulo: `packages/inertia-table/README.md`
- Ejemplo subclase: `app/Tables/Maintenances/UsersIndexTable.php`
- Ejemplo controller: `app/Http/Controllers/Maintenances/UserController.php`
- Ejemplo page React: `resources/js/pages/maintenances/users/index.tsx` → `Inertia::render('maintenances/users/index', …)`
- Tests PHP de referencia: `tests/Feature/Tables/Maintenances/UsersIndexTableTest.php`
