# omakase: How I build things

Author: Theodor Vararu

Last updated: 01/04/2026

This is a curated set of recipes I use across multiple projects.

It's not really a SKILL.md, not really a template. It's stuff I've had to ask
Claude to "hey can you copy how we do FooThing in ../project."

It's not broken up into multiple files, because it's designed to be copy and
pasted as one chunk, whenever I need to align my projects to the same
conventions, or when I'm scaffolding a new one.

## rust + bun

I use this simple heuristic to decide what to use for a new project:

1. `rust` if it requires knowing what a pointer is

2. `bun` for everything else

Some projects use both.

## mise

All my projects use **mise**. The
[/mise](https://github.com/tvararu/.dotfiles/blob/master/claude/.claude/skills/mise/SKILL.md)
Claude skill in my dotfiles has some more details about the exact setup.

### mise: Tools

```toml
[tools]
"cargo:cargo-llvm-cov" = "latest"
"npm:@biomejs/biome" = "latest"
blender = "latest"
bun = "latest"
godot = "latest"
hk = "latest"
jq = "latest"
pkl = "latest"
rust = "latest"

[settings]
npm.package_manager = "bun"
```

If I need a tool to do something in a project, I pull it in using `mise`. I
avoid `brew` or `apt` as it's more portable this way.

I don't pull in linters/formatters/miscellaneous tools through `package.json` if
I can avoid it. This reduces dependabot noise.

`npm.package_manager = "bun"` to prevent accidentally mixing `node`.

### mise: Tasks

```toml
[tasks."rs:test"]
description = "Run Rust tests"
run = "cargo test --workspace"

[tasks."ts:test"]
description = "Run TypeScript tests"
raw = true
run = "bun test"

[tasks.test]
description = "Run all tests"
run = "mise rs:test ::: ts:test"
```

I wrap all common scripts with `mise` convenience wrappers. This way I don't
have to think about `package.json` scripts, Rake tasks, or anything else. `mise
build` and `mise dev` are muscle memory that's shared across all my projects.

Tasks can be namespaced with colons.

Complex tasks (more than 3 lines) go into `.mise/tasks`.

### mise: Environment and Secrets

```toml
[tools]
age = "latest"

[settings]
age.key_file = "config/secret.key"

[env]
ADMIN_PASSWORD = { age = "abcdefghijklmnopqrstuvwxyzsecretthing" }
API_KEY = { age = "abcdefghijklmnopqrstuvwxyzsecretthing" }
SOME_SETTING = "false"
```

`mise` replaces `dotenv`.

I use `age` to encrypt secrets which I then safely commit to version control. I
use this for both local development and production secrets, and create different
`mise.production.toml` and `config/secrets/production.key`s as necessary.

If `config/secret.key` isn't available, `mise.local.toml` can disable it:

```toml
[settings]
age.strict = false
disable_tools = ["age"]
```

Or you can use env vars:

```sh
MISE_AGE_STRICT=false MISE_DISABLE_TOOLS=age mise build
```

## rust

I use a `crates/` folder with `{project}-{name}` for packages.

I use defaults for `fmt` and `clippy`.

### rust: Coverage

```toml
[tools]
"cargo:cargo-llvm-cov" = "latest"
jq = "latest"

[tasks."rs:test:coverage"]
description = "Run Rust tests with coverage summary"
run = "cargo llvm-cov --workspace --json | .mise/tasks/coverage-fmt"
```

`.mise/tasks/coverage-fmt`:

```sh
#!/bin/sh
# Reads cargo-llvm-cov JSON from stdin and prints a coverage table.
jq -r '
  .data[0] as $d |
  ($d.files[0].filename | split("/crates/")[0] + "/crates/") as $prefix |
  ($d.files | map(.filename | ltrimstr($prefix) | length) | max + 3) as $w |
  ($d.files | map("\(.summary.lines.covered)/\(.summary.lines.count)" | length) | max + 3) as $lw |
  def pad(n): . + ((n - length) * " ");
  ("-" * $w + "|" + "-" * $lw + "|" + "---------"),
  (" File" | pad($w)) + "|" + (" Lines" | pad($lw)) + "| % Lines",
  ("-" * $w + "|" + "-" * $lw + "|" + "---------"),
  (" All files" | pad($w)) + "|" + (" \($d.totals.lines.covered)/\($d.totals.lines.count)" | pad($lw)) + "| \($d.totals.lines.percent * 100 | round | . / 100)",
  ($d.files[] |
    (.filename | ltrimstr($prefix)) as $name |
    ("  " + $name | pad($w)) + "|" + (" \(.summary.lines.covered)/\(.summary.lines.count)" | pad($lw)) + "| \(.summary.lines.percent * 100 | round | . / 100)"
  ),
  ("-" * $w + "|" + "-" * $lw + "|" + "---------")
'
```

## bun

I like `bun`. It's fast and comes with most things I need, so I don't need to
install many deps.

### bun: No dependencies if possible

```json
{
  "name": "project",
  "module": "index.ts",
  "type": "module",
  "private": true,
  "devDependencies": {
    "@types/bun": "latest"
  },
  "peerDependencies": {
    "typescript": "^6"
  }
}
```

This is the default `package.json` that `bun init -y` generates.

I will go to great lengths to not add anything to this file. No dependencies.
This is in stark contrast to 99% of TypeScript projects.

Bun comes with a lot of things, and it's easy to forget and end up importing
`@aws-sdk/client-s3` for no reason. A quick overview of what's built-in:

- HTTP server
- Testing
- JSX
- TypeScript
- fetch
- SQLite
- S3
- Redis
- JSONL
- YAML
- Markdown
- TOML

I will pull in a dependency if it's solving a big problem, like `react`, or
`transformers.js`.

### bun: tsconfig.json

```json
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleDetection": "force",

    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true,
    "noEmit": true,

    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    "noPropertyAccessFromIndexSignature": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    "incremental": true,
    "tsBuildInfoFile": "./tmp/.tsbuildinfo"
  }
}
```

This is the default `tsconfig.json` that comes with `bun init -y` with the
following changes:

- Stricter flags flipped to true
- Incremental compilation enabled for a speed-up
- React removed

### bun: Biome

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.9/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "formatter": { "indentStyle": "space" },
  "linter": {
    "enabled": true,
    "rules": { "complexity": { "useLiteralKeys": "off" } }
  },
  "files": { "includes": ["src/**"] },
  "plugins": ["./config/biome.grit"]
}
```

Default formatter settings except for spaces to indent as it's the `prettier`
default.

Default linter settings except for `useLiteralKeys` set to `"off"` because
it conflicts with tsconfig's `noPropertyAccessFromIndexSignature`.

`config/biome.grit`:

```grit
`mock.module($args)` as $call where {
    register_diagnostic(
        span = $call,
        message = "mock.module() is banned — use dependency injection",
        severity = "error"
    )
}
```

`mock.module()` leaks across test files, so I ban it using a grit rule.

### bun: Testing

I use colocated tests, `foo.ts` -> `foo.test.ts`.

```toml
[tasks."ts:test:coverage"]
description = "Run TypeScript tests with coverage"
raw = true
run = "bun test --coverage"

