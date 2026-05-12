## Settings file structure

Settings live under `config/settings/`. Convention:

- `config/settings/base.cr` — shared defaults, all environments
- `config/settings/development.cr` — dev overrides
- `config/settings/production.cr` — production overrides
- `config/settings/test.cr` — test overrides (REQUIRED if running specs)

Shared settings use `Marten.configure do |config|`. Environment-specific settings use the env label:

```crystal
# base.cr
Marten.configure do |config|
  config.secret_key = "change-me"
  config.installed_apps = [MartenAuth::App, MartenTurbo::App] of Marten::Apps::Config.class
  # ...
end

# production.cr
Marten.configure :production do |config|
  config.debug = false
  config.secret_key = ENV.fetch("MARTEN_SECRET_KEY")
  # ...
end
```

`MARTEN_ENV` env var selects the environment. Defaults to `development` if unset. The label must match the symbol passed to `Marten.configure`.

Both blocks run: base first, then the matching env block on top.

**Do not alter settings outside a `Marten.configure` block.** Most settings are read at startup; changes after setup have no effect.

---

## Initializers

`config/initializers/*.cr` runs AFTER both settings files AND model autoload.

Use initializers for anything that depends on model classes being available:

```crystal
# config/initializers/auth.cr
Marten.settings.auth.user_model = User
```

**`config.auth.user_model = User` MUST go in an initializer, NOT in `base.cr`.** The settings files load before models are autoloaded, so `User` does not exist yet at that point.

---

## Core settings

```crystal
# Required — use a long random string; never commit to source control
config.secret_key = ENV.fetch("MARTEN_SECRET_KEY")

# Apps whose models, migrations, assets, and templates are registered
config.installed_apps = [
  MartenAuth::App,
  MartenTurbo::App,
] of Marten::Apps::Config.class

# Middleware — ORDER MATTERS (see http.md)
config.middleware = [
  Marten::Middleware::Session,
  MartenAuth::Middleware,      # must come after Session
  Marten::Middleware::Flash,
  Marten::Middleware::GZip,
  Marten::Middleware::XFrameOptions,
  Marten::Middleware::ReferrerPolicy,
]

config.host  = "127.0.0.1"   # default; use "0.0.0.0" in production containers
config.port  = 8000           # default
config.debug = false          # default; set true in development

# Production MUST set this. Comma-separated env var pattern:
config.allowed_hosts = ENV.fetch("MARTEN_ALLOWED_HOSTS")
  .split(",").map(&.strip).reject(&.empty?)
# In debug mode, defaults automatically to [".localhost", "127.0.0.1", "[::1]"]

# Behind a reverse proxy:
config.use_x_forwarded_host  = true   # trust X-Forwarded-Host
config.use_x_forwarded_port  = true   # trust X-Forwarded-Port
config.use_x_forwarded_proto = true   # trust X-Forwarded-Proto (https detection)

# Optional — deploy path differs from build path (e.g. Heroku)
config.root_path = "/app"

# Unix socket instead of TCP (better perf behind Nginx/Caddy on same machine)
config.socket = "/tmp/marten.sock"
```

`config.secret_key` is used for HMAC signing of cookies, sessions, and password-reset tokens. **Must be set to a unique, secret, unpredictable value in production.**

---

## Database settings

```crystal
# Default database — SQLite
config.database do |db|
  db.backend = :sqlite
  db.name    = Path["app.db"].expand
end

# Default database — PostgreSQL
config.database do |db|
  db.backend  = :postgresql
  db.name     = "my_db"
  db.host     = "localhost"
  db.port     = 5432
  db.user     = "my_user"
  db.password = ENV.fetch("DB_PASSWORD")
end

# From a connection URL (e.g. DATABASE_URL on Heroku/Render)
config.database url: ENV.fetch("DATABASE_URL")

# Second database
config.database :secondary do |db|
  db.backend = :sqlite
  db.name    = "secondary.db"
end
```

