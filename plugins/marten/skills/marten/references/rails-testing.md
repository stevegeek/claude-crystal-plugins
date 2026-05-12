# Rails → Marten testing reference

Porting Rails tests to Marten. Two tiers: handler/integration specs via `Marten::Spec::Client` (in-process, fast), and system/browser specs via LuckyFlow + selenium.cr (separate process).

Default to tier 1. Reach for tier 2 only when the test genuinely exercises browser behaviour — real JS, WebSocket flows, file-upload UI, custom-element interactions.

Code samples use the same books/leaves/leafables domain as `rails-mapping.md`; substitute your own models.

## Two tiers

| Tier | Rails | Marten | When |
|---|---|---|---|
| Handler / integration | `ActionDispatch::IntegrationTest` | `Marten::Spec::Client` (in-process) | ~90% of `*_controller_test.rb` files port here. Fast (~10s for 200 examples), no driver |
| System / browser | `ActionDispatch::SystemTestCase` (Capybara + Selenium) | LuckyFlow + selenium.cr + webdrivers.cr (separate process) | Tests that need real JS / WebSocket / file-upload / custom-element behaviour |

`Marten::Spec.client` is memoized per spec; the cookie jar persists across requests, so one `client.post("/session/create", ...)` keeps you signed in for the rest of the example.

---

## Handler tier

### Scaffolding

```crystal
# spec/spec_helper.cr
ENV["MARTEN_ENV"] = "test"
require "spec"
require "../src/project"
require "marten/spec"
require "./support/**"

# Raw-SQL migrations don't run in specs (see gotcha below). Re-create here.
Spec.before_suite do
  Marten::DB::Connection.default.open do |db|
    db.exec("CREATE VIRTUAL TABLE IF NOT EXISTS leaf_search_index USING fts5(...)")
  end
end
```

```crystal
# spec/support/factories.cr — replaces Rails YAML fixtures
module Spec::Factories
  extend self

  def create_user(email = "u-#{Random::Secure.hex(4)}@example.com", role = "member", **)
    u = Accounts::User.new(name: "Test", email: email, role: role, active: true)
    u.set_password("secret123456")
    u.save!
    u
  end

  def create_book(title : String, editor : Accounts::User? = nil, **)
    book = Books::Book.create!(title: title, ...)
    Accounts::Access.create!(user_id: editor.pk, book_id: book.pk, level: "editor") if editor
    book
  end
end
```

```crystal
# spec/support/sessions.cr — replaces Rails SessionTestHelper
module Spec::Sessions
  def self.sign_in_as(client : Marten::Spec::Client, user : Accounts::User, password = "secret123456")
    response = client.post(
      Marten.routes.reverse("accounts:session_create"),
      data: {"email_address" => user.email.to_s, "password" => password},
    )
    raise "sign-in failed" unless response.status == 302
    user
  end
end
```

### Example handler spec

```crystal
require "../../spec_helper"

describe "BooksIndexHandler" do
  it "lists books the user has access to" do
    kevin = Spec::Factories.create_user
    Spec::Factories.create_book(title: "Handbook", editor: kevin)

    client = Marten::Spec.client
    Spec::Sessions.sign_in_as(client, kevin)
    response = client.get(Marten.routes.reverse("books:index"))

    response.status.should eq(200)
    response.content.should contain("Handbook")
  end
end
```

### Assertion mapping

| Rails | Marten |
|---|---|
| `get root_url` | `client.get(Marten.routes.reverse("root"))` |
| `post url, params: {...}` | `client.post(url, data: {"k" => "v"})` |
| `assert_response :success` | `response.status.should eq(200)` |
| `assert_redirected_to foo_url` | `response.status.should eq(302); response.headers["Location"].should eq(...)` |
| `assert_select "h2", text: "X"` | `response.content.should contain("X")` — no built-in HTML matcher; substring is usually enough |
| `assert_difference -> { Foo.count }, +1 do ... end` | `before = Foo.all.count; ...; Foo.all.count.should eq(before + 1)` |
| Fixtures (`users(:david)`) | Factory helpers (`Spec::Factories.create_user(email: "david@example.com")`) — drop YAML fixtures; per-spec factories are lighter and explicit |
| `cookies[:session_token].present?` | `client.cookies["session_token"]?` (test client memoizes its cookie jar across requests) |
| `sign_in :kevin` | `Spec::Sessions.sign_in_as(client, kevin)` |

