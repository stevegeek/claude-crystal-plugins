## Syntax basics

```html
{{ variable }}            {# output a variable #}
{{ article.title }}       {# dot-notation attribute access #}
{{ name|capitalize }}     {# apply a filter #}
{{ name|default:"Guest" }} {# filter with argument #}
{% tag args %}            {# tag — logic or control flow #}
{# single-line comment #} {# NOT rendered; does NOT span newlines #}
```

**Auto-escaping is on by default.** `<script>` in a variable becomes `&lt;script&gt;`. Use `|safe` to opt out for trusted HTML.

**Footgun — comments do not span newlines.** `{# … #}` and `<!-- … -->` are line-scoped. A `{% tag %}` on the next line still fires even if you think it's "commented out". This bites inside `<script>` and `<style>` blocks too — a `{% csrf_input %}` written in a JS comment is still parsed and executed.


## Variables

```html
{{ name }}              {# simple context key #}
{{ article.title }}     {# attribute lookup #}
{{ article.author.name }} {# chained #}
{{ my_array.0 }}        {# index access — use integer, not bracket notation #}
{{ my_hash.key }}       {# hash key lookup #}
```

Literal values are valid anywhere variables are: `nil`, `true`, `false`, integers (`42`), floats (`3.14`), single- or double-quoted strings.

**Footgun — models need `template_attributes`.**  
Custom classes and model fields are only accessible from templates if the class includes `Marten::Template::Object` and declares `template_attributes :field1, :field2`. Fields not listed there silently resolve to `nil`. See `models.md`.

**Enum values** resolve to their integer. Use the `<name>?` helper in conditions: `{% if color.red? %}`.

Unknown variables silently produce `nil` (empty output, falsey). Enable `config.templates.strict_variables = true` to raise instead.


## Tags

### Control flow

```html
{% if user.admin? %}
  Admin panel
{% elsif user.active? %}
  Welcome back
{% else %}
  Please log in
{% endif %}

{% unless item.published? %}
  Draft — not visible
{% endunless %}

{% for book in books %}
  {{ loop.index }}. {{ book.title }}
{% else %}
  No books found.
{% endfor %}

{% with greeting = "Hello", count = 42 %}
  {{ greeting }}, you have {{ count }} items.
{% endwith %}
```

`loop` variables inside `{% for %}`:

| Variable | Description |
|---|---|
| `loop.index` | 1-indexed position |
| `loop.index0` | 0-indexed position |
| `loop.first?` | `true` on first iteration |
| `loop.last?` | `true` on last iteration |
| `loop.length` | total count |
| `loop.even?` / `loop.odd?` | parity of 0-indexed position |
| `loop.revindex` / `loop.revindex0` | reverse index (1- and 0-based) |
| `loop.parent` | outer loop's `loop` object (nested loops) |

For loops can unpack pairs: `{% for label, url in nav_items %}`.

**Footgun — `{% with %}` uses `=`, not `:`.** `{% url %}` uses `:`. Don't mix them up.

### Inheritance and inclusion

```html
{# base.html #}
<title>{% block title %}My Site{% endblock %}</title>
{% block content %}{% endblock %}

{# child.html #}
{% extend "base.html" %}

{% block title %}{{ page_title }}{% endblock %}

{% block content %}
  {# access parent block content #}
  {% super %} — extra content
{% endblock %}
```

`{% extend %}` must be the first tag in the file.

```html
{% include "partials/button.html" %}
{% include "partials/button.html" with type="primary", text="Click me" %}
{% include "partials/button.html" with type="primary" isolated %}   {# no outer context #}
{% include "partials/button.html" with type="primary" contextual %} {# explicit outer context #}
```

### URLs and assets

```html
{% url "books_index" %}
{% url "book_show" id: book.id %}
{% url "search" q: query, page: 1 %}
{% url "book_show" id: book.id as book_url %}

{% asset "app/app.css" %}
{% asset "javascript/app.js" as js_url %}
```

**Footgun — `{% url %}` uses colons for keyword args**: `id: book.id`, not `id=book.id`.

### Forms and CSRF

```html
<form method="post" action="{% url 'items_create' %}">
  {% csrf_input %}           {# renders <input type="hidden" name="csrftoken" value="..."> #}
  ...
</form>

{# raw token value only: #}
<input type="hidden" name="csrftoken" value="{% csrf_token %}">

{# HTTP method override for PUT/PATCH/DELETE: #}
<form method="post" action="{% url 'item_update' id: item.id %}">
  {% csrf_input %}
  {% method_input "PATCH" %}
  ...
</form>
```

### Variables and assignment

```html
{% assign my_var = "Hello" %}
{% assign my_var = "Hello" unless defined %}  {# no-op if already set #}

{% capture greeting %}
  Hello, {{ name }}!
{% endcapture %}
{{ greeting }}
```

### Caching

