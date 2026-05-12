## Defining a model

Every model is a subclass of `Marten::Model`. Fields are declared with the `field` macro (name, type, options). Every model needs exactly one primary key field.

```crystal
class Article < Marten::Model
  with_timestamp_fields                          # adds created_at / updated_at (date_time, auto-set)

  field :id,      :big_int, primary_key: true, auto: true
  field :title,   :string,  max_size: 255
  field :content, :text
  field :author,  :many_to_one, to: Author, on_delete: :cascade, related: :articles
end
```

`with_timestamp_fields` is exactly equivalent to:

```crystal
field :created_at, :date_time, auto_now_add: true
field :updated_at, :date_time, auto_now: true
```

**UUID primary key pattern** (requires manual initialization):

```crystal
class Token < Marten::Model
  field :id, :uuid, primary_key: true
  after_initialize :set_id
  private def set_id = (@id ||= UUID.random)
end
```

**Abstract base model** (no DB table; fields inherited by children):

```crystal
abstract class TimestampedRecord < Marten::Model
  field :id, :big_int, primary_key: true, auto: true
  with_timestamp_fields
end

class Post < TimestampedRecord
  field :title, :string, max_size: 255
end
```

**Multi-table inheritance** (each model has its own table; joined automatically):

```crystal
class Person < Marten::Model
  field :id,         :big_int, primary_key: true, auto: true
  field :first_name, :string,  max_size: 100
end

class Employee < Person   # own table; person fields accessible via join
  field :company, :string, max_size: 100
end

Employee.filter(first_name: "Jane").first!.first_name  # => "Jane"
person.employee  # => Employee | nil
```

---

## Field types

| Symbol | Crystal type | DB type (PG) | Notes |
|---|---|---|---|
| `:big_int` | `Int64` | `bigint` / `bigserial` | `auto: true` for autoincrement |
| `:int` | `Int32` | `int` / `serial` | `auto: true` for autoincrement |
| `:bool` | `Bool` | `boolean` | |
| `:float` | `Float64` | `double precision` | |
| `:string` | `String` | `varchar(n)` | `max_size:` **required** |
| `:text` | `String` | `text` | `max_size:` optional |
| `:email` | `String` | `varchar(254)` | validates format; `max_size:` overrideable |
| `:url` | `String` | `varchar(200)` | validates URL format |
| `:slug` | `String` | `varchar(50)` | indexed by default; `slugify: :field` auto-generates |
| `:uuid` | `UUID` | `uuid` | |
| `:date` | `Time` | `date` | `auto_now:`, `auto_now_add:` |
| `:date_time` | `Time` | `timestamptz` | `auto_now:`, `auto_now_add:` |
| `:duration` | `Time::Span` | `bigint` (nanoseconds) | |
| `:json` | `JSON::Any` | `jsonb` / `text` | `serializable: MyClass` for typed parsing |
| `:enum` | `MyEnum` | ENUM/check | `values: MyEnum` **required** |
| `:file` | `String` | `text` | stores path; `upload_to:`, `storage:` |
| `:image` | `String` | `text` | requires `crystal-vips` shard |

**Enum example:**

```crystal
enum Status; DRAFT; PUBLISHED; end

class Post < Marten::Model
  field :id,     :big_int, primary_key: true, auto: true
  field :status, :enum,    values: Status
end

Post.last!.status  # => Status::PUBLISHED
```

**JSON with serializable** — pass `serializable: MyClass` (must include `JSON::Serializable`) to get typed objects back instead of `JSON::Any`.

---

## Field options

| Option | Default | Effect |
|---|---|---|
| `null: true` | `false` | allows `NULL` in DB column |
| `blank: true` | `false` | allows empty/blank in validation |
| `default:` | `nil` | sets a default value |
| `unique: true` | `false` | adds a UNIQUE constraint |
| `index: true` | `false` | adds a DB index |
| `primary_key: true` | `false` | marks as PK |
| `db_column:` | field name | override column name |
| `max_size:` | varies | max length (required for `:string`) |
| `min_size:` | `nil` | min length (`:string` only) |
| `editable: false` | `true` | excludes from schema form generation |
| `auto_now: true` | `false` | sets to current time on every save (`:date`, `:date_time`) |
| `auto_now_add: true` | `false` | sets to current time on create only |

