## Basics

QuerySets are lazy and chainable. No DB hit until you iterate, print, or call a terminal method.

```crystal
# All records — returns a QuerySet, not an array
Article.all

# Filter (AND by default; chainable)
Article.filter(title__icontains: "marten").filter(author__first_name: "John")

# Exclude
Article.exclude(title__icontains: "draft")

# Single record — returns nil if not found; raises MultipleRecordsFound if >1 match
author = Author.get(id: 42)          # => Author | nil
author = Author.get!(id: 42)         # => Author  (raises RecordNotFound if nil)

# First / last — nil if empty; bang versions raise NilAssertionError
Article.filter(published: true).first
Article.filter(published: true).last!

# Count / exists?
Article.all.count                                   # SELECT COUNT(*)
Article.filter(title__startswith: "Top").count      # count with filter
Article.all.count(:subtitle)                        # count non-null subtitle values
Article.filter(title__startswith: "Top").exists?    # SELECT EXISTS(...)

# Pluck — column values without loading full records
Post.all.pluck("title", "published")
# => [["First article", true], ["Upcoming article", false]]

# Pick — single record column values
Post.filter(pk: 1).pick("title", "published")  # => ["First article", true] | nil
Post.filter(pk: 1).pick!("title", "published") # raises NilAssertionError if not found

# PKs
Post.all.pks  # => [1, 2, 3]
```

---

## Lookups

Format: `field__predicate: value` (default predicate is `exact`, can be omitted).

```crystal
Article.filter(title: "Hello")                  # exact (default)
Article.filter(title__icontains: "hello")        # case-insensitive substring
Article.filter(created_at__gt: 7.days.ago)       # greater than
Article.filter(tags__in: ["crystal", "marten"])  # value in array
Article.filter(subtitle__isnull: true)           # IS NULL
```

### Predicate table

| Suffix | SQL equivalent | Case-sensitive? |
|---|---|---|
| `exact` / _(omitted)_ | `= value` (or `IS NULL` for nil) | yes |
| `iexact` | `= UPPER(value)` | no |
| `contains` | `LIKE '%value%'` | yes |
| `icontains` | `LIKE UPPER('%value%')` | no |
| `startswith` | `LIKE 'value%'` | yes |
| `istartswith` | `LIKE UPPER('value%')` | no |
| `endswith` | `LIKE '%value'` | yes |
| `iendswith` | `LIKE UPPER('%value')` | no |
| `gt` | `> value` | — |
| `gte` | `>= value` | — |
| `lt` | `< value` | — |
| `lte` | `<= value` | — |
| `in` | `IN (...)` | — |
| `isnull` | `IS NULL` / `IS NOT NULL` | — |

### Cross-table lookups

Use `__` to traverse relations. Marten auto-joins the needed tables.

```crystal
Article.filter(author__first_name: "John")
Article.filter(author__hometown__name__startswith: "New")

# Filter by a QuerySet of related records (subquery)
authors = Author.filter(first_name: "John")
Article.filter(author__in: authors)

# Filter by raw FK id (avoids a join)
Article.filter(author_id: 42)
```

### Complex filters with `q` expressions

```crystal
# OR
Article.filter { q(title__startswith: "Top") | q(title__startswith: "10") }

# NOT
Author.filter { -q(first_name: "Alice") }

# AND + OR + grouping
Article.filter {
  (q(title__startswith: "Top") | q(title__startswith: "10")) & -q(author__first_name: "John")
}

# XOR
qs1 | qs2   # combine two QuerySets with OR
qs1 & qs2   # combine with AND
qs1 ^ qs2   # combine with XOR
```

### Raw SQL predicates inside filter

```crystal
Author.filter("first_name = ?", "John")
Author.filter("first_name = :first_name", first_name: "John")
Post.all.filter { q(category: "news") & q("created_at > ?", Time.local - 7.days) }
```

---

## Ordering

