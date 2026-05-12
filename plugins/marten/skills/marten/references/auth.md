## What marten-auth provides

[`martenframework/marten-auth`](https://github.com/martenframework/marten-auth) is a separate official shard — not part of the core framework. It provides:

- `MartenAuth::User` — abstract base model: `id`, `email` (unique, validated), `password` (bcrypt hash), `created_at`, `updated_at`. Instance methods: `set_password(raw)`, `check_password(raw) : Bool`, `set_unusable_password`, `session_auth_hash`.
- Session-based sign-in middleware (`MartenAuth::Middleware`) that reads the user ID from the session and attaches it to the request.
- Request-level accessors: `request.user`, `request.user!`, `request.user?`, `request.user_id`.
- Top-level functions: `MartenAuth.authenticate`, `sign_in`, `sign_out`, `generate_password_reset_token`, `valid_password_reset_token?`, `update_session_auth_hash`.

`marten gen auth` scaffolds full templates + handlers + emails + migrations. Use it for greenfield. **For existing apps, skip the generator and use the API bare** — that's what the rest of this file covers.

## Installation

```yaml
# shard.yml
dependencies:
  marten_auth:
    github: martenframework/marten-auth
```

```bash
shards install
```

## Wiring

**`src/project.cr`** — require the shard:
```crystal
require "marten_auth"
```

**`config/settings/base.cr`** — register app and middleware. Middleware order matters: `MartenAuth::Middleware` MUST come after `Marten::Middleware::Session`.

```crystal
Marten.configure do |config|
  config.installed_apps = [
    MartenAuth::App,
    # ... your apps
  ] of Marten::Apps::Config.class

  config.middleware = [
    Marten::Middleware::Session,      # must be first
    MartenAuth::Middleware,           # reads session → request.user_id
    Marten::Middleware::Flash,
    Marten::Middleware::GZip,
    Marten::Middleware::XFrameOptions,
  ]
end
```

**`config/initializers/auth.cr`** — point marten-auth at your User class:
```crystal
Marten.settings.auth.user_model = User
```

Do this in an **initializer, not in `settings/base.cr`**. `settings/base.cr` runs before model autoload, so `User` is not yet defined there. Initializers run after everything is loaded.

## Defining your User model

Subclass `MartenAuth::User`. All base fields are inherited — do not redefine `email` or `password`.

```crystal
class User < MartenAuth::User
  field :name, :string, max_size: 255, blank: false, null: false
  field :role, :string, max_size: 32, blank: false, null: false, default: "member"
  field :active, :bool, default: true

  def self.active
    filter(active: true)
  end
end
```

Inherited fields: `id` (big_int PK), `email` (unique email field with built-in format validation), `password` (string, max 128 chars), `created_at`, `updated_at`.

**Do not add a separate email field.** The base `email` field uses the `:email` field type which handles validation automatically.

Run migrations after creating the model: `marten migrate`.

## Password handling

```crystal
# Creating a user — always use set_password, never password=
user = User.new(name: "Alice", email: "alice@example.com", role: "member", active: true)
user.set_password("s3cur3p@ss")
user.save!

# Verifying a password
user.check_password("s3cur3p@ss")  # => true

# Disable login (e.g. deactivating an account)
user.set_unusable_password
user.save!
```

**Never do `user.password = raw_string`** — that writes plaintext into the hash column. `set_password` calls `Crypto::Bcrypt::Password.create` and stores the resulting hash string.

## Sign-in flow

`MartenAuth.authenticate(email, password)` returns `BaseUser?` — it validates credentials but does NOT touch the session. Then `MartenAuth.sign_in(request, user)` persists the user ID in the session (and rotates the CSRF cookie).

```crystal
class SessionsCreateHandler < Marten::Handlers::Schema
  schema SessionSchema
  template_name "sessions/new.html"

  def process_valid_schema
    email    = schema.validated_data["email_address"].as(String)
    password = schema.validated_data["password"].as(String)

    user = MartenAuth.authenticate(email, password)

    # Guard: credentials invalid, or app-level deactivation check
    if user.nil? || !user.as(User).active
      return render("sessions/new.html",
        context: {errors: ["Invalid email or password"]},
        status: 422)
    end

    MartenAuth.sign_in(request, user)
    redirect(reverse("books_index"))
  end
end
```

`sign_in` flushes the old session if the user changed (prevents session fixation) and cycles the session key otherwise.

## Sign-out

```crystal
class SessionsDeleteHandler < Marten::Handler
  def post
    MartenAuth.sign_out(request)   # flushes session, clears request.user
    redirect(reverse("session_new"))
  end
end
```

## request.user — the request-level accessors

`MartenAuth::Middleware` populates `request.user_id` from the session on every request. The user record is loaded lazily on first access of `request.user`.

| Method | Returns | Notes |
|---|---|---|
| `request.user_id` | `String?` | PK as string, or nil if anonymous |
| `request.user` | `MartenAuth::BaseUser?` | nil if anonymous |
| `request.user!` | `MartenAuth::BaseUser` | raises `NilAssertionError` if anonymous |
| `request.user?` | `Bool` | true if authenticated |

The return type is `BaseUser`, not your concrete `User`. Cast when you need app-specific fields:

```crystal
current = request.user.try(&.as(User))   # => User?
current = request.user!.as(User)         # => User (raises if anonymous)
```

## Auth concern for handlers

Put shared auth logic in a module and include it in handlers that need it:

```crystal
module AuthenticationHelpers
  protected def signed_in? : Bool
    !request.user.nil?
  end

  protected def current_user : User?
    request.user.try(&.as(User))
  end

  protected def current_user! : User
    request.user!.as(User)
  end

  protected def require_authentication : Marten::HTTP::Response?
    return nil if signed_in?
    redirect(reverse("session_new"))
  end
end

class BooksHandler < Marten::Handler
  include AuthenticationHelpers
  before_dispatch :require_authentication

  def get
    render("books/index.html", context: {user: current_user!})
  end
end
```

`before_dispatch` returns `nil` to continue or a `Response` to halt. `require_authentication` returns the redirect response when the user is anonymous, halting the dispatch chain.

## Password reset

```crystal
# Generate a signed, expiring token (default expiry: 3 days)
token = MartenAuth.generate_password_reset_token(user)

# Validate the token (checks expiry, user PK, and current password hash)
if MartenAuth.valid_password_reset_token?(user, token)
  user.set_password(new_password)
  user.save!
  MartenAuth.update_session_auth_hash(request, user)  # keep current session alive
end
```

Token expiry is configurable:
```crystal
# config/settings/base.cr
config.auth.password_reset_token_expiry_time = Time::Span.new(hours: 24)
```

The token embeds the current password hash, so it is automatically invalidated when the password changes.

## The `marten gen auth` generator

Scaffolds: handlers (sign-in, sign-up, sign-out, password reset, profile), schemas, emails, templates, migrations, and an `Auth::User` subclass. Also updates `shard.yml`, `project.cr`, `installed_apps`, and adds middleware config.

**Use it for greenfield projects** that don't have an existing auth flow. **Skip it** if you have existing user models, custom session handling, or want full control over templates — it is far easier to wire the shard API bare as shown above than to adapt or discard the generated files.

---

## When to look elsewhere

- Middleware order, sessions, cookies → `http.md`
- Handler `before_dispatch` callbacks → `handlers.md`
- Settings reference → `settings.md`
