## Two CLIs

There are two distinct CLI tools. Confusing them is a common mistake.

**`marten` global CLI** — installed once, used for scaffolding new projects and apps:

```bash
marten new project myblog                       # create new project
marten new project myblog --with-auth           # project + auth app
marten new project myblog --database=postgresql # preconfigure DB
marten new app blog                             # standalone app shard scaffold
```

**`manage.cr` project CLI** — lives inside each project, used for everything else. The canonical invocation is:

```bash
crystal run manage.cr -- <command> [args]
# or via the wrapper:
script/manage <command> [args]
```

`script/manage` is what you use day-to-day. The `marten` global CLI is mostly for project creation.

---

## `script/manage` and `script/serve` wrappers

Every project ships these wrappers because **asdf 0.18 (Go-based rewrite) bypasses the upstream asdf-crystal bash shim** that normally exports `CRYSTAL_LIBRARY_PATH`. Without the export the linker cannot find `libgc` and compilation fails.

`script/manage` (canonical):

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
export CRYSTAL_LIBRARY_PATH="${CRYSTAL_LIBRARY_PATH:+$CRYSTAL_LIBRARY_PATH:}$HOME/.asdf/installs/crystal/1.20.1/embedded/lib"
exec crystal run manage.cr -- "$@"
```

`script/serve` does the same export, then builds `bin/server` if sources are newer than the binary and execs it. It does **not** use `marten serve`; it compiles a production-style binary.

Update the Crystal version string in both scripts when upgrading Crystal.

---

## `marten new` — project scaffolding

```bash
marten new project myblog                       # SQLite by default
marten new project myblog --database=postgresql
marten new project myblog --with-auth           # includes marten-auth app
marten new project myblog --with-image-support  # adds vips shard
marten new app auth                             # standalone app repo (for distribution)
```

To add an app **inside an existing project**, use the generator instead:

```bash
script/manage gen app blog
# Adds src/blog/, wires into installed_apps, src/project.cr, src/cli.cr, config/routes.cr
```

---

## Built-in management commands

Run as `script/manage <cmd>` (or `crystal run manage.cr -- <cmd>`).

| Command | Description |
|---|---|
| `genmigrations [app]` | Diff models vs migration history, write new migration files |
| `genmigrations app --empty` | Write a blank migration for manual editing |
| `migrate [app [version]]` | Apply all pending migrations; target a version to roll back |
| `migrate foo zero` | Unapply all migrations for the `foo` app |
| `listmigrations [app]` | Show applied/unapplied migrations |
| `resetmigrations app` | Squash all migrations for an app into one |
| `gen <generator> [args]` | Run a code generator (see below) |
| `serve` | Dev server with auto-recompile (projects use `script/serve` instead) |
| `routes` | Print all registered routes |
| `seed` | Run the seed file to populate the DB |
| `clearsessions` | Delete expired sessions |
| `collectassets` | Copy assets into the configured storage |
| `play` | Start Crystal playground with project loaded |
| `version` | Print Marten version |

Common `migrate` options: `--fake` (mark applied without running), `--plan` (preview operations), `--db=ALIAS`.

---

## `gen` generators

```bash
script/manage gen                        # list all available generators
script/manage gen model User name:string email:string
script/manage gen model User name:string email:string --app blog
script/manage gen model Article label:string body:text author:many_to_one{User}
script/manage gen model User name:string --no-timestamps
script/manage gen schema ArticleSchema title:string body:string
script/manage gen handler ArticleHandler --app blog
script/manage gen email WelcomeEmail --app blog
script/manage gen app blog               # add app to current project
script/manage gen auth                   # full auth app (adds marten-auth to shard.yml)
script/manage gen secretkey              # generate a new secret_key value
```

Alias: `g` works as a shorthand (`script/manage g model ...`).

Field definition syntax: `name:type`, `name:type{qualifier}`, `name:type:modifier`.  
Examples: `max_size:string{128}`, `author:many_to_one{User}`, `slug:string:uniq:index`.

---

## `manage.cr` — the 4-line entry point

```crystal
require "./src/cli"