```crystal
Article.all.order(:title)             # ASC
Article.all.order("-created_at")      # DESC (string with "-" prefix)
Article.all.order("-published_at", "title")  # multiple keys
Article.all.order(:title).reverse     # flip existing order

# Order by related field
Article.all.order("author__last_name")

# Order by annotation alias
Author.all.annotate { count(:articles) }.order("-articles_count")
```

---

## Slicing / Limits

```crystal
Post.all.limit(10)          # LIMIT 10
Post.all.offset(20)         # OFFSET 20
Post.all.limit(10).offset(20)

# Range slicing (raises IndexError if out of bounds)
Article.all[2..6]           # returns sliced QuerySet or array if already evaluated
Article.all[2..6]?          # returns nil instead of raising

# DISTINCT
Post.all.distinct                   # SELECT DISTINCT
Post.all.distinct(:title)           # SELECT DISTINCT ON (title) — PostgreSQL
Post.all.distinct(:author__name)    # follow association
```

---

## Joins / N+1 Prevention

Use `.join` for single-target relations (many_to_one, one_to_one, reverse one_to_one).
Use `.prefetch` for everything else (many_to_many, reverse many_to_one, collections).

```crystal
# join — single extra SQL join, populates related object inline
article = Article.join(:author).get(id: 42)
article.author  # no extra DB hit

# Deep join
Article.join(:author__hometown).filter(published: true)
Article.join(:author__hometown, :edited_by)  # multiple

# prefetch — separate batch queries per relation
posts = Post.all.prefetch(:tags).to_a
posts[0].tags  # no extra DB hit (tags loaded in one batch)

# Deep prefetch
Author.prefetch(:books__genres, :publisher)

# Prefetch with a custom QuerySet
List.prefetch(:items, query_set: Item.order(:position))
```

| Method | Works with | Strategy |
|---|---|---|
| `join` | many_to_one, one_to_one, reverse one_to_one | SQL JOIN in main query |
| `prefetch` | any relation type | Separate batch query per relation |

---

## Aggregates

```crystal
# Direct aggregates — return a single value
City.all.sum(:population)
City.all.average(:population)
City.all.minimum(:population)
City.all.maximum(:population)
Article.all.count
Article.all.count(:subtitle)   # count non-null values only

# Annotate — add aggregated data to each record in the QuerySet
annotated = Author.all.annotate { count(:articles) }
annotated.each { |a| puts a.annotations["articles_count"] }

# Multiple annotations
Author.all.annotate do
  count(:articles, alias_name: :article_count)
  sum(:articles__score, alias_name: :total_score)
end

# Distinct annotation
Author.all.annotate { sum(:articles__score, distinct: true) }
# or: .annotate { sum(:articles__score).distinct }

# Filter on annotated value (HAVING equivalent)
Author.all.annotate { count(:articles) }.filter(articles_count__gt: 10)

# Order by annotated value
Author.all.annotate { count(:articles) }.order("-articles_count")
```

---

## Mutating

```crystal
# update_all — raw UPDATE, no validation, no callbacks, returns count
Article.filter(title: "Old title").update(title: "New title")  # => Int

# delete — returns count; follows on_delete strategy for related records
Article.filter(title__startswith: "Draft").delete

# Raw delete (skips on_delete logic — risky with FK constraints)
Article.filter(title__startswith: "Draft").delete(raw: true)

# Bulk create — single INSERT, no callbacks, no validation
Post.bulk_create([
  Post.new(title: "First"),
  Post.new(title: "Second"),
], batch_size: 100)

# get_or_create / get_or_create!
tag = Tag.all.get_or_create(label: "crystal")
tag = Tag.all.get_or_create!(label: "crystal") do |t|
  t.active = false
end

# update_or_create / update_or_create!
user = User.all.update_or_create(
  updates: {first_name: "Jane"},
  defaults: {first_name: "Jane", is_admin: true},
  username: "abc"
)
```

---

## Pagination