[tasks."ts:test:slowest"]
description = "Show 10 slowest TypeScript tests"
raw = true
run = '''
bun test --reporter=junit \
  --reporter-outfile=tmp/junit.xml && \
sed -n 's/.*<testcase name="\([^"]*\)".* time="\([^"]*\)".*/\2\t\1/p' \
  tmp/junit.xml | sort -rn | head -10 \
  | awk -F'\t' '{printf "%7.1fms  %s\n",$1*1000,$2}'
'''
```

I often measure what the top 10 slowest tests are. Over 10ms is usually a smell.

I always use `raw = true` for `bun test` as it improves output formatting.

## claude

I use `claude` as my main coding harness.

### claude: AGENTS.md

```sh
ln -s CLAUDE.md AGENTS.md
```

Ensures other harnesses auto-read the CLAUDE.md conventions.

### claude: Session start hook

`.claude/hooks/session-start.sh`:

```bash
#!/bin/bash
set -euo pipefail

[ "${CLAUDE_CODE_REMOTE:-}" = "true" ] || exit 0

if ! command -v mise &>/dev/null; then
  curl -fsSL https://mise.run | sh
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
  export PATH="$HOME/.local/bin:$PATH"
fi

mise trust --yes
mise install
```

`.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

Claude Code on web gets confused about how to set up `mise` properly.

