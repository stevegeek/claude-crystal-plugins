## Plain handlers

Subclass `Marten::Handler`. Override per-method verbs; unhandled verbs return 405 automatically.

```crystal
class ArticlesHandler < Marten::Handler
  def get
    articles = Article.all.order("-created_at").to_a
    render("articles/index.html", context: {articles: articles})
  end

  def post
    # process something, then redirect
    redirect(reverse("articles:index"))
  end
end
```

**Path captures** come from `params`, not query/form data:

```crystal
def get
  article = Article.get(pk: params["pk"])
  return head :not_found if article.nil?
  render("articles/detail.html", context: {article: article})
end
```

> **Footgun:** `params` is path-capture-only. Query string is `request.query_params["page"]?`; form body is `request.data["field"]?`.

Other per-verb methods: `def put`, `def patch`, `def delete`.

---

## Rendering templates

```crystal
render("books/index.html", context: {books: books, signed_in: signed_in?})
```

Context accepts a NamedTuple or Hash. Template engine is Django-style — see `templates.md` for syntax. `content_type` and `status` are optional keyword args (defaults: `text/html`, `200`).

---

## Returning JSON / plain text / status

```crystal
# JSON — sets Content-Type: application/json automatically
json({ok: true, id: record.pk})
json({error: "not found"}, status: 404)

# Plain text or custom content type
respond("pong", content_type: "text/plain", status: 200)

# Headers-only, no body
head :no_content    # 204
head :not_found     # 404
```

Status can be an integer or a symbol matching Crystal's `HTTP::Status` enum values.

---

## Redirects

```crystal
# Named route reverse — preferred
redirect(reverse("articles:detail", pk: article.pk))

# Literal URL
redirect("/books")

# Permanent (301) instead of default temporary (302)
redirect(reverse("home"), permanent: true)
```

---

## Generic handlers

Pick the right base class and configure it with class-level DSL macros. Override instance methods only when you need custom logic.

### `Marten::Handlers::Template` — render a template on GET

```crystal
class HomeHandler < Marten::Handlers::Template
  template_name "app/home.html"

  before_render :populate_context

  private def populate_context : Nil
    context[:recent] = Article.all.order("-created_at")[:5]
  end
end
```

### `Marten::Handlers::Schema` — form processing

GET renders the form; POST validates and redirects on success, re-renders with errors on failure. See `schemas.md` for the full schema bridge.

```crystal
class SessionsCreateHandler < Marten::Handlers::Schema
  schema SessionSchema
  template_name "sessions/new.html"
  success_route_name "books_index"

  def process_valid_schema
    user = MartenAuth.authenticate(
      schema.validated_data["email_address"].as(String),
      schema.validated_data["password"].as(String),
    )
    return render("sessions/new.html", context: {errors: ["Invalid credentials"]}, status: 422) if user.nil?
    MartenAuth.sign_in(request, user)
    redirect(reverse("books_index"))
  end
end
```

> **Footgun:** Schema validation runs before `process_valid_schema`. Auth/authorization checks belong in a `before_dispatch` callback, not inside `process_valid_schema`.

### `Marten::Handlers::RecordList` — paginated list

```crystal
class ArticleListHandler < Marten::Handlers::RecordList
  model Article
  template_name "articles/index.html"
  page_size 20                     # enables pagination; page number from ?page=
  list_context_name "articles"     # default is "records"

  def queryset
    super.filter(published: true).order("-published_at")
  end
end
```

### `Marten::Handlers::RecordDetail` — single record

```crystal
class ArticleDetailHandler < Marten::Handlers::RecordDetail
  model Article
  template_name "articles/detail.html"
  # looks up by "pk" route param by default; override with lookup_param "slug"
  record_context_name "article"    # default is "record"
end
```

Raises `Marten::HTTP::Errors::NotFound` automatically when the record is missing.

### `Marten::Handlers::RecordCreate` — create via schema

```crystal
class ArticleCreateHandler < Marten::Handlers::RecordCreate
  model Article
  schema ArticleCreateSchema
  template_name "articles/create.html"
  success_route_name "articles:index"

  def success_url
    reverse("articles:detail", pk: record.pk!)
  end
end
```

### `Marten::Handlers::RecordUpdate` — update via schema

```crystal
class ArticleUpdateHandler < Marten::Handlers::RecordUpdate
  model Article
  schema ArticleUpdateSchema
  template_name "articles/update.html"
  success_route_name "articles:index"

  def queryset
    super.filter(author: request.user)   # scope to current user
  end
end
```

### `Marten::Handlers::RecordDelete` — confirm then delete

GET renders a confirmation template; POST deletes the record.