**Footgun:** `null` and `blank` are independent. A field can have `null: false` (DB won't store NULL) while `blank: true` (validation allows empty string). These must be coordinated deliberately.

---

## Relationships

### many_to_one (foreign key)

```crystal
class Article < Marten::Model
  field :id,     :big_int, primary_key: true, auto: true
  field :author, :many_to_one, to: Author, on_delete: :cascade, related: :articles
end

article.author        # => Author | nil
article.author!       # => Author (raises if nil)
article.author_id     # => Int64 (raw FK column, no query)
author.articles       # => QuerySet(Article)  [only when related: is set]
author.articles.filter(title__startswith: "Top")
```

### one_to_one

Like `many_to_one` but adds a UNIQUE constraint. Same `on_delete:` and `related:` options.

```crystal
class User < Marten::Model
  field :id,      :big_int, primary_key: true, auto: true
  field :profile, :one_to_one, to: Profile, on_delete: :cascade, related: :user
end

user.profile        # => Profile | nil
profile.user        # => User | nil  (reverse; nil if no user references it)
profile.user!       # => User (raises if nil)
```

### many_to_many

No `on_delete:`. Marten manages a join table automatically.

```crystal
class Article < Marten::Model
  field :tags, :many_to_many, to: Tag, related: :articles
end

article.tags.add(tag1, tag2)
article.tags.remove(tag2)
article.tags.clear
article.tags.filter(label__startswith: "cr")
tag.articles.to_a     # reverse; only available when related: is set
```

### on_delete strategies (many_to_one / one_to_one)

| Value | Behaviour |
|---|---|
| `:do_nothing` | Default — may cause DB FK errors |
| `:cascade` | Delete referencing records first |
| `:protect` | Raise `Marten::DB::Errors::ProtectedRecord` |
| `:set_null` | Set FK column to NULL (requires `null: true`) |

### polymorphic (closed-set)

Use when a field can point to one of a **fixed set** of model types. Contributes two real columns: `<field>_type` (string) and `<field>_id` (bigint/int).

```crystal
class Comment < Marten::Model
  field :id,     :big_int, primary_key: true, auto: true
  field :target, :polymorphic, to: [Article, Recipe], related: :comments
  field :body,   :text
end

comment.target         # => Article | Recipe | nil
comment.target_type    # => "Article" (stored class name string)
comment.target_id      # => Int64

comment.article_target?  # => true/false
comment.article_target   # => Article | nil
comment.article_target!  # => Article (raises if not Article)

Comment.with_article_target  # => QuerySet scoped to Article targets
Comment.with_recipe_target   # => QuerySet scoped to Recipe targets

article.comments.to_a        # backward relation (requires related:)
```

**Important — namespace/paginator conflict:** Marten generates `::Article::Page` as an inner paginator class. If `Article` is nested inside a module (e.g. `Leafables::Article`), generated inner types and the `with_article_target` scope name are based on the **last segment** of the class name. Clashes between short names across namespaces will cause subtle compiler errors. See `gotchas.md` for the full debug story.

### Recursive / self-referential

```crystal
class TreeNode < Marten::Model
  field :id,     :big_int, primary_key: true, auto: true
  field :parent, :many_to_one, to: self, null: true, blank: true
end
```

---

## Validations

Field options contribute validation automatically (`blank: false`, `max_size:`, format for `:email`/`:url`, etc.). Custom rules use the `validate` macro.

```crystal
class User < Marten::Model
  field :id,   :big_int, primary_key: true, auto: true
  field :name, :string, max_size: 128, blank: false

  validate :name_not_reserved

  private def name_not_reserved
    errors.add(:name, "is reserved") if name == "admin"
  end
end
```

**Running validations:**

```crystal
user.valid?         # => bool; runs all validators
user.invalid?       # => !valid?
user.save           # => false if invalid; returns true on success
user.save!          # raises Marten::DB::Errors::InvalidRecord if invalid
user.create!        # same
user.save(validate: false)  # skip — use with extreme caution
```

**Inspecting errors:**

```crystal
user.errors[:name]   # => Array(Error) for that field
user.errors.global   # => Array(Error) with no field
user.errors.add(:name, "too short")
user.errors.add("bad record", type: :cross_field)  # global error
```

Field-level validators run first; custom `validate` methods run after.

---

## Callbacks

Register with the callback macro, passing the method name as a symbol. Multiple registrations for the same event fire in registration order.

```crystal
class Order < Marten::Model
  before_validation :normalize_email
  after_create      :send_confirmation
  after_commit      :enqueue_job, on: :create

  private def normalize_email = (self.email = email.try(&.downcase))
  private def send_confirmation = NotificationMailer.deliver(self)
  private def enqueue_job = OrderJob.enqueue(id)
end
```

| Callback | When it fires |
|---|---|
| `after_initialize` | After `new` or loading from DB |
| `before_validation` | Before running validators |
| `after_validation` | After running validators |
| `before_save` | Before any save (create or update) |
| `before_create` | Before INSERT — note: **after** validation |
| `before_update` | Before UPDATE — note: **after** validation |
| `after_create` | After INSERT completes (before commit) |
| `after_update` | After UPDATE completes (before commit) |
| `after_save` | After create or update (before commit) |
| `before_delete` | Before DELETE |
| `after_delete` | After DELETE (before commit) |
| `after_commit` | After the DB transaction commits |
| `after_rollback` | After the DB transaction rolls back |

`after_commit` and `after_rollback` accept an `on:` argument:

```crystal
after_commit :notify, on: :create
after_commit :notify, on: [:create, :update]  # on: :save is shorthand for both
after_rollback :cleanup, on: :delete
```

**Footgun:** `before_create`/`before_update` run **after** validation. If you need to populate a `null: false` field before validation fires (e.g. derive a slug), use `before_validation` instead.

**Bypass:** `update_columns` / `update_columns!` skip **all** callbacks and validations. Useful for performance; dangerous if business logic depends on lifecycle hooks.

---

## Table options

```crystal
class Article < Marten::Model
  db_table :articles    # override auto-generated table name

  db_index :idx_title_created, field_names: [:title, :created_at]

  db_unique_constraint :uq_room_date, field_names: [:room, :date]

  with_timestamp_fields

  field :id,    :big_int, primary_key: true, auto: true
  field :title, :string,  max_size: 255
  field :room,  :string,  max_size: 50
  field :date,  :date
end
```

- `db_table` — takes a string or symbol; overrides the `<app>_<model_plural>` default
- `db_index` — multi-column index (single-column: use `index: true` on the field)
- `db_unique_constraint` — multi-column unique (single-column: use `unique: true` on the field)
- `with_timestamp_fields` — macro that adds `created_at` / `updated_at` date_time fields

---

## CRUD basics

```crystal
# Create
article = Article.create!(title: "Hello", author: author)

# Read
Article.all
Article.get(id: 42)          # => Article | nil
Article.get!(id: 42)         # raises if not found
Article.filter(title: "Hi")  # QuerySet

# Update
article.title = "Updated"
article.save!
Article.filter(title: "Hi").update(title: "Updated")

# Update specific columns only (skips callbacks and validations)
article.update_columns(title: "Fast update")

# Delete
article.delete
Article.filter(title: "Hi").delete
```

---

## Custom methods and scopes

Instance methods and class-level scopes are plain Crystal methods on the model class.

```crystal
class Article < Marten::Model
  field :id,        :big_int, primary_key: true, auto: true
  field :published, :bool,    default: false
  field :views,     :int,     default: 0

  # Class-level scope
  def self.published
    filter(published: true)
  end

  def self.popular(threshold = 1000)
    filter(views__gte: threshold)
  end

  # Instance method
  def publish!
    self.published = true
    save!
  end
end

Article.published.popular.order("-views")
```

Scopes return query sets and are chainable.

---

## When to look elsewhere

- QuerySet API (`filter`, `exclude`, `order`, `join`, `prefetch_related`, aggregates) → `querying.md`
- Migrations (`genmigrations`, `migrate`, squashing) → `migrations.md`
- Polymorphic field `_type` column shadow gotcha, concern macro caveats, `Leafables::*` namespace conflicts with generated paginator inner classes → `gotchas.md`