### claude: Rust LSP

```sh
claude plugin install rust-analyzer-lsp@claude-plugins-official --scope project
```

### claude: Auto-format Rust on save

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "fp=$(jq -r '.tool_input.file_path'); [[ \"$fp\" == *.rs ]] && rustfmt \"$fp\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### claude: TypeScript LSP

```sh
claude plugin install typescript-lsp@claude-plugins-official --scope project
```

```toml
[tools]
"npm:typescript-language-server" = "latest"
```

The `typescript-language-server` is required in the path.

### claude: Auto-format TypeScript on save

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "fp=$(jq -r '.tool_input.file_path'); [[ \"$fp\" == *.ts ]] && biome format --write \"$fp\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### claude: Prevent accidental commits to main

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.command' | grep -q 'git commit' && [ \"$(git branch --show-current)\" = \"main\" ] && echo 'BLOCK: Do not commit to main. Create a feature branch first.' && exit 2 || exit 0"
          }
        ]
      }
    ]
  }
}
```

### claude: Prevent reading config/secret.key

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|Read",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | grep -q 'secret\\.key' && echo 'BLOCK: Do not read or edit secret files' && exit 2 || exit 0"
          }
        ]
      }
    ]
  }
}
```

### claude: Other plugins

```sh
claude plugin install superpowers@claude-plugins-official --scope project
claude plugin install playwright@claude-plugins-official --scope project

claude plugin install claude-code-setup@claude-plugins-official --scope project
claude plugin install claude-md-management@claude-plugins-official --scope project
claude plugin install feature-dev@claude-plugins-official --scope project
```

Whether I include these depends on the project.

### claude: Permissions

`.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh issue list:*)",
      "Bash(gh issue view:*)",
      "Bash(gh pr checks:*)",
      "Bash(gh pr create:*)",
      "Bash(gh pr diff:*)",
      "Bash(gh pr edit:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr view:*)",
      "Bash(gh release:*)",
      "Bash(gh repo view:*)",
      "Bash(gh run list:*)",
      "Bash(gh run view:*)",
      "Bash(gh search:*)",
      "Bash(git add:*)",
      "Bash(git check-ignore:*)",
      "Bash(git commit:*)",
      "Bash(git fetch:*)",
      "Bash(git log:*)",
      "Bash(git push)",
      "Bash(git rev-parse:*)",
      "Bash(mise build:*)",
      "Bash(mise bundle:*)",
      "Bash(mise ci:*)",
      "Bash(mise deploy:*)",
      "Bash(mise dev:*)",
      "Bash(mise format:*)",
      "Bash(mise help:*)",
      "Bash(mise install:*)",
      "Bash(mise lint:*)",
      "Bash(mise main:*)",
      "Bash(mise tasks:*)",
      "Bash(mise test:*)",
      "Skill(claude-md-management:revise-claude-md)",
      "WebSearch"
    ],
    "deny": [
      "Bash(env)",
      "Bash(env :*)",
      "Bash(export)",
      "Bash(export :*)",
      "Bash(mise env)",
      "Bash(mise env :*)",
      "Bash(printenv:*)",
      "Bash(set)"
    ]
  }
}
```

This is a starter-pack allowing usually non-destructive actions, to pre-empt
approval fatigue.

### claude: Closing loops with screenshots and video recordings

```toml
[tasks."macos:screenshot"]
description = "Take a screenshot of all displays"
run = '''
ts=$(date +%Y%m%d-%H%M%S)
screencapture -x "tmp/screen1-$ts.png" "tmp/screen2-$ts.png" "tmp/screen3-$ts.png"
echo "tmp/screen1-$ts.png tmp/screen2-$ts.png tmp/screen3-$ts.png"
'''

[tasks."linux:screenshot"]
description = "Take a screenshot"
run = '''
f="tmp/screenshot-$(date +%Y%m%d-%H%M%S).png"
grim "$f"
echo "$f"
'''

[tasks."linux:record"]
description = "Record screen for N seconds (default 5)"
usage = 'arg "[seconds]" default="5"'
run = '''
f="tmp/recording-$(date +%Y%m%d-%H%M%S).mp4"
timeout "$usage_seconds" wf-recorder -f "$f" -y 2>/dev/null || true
echo "$f"
'''
```

