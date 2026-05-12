## Models / DB

The cross-cutting list. Sourced from real porting experience (Rails → Marten on the Writebook port). When something seems weird, check here before giving up.

### Polymorphic field — closed-set, with footguns

```crystal
field :leafable, :polymorphic, to: [Leafables::Page, Leafables::Section, Leafables::Picture]
```

- **Single-element `to:` arrays don't compile.** `to: [Page]` types as `Array(Page.class)`, doesn't unify with the macro-expected `Array(Marten::DB::Model.class)`. Pad with a placeholder type, OR annotate `to: [Page] of Marten::DB::Model.class`.
- **Don't prefix `::`.** Marten's macro emits `::{{ type }}.register_reverse_relation(...)`, so `::Page` becomes `::::Page` and fails parse. Plain `Page` or `Foo::Page` works.
- **Marten generates `::<Model>::Page < Marten::DB::Query::Page(Model)`** as a paginator inner class on every `Marten::Model`. So a top-level `class Page < Marten::Model` gets shadowed inside other models' bodies. Compiler error: `Array(Edit::Page.class | Marten::DB::Model.class)`. **Workaround:** put would-be-shadowed models in a namespace (`Leafables::Page`). Same shadow exists for `Paginator`.
- **Method dispatch through the polymorphic union can fail strangely.** `leaf.leafable.try(&.searchable_content)` may error with `for Attachment (compile-time type is Marten::DB::Model+)` even if all targets define the method via an included module. Use explicit case dispatch:
  ```crystal
  case target = leaf.leafable
  when Leafables::Page    then target.searchable_content
  when Leafables::Section then target.searchable_content
  when Leafables::Picture then target.searchable_content
  else                         nil
  end
  ```

### `before_create` runs AFTER validation

If you populate a `null: false` field via a callback, **use `before_validation`, not `before_create`**. Validation fires first and will fail on the still-nil column.

```crystal
# WRONG — fails with "field cannot be null"
before_create :generate_token

# RIGHT
before_validation :generate_token
```

### Concern `macro included` callbacks may silently no-op

Crystal's macro scoping does not always propagate registrations into class-level macro CONSTANTS like `VALIDATION_CALLBACKS` from inside a concern. Marten's own concerns (e.g. `Marten::Handlers::XFrameOptions`) use the pattern and DO work, so it's something subtle about model-vs-handler lifecycle.

```crystal
# Pattern that LOOKS right but didn't actually register on Marten::Model:
module Sluggable
  macro included
    before_validation :populate_slug   # silently doesn't register
  end
end
```

**Workaround:** helper-module pattern. Concerns expose plain methods; each host model declares its own `before_validation`:

```crystal
module SluggableHelpers
  extend self
  def populate(current : String?, source : String) : String
    return current.not_nil! if current && !current.empty?
    source.downcase.gsub(/[^a-z0-9]+/, "-").strip('-')
  end
end

class Book < Marten::Model
  before_validation :populate_slug
  private def populate_slug : Nil
    self.slug = SluggableHelpers.populate(slug, title.to_s)
  end
end
```

### `template_attributes` is required

Models don't auto-expose fields to templates. **You must declare `template_attributes :id, :foo, :bar`** explicitly on each model. Without it, `{{ obj.foo }}` raises `UnknownVariable` (no helpful pointer to the missing declaration).

```crystal
class Book < Marten::Model
  include Marten::Template::CanDefineTemplateAttributes
  field :id, :big_int, primary_key: true, auto: true
  field :title, :string, max_size: 255
  with_timestamp_fields

  template_attributes :id, :title, :created_at, :updated_at
end
```

### `record Foo` (struct) and `class Foo < Marten::Model` clash

If you `record Foo, ...` (Crystal struct macro) anywhere in a project, `class Foo < Marten::Model` later will fail in generators with `Foo is not a class, it's a struct`. Either rename or don't use `record`.

### `String#split(' ')` doesn't collapse whitespace

