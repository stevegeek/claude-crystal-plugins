## What a Schema is

A `Marten::Schema` is the form/data-validation layer. It is **decoupled from models on purpose** — it validates and coerces raw request params (form data or JSON body) into typed Crystal values before you touch a model. Unlike Rails strong-params (which only filter keys) or model-bound forms (which couple validations to persistence), a Schema handles coercion, validation, and error collection in one object that you hand to a handler.

```crystal
class SessionSchema < Marten::Schema
  field :email_address, :string, max_size: 255, required: true
  field :password,      :string, max_size: 255, required: true
end
```

Use a Schema whenever a handler accepts user input that needs validation — sign-in forms, creation forms, API JSON payloads.

---

## Defining a Schema

```crystal
class BookSchema < Marten::Schema
  field :title,    :string, max_size: 255, required: true
  field :subtitle, :string, max_size: 255, required: false
  field :author,   :string, max_size: 255, required: false
end
```

Every field takes: `field <name>, <type>, [options]`. The name becomes the key Marten looks for in the request data.

---

## Field types

| Type        | Coerces to                         | Notes |
|-------------|------------------------------------|-------|
| `:string`   | `String`                           | `strip: true` by default |
| `:int`      | `Int64`                            | `min_value`, `max_value` |
| `:float`    | `Float64`                          | `min_value`, `max_value` |
| `:bool`     | `Bool`                             | |
| `:email`    | `String`                           | validates format, default `max_size: 254` |
| `:url`      | `String`                           | validates format, default `max_size: 200` |
| `:slug`     | `String`                           | chars/numbers/dashes/underscores, default `max_size: 50` |
| `:uuid`     | `UUID`                             | |
| `:date`     | `Time`                             | tries localized formats then settings fallback |
| `:date_time`| `Time`                             | same format-fallback chain |
| `:duration` | `Time::Span`                       | ISO 8601 or `DD.HH:MM:SS` |
| `:file`     | `Marten::HTTP::UploadedFile`       | multipart form only |
| `:image`    | `Marten::HTTP::UploadedFile`       | requires `crystal-vips` shard |
| `:json`     | `JSON::Any` or serializable class  | use `serializable:` option |
| `:enum`     | enum member                        | requires `values: MyEnum` |
| `:array`    | `Array` of member type             | requires `of: :string` etc. |

---

## Field options

```crystal
field :title, :string,
  required: true,      # default true — field must be present in submission
  max_size: 128,       # string/slug/email/url: max char count
  min_size: 1,         # string/slug/email/url: min char count
  strip: true,         # strip leading/trailing whitespace (default true for string types)
  max_value: 100,      # int/float: upper bound
  min_value: 0         # int/float: lower bound
```

**required vs blank:** `required: true` means the key must appear in the submission with a non-empty value. An empty string on a `required` field still fails. If you want to accept empty strings as "present", set `required: false`.

---

## Custom validations

Use the `validate` macro to register a method that runs after all field-level validation. Access already-coerced values via `validated_data`.

```crystal
class SignUpSchema < Marten::Schema
  field :password1, :string, max_size: 128, strip: false
  field :password2, :string, max_size: 128, strip: false

  validate :validate_passwords_match

  private def validate_passwords_match
    return unless validated_data["password1"]? && validated_data["password2"]?
    if validated_data["password1"] != validated_data["password2"]
      errors.add("The two password fields do not match")  # global error
    end
  end
end
```

For a field-specific error: `errors.add(:password2, "Passwords do not match")`.

Multiple `validate` calls are executed in definition order. Custom validations always run after field validations, so `validated_data` holds already-coerced values for any field that passed its own validation.

---

## The Schema-handler bridge

`Marten::Handlers::Schema` is the generic handler that wires a Schema to the GET/POST lifecycle.

```crystal
class BooksNewHandler < Marten::Handlers::Schema
  include AuthenticationHelpers

  before_dispatch :require_authentication   # auth checks go here, NOT in process_valid_schema

  schema       BookSchema
  template_name "books/new.html"

  def process_valid_schema
    title = schema.validated_data["title"].as(String)
    book  = Book.create!(title: title)
    redirect(reverse("book_show", id: book.pk))
  end
end
```