These commands give Claude tools to take screenshots and visually inspect
complex GUI apps.

## git

I add this to `CLAUDE.md`:

```md
## Commits

Use Conventional Commits, title is "what", body is "why":

- Check `git log -n 5` first to match existing style
- Don't use `--oneline`, commit bodies carry important context
- Subject ≤50 chars (including prefix): `feat: Add thing`
- Capitalize after prefix: `feat: Add thing` not `feat: add thing`
- Blank line, then 1-3 sentence description of "why", no bullet points
- Always `git add` and `git commit` as separate commands

## PRs

- Short essay (a few paragraphs) describing why the changes are needed
- Don't hard-wrap PR body, GitHub renders markdown with browser reflow
```

### git: Commit hooks

```toml
[tools]
hk = "latest"
pkl = "latest"
```

`hk.pkl`:

```pkl
amends "package://github.com/jdx/hk/releases/download/v1.38.0/hk@1.38.0#/Config.pkl"
import "package://github.com/jdx/hk/releases/download/v1.38.0/hk@1.38.0#/Builtins.pkl"

hooks {
  ["commit-msg"] {
    fix = true
    steps {
      ["wrap-body"] {
        stage = List()
        fix = #"""
          file="{{commit_msg_file}}"
          subject=$(head -n 1 "$file")
          body=$(tail -n +3 "$file" | grep -v '^#' || true)
          if [ -z "$body" ]; then
            exit 0
          fi
          wrapped=$(echo "$body" | fmt -w 72)
          printf '%s\n\n%s\n' "$subject" "$wrapped" > "$file"
          """#
      }
      ["conventional-commit"] = (Builtins.check_conventional_commit) {
        depends = List("wrap-body")
      }
      ["subject-length"] {
        depends = List("wrap-body")
        check = #"""
          subject=$(head -n 1 "{{commit_msg_file}}")
          len=$(printf '%s' "$subject" | wc -c | tr -d ' ')
          if [ "$len" -gt 50 ]; then
            echo "Subject is $len chars (max 50): $subject" >&2
            exit 1
          fi
          """#
      }
    }
  }
}
```

```sh
hk install
```

This sets up `hk` to check that commits follow the CLAUDE.md guidelines.

### git: gitignore template

```gitignore
.claude/*.local.*
.playwright-mcp
.worktrees
config/secret.key
coverage
mise.local.toml
tmp/*
!tmp/.keep
```

### git: Worktree and feature branch management

```toml
[tasks.main]
description = "Switch to main, pull, and clean up merged branches"
run = '''
set -e
git checkout main
git pull --rebase
git fetch --prune

cleaned=0
for branch in $(git branch -vv | grep ': gone]' | sed 's/^[*+] /  /' | awk '{print $1}'); do
  git branch -D "$branch"
  cleaned=$((cleaned + 1))
done

if [ "$cleaned" -gt 0 ]; then
  echo "Cleaned up ${cleaned} merged branch(es)"
else
  echo "No stale branches to clean up"
fi
'''

[tasks.worktree]
description = "Create a worktree for feature work"
usage = 'arg "<branch>"'
run = '''
set -e
ROOT="$(dirname "$(git rev-parse --path-format=absolute --git-common-dir)")"
DIR="$ROOT/.worktrees/$usage_branch"
[ -d "$DIR" ] && echo "Already exists: $DIR" >&2 && exit 1
git worktree add "$DIR" -b "$usage_branch" main
cd "$DIR"
[ -f "$ROOT/config/secret.key" ] && mkdir -p config && cp "$ROOT/config/secret.key" config/
mise trust
mise install
echo ""
echo "Worktree ready: $DIR"
'''

[tasks."worktree:clean"]
description = "Remove a worktree"
usage = 'arg "<branch>"'
run = """
git worktree remove --force .worktrees/$usage_branch
git branch -D $usage_branch
"""
```

I run `mise main` after merging a piece of work.

`mise worktree` handles setting up a worktree and copying in the encryption key.

## ops

### ops: kamal