Supported backends: `:sqlite`, `:postgresql`, `:mysql`.

Connection pool options (rarely need tuning):

```crystal
db.max_pool_size    = 5    # 0 = unlimited (default)
db.initial_pool_size = 1
db.max_idle_pool_size = 1
db.checkout_timeout = 5.0
```

**Test environment:** Give the test database a distinct name (`:memory:` for SQLite or a different db name for Postgres). Marten will not run specs unless the test db name is explicitly distinct from the development db.

---

## Templates settings

```crystal
config.templates.dirs = [Path["src/templates"].expand.to_s]

config.templates.cached = true   # enable in production; false in dev

config.templates.context_producers = [
  Marten::Template::ContextProducer::Request,  # request, user in templates
  Marten::Template::ContextProducer::Flash,    # flash messages
  Marten::Template::ContextProducer::Debug,    # debug flag
  Marten::Template::ContextProducer::I18n,     # current locale
]

config.templates.app_dirs       = true   # search app templates/ dirs (default)
config.templates.strict_variables = false  # true raises on unknown vars
```

---

## Sessions settings

```crystal
config.sessions.store           = :cookie   # default; also :cache, :db
config.sessions.cookie_name     = "sessionid"
config.sessions.cookie_max_age  = 1_209_600  # 2 weeks in seconds
config.sessions.cookie_secure   = true       # HTTPS only — set in production
config.sessions.cookie_http_only = true      # JS cannot read cookie
config.sessions.cookie_same_site = "Lax"    # "Lax", "Strict", or "None"
config.sessions.cookie_domain   = nil        # set for cross-subdomain sharing
```

---

## CSRF settings

```crystal
config.csrf.protection_enabled = true       # default
config.csrf.cookie_name        = "csrftoken"
config.csrf.cookie_secure      = true       # set in production
config.csrf.cookie_http_only   = false      # keep false — JS reads csrf token
config.csrf.cookie_same_site   = "Lax"
config.csrf.session_key        = "csrftoken"  # only when use_session = true
config.csrf.use_session        = false        # store token in session instead of cookie

# Trust cross-origin CSRF-protected requests:
config.csrf.trusted_origins = ["https://*.example.com"]
```

---

## Assets and media files

```crystal
# Assets (CSS, JS, images — static files)
config.assets.dirs    = [Path["src/assets"].expand.to_s]
config.assets.url     = "/assets/"
config.assets.root    = "assets"           # where collectassets writes to
config.assets.storage = nil                # set to an S3/GCS store for cloud

# Media files (user uploads)
config.media_files.url     = "/media/"
config.media_files.root    = "media"
config.media_files.storage = nil           # set to an S3/GCS store for cloud
```

For cloud storage, assign an instance of a `Marten::Core::Storage::Base` subclass to `storage`; the `root` and `url` settings are then ignored.

---

## Caching

```crystal
# Default — in-memory (per-process, not shared)
config.cache_store = Marten::Cache::Store::Memory.new

# Test — discard all cache operations
config.cache_store = Marten::Cache::Store::Null.new

# Production — use a shard (e.g. marten-cache-redis-store)
config.cache_store = Marten::Cache::Store::Redis.new(url: ENV.fetch("REDIS_URL"))
```

---

## Emailing

```crystal
# Development — print emails to stdout
config.emailing.backend = Marten::Emailing::Backend::Development.new(print_emails: true)

# Test — collect emails in memory for assertion
config.emailing.backend = Marten::Emailing::Backend::Development.new(
  collect_emails: true,
  print_emails: false
)

# Production — use an SMTP shard or provider-specific backend
config.emailing.backend = Marten::Emailing::Backend::SMTP.new(
  host: ENV.fetch("SMTP_HOST"),
  port: 587,
)

config.emailing.from_address = "no-reply@example.com"
```

---

## Internationalization