### Gotchas

1. **`Marten::Spec.setup_databases` uses `schema_editor.sync_models`, not migrations.** Hand-written SQL migrations (FTS5 virtual tables, custom indexes, triggers, raw `ALTER`s) don't run in specs. Re-create them in `Spec.before_suite` and (if they accumulate state) wipe them in `Spec.after_each`. `flush_databases` only knows about Marten model tables.

2. **`:memory:` SQLite is per-connection-pool, not per-database.** Each new connection opening `:memory:` gets its own private database. If a spec's request handler opens an inner connection while a separate query is mid-flight (e.g. raw-SQL search reaching for the pool's second slot), it lands on an empty schema. Fix — point the test DB at a tempfile, not `:memory:`:

   ```crystal
   # config/settings/test.cr
   db.name = File.tempfile("appname-test-", ".db").path
   ```

   Shared across the pool, torn down at process exit. Negligible cost over `:memory:`.

3. **`Marten::Spec::Client` sends no `Accept` header by default, so `request.turbo?` returns true.** `request.turbo?` matches `text/vnd.turbo-stream.html` against `Accept`, and `*/*` (which no-Accept becomes) is treated as a match. Tests of plain-HTML response paths must set `Accept: text/html` explicitly:

   ```crystal
   client.get(url, headers: {"Accept" => "text/html"})
   ```

   Otherwise the handler takes the turbo-stream branch even when no JS client is involved.

4. **`response.headers["Content-Type"]` raises `KeyError`.** Marten exposes content type as a top-level method: `response.content_type`. (Same for `response.status` — no `Status` header exists.)

5. **CSRF is disabled by default on the test client.** Use `Marten::Spec::Client.new(disable_request_forgery_protection: false)` to enable it for tests of the CSRF middleware itself. Otherwise POST flows don't need `{% csrf_input %}` or a `csrftoken` form param.

6. **Generic detail handlers (`RecordDetail`) don't apply Rails-style implicit scoping.** Rails idiomatically writes `Book.accessable(Current.user).find(params[:id])` inside `BooksController#show`. The equivalent `Marten::Handlers::RecordDetail` is pure-pk lookup — no scope. Add a `before_dispatch :ensure_accessable` or override `queryset` explicitly. Ported handler tests catch this.

7. **SQLite3 errors don't descend from `DB::Error`.** `SQLite3::Exception < ::Exception`. Handler/model code that does `rescue DB::Error` to catch unique-constraint violations silently misses SQLite3-raised ones and returns 500. Use `rescue SQLite3::Exception | DB::Error`. Postgres (`PG::PQError`) and MySQL (`MySql::Error`) each have their own hierarchies — the trap repeats per adapter.

8. **Crystal stdlib bcrypt raises on passwords > 72 bytes; Ruby silently truncates.** `MartenAuth::User#set_password` propagates the raise. Port-time fix: length-validate on the schema:

   ```crystal
   field :password, :string, min_size: 8, max_size: 72, required: true
   ```

9. **`Marten::Spec.client` is memoized per spec and cleared by an auto-registered `after_each` hook.** Each `it` gets a fresh cookie jar. For tests needing two clients simultaneously (admin + member, two browser sessions), instantiate `Marten::Spec::Client.new` directly for the second — the global accessor only hands out one.

10. **Mark porting drift as `pending`, not failing.** When a Marten handler doesn't yet implement a Rails-only branch, missing param, or different URL shape, use `pending "description" do ... end` with a `# FIXME(porting gap): ...` comment. Keeps the suite green while the gap stays visible.