Crystal's `"a   b".split(' ')` returns `["a", "", "", "b"]` (Ruby's collapses). When porting Rails code, use `.split(' ').reject(&.empty?)` or `.split(/\s+/)`.

## Forms / schemas

### Use `Marten::Schema`, not raw `request.data` parsing

```crystal
# WRONG — works but loses validation, type coercion, error messaging
def post
  title = (request.data["title"]?.try(&.to_s) || "").strip
  return render(...) if title.empty?
  Book.create!(title: title)
end

# RIGHT
class BookSchema < Marten::Schema
  field :title, :string, max_size: 255, required: true
end

class BooksNewHandler < Marten::Handlers::Schema
  schema BookSchema
  template_name "books/new.html"
  def process_valid_schema
    title = schema.validated_data["title"].as(String)
    Book.create!(title: title)
    redirect(...)
  end
end
```

### `{% csrf_input %}` not `{% csrf_token %}`

- `{% csrf_input %}` → `<input type="hidden" name="csrftoken" value="...">`. Use in forms.
- `{% csrf_token %}` → raw token string. Useful in JS, but **not** in form templates.

### Schema validations fire BEFORE handler logic

In `Marten::Handlers::Schema`, `process_valid_schema` only runs after the schema is validated. If you put auth checks AFTER schema validation in the handler, they run too late. Use `before_dispatch :require_authentication` to run them first.

## Templates

### Multi-line comments don't work the way you expect

`{# … #}` and `<!-- … -->` lexer regexes don't match across newlines. So:

```html
<!-- multi-line comment
  containing {% csrf_input %} as an example
-->
```

…will parse the `csrf_input` tag as real syntax and fail. **Same trap inside `<script>` blocks**: `{% turbo_stream_from %}` written as a literal in a JS comment will be parsed.

**Workarounds:**
- Single-line comments only, OR
- Wrap in `{% verbatim %}…{% endverbatim %}`.

### `{% url %}` colon-syntax for kwargs

```html
{% url 'book_show' id: book.id %}     {# right #}
{% url 'book_show' id=book.id %}      {# wrong — fails parse #}
```

**But** `{% with %}` uses equals:

```html
{% with username=user.name %}{{ username }}{% endwith %}
```

Different tags, different conventions. Easy to mix up.

### No `length` filter

Use `|size`. `|size` works on arrays, hashes, strings.

### `Marten::Template::Context#merge` returns new (no `merge!`)

```crystal
def context
  super.merge({"foo" => bar})  # right — returns new Context
end

# Wrong: super.merge!(...) — undefined method
```

## Routing

### Path-param syntax: `<name:type>`, not `:name`

```crystal
path "/books/<id:int>", BooksShowHandler, name: "book_show"   # right
path "/books/:id", ...                                         # wrong (Rails)
```

Built-in types: `int`, `slug`, `path`, `str`, `uuid`. Custom types via `Marten::Routing::Parameter::Base` subclass.

### `params["id"]?` vs `request.query_params["page"]?`

`params` is for path-routing captures (`<id:int>`). Query-string params live on `request.query_params`. Form params live on `request.data` (or — preferably — `schema.validated_data`).

## Auth (marten-auth)

### Middleware order matters

`MartenAuth::Middleware` reads from the session, so it must come AFTER `Marten::Middleware::Session`:

```crystal
config.middleware = [
  Marten::Middleware::Session,
  MartenAuth::Middleware,    # ← after Session
  Marten::Middleware::Flash,
  ...
]
```

### `auth.user_model` set in an initializer, not settings

`config/settings/base.cr` runs before model autoload, so `User` isn't defined there yet. Set it in `config/initializers/auth.cr`:

```crystal
Marten.settings.auth.user_model = User
```

### `MartenAuth::User#email` is the natural key

`MartenAuth.authenticate(natural_key, password)` looks up by `email` (not username). The base User model has `field :email, :email, unique: true` already; subclasses inherit it. Don't add a separate `email_address` field.

### `set_password` not `password=`