```crystal
paginator = Article.filter(published: true).paginator(10)
# paginator is Marten::DB::Query::Paginator

paginator.page_size    # => 10
paginator.pages_count  # => 6
paginator.total_count  # => 60

page = paginator.page(1)   # 1-indexed
# page is Marten::DB::Query::Page

page.each { |article| puts article }
page.number                # => 1
page.total_count           # => 60
page.previous_page?        # => false
page.previous_page_number  # => nil
page.next_page?            # => true
page.next_page_number      # => 2
```

---

## Raw SQL

### Mapped to model instances

```crystal
# Returns an iterable RawSet — records are mapped to model instances
Article.raw("SELECT * FROM app_article").each { |a| puts a }

# Positional parameters — use ? placeholders
Article.raw("SELECT * FROM app_article WHERE title = ? AND created_at > ?",
  "Hello World!", "2022-10-30")

# Named parameters — use :name placeholders
Article.raw(
  "SELECT * FROM app_article WHERE title = :title",
  title: "Hello World!"
)
```

Table name is `<app_name>_<model_name_underscored>` unless overridden with `db_table`.

### Low-level connection (DDL, FTS5, arbitrary SQL)

```crystal
# Default DB
Marten::DB::Connection.default.open do |db|
  db.scalar("SELECT 1")
  db.exec("CREATE TABLE IF NOT EXISTS fts_index(content TEXT)")
  rows = db.query("SELECT rowid, content FROM fts_index WHERE fts_index MATCH ?", "crystal")
end

# Named DB
Marten::DB::Connection.get(:replica).open do |db|
  db.scalar("SELECT COUNT(*) FROM app_post")
end
```

Use `db.query` for SELECT, `db.exec` for INSERT/UPDATE/DELETE/DDL, `db.scalar` for single-value returns. Never interpolate user input — always use `?` or `:name` placeholders.

Use raw SQL for: FTS5 full-text search, window functions, complex aggregations, DB-specific features not covered by the QuerySet API.

---

## Transactions

```crystal
# Basic transaction — all-or-nothing
Article.transaction do
  article.save!
  audit_log.save!
end

# Can be called on any model class — the class doesn't matter
MyModel.transaction do
  MyModel.create!(foo: "bar")
  OtherModel.create!(baz: "qux")
end

# Manual rollback without propagating an exception
committed = MyModel.transaction do
  MyModel.create!(foo: "bar")
  raise Marten::DB::Errors::Rollback.new("abort") if should_cancel?
end
committed  # => false if rolled back

# Nested transactions — inner block joins the outer transaction (no savepoints)
MyModel.transaction do
  record_a.save!
  MyModel.transaction do   # merged into outer; single commit/rollback
    record_b.save!
  end
end

# Transaction on a specific database
MyModel.transaction(using: :secondary) do
  record.save!
end
```

Callbacks (`after_create`, `after_update`, `after_save`, `after_delete`) run inside the auto-wrapping transaction. Use `after_commit` callbacks for work that must see the committed state (e.g. enqueuing jobs).

---

## Multiple Databases

```crystal
# Query against a non-default DB
Article.using(:replica).filter(published: true)

# Save / delete to a specific DB
tag = Tag.new(label: "crystal")
tag.save(using: :other_db)
tag.delete(using: :other_db)

# Transaction on a specific DB
MyModel.transaction(using: :other_db) { record.save! }

# Low-level connection to a named DB
Marten::DB::Connection.get(:other_db).open { |db| db.scalar("SELECT 1") }
```

Configure additional databases in settings with `config.database :alias_name do |db| ... end`.

---

## Scopes

```crystal
class Post < Marten::Model
  scope :published { filter(is_published: true) }
  scope :recent    { filter(created_at__gt: 1.year.ago) }
  scope :by_author_id { |author_id| filter(author_id: author_id) }
  default_scope { filter(is_published: true) }
end

Post.published.recent           # chainable
Post.by_author_id(42)
Post.unscoped                   # bypasses default_scope
```

---

## When to look elsewhere

- Defining models, fields, relations → `models.md`
- Migrations / raw-SQL DDL → `migrations.md`
- FTS5 example → `gotchas.md` and the marten-writebook codebase