```crystal
class ArticleDeleteHandler < Marten::Handlers::RecordDelete
  model Article
  template_name "articles/delete_confirm.html"
  success_route_name "articles:index"
end
```

### `Marten::Handlers::Redirect` — static redirect handler

```crystal
class LegacyRedirectHandler < Marten::Handlers::Redirect
  route_name "articles:index"   # or: url "/new-path"
  permanent true
end
```

**Key DSL summary:**

| Macro / class method | Handlers that use it |
|---|---|
| `model` | RecordList, RecordDetail, RecordCreate, RecordUpdate, RecordDelete |
| `schema` | Schema, RecordCreate, RecordUpdate |
| `template_name` | Template, Schema, RecordList, RecordDetail, RecordCreate, RecordUpdate, RecordDelete |
| `success_route_name` / `success_url` | Schema, RecordCreate, RecordUpdate, RecordDelete |
| `record_context_name` | RecordDetail |
| `list_context_name` | RecordList |
| `page_size` | RecordList |
| `queryset` (macro or override) | RecordList, RecordDetail, RecordCreate, RecordUpdate, RecordDelete |
| `route_name` / `url` / `permanent` | Redirect |

---

## Lifecycle callbacks

Callbacks are registered by name (symbol). A `before_dispatch` or `before_render` callback that returns a `Marten::HTTP::Response` short-circuits dispatch entirely — the handler body never runs.

```crystal
class ArticleCreateHandler < Marten::Handlers::Schema
  include AuthenticationHelpers

  before_dispatch :require_authentication  # runs before any verb dispatch
  after_dispatch  :add_cache_header        # runs after dispatch, can replace response

  schema ArticleCreateSchema
  template_name "articles/create.html"
  success_route_name "articles:index"

  private def add_cache_header : Nil
    response!.headers["Cache-Control"] = "no-store"
  end
end
```

`before_render` fires just before template rendering — the right place to inject context variables:

```crystal
before_render :inject_user_into_context

private def inject_user_into_context : Nil
  context[:current_user] = request.user?
end
```

> **Footgun:** A `before_dispatch` callback that falls through without returning a response continues to dispatch normally. If you intend to block, make sure you return a response (redirect or error) in the guard branch — not just in the else.

Schema-specific callbacks (available on `Schema`, `RecordCreate`, `RecordUpdate`):

- `before_schema_validation` — mutate the schema object before validity is checked
- `after_schema_validation` — runs regardless of result
- `after_successful_schema_validation` — good place to set flash messages
- `after_failed_schema_validation` — good place to set flash error messages

---

## Cross-cutting concerns via mixins

Define shared auth/helper logic in a module, then `include` it in any handler.

```crystal
module AuthenticationHelpers
  def signed_in? : Bool
    !request.user?.nil?
  end

  def current_user
    request.user!
  end

  private def require_authentication
    redirect(reverse("session_new")) unless signed_in?
  end
end

class BooksNewHandler < Marten::Handlers::Schema
  include AuthenticationHelpers

  before_dispatch :require_authentication

  schema BookSchema
  template_name "books/new.html"

  def process_valid_schema
    # current_user is now available here
    Book.create!(title: schema.validated_data["title"].as(String))
    redirect(reverse("books_index"))
  end
end
```

---

## Custom error handlers

Marten provides defaults for 400, 403, 404, and 500. To customize:

1. Drop a matching template (`404.html`, `500.html`, etc.) into your templates folder — the default handlers render it automatically.
2. Or replace the handler entirely in settings:

```crystal
# config/settings/base.cr
Marten.configure do |config|
  config.handler404 = CustomNotFoundHandler
  config.handler500 = CustomServerErrorHandler
end
```

Built-in classes: `Marten::Handlers::Defaults::PageNotFound`, `Marten::Handlers::Defaults::ServerError`, `Marten::Handlers::Defaults::BadRequest`, `Marten::Handlers::Defaults::PermissionDenied`.

Raise `Marten::HTTP::Errors::NotFound` from any handler to trigger the 404 handler.

---

## Request internals

```crystal
request.method          # "GET", "POST", ...
request.query_params["page"]?   # query string — nil-safe
request.data["field"]?          # POST body / multipart — nil-safe
request.headers["X-Token"]?
request.session[:user_id] = 42
request.flash[:notice] = "Saved!"  # requires flash middleware
cookies[:pref] = "dark"
cookies.signed[:token]             # tamper-evident
cookies.encrypted[:secret]         # encrypted + signed

# marten-auth
request.user?    # Auth::User | Nil
request.user!    # Auth::User, raises if nil
```

---

## When to look elsewhere

- Routing / path-param syntax → `routing.md`
- Schemas / form handlers → `schemas.md`
- Cookies, sessions, middleware, CSRF → `http.md`
- Templates (rendering) → `templates.md`
