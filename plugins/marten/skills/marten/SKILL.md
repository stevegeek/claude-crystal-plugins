---
name: Marten 0.7 (Crystal web framework)
description: This skill should be used when the user is working in a Marten (Crystal language) project — writing or porting code, designing models/handlers/migrations, or asking how to do something in Marten. Triggers on imports of `marten`, files like `manage.cr` / `src/project.cr` / `config/settings/*.cr`, mention of `Marten::Model` / `Marten::Handler` / `Marten::Schema`, or questions like "how do I do X in Marten" / "Marten equivalent of …" / "Crystal Rails port". Covers Marten 0.7 (the active main branch as of 2026); earlier versions only when explicitly asked.
version: 0.1.0
---

# Marten — Crystal web framework

Marten is a Django-shaped MVC framework for Crystal. ORM + handlers + templates + schemas + migrations, batteries included.

You are the Crystal/Marten porting partner. Default to **idiomatic Marten 0.7**. Don't reach for Rails-isms or Django-isms unless the framework genuinely lacks an equivalent.

## When to read which reference

Pull only the file relevant to the task — references are loaded on demand to keep context small.

| Task | Read |
|---|---|
| Defining a model, fields, relations, polymorphic, validations, callbacks | `references/models.md` |
| QuerySets, filtering, joins, raw SQL, transactions, paginators | `references/querying.md` |
| Migrations (auto + hand-written, raw-SQL escape hatch, FTS) | `references/migrations.md` |
| Writing a handler, generic record handlers, lifecycle callbacks, error handlers | `references/handlers.md` |
| Routes, path params, named routes, reverse lookup | `references/routing.md` |
| Cookies, sessions, middlewares, CSRF, security headers | `references/http.md` |
| Forms — Marten Schemas, validation, the schema-handler bridge | `references/schemas.md` |
| Templates — syntax, tags, filters, custom tags, context producers | `references/templates.md` |
| Uploads, file fields, storage backends, static assets | `references/files.md` |
| Auth via the `marten-auth` shard | `references/auth.md` |
| Settings — installed apps, middleware list, db config, all knobs | `references/settings.md` |
| `marten` CLI, generators, management commands, custom commands | `references/cli.md` |
| Testing — spec setup, fixtures, test client | `references/testing.md` |
| Common gotchas + Crystal-specific surprises | `references/gotchas.md` |
| Porting from Rails — idiom mapping (apps, handlers, auth, models, forms, templates, active storage, …) | `references/rails-mapping.md` |
| Porting Rails tests — handler tier + LuckyFlow system tier | `references/rails-testing.md` |

If the user is porting from Rails specifically, `rails-mapping.md` and `rails-testing.md` cover translation recipes for the common idioms. The `rails-to-marten-porter` agent in this plugin wraps both for guided port work.

## Hard rules

These are easy to get wrong; check `references/gotchas.md` for the long list. Top hits:

1. **Marten 0.7 has polymorphic relationships** (`field :owner, :polymorphic, to: [User, Group]`). Closed-set, requires ≥2 types in `to:` or an explicit `of Marten::DB::Model.class` annotation. **Never name a model `Page`** — Marten generates `::<Model>::Page` paginator inner classes that shadow it inside polymorphic-field declarations. Use a namespace.

2. **Forms go through `Marten::Schema`**, not raw `request.data` parsing. Schemas validate AND coerce; access values via `schema.validated_data["field"].as(String)`.

3. **`{% csrf_input %}` (form input), not `{% csrf_token %}`** (raw token value, JS-only).

4. **Models need `template_attributes :id, :foo, …`** to expose fields to templates. Without it, `{{ obj.foo }}` raises `UnknownVariable`.

5. **`{% url 'name' key: val %}`** uses **colon**, not equals. `{% with key=val %}` uses equals. Different tags, different conventions.

6. **`before_create` runs AFTER validation** so it can't populate `null: false` fields. Use `before_validation` instead.

7. **Concern macros that try to register class-level callbacks (`before_validation`, etc.) from inside `macro included` may silently no-op** due to Crystal macro scoping. If a callback isn't firing, declare it in the host model directly, OR use a helper module that the host calls.

8. **No `length` template filter — use `|size`**.

9. **Model concerns use `macro included`** for `extend ActiveSupport::Concern; included do ... end` style. Class methods from concerns: define directly in the module body, no `class_methods do ... end` wrapper.

10. **Marten `:file`/`:image` field is single-attachment-per-attribute.** No blob/attachment indirection (no `has_one_attached` / `has_many_attached`). For multi-attachment + variants, model an `Attachment` table yourself with a polymorphic `record` field.

## Project shape

A typical Marten project (used as the mental model when answering):

```
project/
├── manage.cr                     # CLI entry: `crystal run manage.cr -- migrate` etc.
├── config/
│   ├── routes.cr                 # Marten.routes.draw do ... end
│   ├── initializers/             # runs after settings + models load
│   └── settings/
│       ├── base.cr               # Marten.configure do |config|
│       ├── development.cr        # Marten.configure :development do |config|
│       ├── production.cr
│       └── test.cr
└── src/
    ├── project.cr                # requires marten + project structure
    ├── server.cr                 # require ./project; Marten.start
    ├── cli.cr                    # require ./project + marten/cli + migrations
    ├── apps/                     # optional: multi-app projects
    ├── handlers/
    ├── schemas/
    ├── models/
    ├── templates/                # served by Marten template engine
    ├── migrations/               # auto-generated by `manage genmigrations`
    └── assets/                   # served at /assets/ in dev via the dev asset handler
```

`Marten::App` configs in `config.installed_apps` enable framework apps (e.g. `MartenAuth::App`, `MartenTurbo::App`).

## Stack defaults to recommend

When advising on shard choices for a new Marten app:

- **Database:** `crystal-lang/crystal-sqlite3` (dev) → SQLite or PostgreSQL via `crystal-pg` (prod)
- **Auth:** `martenframework/marten-auth` (don't hand-roll session/cookie/bcrypt)
- **Background jobs (if needed):** `mosquito-cr/mosquito` (Redis-backed)
- **Markdown rendering:** `icyleaf/markd` (CommonMark)
- **Server-side syntax highlighting:** `ralsina/tartrazine` (Pygments/Chroma port, 241 langs)
- **Image processing:** `naqvis/crystal-vips` (libvips bindings)
- **WebSockets / Action-Cable wire protocol:** `cable-cr/cable` + `marten-cable` (workspace shard)
- **Hotwire / Turbo Streams:** `treagod/marten-turbo` (or our fork at workspace level for Turbo 8 features)
- **Encoded IDs (slug-style PKs):** `marten-encoded-id` (workspace shard)
- **Linting:** `crystal-ameba/ameba`
- **Test data:** `kibordg/faker`

## Versioning

This skill targets Marten 0.7 (`main` branch as of 2026). Marten 0.6.3 is the latest tagged release; 0.7-dev introduces:

- **Polymorphic field** (`field :foo, :polymorphic`)
- Various refinements; see `marten-src/docs/docs/the-marten-project/release-notes/0.7.md` for the full list

If user specifies an earlier version, check the matching release-notes file for what's missing.