Lifecycle:
1. **GET** — renders `template_name` with an empty (unbound) schema in context as `schema`.
2. **POST** — binds `request.data` to the schema, calls `schema.valid?`.
   - Valid → calls `process_valid_schema`.
   - Invalid → re-renders `template_name` with the bound schema (errors populated).

**Validations fire before `process_valid_schema`.** Do not put authentication or authorization logic in `process_valid_schema`; put it in a `before_dispatch` callback.

---

## Reading validated data

After `valid?` returns true, use typed accessors (preferred) or the hash:

```crystal
# Type-safe generated accessors (recommended)
schema.title    # => String | Nil
schema.title!   # => String  (raises NilAssertionError if nil)
schema.title?   # => Bool

# Hash access — requires manual cast
schema.validated_data["title"].as(String)
schema.validated_data["subtitle"]?.as(String?)   # ? key-access for optional fields

# Raw bound value (pre-coercion, for template re-rendering)
schema["title"].value
```

`validated_data` is a `Hash(String, Bool | Float64 | Int64 | JSON::Any | ...)` — the union type means `.as(T)` is required for type-safe use.

Errors:

```crystal
schema.errors             # ErrorSet
schema.errors[:title]     # Array of errors for :title field
schema.errors.global      # Array of non-field errors
schema.title.errored?     # Bool — true if the field has any errors (template helper)
```

---

## Initial / pre-fill data

Override `initial_data` in the handler to pre-populate fields (useful for edit forms):

```crystal
class BooksEditHandler < Marten::Handlers::Schema
  schema BookSchema
  template_name "books/edit.html"

  def initial_data
    {"title" => book.title, "subtitle" => book.subtitle.to_s}
  end

  def process_valid_schema
    book.update!(title: schema.title!, subtitle: schema.subtitle)
    redirect(reverse("book_show", id: book.pk))
  end

  private def book
    @book ||= Book.get!(pk: params["id"])
  end
end
```

---

## Custom context

To pass extra variables to the template alongside the schema, override `context`:

```crystal
def context
  super.merge({"book" => book, "signed_in" => signed_in?})
end
```

Use `merge` (returns a new hash), **not** `merge!` (mutates in place and can silently drop the parent context). See `gotchas.md` for more on context pitfalls.

---

## RecordCreate / RecordUpdate generic handlers

These extend `Marten::Handlers::Schema` and handle the model create/update automatically:

```crystal
class PostCreateHandler < Marten::Handlers::RecordCreate
  model            Post
  schema           PostSchema
  template_name    "posts/new.html"
  success_route_name "posts_index"
  record_context_name "post"    # key under which the new record is in context
end
```

`RecordUpdate` takes the same options and additionally loads the existing record via `pk` route param. Both call `process_valid_schema` which you can override for custom post-save behavior.

For the full list of generic handlers and their options, see `handlers.md`.

---

## File uploads via Schema

Use `:file` for uploads. The form must use `enctype="multipart/form-data"`.

```crystal
class AvatarSchema < Marten::Schema
  field :avatar, :file, required: true, max_name_size: 255
end

class AvatarUploadHandler < Marten::Handlers::Schema
  schema AvatarSchema
  template_name "avatar/upload.html"

  def process_valid_schema
    uploaded = schema.validated_data["avatar"].as(Marten::HTTP::UploadedFile)
    # pass uploaded to a storage backend
    redirect(reverse("profile"))
  end
end
```

HTML:

```html
<form method="post" enctype="multipart/form-data">
  <input type="hidden" name="csrftoken" value="{% csrf_token %}" />
  <input type="file" name="{{ schema.avatar.id }}" />
  <button>Upload</button>
</form>
```

For storage backends, signed URLs, and file persistence, see `files.md`.

---

## When to look elsewhere

- Handlers that wrap Schemas (`Marten::Handlers::Schema`, `RecordCreate`, etc.) → `handlers.md`
- File-upload IO + storage → `files.md`
- CSRF on form posts → `http.md`
