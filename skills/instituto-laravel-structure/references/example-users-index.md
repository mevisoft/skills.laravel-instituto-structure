# Example: Users index (read-only template)

> Full CRUD: see `example-student.md` when it exists. Copy this for index/listado screens.

## Files

| Piece | Path |
|-------|------|
| Table | `app/Tables/Maintenances/UsersIndexTable.php` |
| Controller | `app/Http/Controllers/Maintenances/UserController.php` |
| Page | `resources/js/pages/maintenances/users/index.tsx` |
| Route | `routes/maintenances.php` → `maintenances.users.index` |
| Nav | `config/navigation.php` |
| Tests | `tests/Feature/Maintenances/UserIndexTest.php`, `tests/Feature/Tables/Maintenances/UsersIndexTableTest.php` |

## Controller

```php
return Inertia::render('maintenances/users/index', [
    'users_table' => UsersIndexTable::make()->toInertiaProp($request),
]);
```

## React

```tsx
import { DataTable, type InertiaTableResource } from '@inertia-table';

<DataTable resource={users_table} only={['users_table']} />
```

## Permissions

`Maintenances\UserController` → auto `maintenances.users.read` on `index()`.