Raw `password=` would store the plaintext into the bcrypt-hash column. Use `user.set_password(raw)` to encrypt + assign. `user.check_password(raw)` for verification.

## CLI / build

### `script/cr` and `script/serve` wrappers exist for a reason

asdf 0.18 (the Go rewrite) doesn't propagate `CRYSTAL_LIBRARY_PATH` from the asdf-crystal bash wrapper, so direct `crystal build` / `crystal run` calls fail with `ld: library 'gc' not found`. Every workspace project ships a `script/serve` (and sometimes `script/cr` / `script/manage`) wrapper that exports the path before invoking crystal.

```bash
export CRYSTAL_LIBRARY_PATH="$HOME/.asdf/installs/crystal/1.20.1/embedded/lib"
```

### Shard name mismatches break requires

Shard `name:` field uses underscores; the entry-point file at `src/<name>.cr` must match. `name: marten_cable` requires `src/marten_cable.cr`, NOT `src/marten-cable.cr`. Same for require paths — `require "marten_cable"`, not `require "marten-cable"`.

### `shard.override.yml` is gitignored, recreate per machine

Workspace path-pinning lives there. Canonical content for marten-* projects:

```yaml
dependencies:
  marten:
    path: ../marten-src
  encoded_id_cr:
    path: ../encoded-id-cr   # if marten-encoded-id is in shard tree
```

If a transitive dep is path-pinned but a parent shard declares it as a github source, you'll hit `shard name has ambiguous sources` — fix by adding the path-pin to `shard.override.yml`.

## Misc

### `annotation` is a Crystal keyword

If you name a macro arg or method param `annotation`, every USE site of that macro fails parse with a baffling `expecting token 'CONST', not '}'` error. Rename the param.

### `cable-cr/cable` default `route` is broken

`Cable.settings.route` defaults to `Cable.message(:default_mount_path)`, which looks up the wrong key in `INTERNAL`. Set explicitly:

```crystal
Cable.configure do |settings|
  settings.route = "/cable"
end
```

### `pk!` returns `Int32 | Int64` (a union) for `:big_int` PKs, not pure `Int64`

```crystal
field :id, :big_int, primary_key: true, auto: true

book.pk!         # => Int32 | Int64
book.pk!.to_i64  # => Int64
```

If a downstream method is typed `Int64?`, cast explicitly: `book.pk!.to_i64`. Comes up in `reverse(...)` calls and SQL bind-params. Same caveat for any `:big_int` field accessed via the `name!` (not-nil) accessor.

### Template context: NamedTuples auto-convert to dot-accessible Hash values

You can pass `{leaf: leaf, title_match: "...", content_match: "..."}` (a NamedTuple) as a value inside the context Hash, and templates can do `{{ result.leaf }}`, `{{ result.title_match }}` — Marten's template engine converts the NamedTuple to a Hash and dot-notation walks the keys.

This means you can return arrays of NamedTuples from queries and iterate them directly in templates without defining a wrapper type:

```crystal
results = Searchable.search(q)  # => Array(NamedTuple(leaf: Leaf, title_match: String?, content_match: String?))
render("books/searches/create.html", context: {results: results})
```

```html
{% for result in results %}
  <h3>{{ result.title_match|safe }}</h3>
  <p>{{ result.content_match|safe }}</p>
{% endfor %}
```

### FTS5 `highlight` / `snippet` output is HTML — needs `|safe`

SQLite's `highlight(...)` and `snippet(...)` functions return strings containing literal `<mark>...</mark>` tags. Marten templates auto-escape by default, so `{{ result.title_match }}` renders `&lt;mark&gt;...&lt;/mark&gt;` (visible markup, not bold). Use `|safe`:

```html
<h3>{{ result.title_match|safe }}</h3>
```

Same caveat for any other column that legitimately contains HTML you control (e.g., rendered Markdown body — though you should pre-sanitize before storing if user-generated).

### Signed tokens in URL path segments need `Base64.urlsafe_encode`, not `tr`-substitution

