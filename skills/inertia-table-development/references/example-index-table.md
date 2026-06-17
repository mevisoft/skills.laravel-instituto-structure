# Example: Users index (read-only template)

> Plantilla canónica del paquete. Para CRUD completo, complementa con `instituto-laravel-structure`.

## Files

| Piece | Path |
|-------|------|
| Table | `app/Tables/Maintenances/UsersIndexTable.php` |
| Controller | `app/Http/Controllers/Maintenances/UserController.php` |
| Page | `resources/js/pages/maintenances/users/index.tsx` |
| Route | `routes/maintenances.php` → `maintenances.users.index` |
| Tests | `tests/Feature/Tables/Maintenances/UsersIndexTableTest.php` |

## Table

```php
final class UsersIndexTable extends Table
{
    protected string $table_name = 'users';
    protected ?string $default_sort = '-created_at';

    public function resource(): Builder|Relation
    {
        return User::query()->with('roles');
    }

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
}
```

## Controller

```php
return Inertia::render('maintenances/users/index', [
    'users_table' => UsersIndexTable::make()->toInertiaProp($request),
]);
```

## React

```tsx
import { DataTable, type InertiaTableResource } from '@inertia-table';

type Props = {
    users_table: InertiaTableResource;
};

export default function Index({ users_table }: Props) {
    return (
        <DataTable resource={users_table} only={['users_table']} />
    );
}
```

## Query string (table name `users`)

```
/maintenances/users?users[search]=alice&users[page]=1&users[sort]=-created_at
```

## Tests (Pest)

```php
$prop = UsersIndexTable::make()->toInertiaProp(
    Request::create('/maintenances/users', 'GET', ['users' => ['search' => 'alice']])
);

expect($prop['pagination']['total'])->toBe(1);
```