Marten.setup
Marten::CLI.run
```

This is the complete `manage.cr`. It requires `src/cli.cr`, sets up the project, then runs the CLI dispatcher.

---

## `src/cli.cr`

```crystal
require "./project"

require "marten/cli"

require "./migrations/**"
```

Requires `src/project.cr` (all app code), loads the Marten CLI, then requires migration files so the CLI knows about them. App-specific `cli.cr` files follow the same pattern but scoped to their directory.

---

## `src/project.cr` — require order matters

```crystal
require "marten"
require "sqlite3"          # or pg / mysql
# third-party shards
require "marten_auth"
require "marten_turbo"
require "marten_cable"

# Settings and initializers first
require "../config/settings/base"
require "../config/settings/**"
require "../config/initializers/**"
require "../config/routes"

# App code — concerns/helpers before the things that use them
require "./schemas/**"
require "./models/**"
require "./handlers/**"

# Cable channels last: reference models, must precede Marten.start
require "./channels/**"
```

Order is strict: settings before models, models before handlers, channels last.

---

## Apps — multi-app projects

Each app lives under `src/<label>/` and defines:

```crystal
# src/blog/app.cr
require "./emails/**"
require "./handlers/**"
require "./models/**"
require "./routes"
require "./schemas/**"

module Blog
  class App < Marten::App
    label "blog"
  end
end
```

Register in `config/settings/base.cr`:

```crystal
config.installed_apps = [
  BlogApp,
  AuthApp,
]
```

Each app has its own `migrations/` folder. `genmigrations`/`migrate` operate per-app or across all apps.

App order in `installed_apps` determines template lookup order — namespace your templates (`blog/post.html`) to avoid conflicts.

---

## Custom management commands

Subclass `Marten::CLI::Command`. Place the file in `src/<app>/cli/`. Require it from the app's `cli.cr`.

```crystal
# src/blog/cli/reindex.cr
class ReindexCommand < Marten::CLI::Command
  command_name :reindex
  help "Re-index all blog posts for search"

  @app_label : String?

  def setup
    on_option_with_arg("a", "app", "label", "Limit to one app") { |v| @app_label = v }
  end

  def run
    print("Re-indexing...")
    # call your indexing logic here
    print(style("Done.", fore: :green))
  end
end
```

Key methods:

| Method | Purpose |
|---|---|
| `help "text"` | Help string shown in `marten help` |
| `command_name :name` | Override inferred name (default: class name snake-cased) |
| `command_aliases :alias` | Short alias |
| `on_argument(:name, "desc") { \|v\| }` | Positional argument |
| `on_option("flag", "desc") { }` | Boolean flag |
| `on_option_with_arg("f", "flag", "arg", "desc") { \|v\| }` | Flag with value |
| `print(msg)` | Write to stdout |
| `print_error(msg)` | Write to stderr |
| `print_error_and_exit(msg)` | Write to stderr and exit 1 |
| `style(msg, fore: :green, mode: :bold)` | Colorize output |

Custom commands become available automatically once the app is in `installed_apps` and required.

---

## Common dev workflows

```bash
# After adding or changing a model:
script/manage genmigrations
script/manage migrate

# Boot the dev server:
script/serve

# Inspect data interactively (requires icr):
script/manage shell

# Inspect all routes:
script/manage routes

# Generate a new model + migration in one pass:
script/manage gen model Post title:string body:text author:many_to_one{User}
script/manage genmigrations
script/manage migrate

# Roll back the last migration for the blog app:
script/manage migrate blog <previous_migration_version>
```

---

## When to look elsewhere

- Migrations CLI in detail → `migrations.md`
- Settings configuration → `settings.md`
- Custom generators (less common) → Marten docs generators reference