```html
{% cache "sidebar" 3600 %}
  ...expensive content...
{% endcache %}

{% cache "user-sidebar" 3600 current_locale user.id %}
  ...varies by locale and user...
{% endcache %}
```

### i18n

```html
{% translate "nav.home" %}
{% t "greeting" name: user.name %}      {# t is an alias for translate #}
{% l "key" %}                           {# l is an alias for localize #}
{% localize created_at format: "short" %}
{% local_time "%Y-%m-%d" %}
```

See Marten i18n docs for locale file setup (out of scope here).

### Misc

```html
{% verbatim %}
  Sent to browser as-is: {{ not_a_variable }}, {% not_a_tag %}.
  Useful when embedding JS template syntax like Vue/Alpine {{ }} blocks.
{% endverbatim %}

{% spaceless %}
  <p>  <a href="/">Home</a>  </p>
{% endspaceless %}
{# outputs: <p><a href="/">Home</a></p> #}

{% escape off %}
  {{ article.trusted_html_body }}
{% endescape %}
```

### Full tag reference table

| Tag | Notes |
|---|---|
| `{% assign x = val %}` | Set a context variable |
| `{% asset "path" %}` | Output asset URL |
| `{% block name %}` | Define overridable block |
| `{% cache "key" secs %}` | Cache fragment |
| `{% capture x %}…{% endcapture %}` | Capture rendered block to variable |
| `{% csrf_input %}` | Hidden CSRF `<input>` element |
| `{% csrf_token %}` | Raw CSRF token value |
| `{% escape on/off %}…{% endescape %}` | Toggle auto-escaping |
| `{% extend "base.html" %}` | Inherit from base template (must be first) |
| `{% for x in xs %}…{% else %}…{% endfor %}` | Loop with optional empty fallback |
| `{% if cond %}…{% elsif %}…{% else %}…{% endif %}` | Conditional |
| `{% include "partial.html" with k=v %}` | Include partial |
| `{% l "key" %}` / `{% localize val %}` | Localize a value |
| `{% local_time "%Y-%m-%d" %}` | Current local time |
| `{% method_input "PATCH" %}` | Hidden `_method` override input |
| `{% spaceless %}…{% endspaceless %}` | Strip whitespace between tags |
| `{% super %}` | Render parent block content (inside `{% block %}`) |
| `{% t "key" k: v %}` / `{% translate %}` | i18n translation lookup |
| `{% unless cond %}…{% endunless %}` | Inverse if |
| `{% url "name" k: v %}` | Reverse URL lookup |
| `{% verbatim %}…{% endverbatim %}` | Emit template syntax literally |
| `{% with x = v %}…{% endwith %}` | Scoped variable block |


## Filters

```html
{{ name|capitalize }}          {# "hello world" → "Hello world" #}
{{ name|upcase }}              {# "hello" → "HELLO" #}
{{ name|downcase }}            {# "Hello" → "hello" #}
{{ slug|underscore }}          {# "FooBar" → "foo_bar" #}
{{ html|safe }}                {# bypass auto-escaping #}
{{ html|escape }}              {# force-escape (usually redundant) #}
{{ bio|linebreaks }}           {# \n → <br /> #}
{{ tags|join:", " }}           {# ["a","b"] → "a, b" #}
{{ csv|split:"," }}            {# "a,b" → ["a","b"] #}
{{ items|size }}               {# count of string chars or array elements #}
{{ name|default:"Anonymous" }} {# fallback when nil/false/0 #}
{{ created_at|time:"%Y-%m-%d" }} {# format a Time value #}
```

**Footgun — there is no `|length` filter. Use `|size`.**

| Filter | Arg | Notes |
|---|---|---|
| `capitalize` | — | First char upper, rest lower |
| `default:"x"` | fallback | Returns fallback when value is nil, false, or 0 |
| `downcase` | — | All lowercase |
| `escape` | — | HTML-escape special chars |
| `join:"sep"` | separator | Array → string |
| `linebreaks` | — | Newlines → `<br />` |
| `safe` | — | Mark as trusted HTML, skip auto-escaping |
| `size` | — | String length or array count |
| `split:"sep"` | separator | String → array |
| `time:"%fmt"` | Crystal `Time::Format` string | Format a `Time` value |
| `underscore` | — | CamelCase → snake_case |
| `upcase` | — | All uppercase |

Chaining: `{{ name|downcase|capitalize }}` — left to right.


## Operators

Used in `{% if %}` and `{% unless %}` conditions.

```html
{% if score == 100 %}Perfect{% endif %}
{% if score != 0 && lives > 0 %}Still playing{% endif %}
{% if !user.admin? %}No access{% endif %}
{% if "crystal" in languages %}Supported{% endif %}
{% if count >= 10 %}Many{% elsif count > 0 %}Some{% else %}None{% endif %}
```

