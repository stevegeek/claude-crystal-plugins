# Rails → Marten porting reference

Mapping from Rails idioms to idiomatic Marten 0.7 equivalents. Each section is a self-contained recipe.

Code examples use a books domain throughout: books contain `Leaf` rows (a polymorphic join), and each leaf points at one of three content types — `Page`, `Section`, or `Picture`. Substitute your own model names. The leaf/leafable polymorphism shows up in multiple sections because it triggers most of the interesting gotchas.

For testing-specific porting (handler specs, system tests, fixtures → factories), see `rails-testing.md` in this skill.

## Index

- [Project organization — apps](#project-organization)
- [Handlers — prefer generic handlers](#handlers)
- [Authentication](#authentication)
- [Models](#models) — polymorphic, delegated types, concerns, callbacks
- [Forms](#forms)
- [Routing](#routing)
- [Templates](#templates)
- [Frontend](#frontend)
- [Markdown](#markdown)
- [Active Storage](#active-storage)
- [Search](#search)
- [Background jobs](#background-jobs)

---

## Project organization — apps

The biggest mental-model shift. Marten projects compose one or more **apps** (Marten's term for what Rails has informally — engines + namespaces + concerns). Every model, handler, schema, template, and migration belongs to exactly one app, identified by a `label`. The label becomes the table prefix (`books_book`), the migration namespace (`Migration::Books::V…`), and the template path prefix (`books/books/index.html`).

A new project gets an implicit `main` app at `src/`. As soon as the domain has natural boundaries, split into apps — moving later costs migrations that rename tables and updates to every module reference.

### Layouts

Rails has no enforced split:

```
app/
├── controllers/
├── models/
└── views/
```

Marten splits per-app:

```
src/
├── books/                  # Books app
│   ├── app.cr              # class Books::App < Marten::App; label "books"; end
│   ├── routes.cr           # Books::ROUTES = Marten::Routing::Map.draw do … end
│   ├── cli.cr              # require "./migrations/**"
│   ├── models/             # Books::Book, Books::Leaf, …
│   │   └── leafables/      # Books::Leafables::{Page, Section, Picture}
│   ├── handlers/
│   ├── schemas/
│   ├── templates/          # books/<file>.html
│   ├── migrations/
│   └── concerns/
├── accounts/               # Accounts app — User, sessions, profiles
├── server.cr               # Marten.start
├── project.cr              # require "./books/app"; require "./accounts/app"
└── cli.cr
```

Each app is self-contained. The directory IS the boundary — Marten treats every model under `src/books/` as belonging to the books app via the closest `app.cr` ancestor.

### `app.cr` skeleton

```crystal
# src/books/app.cr
require "./models/**"
require "./schemas/**"
require "./handlers/**"
require "./concerns/**"
require "./routes"

module Books
  class App < Marten::App
    label "books"
  end
end
```

The label prefixes table names (`books_book`), prefixes migration class namespaces (`Migration::Books::V20260507091523`), and identifies the app for `installed_apps` ordering (template/asset lookup precedence).

### `routes.cr` per app

```crystal
# src/books/routes.cr
module Books
  ROUTES = Marten::Routing::Map.draw do
    path "", BooksIndexHandler, name: "index"
    path "/new", BooksNewHandler, name: "new"
    path "/<id:int>", BooksShowHandler, name: "show"
    path "/<book_id:int>/pages/new", PagesNewHandler, name: "pages_new"
  end
end
```

Project-level `config/routes.cr` mounts the apps:

```crystal
Marten.routes.draw do
  path "/", include: Accounts::ROUTES
  path "/books", include: Books::ROUTES
end
```

Reverse routing uses `<app_name>:<route_name>`: `reverse("books:show", id: 42)`. In templates: `{% url 'books:show' id: book.id %}`.

### `installed_apps`

```crystal
# config/settings/base.cr
config.installed_apps = [
  MartenAuth::App,
  Books::App,
  Accounts::App,
] of Marten::Apps::Config.class
```

Cross-app references use the full module path: `Books::Book`, `Accounts::User`. Polymorphic `to:` lists, `many_to_one` `to:` args, and `is_a?` checks all use the full namespace.

### Gotchas

1. **Label collisions** between apps produce baffling errors at migration time. Pick distinct labels.
2. **`installed_apps` order matters for template lookup.** If two apps define `home.html`, the first-listed wins. Namespace all template paths (`books/home.html`, not `home.html`).
3. **`Marten::Routing::Map.draw do ... end`** — note `.draw`, not `.new`. Per-app maps use this; project-level `Marten.routes.draw` is a convenience wrapper.
4. **Changing an app label renames its tables** in the next auto-generated migration. Review the diff before applying if data matters.
5. **The polymorphic-shadow gotcha (below) is per-app.** Each model in each app gets its own `::<Model>::Page` paginator. Always namespace would-be-shadowed content types.
6. **The `Marten::App` subclass must live in the app's directory, not a parent.** The directory containing `app.cr` is the directory whose models/migrations/templates the app owns.

### When to split

| Project state | Action |
|---|---|
| Prototype / 1–2 models / spike | Stay in `main` |
| 3+ models with clear domain boundaries | Split |
| Models reusable across projects | Extract as own app, package as shard |
| Single-tenant CRUD, no obvious boundaries | Stay in `main` until boundaries emerge |

---

## Handlers — prefer generic handlers

Marten's `Marten::Handlers::*` family covers ~90% of CRUD. Write a plain `Marten::Handler` subclass only when custom dispatch logic genuinely doesn't fit the generics.

### Mapping table

| Task | Generic handler | Replaces |
|---|---|---|
| List with optional pagination | `Marten::Handlers::RecordList` | `def get; @records = Model.all.to_a; render(...); end` |
| Show one record | `Marten::Handlers::RecordDetail` | `def get; @record = Model.get(pk: params["id"]); render(...); end` |
| Render a static template | `Marten::Handlers::Template` | plain handler with `render(...)` |
| Create from a form | `Marten::Handlers::RecordCreate` | manual `Model.create!(schema.validated_data…)` |
| Update from a form | `Marten::Handlers::RecordUpdate` | manual `record.update!(…)` |
| Delete with confirmation | `Marten::Handlers::RecordDelete` | manual `record.delete` + redirect |
| Just redirect | `Marten::Handlers::Redirect` | `redirect(…)` |
| Form processing, no model | `Marten::Handlers::Schema` | manual `request.data` parsing |

### Example: list handler

```crystal
class BooksIndexHandler < Marten::Handlers::RecordList
  include AuthenticationHelpers

  model Book
  template_name "books/index.html"
  list_context_name "books"
  page_size 20                           # enables ?page=N pagination

  def queryset
    super.order(:title)                  # default is Model.all; chain modifiers in
  end

  before_render :inject_signed_in

  private def inject_signed_in : Nil
    context[:signed_in] = signed_in?
  end
end
```

Free wins from the generic:

- `?page=2` pagination, with `paginator` / `page` available in the template
- `list_context_name` makes the template variable conventionally named (`books`, not `records`)
- `queryset` is the right hook for filtering/scoping
- `before_render` is the canonical place to inject extra context

### Example: create handler

```crystal
class BooksNewHandler < Marten::Handlers::RecordCreate
  include AuthenticationHelpers
  before_dispatch :require_authentication

  model Book
  schema BookSchema
  template_name "books/new.html"

  def success_url
    reverse("books:show", id: record.pk!)   # `record` is the just-created instance
  end
end
```

The generic class auto-instantiates the model from `schema.validated_data`, calls `record.save!`, exposes `record` for `success_url`, and re-renders the template with `schema.errors` populated on validation failure.

### When NOT to use a generic handler

When the action does multi-record work inside a transaction (e.g. one form submit creates a Leaf + a Markdown + a Leafable in one go). A plain `Marten::Handlers::Schema` with `process_valid_schema` doing the transaction is clearer than fighting `RecordCreate`'s single-record assumption.

### Auth-gating

All generic handlers honor lifecycle callbacks:

```crystal
include AuthenticationHelpers
before_dispatch :require_authentication
```

Returning a redirect from `require_authentication` short-circuits the handler before any DB work.

### Gotchas

1. **`pk!` returns `Int32 | Int64`** for `:big_int` primary keys (a union, not pure `Int64`). Cast with `.to_i64` if you need `Int64?` downstream.
2. **`success_route_name` is for static reverse-routing only.** For dynamic URLs (`reverse("books:show", id: record.pk!)`), override `success_url`.
3. **`record_context_name` / `list_context_name` only set the template variable name.** The handler's `record` / `records` accessors are unchanged. Scope via `queryset`; don't override `records`.
4. **Generic detail handlers don't apply Rails-style implicit scoping.** Where Rails idiomatically writes `Book.accessable(Current.user).find(params[:id])`, `Marten::Handlers::RecordDetail` is pure-pk lookup. Add a `before_dispatch` check or override `queryset` for access control.

---

## Authentication

Use the `marten-auth` shard. Don't hand-roll session/cookie/bcrypt — known footguns (timing attacks on token comparison, session fixation, password-reset token entropy) and the shard wraps the same bcrypt you'd reach for anyway.

### Setup

```yaml
# shard.yml
dependencies:
  marten_auth:
    github: martenframework/marten-auth
```

```crystal
# config/settings/base.cr
config.installed_apps = [MartenAuth::App] of Marten::Apps::Config.class

config.middleware = [
  Marten::Middleware::Session,
  MartenAuth::Middleware,                 # MUST come after Session
  Marten::Middleware::Flash,
  # …
]
```

```crystal
# config/initializers/auth.cr — runs after models autoload
Marten.settings.auth.user_model = User
```

```crystal
# src/models/user.cr
class User < MartenAuth::User
  # Inherited: id, email (unique, format-validated), password (bcrypt hash),
  # created_at, updated_at. Class methods: get_by_natural_key, authenticate.
  # Instance methods: set_password, check_password, set_unusable_password,
  # session_auth_hash.

  field :name, :string, max_size: 255, blank: false, null: false
  field :role, :string, max_size: 32, default: "member"
  field :active, :bool, default: true
end
```

### Sign in / sign out

```crystal
class SessionsCreateHandler < Marten::Handlers::Schema
  schema SessionSchema
  template_name "sessions/new.html"

  def process_valid_schema
    user = MartenAuth.authenticate(
      schema.validated_data["email_address"].as(String),
      schema.validated_data["password"].as(String),
    )
    return render(...) if user.nil? || !user.as(User).active

    MartenAuth.sign_in(request, user)
    redirect(reverse("books:index"))
  end
end

class SessionsDeleteHandler < Marten::Handler
  def post
    MartenAuth.sign_out(request)
    redirect(reverse("accounts:session_new"))
  end
end
```

### Handler helpers

```crystal
module AuthenticationHelpers
  protected def signed_in? : Bool
    !request.user.nil?
  end

  protected def current_user : User?
    request.user.try(&.as(User))
  end

  protected def require_authentication : Marten::HTTP::Response?
    return nil if signed_in?
    redirect(reverse("accounts:session_new"))
  end
end
```

### Mapping table

| Rails | Marten |
|---|---|
| `has_secure_password` | inherit `MartenAuth::User`; `user.set_password(raw)` / `user.check_password(raw)` |
| `User.find_by(email: …)` + `bcrypt.is_password?` | `MartenAuth.authenticate(email, password)` returns `User?` |
| `start_new_session_for(user)` + signed cookie | `MartenAuth.sign_in(request, user)` (uses Marten's session backend; no separate Session model) |
| `terminate_current_session` | `MartenAuth.sign_out(request)` |
| `Current.user` (fiber-local class state) | `request.user` / `request.user!` / `request.user?` set per-request by `MartenAuth::Middleware` |
| `before_action :require_authentication` | `before_dispatch :require_authentication` |
| Custom `Session` table with metadata (user_agent, ip, last_active_at) | Marten's session is opaque; track those in a separate audit table |

### Gotchas

1. **Middleware order**: `MartenAuth::Middleware` must come after `Marten::Middleware::Session` (it reads from the session).
2. **`auth.user_model` is set in an initializer**, not in main settings — settings run before model autoload.
3. **`MartenAuth::User#email` is type `:email`** with built-in format validation. Don't re-declare as `:string`.
4. **Don't run `marten gen auth`** if you have custom auth flows. The generator scaffolds templates + handlers + emails on top of the shard; undoing the scaffold is more work than starting from the bare shard.
5. **Crystal stdlib bcrypt raises on passwords > 72 bytes; Ruby silently truncates.** `MartenAuth::User#set_password` propagates the raise. If the Rails app relied on silent truncation, length-validate on the schema: `field :password, :string, min_size: 8, max_size: 72, required: true`.

---

## Models

### Polymorphic associations

Rails:

```ruby
class Leaf < ApplicationRecord
  belongs_to :leafable, polymorphic: true
end
```

Open-ended: any model can be a leafable at runtime; type stored as a string.

Marten:

```crystal
class Leaf < Marten::Model
  field :leafable, :polymorphic,
    to: [Leafables::Page, Leafables::Section, Leafables::Picture]
end
```

Closed-set: target classes listed at compile time. Auto-generates `leafable_type` (string) + `leafable_id` (int) columns plus typed getters/setters.

Per-target sugar auto-generated by the macro:

```crystal
leaf.leafable                 # => Leafables::Page | Leafables::Section | Leafables::Picture | Nil
leaf.leafable = some_page     # writes the type + id columns
leaf.leafable_class           # => Leafables::Page.class | … | Nil
leaf.page_leafable            # => the leafable as Leafables::Page?
leaf.page_leafable!           # => raises if not a Page
leaf.page_leafable?           # => Bool
Leaf.with_page_leafable       # => filter scope
```

#### Mapping table

| Rails | Marten |
|---|---|
| `belongs_to :foo, polymorphic: true` | `field :foo, :polymorphic, to: [Klass1, Klass2, …]` |
| `record.foo_type` (any string) | `record.foo_type` (constrained to listed class names) |
| `record.foo` (resolves via class lookup) | `record.foo` (returns typed union) |

#### Gotchas (these compile-error in subtle ways)

1. **Marten generates `::<Model>::Page < Marten::DB::Query::Page(Model)`** as an inner paginator class on every `Marten::Model`. A top-level `class Page < Marten::Model` is shadowed inside another model's body. Compiler error from `polymorphic.cr`:

   ```
   Error: expected argument 'to' to 'Marten::DB::Field::Polymorphic.new' to be
   Array(Marten::DB::Model.class), not Array(Edit::Page.class | Marten::DB::Model.class)
   ```

   Fix: put would-be-shadowed types in a namespace (`module Leafables; class Page < Marten::Model; … end; end`). References resolve to `::Leafables::Page` from any nested scope.

2. **Single-element polymorphic `to:` arrays don't compile.** `to: [Leafables::Page]` types as `Array(Leafables::Page.class)`, which doesn't unify with the macro's expected `Array(Marten::DB::Model.class)`. Two element types unify via the parent class.

   Fix: list at least two types, OR annotate `to: [Page] of Marten::DB::Model.class`.

3. **Don't `::`-prefix the type list.** The polymorphic macro internally emits `::{{ type }}.register_reverse_relation(...)` — `::Page` becomes `::::Page` and fails with `unexpected token: "::"`. Plain unprefixed `Page` (or `Foo::Page`) works.

4. **Method dispatch through the polymorphic union doesn't always work.** Calling `leaf.leafable.try(&.searchable_content)` where the union is `Page | Section | Picture | Nil` may fail with a strange `compile-time type is Marten::DB::Model+` error even when all three types define the method via an included module. Workaround — explicit case dispatch:

   ```crystal
   case target = leaf.leafable
   when Leafables::Page    then target.searchable_content
   when Leafables::Section then target.searchable_content
   when Leafables::Picture then target.searchable_content
   else                         nil
   end
   ```

   Verbose but unambiguous. Crystal's union-method dispatch is weaker than expected when the union members originate from a polymorphic field rather than direct typing.

### Delegated types

Rails:

```ruby
class Leaf < ApplicationRecord
  delegated_type :leafable, types: %w[Page Section Picture], dependent: :destroy
end
```

Provides `leaf.page`, `leaf.page?`, `leaf.section`, `leaf.section?`, `leaf.picture`, `leaf.picture?`.

Marten's polymorphic field auto-generates analogous accessors but with different names: `page_leafable`, `section_leafable`, `picture_leafable`. Porting templates verbatim breaks because `{{ leaf.page }}` ≠ `{{ leaf.page_leafable }}`. Layer a small macro on top so the model reads like Rails:

```crystal
# src/books/concerns/delegated_type.cr
module Books
  module DelegatedType
    macro delegated_type(field, types)
      {% for type in types.expressions %}
        {% short = type.stringify.split("::").last.underscore.id %}

        def {{ short }}
          {{ short }}_{{ field.id }}
        end

        def {{ short }}? : ::Bool
          {{ short }}_{{ field.id }}?
        end
      {% end %}
    end
  end
end
```

Usage:

```crystal
class Leaf < Marten::Model
  include DelegatedType

  field :leafable, :polymorphic, to: [Leafables::Page, Leafables::Section, Leafables::Picture]
  delegated_type :leafable, types: [Leafables::Page, Leafables::Section, Leafables::Picture]
end
```

`leaf.page` / `leaf.page?` / `leaf.section` / `leaf.section?` now work in templates.

#### Gotcha

**Macros defined inside `macro included` blocks don't become callable on the host class.** Define `macro delegated_type` directly on the module, not nested inside `macro included`. Marten's own `template_attributes` macro shows the working pattern.

### Concerns

Rails:

```ruby
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_save :populate_slug
  end

  def populate_slug; …; end

  class_methods do
    def find_by_slug(s); find_by(slug: s); end
  end
end
```

Marten/Crystal uses `macro included` for `included do` blocks, and module-level `def self.foo` for `class_methods`.

**But:** Crystal's macro scoping does not propagate registrations into model-level callback constants (`VALIDATION_CALLBACKS`, etc.) from inside a concern's `macro included`. The pattern looks correct, the macro runs, the callback silently no-ops:

```crystal
# Looks right; doesn't register:
module Sluggable
  macro included
    before_validation :populate_slug    # silent no-op
  end
end
```

(Handler-side concerns using the same pattern work — the issue is specific to model lifecycle macros.)

**Workaround:** use **helper modules** (no callbacks), and have each host model declare its own `before_validation`:

```crystal
module SluggableHelpers
  extend self

  def populate_if_blank(current : String?, source : String, fallback = "untitled") : String
    return current.not_nil! if !current.nil? && !current.empty?
    derived = source.downcase.gsub(/[^a-z0-9]+/, "-").strip('-')
    derived.empty? ? fallback : derived
  end
end

class Book < Marten::Model
  field :slug, :string, max_size: 255

  before_validation :populate_slug

  private def populate_slug : Nil
    self.slug = SluggableHelpers.populate_if_blank(slug, title.to_s)
  end
end
```

### Commit callbacks

Marten 0.7 ships `after_create_commit`, `after_update_commit`, `after_save_commit`, `after_delete_commit`, `after_create_rollback`, and so on. Drop-in for Rails equivalents:

```crystal
class Leaf < Marten::Model
  after_create_commit  :create_in_search_index
  after_update_commit  :update_in_search_index
  after_delete_commit  :remove_from_search_index
end
```

### `before_validation` vs `before_create` for `null: false` fields

If a callback populates a `null: false` field, use **`before_validation`, not `before_create`**. Marten runs validation before save, so `before_create` (which fires after validation) is too late — validation already failed on the null field.

---

## Forms

Strong params → `Marten::Schema`. Schemas validate AND coerce; access via `schema.validated_data["field"].as(String)`.

Rails:

```ruby
def book_params
  params.require(:book).permit(:title, :subtitle, :author, :theme)
end
```

Marten:

```crystal
class BookSchema < Marten::Schema
  field :title, :string, max_size: 255, required: true
  field :subtitle, :string, required: false
  field :author, :string, required: false
  field :theme, :string, max_size: 32, required: false
end

class BooksNewHandler < Marten::Handlers::Schema
  schema BookSchema
  template_name "books/new.html"

  def process_valid_schema
    title = schema.validated_data["title"].as(String)
    book = Book.create!(title: title, …)
    redirect(reverse("books:show", id: book.pk))
  end
end
```

| Rails | Marten |
|---|---|
| Strong params (`require/permit`) | `Marten::Schema` field declarations (typed) |
| `model.errors.full_messages` | `schema.errors` populated by Marten |
| `respond_to do { format.html / format.json }` | Match on `request.accepted_media_type` (no DSL sugar) |
| `params.require(:foo)` | `schema.validated_data["foo"].as(String)` |

Schemas catch coercion errors (e.g. `:int` field with non-numeric input) at validation time, not later in the handler.

---

## Routing

| Rails | Marten |
|---|---|
| `resources :books, only: [...]` | One `path` per action: `path "/books", BooksIndexHandler, name: "index"` |
| `book_path(book)` | `reverse("books:show", id: book.pk)` |
| `direct :foo do |b| … end` (inline named route) | A plain Crystal method that calls `Marten.routes.reverse(...)` |
| `dom_id(record)` → `"book_42"` | `"#{record.class.name.underscore}_#{record.pk}"` |
| `get "/:id", constraints: { id: /\d+/ }` | `path "/<id:int>", Handler` (typed path params) |

**Syntactic gotcha:** `{% url 'name' key: val %}` uses a **colon**. `{% with key=val %}` uses **equals**. Different tags, different conventions.

---

## Templates

ERB → Django-style Marten templates.

| ERB | Marten |
|---|---|
| `<%= @foo %>` | `{{ foo }}` |
| `<%= h(foo) %>` (auto in Rails 7+) | `{{ foo }}` (auto-escaped) |
| `<%== foo %>` (raw) | `{{ foo|safe }}` |
| `<%= form_with model: @book do %>` | Plain HTML `<form method="post">` + `{% csrf_input %}` |
| `<%= csrf_meta_tags %>` | `{% csrf_input %}` (form input, not meta tag) |
| `<%= render 'shared/header' %>` | `{% include "shared/_header.html" %}` |
| `<%= link_to 'X', book %>` | `<a href="{% url 'books:show' id: book.id %}">X</a>` |
| `<%= image_tag "logo.svg" %>` | `<img src="{% asset 'images/logo.svg' %}">` |
| `<%= @items.length %>` | `{{ items|size }}` (no `length` filter) |
| `<% @books.each do |book| %>` | `{% for book in books %}` … `{% endfor %}` |
| `<%= render layout: 'card' do %>...<% end %>` | `{% block card %}` + `{% extend %}` |
| `<%= number_with_delimiter(n) %>` | Custom filter or compute server-side |

### Gotchas

1. **`{# … #}` and `<!-- … -->` don't span newlines.** Multi-line "comments" containing `{% tag %}` parse the tag as real syntax. Same trap inside `<script>` blocks. Wrap with `{% verbatim %}…{% endverbatim %}` or paraphrase.

2. **Models need explicit `template_attributes :name, :foo, :bar`** to expose fields. Without it, `{{ book.title }}` raises `UnknownVariable`. Rails' instance methods are callable from views by default; Marten's are not.

3. **`{% csrf_input %}` not `{% csrf_token %}`.** The former emits `<input type="hidden" name="csrftoken" value="...">`; the latter emits the bare token value (useful only in JS).

4. **`{% include … with k=v, k2=v2 %}` requires commas between assignments.** Spaces alone aren't separators. Without commas: `Filter expression ends with characters that cannot be parsed properly: …`.

5. **Boolean literals are lowercase.** `true` / `false` / `nil` — never `True` / `False` / `None`. Anything else is treated as a variable lookup (silently resolves to nil if absent).

6. **Turbo broadcast partials should not include `{% turbo_stream %}`.** `MartenTurbo.broadcast_*_to(stream, partial: "...")` wraps the partial body in `<turbo-stream action="…" target="…">` itself. Nesting a turbo_stream tag inside produces nested `<turbo-stream>` markup. Rails behaves the same way despite the `.turbo_stream.erb` convention.

---

## Frontend

| Rails | Marten |
|---|---|
| `propshaft` (asset digesting) | None — serve under `/assets/...` from `src/assets/` (via `config.assets.dirs`); cache headers via reverse proxy |
| `importmap-rails` (`config/importmap.rb` + `<%= javascript_importmap_tags %>`) | Inline `<script type="importmap">` in `base.html`, hand-curated; pin npm packages via jsdelivr/unpkg ESM URLs |
| `pin_all_from "app/javascript/controllers"` | Loop or copy-paste each pin with `{% asset 'javascript/controllers/foo.js' %}` |
| `Stimulus.application.start()` + auto-discovery | Manual `Stimulus.register("foo-bar", FooBarController)` for each |
| `@hotwired/turbo-rails` JS package | `@hotwired/turbo` (no `-rails`) — drop the Rails-specific bits |
| `<%= javascript_include_tag "application" %>` | A `<script type="module">` block with explicit imports |

### Gotchas

1. **`<turbo-cable-stream-source>` ships in `@hotwired/turbo-rails`'s npm package**, not `@hotwired/turbo`. For WS-driven Turbo Streams, define the ~20-line custom element yourself in `base.html`.
2. **`@rails/request.js` reads CSRF from `<meta name="csrf-token">`** and sends it as a header. Marten expects a `csrftoken` form param/cookie. POSTs from `@rails/request.js` will fail CSRF until reworked, or until you emit a compatible meta tag and adjust Marten's CSRF source.
3. **`@hotwired/stimulus-loading` (eager-load)** ships with `stimulus-rails`. No CDN equivalent; register controllers explicitly.

---

## Markdown

`redcarpet` + `rouge` → `markd` + `tartrazine`.

| Rails | Marten/Crystal |
|---|---|
| `redcarpet (~> 3.6)` (CommonMark + extensions) | [`icyleaf/markd`](https://github.com/icyleaf/markd) (CommonMark, smaller surface) |
| `Redcarpet::Markdown.new(html, opts)` | `Markd.to_html(source, Markd::Options.new(smart: true))` |
| `Redcarpet::Render::HTML` subclass with `header`/`image`/`block_code` overrides | Post-process the HTML string with regex (no AST hooks in markd 0.5) |
| `rouge` (200+ language syntax highlighting) | [`tartrazine`](https://github.com/ralsina/tartrazine) (Pygments/Chroma port, 241 languages) |
| `Rouge::Plugins::Redcarpet` mixin | Regex `<pre><code class="language-X">…</code></pre>` blocks, pipe through `Tartrazine::Html` formatter |

### Gotchas

- markd's code-block output is `<pre><code class="language-X">code-with-html-entities</code></pre>`. Decode entities before passing to tartrazine.
- markd ships no render-hook callbacks; post-process HTML with regex for headings + images. Brittle but practical.
- Image wrapping (e.g. lightbox): turn `<img>` into `<a href><img></a>` with a regex. Safe because markd-emitted images have a stable shape.

---

## Active Storage

Rails Active Storage uses three tables (`active_storage_blobs`, `active_storage_attachments`, `active_storage_variant_records`) and computes variants on-demand. Collapse into one polymorphic `Attachment` row, and pre-compute variants at upload time using `crystal-vips`.

```crystal
class Attachment < Marten::Model
  field :record, :polymorphic, to: [Book, Leafables::Picture, Markdown]
  field :name, :string                 # "cover" | "image" | "uploads"
  field :file, :file, upload_to: "attachments"
  field :variant_of, :many_to_one, to: Attachment, blank: true, null: true
  field :variation_kind, :string, blank: true, null: true   # "large" | …
  # plus content_type, byte_size, slug
end
```

```crystal
AttachmentHelpers.attach(
  record: picture, name: "image", uploaded_file: io,
  variants: {"large" => {max_dimension: 1500}},
)
```

The variant pipeline opens the original through Marten's storage backend, pipes it through `Vips::Image.new_from_file → resize → write_to_file`, then saves a second `Attachment` row with `variant_of_id` pointing at the original.

### Mapping table

| Rails | Marten |
|---|---|
| `has_one_attached :cover` | `AttachmentHelpers.find_one(record: book, name: "cover")` |
| `has_many_attached :uploads` | `AttachmentHelpers.find_many(record: markdown, name: "uploads")` |
| `attachable.variant :large, resize_to_limit: [1500, 1500]` | `attach(..., variants: {"large" => {max_dimension: 1500}})` (pre-computed) |
| `attachment.url` | `attachment.file.url` |
| `record.cover.purge` | `attachment.delete` (cascade-deletes variants via FK) |
| `service: :amazon` (S3) | `Marten.settings.media_files.storage = MyS3Storage.new(...)` (subclass `Marten::Core::Storage::Base`) |

### Constraints

- **No on-demand variants.** Every variant kind is computed at upload — saves a job-runner dependency. For on-demand variants, plumb a job system (mosquito) and rebuild this layer.
- **`crystal-vips` requires libvips installed.** Falls back gracefully when the original isn't a vips-readable image (e.g. PDF).

### Gotcha: required-on-first-save vs required-always

Rails' `has_one_attached` doesn't validate "image present" by default — the model exists with no attachment, and the form's `file_field` is plain HTML. The natural Marten translation `field :image, :file, required: true` on the schema **breaks inline-create-then-edit flows** because the empty record has no image yet, and the user can't save other fields without re-uploading.

Right shape: `:image, :file, required: false` on the schema; enforce "image required on first save" in the handler if needed (or allow image-less records and render a placeholder). The handler checks `schema.validated_data["image"]?.as(Marten::HTTP::UploadedFile?)` and only calls `AttachmentHelpers.attach` when present.

---

## Search

Marten has no built-in full-text search. Use SQLite FTS5 (or PostgreSQL `tsvector`) via raw SQL migrations.

```crystal
# src/migrations/202605070000001_create_leaf_search_index.cr
class Migration::Main::V202605070000001 < Marten::Migration
  def plan
    execute(
      "CREATE VIRTUAL TABLE leaf_search_index USING fts5(
        title, content, content='', tokenize='porter unicode61 remove_diacritics 2'
      )",
      "DROP TABLE leaf_search_index",
    )
  end
end
```

```crystal
module Searchable
  # Note: see Concerns gotcha — model-level callbacks inside `macro included`
  # may silently no-op. Inline `after_*_commit` directly in the host model
  # if registration doesn't take.

  def self.search(terms : String?)
    sql = <<-SQL
      SELECT main_leaf.id,
             highlight(leaf_search_index, 0, '<mark>', '</mark>') AS title_match,
             snippet(leaf_search_index, 1, '<mark>', '</mark>', '...', 20) AS content_match
      FROM main_leaf JOIN leaf_search_index ON main_leaf.id = leaf_search_index.rowid
      WHERE main_leaf.status = 'active' AND leaf_search_index MATCH ?
      ORDER BY bm25(leaf_search_index, 2.0)
      LIMIT 50
    SQL

    Marten::DB::Connection.default.open do |db|
      db.query(sql, terms) do |rs| ... end
    end
  end
end
```

| Rails | Marten |
|---|---|
| `connection.execute(sql)` | `Marten::DB::Connection.default.open { |db| db.exec/query(sql, *args) }` |
| `Arel.sql("bm25(...)")` | Plain string literal (no Arel) |
| `def search(terms); …; end` as a scope | Module method on a concern, or class method on the model |

Marten's raw-SQL escape hatch is the same `crystal-db` API used elsewhere in the Crystal ecosystem. Bind params with `?` placeholders.

---

## Background jobs

Marten has no built-in job system. For Rails apps that used Resque/Sidekiq purely for Active Storage analyze/variant work, the recommended replacement is **pre-computed variants at upload time** (see [Active Storage](#active-storage)) — no job runner needed for the common case.

If a job system is genuinely needed: [`mosquito-cr/mosquito`](https://github.com/mosquito-cr/mosquito) is the Crystal-native Redis-backed equivalent. Job classes are plain Crystal classes with a `perform` method; the worker is shipped as `bin/mosquito_runner`.