```toml
[tools]
"gem:kamal" = "latest"
```

Example single-server deploy with local docker registry and a `pgvector`
database:

`config/deploy.yml`:

```yaml
service: foo
servers:
  web:
    - foo.com
proxy:
  ssl: true
  host: foo.com
registry:
  server: "localhost:5555"
env:
  secret:
    - RAILS_MASTER_KEY
  clear:
    WEB_CONCURRENCY: 2
builder:
  arch: amd64
accessories:
  pg18:
    image: pgvector/pgvector:pg18
    host: foo.com
    env:
      clear:
        POSTGRES_USER: foo
      secret:
        - POSTGRES_PASSWORD
    directories:
      - pg18:/var/lib/postgresql/18/docker
```

`.kamal/secrets` daisy-chains encrypted secrets from `mise`, which is why it's
important to run `kamal` commands via `mise` wrappers:

```sh
RAILS_MASTER_KEY=$RAILS_MASTER_KEY
POSTGRES_PASSWORD=$POSTGRES_PASSWORD
```

### ops: Mise tasks

```toml
[tasks.deploy]
description = "Deploy via Kamal"
raw = true
run = "kamal deploy"

[tasks."ops:setup"]
description = "First-time Kamal setup"
raw = true
run = "kamal setup"

[tasks."ops:logs"]
description = "Tail logs from deployment"
raw = true
usage = '''
flag "-f --follow"
arg "[lines]" default="50"
'''
run = 'kamal app logs --lines $usage_lines ${usage_follow:+-f}'

[tasks."ops:console"]
description = "Open a console in the container"
raw = true
run = "kamal app exec --interactive --reuse 'bash'"

[tasks."ops:restart"]
description = "Restart the app container"
raw = true
run = "kamal app boot"
```

### ops: Grouped dependabot updates

`.github/dependabot.yml`:

```yml
version: 2
updates:
  - package-ecosystem: <bun|cargo|github-actions|docker|bundler> # Pick one
    directory: /
    schedule:
      interval: weekly
      day: monday
    commit-message:
      prefix: "chore(deps):"
    ignore:
      - dependency-name: "*"
        update-types:
          - version-update:semver-major
    groups:
      minor-and-patch:
        update-types:
          - minor
          - patch
```

This ensures dependabot groups updates into one PR instead of many.

### ops: Local CI with signoff

```toml
[tools]
gh = "latest"

[tasks.ci]
description = "Run all CI checks in parallel"
depends = ["typecheck", "test", "format", "lint"]
run = "gh signoff ci 2>/dev/null || true"
```

```sh
gh extension install basecamp/gh-signoff
gh signoff install
```

This runs CI locally and signs off on GitHub using
[basecamp/gh-signoff](https://github.com/basecamp/gh-signoff).

### ops: Local database with docker compose

`docker-compose.yml`:

```yaml
services:
  pg18:
    image: pgvector/pgvector:pg18
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: myapp
    volumes:
      - ./storage/pg18:/var/lib/postgresql/18/docker
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "myapp"]
      interval: 100ms
      timeout: 500ms
      retries: 30
```

I prefer running things like Postgres or Redis via docker containers instead of
`mise` tools.

```toml
[tasks.dev]
description = "Start development server"
depends = ["pg:up"]
run = "bin/setup"

[tasks."pg:up"]
description = "Start PostgreSQL if not running"
run = "docker compose up -d pg18 --wait"

[tasks."pg:down"]
description = "Stop PostgreSQL"
run = "docker compose down"

[tasks."pg:logs"]
description = "Show PostgreSQL logs"
run = "docker compose logs -f pg18"

[tasks."pg:psql"]
description = "Open PostgreSQL console"
run = "docker compose exec pg18 psql -U myapp -d myapp_development"

[tasks."pg:dump"]
description = "Dump main development database"
run = '''
docker compose exec pg18 pg_dump -U myapp -Fc myapp_development > backup.dump
'''

[tasks."pg:restore"]
description = "Restore local database from backup file"
usage = 'arg "<backup_file>"'
run = '''
docker compose exec -T pg18 pg_restore -U myapp -d myapp_development \
  --clean --if-exists < {{arg(name="backup_file")}}
'''
```
