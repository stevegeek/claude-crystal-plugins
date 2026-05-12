# Claude Crystal Plugins

A Claude Code marketplace providing plugins for Crystal language development.

## Installation

Add this marketplace to Claude Code:

```bash
/plugin marketplace add stevegeek/claude-crystal-plugins
```

Or add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": [
    "stevegeek/claude-crystal-plugins"
  ]
}
```

Then browse and install plugins:

```bash
/plugin
```

## Available Plugins

- **marten** - Skill for the [Marten](https://martenframework.com) web framework: Django-shaped MVC for Crystal with ORM, handlers, schemas, templates, and migrations. Includes Rails-to-Marten porting guidance.

---

# marten

A skill for working with [Marten](https://martenframework.com), a Django-shaped MVC web framework for Crystal. ORM + handlers + templates + schemas + migrations, batteries included.

## What is Marten?

Marten is a full-stack Crystal web framework that takes its shape from Django: declarative models with auto-generated migrations, class-based handlers (controllers), a template engine, form schemas, and a built-in admin/auth ecosystem. If you've worked in Django or Rails, the mental model carries over - this skill helps with the parts that don't.

- Project: <https://martenframework.com>
- Repo: <https://github.com/martenframework/marten>
- Targets Marten 0.7 (the active `main` branch as of 2026)

## Installation

```bash
/plugin install marten@stevegeek-crystal-marketplace
```

Or in `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "marten@stevegeek-crystal-marketplace": true
  }
}
```

## Skill

The plugin contains one skill, `marten`, with progressive disclosure: a short top-level overview, hard-rules list, and stack recommendations point to detailed reference files for each subsystem. Claude loads only the references it needs for the task at hand.

References cover:

- **models.md** - fields, relations, polymorphic, validations, callbacks
- **querying.md** - QuerySets, filtering, joins, raw SQL, transactions, paginators
- **migrations.md** - auto + hand-written migrations, raw-SQL escape hatch, FTS
- **handlers.md** - handler classes, generic record handlers, lifecycle callbacks, error handlers
- **routing.md** - routes, path params, named routes, reverse lookup
- **http.md** - cookies, sessions, middlewares, CSRF, security headers
- **schemas.md** - Marten Schemas, validation, the schema-handler bridge
- **templates.md** - template syntax, tags, filters, custom tags, context producers
- **files.md** - uploads, file fields, storage backends, static assets
- **auth.md** - the `marten-auth` shard
- **settings.md** - installed apps, middleware list, db config, all knobs
- **cli.md** - `marten` CLI, generators, management commands, custom commands
- **testing.md** - spec setup, fixtures, test client
- **gotchas.md** - common Crystal-specific surprises

## Quick example

```crystal
# src/models/article.cr
class Article < Marten::Model
  field :id, :big_int, primary_key: true, auto: true
  field :title, :string, max_size: 200
  field :body, :text
  field :author, :many_to_one, to: User, on_delete: :cascade
  field :published_at, :date_time, blank: true, null: true

  template_attributes :id, :title, :body, :author, :published_at
end

# src/handlers/articles/list_handler.cr
class Articles::ListHandler < Marten::Handler
  def get
    render("articles/list.html", context: {articles: Article.all})
  end
end
```

---

## License

Unlicense
