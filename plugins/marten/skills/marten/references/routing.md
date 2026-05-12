## Defining routes

Routes live in `config/routes.cr`. The `draw` block is the only place to register them — Marten has no nested-resource DSL (no `resources :books` shorthand).

```crystal
Marten.routes.draw do
  path "/", HomeHandler, name: "home"
  path "/books", BooksIndexHandler, name: "books_index"
  path "/books/<id:int>", BooksShowHandler, name: "book_show"
end
```

Each `path` call takes: pattern string, handler class, `name:` keyword. The name is used for reverse routing and must be unique across the map.

## Path parameters

Syntax: `<name>` (str default) or `<name:type>`.

| Type | Matches | Crystal type |
|------|---------|--------------|
| `str` / `string` | Any non-empty string, no `/` | `String` |
| `int` | Zero or positive integer | `UInt64` |
| `slug` | ASCII letters, digits, `-`, `_` | `String` |
| `path` | Any non-empty string including `/` | `String` |
| `uuid` | Valid UUID string | `UUID` |

```crystal
path "/posts/<slug:slug>", PostHandler, name: "post_show"
path "/files/<filepath:path>", FileHandler, name: "file_show"
path "/users/<id:uuid>", UserHandler, name: "user_show"
```

In the handler, captures are in `params`, not the query string:

```crystal
def get
  id = params["id"]                    # route capture — UInt64 for :int
  q  = request.query_params["q"]?      # query string — ?q=foo
end
```

## Multiple captures and ordering

Declare more-specific paths **before** patterns with captures that could match them.

```crystal
path "/books/new", BooksNewHandler, name: "books_new"        # MUST come first
path "/books/<id:int>", BooksShowHandler, name: "book_show"  # :int won't match "new", but be explicit

path "/books/<book_id:int>/pages/<id:int>", PageShowHandler, name: "page_show"
```

Ordering matters when a literal segment could be swallowed by a slug or str parameter:

```crystal
path "/items/new",              ItemsCreateHandler, name: "items_create"  # before slug
path "/items/<encoded_id:slug>", ItemsShowHandler,  name: "item_show"     # slug would match "new"
```

## Reverse routing

```crystal
# Anywhere in Crystal code:
Marten.routes.reverse("book_show", id: 42)      # => "/books/42"
Marten.routes.reverse("home")                   # => "/"

# Inside a handler (shorthand):
reverse("page_show", book_id: 1, id: 5)         # => "/books/1/pages/5"
```

**In templates — use a colon, not `=`:**

```html
{% url 'book_show' id: book.id %}
{% url 'page_show' book_id: book.id id: page.id %}
```

The separator is `:` (keyword argument style), not `=`. Using `=` is a common mistake.

## Sub-route maps (included routes)

Define a `Marten::Routing::Map` constant and pass it as the second argument to `path`.

```crystal
ARTICLE_ROUTES = Marten::Routing::Map.draw do
  path "",              ArticlesHandler,      name: "list"
  path "/create",       ArticleCreateHandler, name: "create"
  path "/<pk:int>",     ArticleDetailHandler, name: "detail"
  path "/<pk:int>/edit", ArticleUpdateHandler, name: "update"
end

Marten.routes.draw do
  path "/", HomeHandler, name: "home"
  path "/articles", ARTICLE_ROUTES, name: "articles"
end
```

Route names are prefixed: `articles:list`, `articles:detail`, etc.

```crystal
reverse("articles:detail", pk: 42)  # => "/articles/42"
```

The `name:` on the `path` that includes the sub-map is optional but prevents name collisions.

## Dev-only routes

Asset and media-file serving for development only:

```crystal
if Marten.env.development?
  path "#{Marten.settings.assets.url}<path:path>",
    Marten::Handlers::Defaults::Development::ServeAsset,
    name: "asset"
  path "#{Marten.settings.media_files.url}<path:path>",
    Marten::Handlers::Defaults::Development::ServeMediaFile,
    name: "media_file"
end
```

These use the `path` parameter type so they match nested file paths with slashes.

## Custom path parameter types

Subclass `Marten::Routing::Parameter::Base` and implement three methods:

```crystal
class YearParameter < Marten::Routing::Parameter::Base
  def regex : Regex
    /[12][0-9]{3}/
  end

  def loads(value : ::String) : UInt64
    value.to_u64
  end

  def dumps(value) : Nil | ::String
    value.is_a?(Int) ? value.to_s : nil
  end
end

Marten::Routing::Parameter.register(:year, YearParameter)
```

Then use it in routes:

```crystal
path "/archive/<year:year>", ArchiveHandler, name: "archive"
```

- `regex` — matched against the raw URL segment
- `loads` — raw string → Crystal object stored in `params`
- `dumps` — Crystal object → string for reverse routing; return `nil` to signal a `NoReverseMatch` error

The `marten-encoded-id` shard ships pre-built parameter types for encoded/obfuscated integer IDs using this same mechanism.

## When to look elsewhere

- Handler bodies that match these routes → `handlers.md`
- Custom param-type for encoded slug ids → look at the `marten-encoded-id` shard