| Operator | Meaning |
|---|---|
| `==` | Equals |
| `!=` | Not equals |
| `<` / `>` | Less / greater than |
| `<=` / `>=` | Less-or-equal / greater-or-equal |
| `&&` | Logical AND |
| `\|\|` | Logical OR |
| `!` / `not` | Logical negation |
| `in` | Substring or array membership |


## Context

Context values are passed from a handler using `render(template, context: {...})`:

```crystal
# handler
render "books/index.html", context: { books: Book.all, page_title: "Books" }
```

In the template, keys are accessed by name: `{{ books }}`, `{{ page_title }}`. Hash keys must be strings (symbol keys in named tuples are converted automatically).

Rendering programmatically:

```crystal
template = Marten.templates.get_template("foo/bar.html")
template.render({"foo" => "bar"})
template.render({ foo: "bar" })  # named tuple also works
```


## Context producers

Context producers automatically inject variables into every template context. Configured in `config/settings/base.cr`:

```crystal
config.templates.context_producers = [
  Marten::Template::ContextProducer::Request,
  Marten::Template::ContextProducer::Flash,
  Marten::Template::ContextProducer::Debug,
  Marten::Template::ContextProducer::I18n,
]
```

| Producer | Variable(s) injected | Notes |
|---|---|---|
| `Request` | `request` | Current `HTTP::Request` object; no-op if no request in context |
| `Flash` | `flash` | Flash store for the current request; no-op if no request |
| `Debug` | `debug` | `true` / `false` based on `config.debug` setting |
| `I18n` | `locale`, `available_locales` | Current locale string and array of all configured locales |

**Custom context producer:**

```crystal
class CurrentUserProducer < Marten::Template::ContextProducer
  def produce(request : Marten::HTTP::Request? = nil)
    return nil if request.nil?
    user = request.session["user_id"]? ? User.get(id: request.session["user_id"]) : nil
    {"current_user" => user}
  end
end
```

Register it by adding the class to `config.templates.context_producers`.


## Custom tags

```crystal
class GreetTag < Marten::Template::Tag::Base
  include Marten::Template::Tag::CanSplitSmartly

  def initialize(parser : Marten::Template::Parser, source : String)
    parts = split_smartly(source)
    raise Marten::Template::Errors::InvalidSyntax.new("greet requires a name arg") if parts.size < 2
    @name_expr = Marten::Template::FilterExpression.new(parts[1])
  end

  def render(context : Marten::Template::Context) : String
    "Hello, #{@name_expr.resolve(context)}!"
  end
end

Marten::Template::Tag.register("greet", GreetTag)
```

Use in template: `{% greet user.name %}`.

- `#initialize` runs at parse time — validate syntax and build expressions here.
- `#render` runs at render time — resolve expressions against context and return a `String`.
- For closable tags, call `parser.parse(up_to: %w(endmytag))` then `parser.shift_token` to consume the closing tag.
- Assign-to-variable support: look for `as varname` in `parts` and write to `context[var] = result; return ""`.


## Custom filters

```crystal
class TruncateFilter < Marten::Template::Filter::Base
  def apply(value : Marten::Template::Value, arg : Marten::Template::Value? = nil) : Marten::Template::Value
    max = arg ? arg.to_s.to_i : 100
    Marten::Template::Value.from(value.to_s[0, max])
  end
end

Marten::Template::Filter.register("truncate", TruncateFilter)
```

Use: `{{ article.body|truncate:200 }}`.

If your filter outputs HTML that should not be re-escaped, wrap the result in `Marten::Template::SafeString.new(str)` before passing to `Value.from`.


## Loaders

Template resolution order follows the configured loaders. Default behavior: app `templates/` directories (`templates.app_dirs = true`) plus any paths in `templates.dirs`.

Override with `templates.loaders` for full control:

```crystal
# config/settings/base.cr
config.templates.loaders = [
  Marten::Template::Loader::FileSystem.new("/path/to/shared/templates"),
  Marten::Template::Loader::AppDirs.new,
] of Marten::Template::Loader::Base
```

| Loader | Purpose |
|---|---|
| `FileSystem.new("/path")` | Load from a filesystem directory |
| `AppDirs.new` | Load from each installed app's `templates/` folder |
| `Cached.new([...loaders])` | Cache compiled templates in memory (wrap another loader) |

**Custom loader** — subclass `Marten::Template::Loader::Base` and implement `#get_template_source(template_name) : String`. Raise `Marten::Template::Errors::TemplateNotFound` when the template does not exist.

```crystal
class InMemoryLoader < Marten::Template::Loader::Base
  def initialize(@store : Hash(String, String))
  end

  def get_template_source(template_name) : String
    @store[template_name]? || raise Marten::Template::Errors::TemplateNotFound.new(template_name)
  end
end
```


## When to look elsewhere

- Handlers that render templates and pass context → `handlers.md`
- Models — `template_attributes` declaration to expose fields → `models.md`
- i18n / translations — see Marten docs (out of scope)