---

## System tier (LuckyFlow)

LuckyFlow is the Capybara-shaped DSL; the underlying stack is `selenium.cr` + `webdrivers.cr` (auto-downloads `chromedriver` on first run, keyed to whichever Chrome is on PATH). No driver setup beyond `shards install`.

### Scaffolding

```yaml
# shard.yml
development_dependencies:
  lucky_flow:
    github: luckyframework/lucky_flow
  webdrivers:
    github: crystal-loot/webdrivers.cr
```

```crystal
# config/settings/test.cr
db.name = ENV["MARTEN_TEST_DB"]? || File.tempfile("appname-test-", ".db").path
config.host = "127.0.0.1"
config.port = (ENV["MARTEN_TEST_PORT"]? || "8765").to_i
```

```crystal
# config/routes.cr — assets must be served in test env so JS loads
if Marten.env.development? || Marten.env.test?
  path "#{Marten.settings.assets.url}<path:path>",
       Marten::Handlers::Defaults::Development::ServeAsset, name: "asset"
end
```

```crystal
# spec/system/spec_helper.cr
ENV["MARTEN_ENV"] = "test"
ENV["MARTEN_TEST_DB"]   ||= File.tempfile("appname-system-", ".db").path
ENV["MARTEN_TEST_PORT"] ||= "8765"

require "spec"
require "lucky_flow"
require "../../src/project"
require "marten/spec"

LuckyFlow.configure { |s| s.base_uri = "http://127.0.0.1:#{ENV["MARTEN_TEST_PORT"]}" }
Habitat.raise_if_missing_settings!
LuckyFlow::Spec.setup

Spec.before_suite do
  # Sync schema + recreate raw-SQL tables (FTS5 etc.) in the shared tempfile.
  Marten::DB::Connection.default.open { |db| db.exec("CREATE VIRTUAL TABLE ...") }

  # Spawn the real server. It picks up MARTEN_ENV + MARTEN_TEST_DB +
  # MARTEN_TEST_PORT from env and lands on the same SQLite file.
  process = Process.new("./bin/server", env: {
    "MARTEN_ENV"       => "test",
    "MARTEN_TEST_DB"   => ENV["MARTEN_TEST_DB"],
    "MARTEN_TEST_PORT" => ENV["MARTEN_TEST_PORT"],
  }, output: log, error: log)
  LuckyFlow.wait_for_server(timeout: 10.seconds)
end

Spec.after_suite { process.signal(::Signal::TERM) rescue nil }
```

Plus a `script/spec_system` wrapper that pre-builds `bin/server` (cold compile ~30s, warm rebuild ~3s) before invoking the spec runner. Per-spec wall clock is ~10s once the binary is built and Chrome is warmed.

### DSL shim

Crystal Spec runs `it` blocks at top level with no implicit instance scope, so the DSL ships as top-level free functions: `visit`, `fill_in`, `click_on`, `assert_text`, `assert_selector`, `assert_current_path`, `execute_script`, `sign_in(user)`, `wait_until`. Internally they delegate to a memoized `LuckyFlow.new` wrapper. ~150 LOC; mirrors the most-used 12 Capybara verbs.

### Gotchas

1. **Cross-process DB sharing**: the spec process and server process must hit the same SQLite file. `File.tempfile(...)` per process creates different paths; instead pick one path in the spec helper and pass it to the subprocess via env var. The test env reads `ENV["MARTEN_TEST_DB"]` in its `config.database` block. Without this, the server runs against an empty (schema-but-no-rows) DB while the spec process holds the real data.

2. **Assets must be served in test env**: Marten's default routes mount `ServeAsset` only when `Marten.env.development?`. Without it, system specs that depend on Turbo / Stimulus / custom-element JS see 404s and the JS never attaches. Two-line route fix: gate on `development? || test?`.

