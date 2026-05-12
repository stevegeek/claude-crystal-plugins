## Request basics

`request` is a `Marten::HTTP::Request` instance. Key methods:

| Method | Returns |
|---|---|
| `request.method` | `"GET"`, `"POST"`, etc. |
| `request.headers` | `Marten::HTTP::Headers` — hash-like, e.g. `request.headers["Authorization"]?` |
| `request.url` | Full request URL string |
| `request.body` | Raw body string |
| `request.query_params` | `Marten::HTTP::Params::Query` — GET params |
| `request.data` | `Marten::HTTP::Params::Data` — form/POST data |
| `request.cookies` | Cookie store |
| `request.session` | Session store (requires `Marten::Middleware::Session`) |
| `request.flash` | Flash store (requires `Marten::Middleware::Flash`) |

---

## Cookies

Read:

```crystal
request.cookies[:foo]   # raises KeyError if missing
request.cookies[:foo]?  # returns nil if missing
request.cookies.fetch(:foo, "default")
```

Write (simple — no expiry, path `/`, not secure):

```crystal
request.cookies[:foo] = "bar"
```

Write with options:

```crystal
request.cookies.set(
  :foo,
  "bar",
  expires:   2.days.from_now,  # or use max_age: 86400
  path:      "/",
  domain:    nil,
  secure:    true,
  http_only: true,
  same_site: "lax"             # "lax" | "strict"
)
```

Delete (pass same `path`/`domain`/`same_site` used at creation):

```crystal
request.cookies.delete(:foo)
```

### Signed cookies (HMAC — tamper-detected, not encrypted)

Values signed with HMAC-SHA256. Client can read the value, but cannot forge it.

```crystal
request.cookies.signed[:foo]?
request.cookies.signed[:foo] = "bar"
request.cookies.signed.set(:foo, "bar", expires: 1.hour.from_now)
request.cookies.signed.delete(:foo)
```

### Encrypted cookies (AES-256-CBC + HMAC — unreadable by client)

```crystal
request.cookies.encrypted[:foo]?
request.cookies.encrypted[:foo] = "bar"
```

> Only cookie **values** are signed/encrypted. Cookie names are never signed.

---

## Sessions

Requires `Marten::Middleware::Session` in the middleware chain. Default store is `:cookie` (encrypted, 4 KB limit).

```crystal
request.session[:user_id] = "42"   # only string values
request.session[:user_id]?         # nil-safe read
request.session.delete(:user_id)
request.session.empty?
request.session.flush              # clear all data + regenerate key
```

Expiry:

```crystal
request.session.expires_at = 2.days.from_now
request.session.expires_in = 2.hours
request.session.expires_at_browser_close = true
```

Inside a handler, `session` shorthand works: `session[:foo] = "bar"`.

Configure session cookie via `config.sessions.*`:

```crystal
config.sessions.store             # :cookie (default), or external shard
config.sessions.cookie_name       # default "sessionid"
config.sessions.cookie_max_age    # default 2 weeks
config.sessions.cookie_secure     # true in production
config.sessions.cookie_http_only  # true recommended
config.sessions.cookie_same_site  # "lax" | "strict"
```

External stores: `marten-db-session` (DB), `marten-redis-session` (Redis).

---

## Flash messages

Requires `Marten::Middleware::Flash` **and** `Marten::Middleware::Session`.

Flash values survive exactly one request hop — set in handler A, read in handler B, then cleared.

```crystal
# Set (usually before a redirect)
flash[:notice] = "Saved successfully."
flash[:error]  = "Something went wrong."

# Read (in next request or same request via flash context producer in templates)
flash[:notice]?

# Keep or discard explicitly
flash.keep          # keep all for another hop
flash.keep(:notice) # keep one key only
flash.discard       # drop all immediately
```

In templates the flash context producer exposes `flash` automatically.

---

## Middleware

Activate in settings:

```crystal
config.middleware = [
  Marten::Middleware::SSLRedirect,         # must be early
  Marten::Middleware::GZip,               # compress before others read body
  Marten::Middleware::ContentSecurityPolicy,
  Marten::Middleware::XFrameOptions,
  Marten::Middleware::ReferrerPolicy,
  Marten::Middleware::StrictTransportSecurity,
  Marten::Middleware::Session,            # session must precede Flash + auth
  Marten::Middleware::Flash,              # depends on Session
  MartenAuth::Middleware,                 # depends on Session
  Marten::Middleware::I18n,
]
```

### Built-in middleware reference

| Middleware class | Purpose | Notes |
|---|---|---|
| `Marten::Middleware::Session` | Loads/saves session store per-request | Required by Flash and auth |
| `Marten::Middleware::Flash` | Initialises flash store from session | Must come **after** Session |
| `Marten::Middleware::GZip` | Compresses responses ≥ 200 bytes | Place early; BREACH mitigated |
| `Marten::Middleware::XFrameOptions` | Sets `X-Frame-Options` header | Default `DENY`; clickjacking guard |
| `Marten::Middleware::ReferrerPolicy` | Sets `Referrer-Policy` header | Configurable via `config.referrer_policy` |
| `Marten::Middleware::ContentSecurityPolicy` | Sets `Content-Security-Policy` header | Configure directives in settings |
| `Marten::Middleware::SSLRedirect` | Redirects HTTP → HTTPS (301) | Enable only when TLS is live |
| `Marten::Middleware::StrictTransportSecurity` | Sets `Strict-Transport-Security` | Set `max_age` only after HTTPS is stable |
| `Marten::Middleware::I18n` | Activates locale from `Accept-Language` / cookie | — |
| `Marten::Middleware::AssetServing` | Serves collected static assets | Place first; dev/staging only |
| `Marten::Middleware::MethodOverride` | Maps hidden `_method` param to HTTP verb | Place early so other middleware sees it |