`Marten::Core::Signer#sign(value)` returns a string of the form `base64(payload)--hex_hmac`. Putting that directly into a URL path segment is unsafe: the underlying base64 may contain `+`, `/`, `=` characters, AND the `--` separator can collide with path-normalization rules. The intuitive fix `signed.tr("+/=", "-_.")` is **broken** for two reasons:

1. The token already contains `--` as a separator — any later `tr` that targets `-` corrupts it.
2. Path normalisation (server-side or CDN-side) collapses any `..` sequence — substituting `=` with `.` lands you with `..` and a 400.

**Fix:** wrap the entire signed string in `Base64.urlsafe_encode(signed, padding: false)`. The outer URL-safe encoding makes the whole token opaque alphanumeric + `-_`. Reverse with `Base64.decode_string` before passing to `signer.unsign`.

```crystal
# Generating
signer = Marten::Core::Signer.new("transfer/" + Marten.settings.secret_key)
signed = signer.sign(user.id.to_s, expires: 4.hours.from_now)
url_token = Base64.urlsafe_encode(signed, padding: false)

# Verifying
signed = Base64.decode_string(url_token)
user_id = signer.unsign(signed)
```

### `I18n.translate` does NOT take a `locale:` kwarg — use `I18n.with_locale(...) { ... }`

The `crystal-i18n` shard's `I18n.translate` signature looks like it'd accept a locale, but it doesn't:

```crystal
def self.translate(key, params = nil, count = nil, scope = nil, default = nil, **kwargs)
```

The `**kwargs` is for interpolation params, not locale switching. Passing `locale: "es"` silently falls through to the current (default) locale. **Use the block form to switch:**

```crystal
# WRONG — silently returns the default-locale translation
I18n.translate("field_labels.email_address", locale: "es")

# RIGHT
I18n.with_locale("es") do
  I18n.translate("field_labels.email_address")
end
```

Bit me when porting Writebook's translation_button popover (shows the same field label in 6 languages side-by-side) — `translate(..., locale:)` returned English for all.

### `Cable::Handler` must sit ABOVE `Marten::Server::Handlers::Middleware`, not between Middleware and Routing

If you wire cable into Marten via `MartenCable.use(...)` (or hand-roll the handler chain), the `Cable::Handler` MUST be inserted **above** Marten's `Middleware` handler. On a WebSocket upgrade, cable hijacks the socket via `ws.call(context)` and returns without writing to `context.marten.response`. If cable sits BELOW middleware, Marten's `Middleware#get_final_response` then runs `context.marten.response.not_nil!` after `call_next` returns and crashes:

```
Nil assertion failed (NilAssertionError)
  from src/nil.cr:113:7 in 'not_nil!'
  from lib/marten/src/marten/server/handlers/middleware.cr:30:11 in 'get_final_response'
```

Correct chain (cable ABOVE middleware):

```
HTTP::ErrorHandler
DebugLogger / Logger
Marten::Server::Handlers::Error      # pass-through, doesn't read response
Cable::Handler                        # hijacks socket on WS upgrade
Marten::Server::Handlers::Middleware
Marten::Server::Handlers::Routing
```

`Error` and `DebugLogger` only `call_next` and `rescue` — they don't touch `context.marten.response`, so cable short-circuiting above them is safe. For non-WS requests, `Cable::Handler#call` falls through via `call_next(context)` and the chain proceeds into Middleware → Routing unchanged.

The marten-cable shard's `MartenCable.use` macro was originally wrong (placed cable BELOW middleware). Fixed in workspace-pinned `marten-cable@main` (commit pending).

### `<turbo-cable-stream-source>` ships in `@hotwired/turbo-rails`, not `@hotwired/turbo`

The base Turbo package has no Action Cable client. Apps that don't pull `turbo-rails` JS need a ~20-line custom-element shim using `@rails/actioncable` + `Turbo.connectStreamSource`. See `example_app/src/templates/base.html` for the template.
