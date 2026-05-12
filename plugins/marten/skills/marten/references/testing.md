## Test framework

Crystal's stdlib `spec` library. Spec files live anywhere under `spec/` and must be named `*_spec.cr`. Marten adds `marten/spec`, which installs the before/after hooks that create and flush the test database automatically.

---

## spec_helper.cr

```crystal
# spec/spec_helper.cr
ENV["MARTEN_ENV"] = "test"

require "spec"
require "marten"
require "marten/spec"

require "../src/project"
```

Setting `ENV["MARTEN_ENV"] = "test"` (or running `MARTEN_ENV=test crystal spec`) loads `config/settings/test.cr`. Every spec file starts with `require "./spec_helper"`. For nested folders, chain helpers:

```crystal
# spec/models/spec_helper.cr
require "../spec_helper"
```

Real-world shards (marten-encoded-id, marten-turbo) also call `Marten.configure :test do ... end` directly in `spec_helper.cr` when no separate `config/` tree exists — valid for library specs, not typical app layout.

---

## Test database lifecycle

`marten/spec` hooks create and flush the test DB before the suite runs. Each spec is expected to leave the DB clean; rely on the between-spec flush rather than manual teardown.

**`config/settings/test.cr` must set a distinct DB name — Marten refuses to run specs if the name matches dev/prod:**

```crystal
# config/settings/test.cr
Marten.configure :test do |config|
  config.secret_key = "insecure-test-key"

  config.database do |db|
    db.backend = :sqlite
    db.name    = ":memory:"           # SQLite: fastest option
    # db.name  = "myapp_test"         # Postgres: must differ from dev name
  end
end
```

---

## Model specs

Plain `create!` + assertions. No special setup beyond the spec_helper.

```crystal
require "./spec_helper"

describe Book do
  it "auto-populates slug from title" do
    book = Book.create!(title: "Hello World")
    book.slug.should eq "hello-world"
  end

  it "raises on missing required field" do
    expect_raises(Marten::DB::Errors::InvalidRecord) do
      Book.create!(title: "")
    end
  end

  it "find_by returns nil for unknown id" do
    Book.get(id: 999999).should be_nil
  end
end
```

---

## Schema specs

Instantiate with `Marten::HTTP::Params::Data`, call `.valid?`, inspect `.errors`.

```crystal
require "./spec_helper"

describe BookSchema do
  it "rejects an empty title" do
    schema = BookSchema.new(Marten::HTTP::Params::Data.new({"title" => [""]}))
    schema.valid?.should be_false
    schema.errors[:title].should_not be_empty
  end

  it "accepts a valid payload" do
    schema = BookSchema.new(Marten::HTTP::Params::Data.new({"title" => ["Dune"]}))
    schema.valid?.should be_true
  end
end
```

---

## Handler specs — the test client

`Marten::Spec.client` returns a memoized `Marten::Spec::Client` reset after each spec. Use it for all handler testing; do not construct `Marten::HTTP::Request` by hand unless testing middleware directly.

```crystal
require "./spec_helper"

describe BooksHandler do
  describe "#get" do
    it "lists books" do
      Book.create!(title: "Dune")
      response = Marten::Spec.client.get(Marten.routes.reverse("books_index"))
      response.status.should eq 200
      response.content.should contain "Dune"
    end
  end

  describe "#post" do
    it "creates a book and redirects" do
      url = Marten.routes.reverse("books_create")
      response = Marten::Spec.client.post(url, data: {"title" => "Foundation"})
      response.status.should eq 302
      Book.filter(title: "Foundation").exists?.should be_true
    end
  end
end
```

Key properties:
- No server needed — the client runs the middleware + routing chain in-process.
- **CSRF checks are disabled by default.** To enable: `Marten::Spec::Client.new(disable_request_forgery_protection: false)`.
- Cookies and session values persist across calls within one client instance.
- Handler exceptions surface directly in the spec (use `expect_raises` to assert on them).

Accessing session / flash before a request:

```crystal
Marten::Spec.client.session["cart_id"] = "abc"
response = Marten::Spec.client.get(Marten.routes.reverse("cart"))

Marten::Spec.client.flash[:notice].should eq "Item added"
```

---

## Authentication in handler specs

With [marten-auth](https://github.com/martenframework/marten-auth), use the built-in helpers:

```crystal
user = Auth::User.create!(email: "test@example.com") do |u|
  u.set_password("insecure")
end

Marten::Spec.client.sign_in(user)
response = Marten::Spec.client.get(Marten.routes.reverse("auth:profile"))
response.status.should eq 200

Marten::Spec.client.sign_out
```

`sign_in` writes the user PK into the client's session store; `sign_out` flushes it.

---

## Constructing requests directly (low-level / middleware specs)

When `Marten::Spec.client` is too high-level (e.g., testing a handler method in isolation), build the request manually — the pattern used in marten-turbo's own handler specs:

```crystal
request = Marten::HTTP::Request.new(
  ::HTTP::Request.new(
    method: "POST",
    resource: "",
    headers: HTTP::Headers{
      "Host"         => "example.com",
      "Content-Type" => "application/x-www-form-urlencoded",
      "Accept"       => "text/html",
    },
    body: "name=foo"
  )
)
handler = MyHandler.new(request)
response = handler.post
response.should be_a Marten::HTTP::Response::Found
```

Prefer `Marten::Spec.client` for integration-style tests; reserve direct instantiation for unit-testing handler methods or middleware logic.

---

## Fixtures / test helpers

Marten ships no fixture loader. Keep setup code in `spec/support/`:

```crystal
# spec/support/factories.cr
def create_book(title = "Default Title", **attrs)
  Book.create!(title: title, **attrs)
end

def create_user(email = "user@example.com")
  Auth::User.create!(email: email) { |u| u.set_password("insecure") }
end
```

Require from spec_helper: `require "./support/factories"`. Never share record instances across specs — create fresh records per `it` block and let the between-spec flush clean up.

---

## Email collection

Configure in `config/settings/test.cr`:

```crystal
config.emailing.backend = Marten::Emailing::Backend::Development.new(
  collect_emails: true,
  print_emails:   false,
)
```

Inspect in specs via `Marten::Spec.delivered_emails` (auto-reset after each spec):

```crystal
it "sends a welcome email" do
  WelcomeEmail.new(user: user).deliver
  Marten::Spec.delivered_emails.size.should eq 1
  Marten::Spec.delivered_emails[0].subject.should eq "Welcome!"
end
```

---

## Running specs

```shell
crystal spec                          # full suite
crystal spec spec/models/book_spec.cr # single file
crystal spec --example "lists books"  # filter by description
```

Use `fit` / `fdescribe` for focus (run only focused examples). Projects using asdf Crystal 0.18+ often wrap this in `script/spec` to handle `CRYSTAL_LIBRARY_PATH` — see `cli.md`.

---

## Things to look elsewhere

- Settings (`test.cr` config) → `settings.md`
- CLI invocation (`script/spec` wrapper) → `cli.md`
- Auth-specific test patterns → `auth.md`
