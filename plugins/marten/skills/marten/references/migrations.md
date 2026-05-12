## Generating migrations

```bash
crystal run manage.cr -- genmigrations        # all apps
crystal run manage.cr -- genmigrations my_app # one app
crystal run manage.cr -- genmigrations my_app --empty  # empty skeleton
# or via wrapper:
script/manage genmigrations
```

Marten diffs declared model definitions against the existing migration history and writes new files into `src/<app>/migrations/`. Auto-detects: new/dropped tables, added/removed/renamed columns, indexes, unique constraints, and cross-app FK dependencies.

**Always read the generated file before applying.** Auto-detection sometimes emits a `delete_table` + `create_table` pair when a rename or column change was intended. Correct it manually before running `migrate`.

---

## Migration class structure

```crystal
# src/migrations/202605061700001_add_rooms_and_link_messages.cr
class Migration::Main::V202605061700001 < Marten::Migration
  depends_on :main, "202605061600001_create_main_message"

  def plan
    delete_table :main_message

    create_table :main_room do
      column :id, :big_int, primary_key: true, auto: true
      column :name, :string, max_size: 100
      column :slug, :string, max_size: 100, unique: true
      column :created_at, :date_time
      column :updated_at, :date_time
    end

    create_table :main_message do
      column :id, :big_int, primary_key: true, auto: true
      column :body, :text
      column :room_id, :reference, to_table: :main_room, to_column: :id, null: false
      column :created_at, :date_time
      column :updated_at, :date_time
    end
  end
end
```

- Class name: `Migration::<AppPascal>::V<timestamp>`
- Inherits `Marten::Migration`
- `depends_on :app_label, "migration_name"` — one call per dependency; auto-generated for FK relationships
- `def plan` — bidirectional by default (Marten reverses ops in reverse order on rollback)
- Use `def forward` / `def backward` instead of `plan` when forward and backward logic differ

---

## Operation DSL reference

| Operation | Signature |
|---|---|
| `create_table` | `create_table :tbl do; column ...; index ...; unique_constraint ...; end` |
| `delete_table` | `delete_table :tbl` |
| `rename_table` | `rename_table :old, :new` |
| `add_column` | `add_column :tbl, :col, :type, **opts` |
| `remove_column` | `remove_column :tbl, :col` |
| `change_column` | `change_column :tbl, :col, :type, **opts` |
| `rename_column` | `rename_column :tbl, :old_col, :new_col` |
| `add_index` | `add_index :tbl, :idx_name, [:col1, :col2]` |
| `remove_index` | `remove_index :tbl, :idx_name` |
| `add_unique_constraint` | `add_unique_constraint :tbl, :cname, [:col1, :col2]` |
| `remove_unique_constraint` | `remove_unique_constraint :tbl, :cname` |
| `execute` | `execute(forward_sql, backward_sql)` |
| `run_code` | `run_code :forward_method, :backward_method` |

---

## `execute` for raw SQL

Use `execute(forward_sql, backward_sql)` when DDL is outside Marten's model — virtual tables, extensions, custom types, etc. Auto-gen **never** produces these; write the migration by hand.

```crystal
# src/migrations/202605070000001_create_leaf_search_index.cr
class Migration::Main::V202605070000001 < Marten::Migration
  def plan
    execute(
      <<-SQL,
        CREATE VIRTUAL TABLE leaf_search_index USING fts5(
          title,
          content,
          content='',
          tokenize='porter unicode61 remove_diacritics 2'
        )
      SQL
      "DROP TABLE leaf_search_index"
    )
  end
end
```

Always supply the backward SQL so `migrate <app> <version>` can roll back cleanly. If rollback is intentionally a no-op, pass an empty string as the second argument.

---

## Applying / rolling back

```bash
crystal run manage.cr -- migrate                        # apply all pending
crystal run manage.cr -- migrate my_app                 # apply one app only
crystal run manage.cr -- migrate my_app 202605061700001 # roll back to that version
crystal run manage.cr -- migrate my_app zero            # unapply all for app
crystal run manage.cr -- migrate my_app 202605061700001 --fake  # mark without executing
```

---

## Multi-app / multi-db notes

- Migration class namespace mirrors the app label: `main` app → `Migration::Main::V…`, `blog` app → `Migration::Blog::V…`.
- Files live under `src/<app>/migrations/` for multi-app projects, or `src/migrations/` for single-app projects.
- Marten uses the `default` DB connection unless the app's model specifies an alternative connection via `self.db_connection`. Migrations follow the same connection routing.
- Cross-app FK deps are expressed with multiple `depends_on` calls and resolved automatically by `genmigrations`.

---

## Common patterns

**Adding a non-null column with a default (three-step)**

```crystal
# Step 1: add nullable
add_column :articles, :status, :string, max_size: 32, null: true

# Step 2: backfill via run_code
run_code :backfill_status

def backfill_status
  Marten::DB::Connection.default.open do |db|
    db.exec("UPDATE articles SET status = 'draft' WHERE status IS NULL")
  end
end

# Step 3: tighten in a follow-up migration
change_column :articles, :status, :string, max_size: 32, null: false
```

**Raw-SQL backfill via `execute`**

```crystal
execute(
  "UPDATE articles SET status = 'draft' WHERE status IS NULL",
  "" # no-op on rollback
)
```

**Dropping a column safely**

1. Remove the field from the model definition.
2. Run `genmigrations` — Marten emits `remove_column`.
3. Review that it targets the right table/column, then `migrate`.

---

## Seed / fixture data

Do **not** embed seed data in migrations. Migrations run in all environments and are hard to skip. Use a dedicated seeder script (`crystal run manage.cr -- seed`) or test-fixture helpers instead. The only data that belongs in a migration is a backfill required for schema correctness (e.g. populating a newly non-null column before the constraint is added).

---

## When to look elsewhere

- Defining models the migrations diff against → `models.md`
- QuerySets / raw SQL at runtime → `querying.md`