```crystal
config.i18n.default_locale    = :en
config.i18n.available_locales = [:en, :fr]
config.i18n.locale_cookie_name = "marten_locale"

# Simple fallback for all locales:
config.i18n.fallbacks = ["en"]

# Per-locale fallback chains:
config.i18n.fallbacks = {"fr-CA" => ["fr", "en"], "en-CA" => ["en-US", "en"]}
```

Requires `Marten::Middleware::I18n` in the middleware stack to activate locale from cookie/header.

---

## Security headers

```crystal
# X-Frame-Options middleware
config.x_frame_options = "DENY"    # or "SAMEORIGIN"

# Referrer-Policy middleware
config.referrer_policy = "same-origin"

# Content-Security-Policy middleware
config.content_security_policy.default_src = [:self]
config.content_security_policy.script_src  = [:self, "cdn.example.com"]
config.content_security_policy.report_only = false  # true = report without blocking

# HTTP Strict-Transport-Security middleware
config.strict_transport_security.max_age            = 31_536_000  # 1 year
config.strict_transport_security.include_sub_domains = true
config.strict_transport_security.preload            = false

# SSL redirect middleware (redirects HTTP → HTTPS)
config.ssl_redirect.host             = nil   # nil = use request host
config.ssl_redirect.exempted_paths   = [/^\/health\/$/]
```

---

## Auth (marten-auth)

`config.auth.user_model` **must** be set in an initializer, not in `base.cr`:

```crystal
# config/initializers/auth.cr
Marten.settings.auth.user_model = User
```

Other auth settings (set in `base.cr` or `production.cr`):

```crystal
config.auth.password_reset_token_expiry_time = 72.hours
```

---

## Per-environment patterns

### `production.cr` — condensed real-world pattern

```crystal
Marten.configure :production do |config|
  config.debug = false
  config.host  = "0.0.0.0"
  config.port  = 8000

  # Secrets from environment — never hardcode
  config.secret_key     = ENV.fetch("MARTEN_SECRET_KEY")
  config.allowed_hosts  = ENV.fetch("MARTEN_ALLOWED_HOSTS")
    .split(",").map(&.strip).reject(&.empty?)

  # Secure cookies
  config.sessions.cookie_secure    = true
  config.sessions.cookie_http_only = true
  config.csrf.cookie_secure        = true
  config.csrf.cookie_http_only     = true

  # Template caching
  config.templates.cached = true

  # Database from URL if provided
  config.database url: ENV.fetch("DATABASE_URL") if ENV["DATABASE_URL"]?
end
```

### `test.cr` — required for specs

```crystal
Marten.configure :test do |config|
  # MUST be different from dev db — Marten enforces this
  config.database do |db|
    db.name = ":memory:"   # SQLite in-memory; or a separate named db for Postgres
  end

  config.allowed_hosts = ["127.0.0.1"]
  config.cache_store   = Marten::Cache::Store::Null.new

  # Collect emails so specs can assert on Marten::Emailing::Backend::Development.collected_emails
  config.emailing.backend = Marten::Emailing::Backend::Development.new(
    collect_emails: true,
    print_emails: false
  )
end
```

---

## Things to watch out for

- **`config.secret_key` must be secret in production.** Load via `ENV.fetch("MARTEN_SECRET_KEY")`. A missing or empty secret key breaks signed cookies, sessions, and password-reset tokens.
- **`config.auth.user_model = User` goes in `config/initializers/auth.cr`**, not in any settings file. Settings files run before models are autoloaded; `User` doesn't exist yet.
- **Test settings are mandatory.** Running specs without a `config/settings/test.cr` that sets a distinct database name will fail.
- **Middleware order matters.** `Session` must precede anything that reads the session (e.g., `MartenAuth::Middleware`, `Flash`). See `http.md`.
- **`allowed_hosts` is not auto-set in production.** Debug mode sets a default; production does not. A missing `allowed_hosts` will reject all requests.

---

## When to look elsewhere

- Middleware list + order → `http.md`
- Auth-specific settings → `auth.md`
- Template config + custom context producers → `templates.md`
