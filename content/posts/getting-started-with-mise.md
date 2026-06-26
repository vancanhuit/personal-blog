+++
date = '2026-06-26T20:45:36+07:00'
draft = false
title = 'Getting Started With mise: Dev Tools, Environments, and Tasks'
tags = ['mise', 'devops', 'dev-tools', 'environment', 'task-runner', 'tooling', 'golang', 'golangci-lint']
+++
If you have ever juggled `nvm`, `pyenv`, `rbenv`, a pile of `.env` files, and a `Makefile` in the same project, [`mise`](https://mise.jdx.dev/) (pronounced "meez", short for *mise-en-place*) is worth a look. It replaces all of them with a single tool and a single `mise.toml` config file.

In this post I'll introduce the three major concepts that make `mise` useful day to day:

1. [Dev tools](https://mise.jdx.dev/dev-tools/) — install and switch between runtimes like Node, Python, and Go.
2. [Environments](https://mise.jdx.dev/environments/) — load the right environment variables per directory.
3. [Tasks](https://mise.jdx.dev/tasks/) — a built-in task runner for builds, tests, linting, and scripts.

Then I'll walk through a small but fully runnable example that ties all three together.

## What is mise?

`mise` is a polyglot tool manager, environment manager, and task runner rolled into one binary. It is a spiritual successor to [asdf](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html) (and is compatible with asdf's `.tool-versions` files), but written in Rust and considerably faster.

The core idea: you declare what a project needs in a `mise.toml` file, and `mise` makes sure those tools, versions, and environment variables are active whenever you're inside that directory.

## Install mise

On Linux or macOS:

```bash
curl https://mise.run | sh
```

By default `mise` installs to `~/.local/bin`. Verify it:

```bash
~/.local/bin/mise --version
# mise 2026.x.x
```

> See the [installation guide](https://mise.jdx.dev/installing-mise.html) for other methods: Homebrew, `apt`, `dnf`, Snap, Nix, Windows, and more.

### Activate mise (recommended)

`mise exec` works for one-off commands, but for interactive shells you'll want to *activate* `mise` so tools and env vars load automatically as you `cd` around.

Add the appropriate line to your shell's rc file:

```bash
# bash
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc

# zsh
echo 'eval "$(~/.local/bin/mise activate zsh)"' >> ~/.zshrc

# fish
echo '~/.local/bin/mise activate fish | source' >> ~/.config/fish/config.fish
```

Restart your shell, then run a health check:

```bash
mise doctor
```

> For CI/CD, IDEs, and non-interactive scripts, `mise` also supports [shims](https://mise.jdx.dev/dev-tools/shims.html) instead of shell activation.

## Concept 1: Dev tools

`mise` manages multiple versions of programming language runtimes and CLI tools on the same machine, then switches between them automatically based on the directory you're in.

Tools are declared in the `[tools]` section of `mise.toml`:

```toml
# mise.toml
[tools]
node = '24'
python = '3'
go = 'latest'
```

You rarely write this by hand. The `mise use` command adds tools for you and installs them:

```bash
# add to the current project's mise.toml
mise use node@24

# install globally (~/.config/mise/config.toml)
mise use --global node@24
```

To run a tool once without installing it permanently, use `mise exec` (aliased to `mise x`):

```bash
mise exec node@26 -- node -v
# downloads node 26 if needed, then prints v26.x.x
```

### How version switching works

Once `mise` is activated, it walks up the directory tree looking for config files (`mise.toml`, `.tool-versions`, `.node-version`, etc.), resolves the requested versions, and prepends the correct binaries to your `PATH`:

```bash
# inside a project pinned to node 20
echo $PATH
# /home/user/.local/share/mise/installs/node/20.x.x/bin:/usr/local/bin:...

node -v
# v20.x.x
```

Move to another project pinned to a different version and `node` changes automatically. Tools come from multiple [backends](https://mise.jdx.dev/dev-tools/backend_architecture.html) — core, `aqua`, `npm`, `pipx`, `cargo`, `github`, and asdf plugins — so the same `mise use` workflow installs almost anything. Browse the [registry](https://mise.jdx.dev/registry.html) to see what's available.

## Concept 2: Environments

`mise` can load environment variables per project from the `[env]` section of `mise.toml`:

```toml
# mise.toml
[env]
NODE_ENV = 'production'
APP_PORT = '3000'
```

When `mise` is activated, these are set automatically when you `cd` into the project and unset when you leave. You can also manage them from the CLI:

```bash
mise set NODE_ENV=development
mise set
# key       value        source
# NODE_ENV  development  mise.toml

mise unset NODE_ENV
```

A few handy features:

* **Unset a variable** inherited from a parent config by setting it to `false`:

  ```toml
  [env]
  NODE_ENV = false
  ```

* **Provide a fallback** that only applies if the variable isn't already set, using `default`:

  ```toml
  [env]
  NODE_ENV = { default = "development" }
  ```

* **Load a dotenv file** with the `_.file` directive:

  ```toml
  [env]
  _.file = '.env'
  ```

* **Redact secrets** so they don't leak into task output:

  ```toml
  [env]
  SECRET = { value = "my_secret", redact = true }
  ```

Because env vars resolve alongside tools, you get a single source of truth for "what does this project need to run" — versions *and* configuration together.

## Concept 3: Tasks

`mise` has a built-in task runner, so you can replace a `Makefile` or a pile of npm scripts. Tasks run with the full `mise` context loaded — your tools and env vars are already on `PATH`.

Define tasks in the `[tasks]` section:

```toml
# mise.toml
[tasks.build]
description = "Build the project"
run = "go build ./..."

[tasks.test]
description = "Run the test suite"
run = "go test ./..."
depends = ["build"]
```

Run them with `mise run` (aliased to `mise r`):

```bash
mise run build
mise run test     # runs build first because of `depends`
mise tasks        # list all available tasks
```

### Task dependencies

The `depends` array is what turns a list of tasks into a build graph. When you run a task, `mise` first runs everything in its `depends` list, and because dependencies execute **in parallel by default**, independent steps don't block each other:

```toml
# mise.toml
[tasks.lint]
run = "golangci-lint run ./..."

[tasks.test]
run = "go test ./..."

[tasks.ci]
description = "Run lint and test, then build"
depends = ["lint", "test"]   # these two run concurrently
run = "go build ./..."        # runs only after both succeed
```

Running `mise run ci` executes `lint` and `test` at the same time, waits for both to finish, and only then runs `ci`'s own `run` command. There are related keys too — `depends_post` for cleanup steps that run afterwards, and `wait_for` for soft ordering without forcing a task to run. See [task dependencies](https://mise.jdx.dev/tasks/task-configuration.html) for the full list.

Some other highlights:

* **File watching** with `mise watch` to rerun a task when sources change.
* **File tasks** — instead of inlining shell into TOML strings, you can drop an executable script into a `mise-tasks/` directory and get proper syntax highlighting and linting:

  ```bash
  #!/usr/bin/env bash
  #MISE description="Build the CLI"
  go build ./...
  ```

Tasks also receive useful variables like `MISE_PROJECT_ROOT` and `MISE_TASK_NAME`, so scripts can be location-independent.

## A runnable example

Let's combine all three concepts into one small Go web project you can actually run. We'll also add [`golangci-lint`](https://golangci-lint.run/) as an additional tool to show how `mise` manages more than just language runtimes.

### 1. Create the project

```bash
mkdir mise-demo && cd mise-demo
```

### 2. Add the tools

Pin the exact Go version for this project. This creates `mise.toml` and installs Go if needed:

```bash
mise use go@1.26.4
```

Then add `golangci-lint` at a specific version. It lives in `mise`'s [registry](https://mise.jdx.dev/registry.html), so the same `mise use` workflow installs it:

```bash
mise use golangci-lint@2.12.2
```

### 3. Configure environment and tasks

Open the generated `mise.toml` and make it look like this:

```toml
# mise.toml
[settings]
lockfile = true

[tools]
go = "1.26.4"
golangci-lint = "2.12.2"

[env]
APP_NAME = "mise-demo"
APP_PORT = "8080"

[tasks.build]
description = "Compile the web server"
run = "go build -o bin/server ."

[tasks.lint]
description = "Run golangci-lint"
run = "golangci-lint run ./..."

[tasks.serve]
description = "Run the web server"
run = "go run ."

[tasks.ci]
description = "Lint and build (lint runs first)"
depends = ["lint"]
run = "go build -o bin/server ."
```

The `[tasks.ci]` task uses `depends` to guarantee linting happens before the build — a small example of the [task dependencies](#task-dependencies) covered above.

### 4. Lock the tool versions with `mise.lock`

We set `lockfile = true` above, but lockfiles are **not** generated automatically. Run `mise lock` (or just `mise install`) to create one:

```bash
mise lock
```

This writes a `mise.lock` file next to `mise.toml`, pinning the exact resolved versions, checksums, and per-platform download URLs (truncated here for brevity):

```toml
# mise.lock
# @generated - this file is auto-generated by `mise lock`

[[tools.go]]
version = "1.26.4"
backend = "core:go"

[tools.go."platforms.linux-x64"]
checksum = "sha256:1153d3d50e0ac764b447adfe05c2bcf08e889d42a02e0fe0259bd47f6733ad7f"
url = "https://dl.google.com/go/go1.26.4.linux-amd64.tar.gz"

[[tools.golangci-lint]]
version = "2.12.2"
backend = "aqua:golangci/golangci-lint"

[tools.golangci-lint."platforms.linux-x64"]
checksum = "sha256:8df580d2670fed8fa984aac0507099af8df275e665215f5c7a2ae3943893a553"
url = "https://github.com/golangci/golangci-lint/releases/download/v2.12.2/golangci-lint-2.12.2-linux-amd64.tar.gz"
provenance = "github-attestations"
```

Commit `mise.lock` alongside `mise.toml`. Now even loose specs (like `go = "1"`) resolve to the locked version on every machine, giving you reproducible installs. In CI you can go a step further with the [`locked`](https://mise.jdx.dev/configuration/settings.html#locked) setting (or `mise install --locked`) to fail fast if the lockfile is missing or incomplete.

### 5. Add the application code

Initialize the Go module:

```bash
go mod init mise-demo
```

Create `main.go`:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"runtime"
)

func main() {
	name := envOr("APP_NAME", "unknown")
	port := envOr("APP_PORT", "8080")

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if _, err := fmt.Fprintf(w, "Hello from %q, served by Go %s\n", name, runtime.Version()); err != nil {
			log.Printf("write response: %v", err)
		}
	})

	addr := ":" + port
	log.Printf("%s listening on http://localhost%s", name, addr)
	log.Fatal(http.ListenAndServe(addr, nil))
}

func envOr(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

Note that the handler checks the error returned by `fmt.Fprintf`. This isn't just good practice — `golangci-lint`'s default `errcheck` linter would otherwise flag the unchecked return value and the `ci` task would fail at the lint step.

### 6. Run it

```bash
# confirm the tools are active
mise exec -- go version
# go version go1.26.4 ...
mise exec -- golangci-lint version
# golangci-lint has version 2.12.2 ...

# list the tasks mise discovered
mise tasks
# build  Compile the web server
# ci     Lint and build (lint runs first)
# lint   Run golangci-lint
# serve  Run the web server

# lint then build in one shot (lint runs first via `depends`)
mise run ci

# start the server (uses APP_PORT from [env])
mise run serve
# mise-demo listening on http://localhost:8080
```

In another terminal:

```bash
curl localhost:8080
# Hello from "mise-demo", served by Go go1.26.4
```

That's the whole loop: `mise.toml` declares the **tools** (Go 1.26.4 plus `golangci-lint` 2.12.2), the **environment** (`APP_NAME`, `APP_PORT`), and the **tasks** (`build`, `lint`, `serve`, `ci`); `mise.lock` pins exactly what gets installed. Anyone who clones the repo and runs `mise install` followed by `mise run ci` gets the exact same setup — no README full of "first install the right Go version, then `go install golangci-lint`, then..." steps.

### 7. Clean up (optional)

```bash
cd .. && rm -rf mise-demo
```

## Bonus: global quality-of-life CLI tools

So far we've used `mise` for per-project toolchains, but it's just as handy for the CLI tools you want available *everywhere*. The `-g` (`--global`) flag installs a tool and records it in your global config at `~/.config/mise/config.toml` instead of a project `mise.toml`.

Here's a starter kit of quality-of-life tools, each pulled from `mise`'s [registry](https://mise.jdx.dev/registry.html) across different backends (core, aqua, npm, GitHub releases):

```bash
mise use -g lazygit@latest    # terminal UI for git
mise use -g fzf@latest        # fuzzy finder
mise use -g ripgrep@latest    # fast grep (rg)
mise use -g fd@latest         # fast, friendly find
mise use -g gh@latest         # GitHub CLI
mise use -g copilot@latest    # GitHub Copilot CLI
mise use -g neovim            # hyperextensible Vim-based editor
```

Each command updates the same global config:

```toml
# ~/.config/mise/config.toml
[tools]
lazygit = "latest"
fzf = "latest"
ripgrep = "latest"
fd = "latest"
gh = "latest"
copilot = "latest"
neovim = "latest"
```

Because this file lives in your dotfiles location, you can commit it to your dotfiles repo and reproduce your entire CLI toolbox on a new machine with a single command:

```bash
mise install
```

A couple of tips:

* Run `mise ls` to see everything `mise` manages (global and local) and which versions are active.
* Pin a tool to a real version instead of `latest` (e.g. `mise use -g fzf@0.56.3`) when you want reproducibility over always-newest.
* Upgrade everything later with `mise upgrade`.

## Why this matters

The payoff is reproducibility with almost no ceremony. A single committed `mise.toml` answers three questions at once:

* Which tool versions does this project use?
* Which environment variables does it expect?
* How do I build, test, and run it?

New contributors run `mise install` and they're ready — and because `mise.lock` pins exact versions, they get the same toolchain you do. CI runs the same `mise run` commands as your laptop. And you stop maintaining three separate tools to do one job.

## Useful links

* [mise — Getting Started](https://mise.jdx.dev/getting-started.html)
* [Dev Tools](https://mise.jdx.dev/dev-tools/)
* [Environments](https://mise.jdx.dev/environments/)
* [Tasks](https://mise.jdx.dev/tasks/)
* [Configuration reference](https://mise.jdx.dev/configuration.html)
* [Lockfiles (`lockfile` setting)](https://mise.jdx.dev/configuration/settings.html#lockfile)
* [Tool registry](https://mise.jdx.dev/registry.html)
* [Comparison to asdf](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)