### Order rules

- `Session` must precede `Flash` and any auth middleware.
- `GZip` should precede middleware that reads response body.
- `SSLRedirect` should be first so unauthenticated redirects still happen.
- `AssetServing` must be first if used.

---

## Custom middleware

```crystal
class RequestIdMiddleware < Marten::Middleware
  def call(request : Marten::HTTP::Request, get_response : Proc(Marten::HTTP::Response)) : Marten::HTTP::Response
    request.headers["X-Request-Id"] = Random::Secure.hex(8)
    response = get_response.call
    response.headers["X-Request-Id"] = request.headers["X-Request-Id"]
    response
  end
end
```

Register it in `config.middleware` at the appropriate position.

---

## CSRF protection

Enabled by default via `Marten::Handlers::Concerns::RequestForgeryProtection` (included in `Marten::Handler`).

The token cookie is named **`csrftoken`** (lowercase, no underscore — not `_csrf` or `XSRF-TOKEN`). The token rotates on every response where it is used.

### Forms

```html
<form method="post">
  {% csrf_input %}
  <!-- renders: <input type="hidden" name="csrftoken" value="..."> -->
</form>
```

Or manually with the raw tag:

```html
<input type="hidden" name="csrftoken" value="{% csrf_token %}">
```

### AJAX

Send as `X-CSRF-Token` header. Read from cookie (only if `csrf.cookie_http_only = false`):

```javascript
const csrfToken = Cookies.get("csrftoken");
fetch("/api/action", { method: "POST", headers: { "X-CSRF-Token": csrfToken } });
```

Or embed via template tag if the cookie is http-only:

```html
<script>const csrfToken = "{% csrf_token %}";</script>
```

### Per-handler enable/disable

```crystal
class PublicApiHandler < Marten::Handler
  protect_from_forgery false   # disable for this handler
end

class FormHandler < Marten::Handler
  protect_from_forgery true    # explicit enable (if globally disabled)
end
```

Global toggle: `config.csrf.protection_enabled = false` (not recommended).

### CSRF cookie settings

```crystal
config.csrf.cookie_name       # default "csrftoken"
config.csrf.cookie_secure     # true in production
config.csrf.cookie_http_only  # false = JS-readable (needed for SPA AJAX)
config.csrf.cookie_same_site  # "lax" | "strict"
config.csrf.trusted_origins   # ["https://app.example.com"] for cross-origin
```

> **SameSite warning:** `"strict"` blocks the CSRF cookie on cross-site navigations, breaking OAuth callbacks and any flow where the user arrives from a third-party redirect. Use `"lax"` for most apps.

---

## Security headers

### X-Frame-Options (clickjacking)

Default: `DENY`. Configure globally:

```crystal
config.x_frame_options = "SAMEORIGIN"  # or "DENY"
```

Exempt a specific handler from the header:

```crystal
class EmbeddableWidget < Marten::Handler
  exempt_from_x_frame_options true
end
```

### Referrer-Policy

```crystal
config.referrer_policy = "strict-origin-when-cross-origin"
```

### Content Security Policy

Enable the middleware, then configure directives:

```crystal
config.content_security_policy.default_policy.default_src = [:self]
config.content_security_policy.default_policy.script_src  = [:self, :https]
config.content_security_policy.default_policy.style_src   = [:self, "cdn.example.com"]
config.content_security_policy.default_policy.img_src     = [:self, :data]
config.content_security_policy.default_policy.font_src    = [:self]
config.content_security_policy.default_policy.connect_src = [:self]
```

Enable nonces for inline scripts/styles (avoids `unsafe-inline`):

```crystal
config.content_security_policy.nonce_directives = ["script-src", "style-src"]
```

Use in templates:

```html
<script nonce="{{ request.content_security_policy_nonce }}">...</script>
```

Per-handler CSP override:

```crystal
class SpecialHandler < Marten::Handler
  content_security_policy do |csp|
    csp.default_src = {:self, "cdn.example.com"}
  end
end

class NoCspHandler < Marten::Handler
  exempt_from_content_security_policy true
end
```

---

## Production checklist

From `marten-writebook/config/settings/production.cr` and security docs:

```crystal
Marten.configure :production do |config|
  config.secret_key = ENV.fetch("MARTEN_SECRET_KEY")
  config.allowed_hosts = ENV.fetch("MARTEN_ALLOWED_HOSTS").split(",").map(&.strip)

  config.sessions.cookie_secure    = true
  config.sessions.cookie_http_only = true

  config.csrf.cookie_secure    = true
  config.csrf.cookie_http_only = true

  # Add these middlewares to the chain:
  # Marten::Middleware::SSLRedirect
  # Marten::Middleware::StrictTransportSecurity (set max_age after confirming TLS)
  # Marten::Middleware::ContentSecurityPolicy
end
```

Key items:
- `cookie_secure: true` — session and CSRF cookies HTTPS-only
- `cookie_http_only: true` — blocks JS access (SPA AJAX needs `false` for CSRF cookie)
- SSL redirect + HSTS — start HSTS with small `max_age` (e.g. 3600) before committing to a large value
- Defined CSP policy — at minimum `default_src :self`
- `allowed_hosts` set explicitly — prevents HTTP Host Header attacks

---

## When to look elsewhere

- Auth (sign-in/out, marten-auth) → `auth.md`
- Settings reference for all the cookie/session/CSP knobs → `settings.md`
- Handler bodies that consume cookies/sessions → `handlers.md`