3. **`request.turbo?` matches `*/*`**: same trap as the handler tier — bites system specs harder because real browsers always send a wildcard. Handler endpoints that branch on `request.turbo?` take the 204/turbo-stream path even on plain `<form>` submits. Workarounds: (a) re-navigate manually after submit (the save persists before the 204); (b) POST via `fetch()` with explicit `Accept: text/html` to force the redirect path; (c) tighten `marten-turbo`'s `accepts?` to not match via `*/*`.

4. **Form-associated custom elements need their JS to participate in submissions.** A `<house-md name="body">` (or similar) only contributes to `FormData` once the custom element is *registered* — i.e. once its JS module has loaded and `customElements.define(...)` has run. Setting `.value` on an un-upgraded element via `execute_script` is a no-op for form submit. Either ensure JS loads (gotcha #2 above) or build the `FormData` explicitly via `fetch()` and bypass the custom element layer.

5. **LuckyFlow's abstract `Driver` doesn't expose `execute_script`**. Only `LuckyFlow::Selenium::Driver` has it (via the underlying `session.document_manager`), and the session is behind a `private getter`. Reopen the class to surface a public bridge:

   ```crystal
   abstract class LuckyFlow::Selenium::Driver
     def execute_script(script : String) : String
       session.document_manager.execute_script(script)
     end
   end
   ```

   The return value is the script's `return …`, JSON-stringified if it's not already a string. Useful for polling `window.__flag`-style sync points after async fetches.

6. **`click_on` needs aria-label / title fallback.** Capybara's `click_on` falls back through visible text → aria-label → title → id → name. Cosmetic icon-only buttons (an SVG plus a screen-reader span and `aria-label="Create book"`) don't match visible-text xpath. Include `[@aria-label=…] or [@title=…]` in the locator chain.

7. **Anonymous-session simulation**: Rails' `using_session "public" do … end` boots a fresh Capybara session. LuckyFlow has no direct equivalent; the simplest replacement is `execute_script` to clear the auth cookie:

   ```js
   document.cookie = 'sessionid=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;'
   ```

   Then visit the public URL. Marten's session cookie name is `sessionid` by default.

8. **`schema_editor.sync_models` doesn't run in the spawned server.** The system-spec process initializes the schema via `Marten::Spec.setup_databases` (`before_suite`). The server subprocess does NOT — it just runs `Marten.start`. Both processes share the SQLite file, so the server inherits whatever the spec process built. Don't try to `setup_databases` from the server; let the spec process own schema lifecycle.

9. **Split `script/spec` and `script/spec_system`.** The default `script/spec` runner globs `spec/**/*_spec.cr` and tries to compile every spec helper. System specs require LuckyFlow + a built `bin/server` and tank dev-loop time. Exclude `spec/system/` from `script/spec`; have `script/spec_system` run only that dir and build `bin/server` first. ~10x dev-loop speedup for the common (handler) case.

10. **`Time.monotonic` is deprecated** in current Crystal — use `Time.instant`. Comes up immediately writing retry loops for assertion helpers (`assert_text` with implicit-wait semantics).

---

## What to deliberately skip from Rails test-land

- **YAML fixtures.** Marten has no fixture loader equivalent. Replace with per-spec factory modules. Specs read more honestly (no "what's in `:kevin`?") and run faster.
- **`assert_select` (HTML-aware matchers).** No equivalent in Crystal's stdlib spec. Substring matching on `response.content` is enough for ~95% of cases; for the remainder, post-parse with `XML.parse_html` and walk the tree.
- **`parallelize(workers: :number_of_processors)`.** Marten's `Spec.after_each` flushes the shared database — parallel processes racing on one tempfile DB conflict. Dev-loop run time rarely needs parallelism at handler-tier scale (~10s for 200+ examples warm).
- **`travel_to` / `freeze_time`.** Use the `timecop` shard ([`mverzilli/crystal-timecop`](https://github.com/mverzilli/crystal-timecop)) when needed — Marten's own specs depend on it. Out of scope for most handler-tier porting.
