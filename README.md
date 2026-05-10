# claude-in — containerised Claude Code, ready for many projects

A wrapper that runs [Claude Code](https://github.com/anthropics/claude-code) in a single long-running Docker container, mounts multiple projects side-by-side for cross-project context via a hub `CLAUDE.md`, and adds opt-in git-worktree sandboxes where you can grant Claude full permissions without risking your main checkout. Designed to keep language runtimes off the host — PHP, Python, Java, whatever your projects use — the projects stay in their own docker-compose stacks and Claude reaches them through a mounted docker socket.

---

## Why

If you:

- juggle several related projects and want Claude to reason about them together (cross-project refactors, "how does service A talk to B?");
- prefer not to install a zoo of language runtimes on your host because each project already ships its own docker-compose stack;
- want to grant Claude broad permissions for larger tasks without trusting it with your working tree;
- work on Linux, macOS, or WSL2,

this gives you a reproducible environment for Claude Code with one entry point: `claude-in`.

---

## How it works (one paragraph)

A single long-running container, `claude-box`, based on Debian. It has Node.js + Claude Code, the Docker CLI (no daemon — the socket is mounted from the host), and a minimal set of tools (git, ripgrep, jq). It bind-mounts `~/workspace/` as `/workspace/`, your `~/.claude/` state directory (sessions, plugins, settings), and `/var/run/docker.sock`. Each project is mounted **twice**: once at `/workspace/projects/<name>/` for Claude's `CLAUDE.md` chain, once at `$HOST_HOME/<name>/` — a mirror whose path matches the host's path, so `docker compose` commands run inside the container hand real host paths to the daemon. A hub `CLAUDE.md` at the workspace root is auto-loaded as a parent of every project session, so Claude always knows the map of projects regardless of the current working directory.

---

## Supported platforms

| Platform                                     | Status        | Notes |
| -------------------------------------------- | ------------- | ----- |
| Linux with native Docker Engine 20.10+       | Supported     | `host.docker.internal` is wired via `extra_hosts: host-gateway` in `compose.yaml`. |
| WSL2 (Ubuntu/Debian) with Docker Desktop     | Primary       | Host networking (corp VPNs etc.) passes through via Docker Desktop's WSL integration. |
| macOS with Docker Desktop                    | Supported     | Same host-networking semantics as WSL2. |
| Windows native                               | Not supported | A Unix-like shell is required. Use WSL2 on Windows. |

The `claude-in` script detects BSD vs GNU `stat` and uses portable `date` formats, so the same file works across all three platforms.

---

## Prerequisites

- Docker Engine **20.10+** or Docker Desktop.
- Git **2.17+** (for `git worktree`).
- Bash **4.4+**.
- Either an active Claude subscription or `ANTHROPIC_API_KEY` exported in your shell.

---

## Install

Clone (or copy) this directory to `~/workspace/`:

```bash
git clone <repo> ~/workspace
cd ~/workspace
```

Create your per-user configuration from the templates:

```bash
cp CLAUDE.md.example CLAUDE.md                              # the hub — describe your projects
cp .mcp.json.example .mcp.json                              # MCP servers (optional)
cp .container/compose.local.yaml.example .container/compose.local.yaml
$EDITOR .container/compose.local.yaml                        # add bind-mounts for your projects
```

Put `claude-in` on your `PATH`. In `~/.bashrc` (or `~/.zshrc`):

```bash
export PATH="$HOME/workspace/.container/bin:$PATH"
```

Reload: `source ~/.bashrc`.

First launch:

```bash
claude-in
```

Docker builds the image (2–5 min on first run), starts `claude-box`, and drops you into Claude Code at `/workspace`. Subsequent launches are instant — the container stays up in the background (`restart: unless-stopped`).

---

## Daily commands

| Command                              | Effect |
| ------------------------------------ | ------ |
| `claude-in`                          | Enter `/workspace` (cross-project context), resume the last session. |
| `claude-in <name>`                   | Enter `projects/<name>` or `sandbox/<name>`. The wrapper figures out which. |
| `claude-in --new [<name>]`           | Start a fresh session instead of resuming. |
| `claude-in --shell [<name>]`         | Open bash inside the container (debugging mounts, GID, network). |
| `claude-in --update`                 | Rebuild the image with `--pull` and recreate the container. Picks up the latest Claude Code. |
| `claude-in --update --full`          | Same, but with `--no-cache --pull`. Picks up Debian base-image security updates. |
| `claude-in --down`                   | Stop and remove the container. |

### Sandbox commands

| Command                                          | Effect |
| ------------------------------------------------ | ------ |
| `claude-in --sandbox <project>`                  | Create a fresh sandbox from `~/<project>`, enter it with `bypassPermissions`. |
| `claude-in --sandbox <project> --continue`       | Same, but carry the most recent session from `projects/<project>`. |
| `claude-in --sandbox <project> --isolated-claude`| Same, but with an ephemeral `~/.claude/` scoped to this sandbox — host memory, plugins, skills, and login stay out of reach. Mutually exclusive with `--continue`. |
| `claude-in --sandbox-list`                       | List active sandboxes. |
| `claude-in --sandbox-prune`                      | Remove sandboxes whose branch has been merged into its base. |
| `claude-in --sandbox-remove <name> [--force]`    | Remove a specific sandbox. Worktrees with uncommitted changes need `--force`. |

### Autonomous commands

| Command                                          | Effect |
| ------------------------------------------------ | ------ |
| `claude-auto <project> <task-source ...> [--stand]` | Unattended dev → review loop. Task is composed from Jira ticket(s), Confluence page(s), inline prompt, or file. `--stand` adds a live docker compose stack the reviewer (and dev rounds 2+) can hit. Beta — see [Autonomous mode](#autonomous-mode-claude-auto). |
| `claude-auto --help`                             | Show all flags. |

---

## Adding a project

1. The project already lives somewhere under `$HOME` on the host, e.g. `~/my-api/`.
2. Add two lines to `~/workspace/.container/compose.local.yaml`:

   ```yaml
   services:
     claude:
       volumes:
         - ${HOME}/my-api:/workspace/projects/my-api
         - ${HOME}/my-api:${HOME}/my-api
   ```

   Two lines, not a duplicate. The first gives Claude context (the `CLAUDE.md` chain resolves from `/workspace/…`). The second is the **host-path mirror** — without it, `docker compose up` invoked inside the container won't find files referenced by relative bind-mounts in the project's own `docker-compose.yml`. Both are required.

3. Recreate the container:

   ```bash
   cd ~/workspace/.container && docker compose -f compose.yaml -f compose.local.yaml up -d
   ```

   Or `claude-in --update` — same thing plus an image rebuild.

4. If the project doesn't have its own `CLAUDE.md` yet, copy the template:

   ```bash
   cp -r ~/workspace/projects/_TEMPLATE/. ~/my-api/
   ```

   Fill it in for the project's stack — what it is, which commands build/test/run it, which services live where.

5. Add a line to `~/workspace/CLAUDE.md` under `## Projects` so every Claude session knows this project exists.

6. Run: `claude-in my-api`.

---

## Adding a read-only reference

Useful for external services you need to read for context but shouldn't modify.

1. In `compose.local.yaml`:

   ```yaml
   - ${HOME}/external-service:/workspace/references/external-service:ro
   - ${HOME}/external-service:${HOME}/external-service:ro
   ```

   `:ro` on both mounts — the kernel enforces read-only even if Claude tries to write.

2. Recreate the container.

3. Add a line to the hub `CLAUDE.md` under `## References`.

`claude-in` doesn't enter references directly (by design — you don't work in them). To read them, use `claude-in --shell` and `cd /workspace/references/<name>`, or run `claude-in` at the workspace root and ask Claude to read from there.

---

## Sandbox — full workflow

### When to use it

When you want to grant Claude wide permissions (large refactors, risky experiments, migrations) but don't want to trust it with your main working tree. Every change goes into a git branch in a separate worktree — you review before merging, or throw the whole branch away.

### Create

```bash
claude-in --sandbox my-api
```

What happens:

1. `git -C ~/my-api worktree add ~/workspace/sandbox/my-api-<YYMMDD-HHMM> -b claude/my-api-<YYMMDD-HHMM>`.
2. Claude starts with `--permission-mode bypassPermissions` — all prompts disabled.
3. `COMPOSE_PROJECT_NAME=<sandbox-name>` is injected, so any docker-compose stacks run with that prefix — they don't collide with the production copy's containers or named volumes.

If the project isn't a git repo, the wrapper falls back to `cp -r` (mode=copy). Review is then by hand (`diff -r`, `rsync`).

### Isolated-claude mode

Add `--isolated-claude` to sandbox creation:

```bash
claude-in --sandbox my-api --isolated-claude
```

The wrapper swaps `HOME` inside the container to an ephemeral directory scoped to this sandbox (`sandbox/.meta/<name>-home/`), so Claude Code reads and writes `.claude/` relative to that — your host `~/.claude/` is untouched.

Tradeoffs:

- **No host memory** — `MEMORY.md` and everything under `~/.claude/memory/` is unavailable.
- **No host plugins or skills** — fresh install, only what ships with Claude Code itself.
- **No host settings or hooks** — starts with defaults.
- **No OAuth login carry-over** — set `ANTHROPIC_API_KEY` in your shell so it flows through the env, or log in inside the isolated session.
- **Ephemeral sessions** — they live in `sandbox/.meta/<name>-home/.claude/` and vanish with `--sandbox-remove`.

`--isolated-claude` is mutually exclusive with `--continue` — the isolated HOME is empty, so there's no session to carry in. Resuming an isolated sandbox (`claude-in <sandbox-name>`) is auto-detected from its metadata — don't repeat the flag.

Fit: when a sandbox pulls in code you haven't audited (an external branch, a CI artifact). Doesn't eliminate the `/var/run/docker.sock` attack surface — see **Security** below.

### Work

Claude does whatever it needs inside the sandbox. To bring up the project stack:

```bash
(cd $HOST_HOME/workspace/sandbox/my-api-<stamp> && docker compose up -d)
```

`$HOST_HOME` is injected into the container's environment by `compose.yaml` — it holds the host user's home directory regardless of username.

### Re-enter an existing sandbox

Pass the full sandbox name:

```bash
claude-in my-api-260421-1530     # --continue is implicit
```

### Review and merge

```bash
cd ~/workspace/sandbox/my-api-260421-1530
git diff main                   # inspect

cd ~/my-api
git checkout main
git merge claude/my-api-260421-1530
```

Pick specific commits with `git cherry-pick <hash>`.

### Clean up

```bash
claude-in --sandbox-prune                      # bulk: remove all merged sandboxes
claude-in --sandbox-remove my-api-260421-1530  # targeted: remove a specific one
claude-in --sandbox-remove <name> --force      # override git's dirty-tree safety check
```

`--sandbox-prune` uses `git merge-base --is-ancestor` to detect merged branches, so **squash-merges aren't caught** — use `--sandbox-remove` for those.

### What isn't isolated

- **Host ports** — production and sandbox stacks that want the same port will conflict. Stop production first, or add a `docker-compose.override.yml` in the worktree with different ports.
- **Named volumes** — prefixed by `COMPOSE_PROJECT_NAME` (so they're separate), but they start empty. If you need production data, dump and restore manually.
- **External APIs** — real (Jira, Confluence, third-party). Sandbox doesn't proxy.
- **`~/.claude/` (default)** — shared across the whole workspace. `bypassPermissions` means Claude could in theory edit its own state there. Pass `--isolated-claude` at sandbox creation to swap in an ephemeral `.claude/` scoped to that sandbox.
- **`/workspace/CLAUDE.md`** (the hub) — outside the worktree; edits apply to the real hub.

---

## Security

Sandbox mode runs Claude with `--permission-mode bypassPermissions`. This disables all per-action prompts in Claude Code, including for Bash commands. The container's reach is bounded by its bind mounts, but you're explicitly opting into:

- Full write access to the sandbox worktree (intentional).
- Write access to `~/.claude/` (session state, plugins) via the mount.
- Full access to the host's Docker daemon via the mounted socket. Claude can `docker run`, `docker exec`, `docker volume rm` on any container, including `claude-box` itself, and equivalently run privileged containers. This is an acknowledged sharp edge of using the docker-socket pattern.

Mitigations built into the design:

- **Read-only mounts** (`:ro`) for external references — kernel-enforced write protection.
- **`COMPOSE_PROJECT_NAME` isolation** — sandbox stacks don't collide with production by default.
- **No SSH keys mounted** — local `git commit` works; `git push`/`pull` do not.
- **No ingress** — the container only exposes what Docker Desktop does (nothing by default, unless you add port mappings).
- **`--isolated-claude` for sandboxes** — ephemeral `HOME` swap keeps host `~/.claude/` (memory, plugins, login, settings) out of reach of the sandbox agent.

If you need stricter isolation, run separate `claude-box` instances per project, each with its own `container_name` and image name — the current design uses a single shared container.

---

## Autonomous mode (`claude-auto`)

Run a dev → review loop end-to-end with no human in the iteration. Built on top of the sandbox infrastructure: a fresh worktree, two headless Claude Code sessions, structured handoff via a `VERDICT: PASS|FAIL` trailer, up to 4 rounds. **Beta** — has been smoke-tested but not battle-hardened; treat the diff as untrusted and review before merging.

### When it fits

- Task is well-defined: a Jira ticket, a Confluence spec, a written brief, or any combination. Both agents fetch external sources via the jira MCP — the orchestrator stays MCP-free and just composes the prompt.
- You want to start the run, walk away, and come back to either a green merge candidate or a sandbox you can continue manually.
- Acceptance is testable from code (functional spec, contracts, expected output). Open-ended refactors and exploratory work fit the manual flow better.

### Run

```bash
claude-auto <project> [<jira-key>] [task-source ...] [--rounds N=3] [--name NAME]
```

Task sources are combinable; **at least one is required**:

| Source | Flag | Repeatable | What the agents do with it |
| ------ | ---- | ---------- | --------------------------- |
| Jira issue | `--jira KEY` | yes | fetch via `mcp__jira__jira_get_issue` |
| Confluence page | `--confluence PAGE_ID` | yes | fetch via `mcp__jira__confluence_get_page` |
| Inline brief | `--prompt "TEXT"` | no | included verbatim in the prompt |
| File brief | `--prompt-file PATH` (`-` = stdin) | no | included verbatim; mutually exclusive with `--prompt` |

Back-compat: the 2nd positional argument is still accepted as a Jira key, equivalent to `--jira <key>`.

Examples:

```bash
# Single Jira ticket (back-compat shorthand)
claude-auto my-api PROJ-1234

# Multiple Jira tickets + a Confluence spec
claude-auto my-api --jira PROJ-1 --jira PROJ-2 --confluence 671980720

# Free-form brief (no Jira at all)
claude-auto my-api --prompt "Add /healthz endpoint returning {status: ok}"

# Hybrid — ticket plus extra constraints
claude-auto my-api --jira PROJ-1234 --prompt "Reuse the existing helper instead of writing a new one."

# Long brief from a file or piped in
claude-auto my-api --prompt-file ./task.md
echo "Refactor X..." | claude-auto my-api --prompt-file -
```

What happens:

1. **Sandbox creation.** Fresh git worktree on branch `claude/<sandbox-name>` from the current branch of `~/<project>`. Same constraint as `claude-in --sandbox`: the project must be bind-mounted into `claude-box`.
2. **Round 1 — dev.** Headless `claude -p` runs in the sandbox cwd with `--permission-mode bypassPermissions`. The composed prompt has each task source as a labeled section (`== Jira ==`, `== Confluence ==`, `== Дополнительные инструкции ==`); the agent reads them all, implements the change, and commits locally.
3. **Round 1 — review.** A separate headless session runs in `/workspace/projects/<project>` (the project's main checkout). It receives the same source blocks plus the base branch reference, reads the diff via Bash, compares the implementation against every source, and ends its output with `VERDICT: PASS` or `VERDICT: FAIL` plus an actionable list.
4. On `PASS` — exit 0, sandbox left in place for human merge.
5. On `FAIL` — feedback feeds into dev's next round (resumed via `--resume <session-uuid>`), then review again. Loop repeats up to `--rounds`.

Both agents run **inside `claude-box`**, just in different cwds. They use distinct session IDs (`--session-id <uuid>` / `--resume <uuid>`) so neither collides with your normal interactive sessions in the same project dir.

### What you see in the terminal

The stream is line-buffered and human-readable:

```
[init session=<uuid> cwd=/workspace/sandbox/<name> mcp=jira]
  -> mcp__jira__jira_get_issue(issue_key=PROJ-1234)
     [ok]
  -> Bash(cmd=git diff develop..HEAD)
     [ok]
The change splits the unit-dependent reference book by …
…
VERDICT: PASS
[done: success cost=$1.85 duration_ms=276351]
```

`-> Tool(arg)` is a tool call; `[ok]` / `[tool error]` is the result. Model text passes through verbatim. The trailing `[done:]` reports total cost and wall time for that round.

### Output layout

```
~/workspace/sandbox/<name>/                            sandbox worktree
~/workspace/sandbox/.meta/<name>.env                   sandbox metadata (claude-in compatible)
~/workspace/sandbox/.meta/<name>-auto/
├── session-ids.env                                    project, sources, dev + reviewer UUIDs, timestamp
├── inline-prompt.txt                                  resolved --prompt / --prompt-file content (if any)
├── round-<N>-dev.prompt.txt                           prompt sent to dev (composed source blocks + steps)
├── round-<N>-dev.log                                  formatted human-readable stream
├── round-<N>-dev.jsonl                                raw stream-json events
├── round-<N>-review.prompt.txt
├── round-<N>-review.log
└── round-<N>-review.jsonl
```

`*.log` mirrors what you saw in the terminal. `*.jsonl` has full event-level detail (per-turn token usage, model, tool inputs/outputs) — useful for cost analysis and post-hoc debugging.

### After the run

- **PASS:** review the diff in `~/workspace/sandbox/<name>/` and `git merge claude/<name>` into the base branch when satisfied. Same workflow as for any sandbox.
- **FAIL after `--rounds`:** continue manually with `claude-in <name>` to enter the sandbox; the dev session is resumable, so Claude remembers everything from the auto run.

Cleanup goes through the normal sandbox path:

```bash
claude-in --sandbox-remove <name>          # targeted
claude-in --sandbox-prune                  # bulk: remove merged sandboxes
```

### Live review stand (`--stand`)

Add `--stand` to bring up the project's `docker-compose.yml` inside the sandbox, isolated from the production stack. The reviewer (and dev rounds 2+) get a port table in their prompt and can `curl` endpoints, `docker exec` into services, read logs — anything you'd do manually to behaviorally validate a change.

How it works:

1. **Before any rounds run**, claude-auto inspects the sandbox's `docker-compose.yml` via `docker compose config --format json` and generates a per-sandbox override file (`<auto-dir>/docker-compose.review-stand.yml`) that remaps every published port to `0` (Docker assigns a free host port). This avoids collisions with the production stack on the same host.
2. **After dev round 1**, the orchestrator runs `<auto-dir>/stand-up.sh`: `docker compose up -d` with `COMPOSE_PROJECT_NAME=<sandbox-name>`, discovers the assigned host ports via `docker compose port`, writes a port table to `<auto-dir>/stand-ports.txt`, and waits up to 60s for each port to open (bash `/dev/tcp/`). If a port doesn't open in time, the run aborts.
3. **Per-project post-up hook (optional)**: if `~/<project>/.container/claude-auto-post-up.sh` exists and is executable, it runs after the stack is healthy and before the reviewer starts. Use it for migrations, fixtures, sanitized snapshot restore — anything the project needs to be testable.
4. **Reviewer prompt** gets a `== Стенд ==` block with the port table and a hint about `docker compose -p <sandbox-name> exec <svc>`. Dev's round 2+ prompt gets the same block (so dev can verify fixes against running services).
5. **Tear-down via `trap EXIT`** — the stack is always brought down (`docker compose down -v`) on PASS, FAIL, error, or Ctrl-C.

Database notes — claude-auto **does not** add a DB-specific abstraction in v1. The project's `docker-compose.yml` decides:

- **Self-contained compose** (DB service in compose) → sandbox spins up its own DB container with an isolated volume (thanks to `COMPOSE_PROJECT_NAME`). Empty by default; load fixtures via the post-up hook if behavioral testing needs them.
- **External DB** (project compose connects to `host.docker.internal:5432` or similar) → sandbox shares the same external DB with production. **Write tasks may corrupt production data** — be mindful.

Caveats specific to `--stand`:

- Requires git project (worktree mode). Copy-mode sandboxes (non-git projects) are rejected.
- v1 looks for `./docker-compose.yml` (or `compose.yaml` / `compose.yml`) in the sandbox root only. Multi-file / monorepo projects need `--stand-compose-file PATH` (planned, not in v1).
- Port collisions: if the project's compose hardcodes ports the override successfully randomizes them, but **named volumes** are isolated by `COMPOSE_PROJECT_NAME` automatically — no per-sandbox volume conflict.
- Resource cost: a self-contained stack with mariadb + opensearch + redis adds ~30–60s of stand-up time and several GB of RAM. Skip `--stand` for refactors or tasks that don't need behavioral validation.
- **No DB profiles yet.** When the QA agent comes (separate role, separate prompt), expect a `--stand-profile <name>` abstraction with predefined dev/qa/snapshot lifecycle steps. v1's `--stand` is the implicit "dev" profile.

### Caveats

- **Cost.** Each round runs two long headless sessions. Real tasks observed at ~$5–10 per dev pass and ~$1–3 per reviewer pass on Opus. Watch the `[done: cost=...]` markers; cap with `--rounds 1` for cheap dry runs. `--stand` adds the cost of the running stack (host resources, not LLM tokens).
- **`bypassPermissions` for both agents.** Headless mode can't ask for per-tool permission interactively, so the default mode would block any MCP call (and a lot of useful Bash). Reviewer doesn't write much in practice, but it can — review the diff before merging. Tighter allowlists via `--allowedTools` are tracked as a follow-up.
- **Verdict discipline.** The reviewer is instructed to end with a strict trailer. If it forgets, `claude-auto` retries the review pass once with a stricter reminder, same session. If the trailer is still missing, the run aborts with a clear error rather than guessing.
- **Non-git projects.** Only git projects work — review needs `git diff <base>..<branch>`. For copy-mode projects, use the manual sandbox flow.
- **Project-specific runtimes.** Dev runs inside `claude-box`, which is deliberately thin (no PHP, Python, JDK baked in). For commands that need a project runtime, dev should use `docker compose exec <service> <cmd>` from `$HOST_HOME/<project>` — same convention as a normal sandbox session.

---

## Configuration files

```
workspace/
├── CLAUDE.md                        # hub — yours, gitignored
├── CLAUDE.md.example                # template
├── .mcp.json                        # project-scope MCP servers — gitignored
├── .mcp.json.example                # template
├── README.md
├── .gitignore
├── .container/
│   ├── Dockerfile                   # claude-box image
│   ├── compose.yaml                 # base (core mounts + sandbox + extra_hosts)
│   ├── compose.local.yaml           # your project mounts — gitignored
│   ├── compose.local.yaml.example   # template
│   └── bin/claude-in                # wrapper script
├── projects/
│   └── _TEMPLATE/                   # skeleton for new projects (tracked)
├── references/                      # bind-mount targets (gitignored)
├── sandbox/                         # sandbox worktrees (gitignored)
│   └── .meta/                       # per-sandbox metadata
└── shared/                          # cross-project notes
```

### How the two compose files fit together

`claude-in` automatically includes `compose.local.yaml` when it's present:

```bash
docker compose -f compose.yaml -f compose.local.yaml up -d
```

If it's missing, only the base runs — fine for a first-time install or reset. Compose's merge rules: list fields like `volumes` are appended across files; other fields are overridden by the later file. So `compose.local.yaml` only needs `services.claude.volumes:` with your mounts.

### Git identity

Your host's `~/.gitconfig` is bind-mounted read-only into the container, so `git commit` inside `claude-box` uses your real name, email, signing keys, and aliases. The mount is read-only — Claude cannot modify your global git config from inside. If you don't have a `~/.gitconfig` yet, set it up on the host before your first commit:

```bash
git config --global user.name 'Your Name'
git config --global user.email 'you@example.com'
```

`claude-in` creates an empty `~/.gitconfig` if one doesn't exist (to keep Docker from turning the missing file into a mount-point directory), so launching the container before you've configured git won't break anything — commits just need identity before they'll succeed.

SSH keys are deliberately **not** mounted. Local `git commit` works; `git push` / `git pull` / `git fetch` do not — remote operations stay on your host.

### Personal toolchain caches

By default, `claude-box` is deliberately thin: no PHP, Python, JDK, or other language runtimes baked in. The assumption is that each project runs in its own `docker-compose` stack and Claude reaches it via `docker compose exec`. In that workflow, `claude-box` doesn't need `~/.gradle`, `~/.m2`, `~/.cache/pip` and friends — those caches live in the project's containers.

If you diverge from that (e.g. you've extended `claude-box` to include a JDK and want to run Gradle directly inside it), `compose.local.yaml.example` ships a commented section listing the common caches by stack (Java/Kotlin, Python, Node, Go, Rust, PHP). Uncomment the ones you need — they're `${HOME}/.gradle:/home/dev/.gradle` style mounts (host-side cache → container user's home) so tools inside find them at the normal `~/.gradle` location.

### Claude Code settings

Model, effort level, plugins, permission modes — they live in `~/.claude/settings.json`, which is mounted into the container. Settings are shared between host and container sessions. A typical setup:

```json
{
  "model": "claude-opus-4-7[1m]",
  "effortLevel": "max",
  "alwaysThinkingEnabled": true,
  "skipDangerousModePermissionPrompt": true
}
```

The last one removes the startup warning in `bypassPermissions` mode — relevant if you use sandboxes.

---

## Updates

Nothing updates automatically. The image is frozen until you rebuild.

- **Claude Code**: `claude-in --update` rebuilds with `npm install -g @anthropic-ai/claude-code` (unpinned) — pulls the current release.
- **Debian + base image**: `claude-in --update --full` (`--no-cache --pull`).
- **Your `~/.claude/` state, plugins, settings**: not in the image, live in the mount — updates apply immediately.

Rule of thumb: `--update` weekly, `--update --full` monthly.

---

## Troubleshooting

**`claude-in: command not found`**
`~/workspace/.container/bin` isn't on your `PATH`. Add to `~/.bashrc` and `source` it.

**`permission denied` on `/var/run/docker.sock` inside the container**
The host's `docker` group GID doesn't match what `group_add` passed in. The wrapper auto-detects it, but if something went wrong:
```bash
stat -c '%g' /var/run/docker.sock   # Linux / WSL2
stat -f '%g' /var/run/docker.sock   # macOS
```
`claude-in --down && claude-in` recreates with the correct GID.

**`not a directory: Are you trying to mount a directory onto a file`**
A compose lifecycle command was invoked from the wrong cwd. Inside the container:
- ❌ `cd /workspace/projects/my-api && docker compose up` — the daemon doesn't know this path.
- ✅ `cd $HOST_HOME/my-api && docker compose up` — the host-path mirror.

**Sandbox can't bind to its port**
Production stack already owns it. Stop production, or add a `docker-compose.override.yml` in the sandbox with different ports.

**VPN / corporate network not reachable**
Check from inside:
```bash
docker exec claude-box curl -s ifconfig.me
```
Containers inherit routing from the host. On Docker Desktop (Mac / WSL2), that's Windows/macOS networking, which usually includes your VPN. On Linux native, the host is the machine itself. If the container can't reach something, neither can the host — fix it on the host.

**MCP server on `localhost:<port>` doesn't respond from inside**
`localhost` inside the container is the container itself. Use `host.docker.internal:<port>` — it's wired via `extra_hosts` in `compose.yaml`.

**Updated the image but `claude` still runs the old version**
You need to recreate the container (`--force-recreate`), which `claude-in --update` does automatically.

---

## Known limitations

- **No `git push`/`pull` from inside** — intentional. SSH keys aren't mounted. Claude can `git commit` locally; pushing is on you.
- **`~/.claude/` is shared by default** with host Claude sessions — convenient (resume on either side) but in `bypassPermissions` mode nothing stops Claude from writing to it. Use `--isolated-claude` on sandbox creation if you want that mount out of the way for a specific session.
- **Auto-memory slug per cwd.** Each sandbox has its own memory slug — memory collected there vanishes with `--sandbox-prune`. Move the relevant files manually before pruning if you care.
- **One `claude-box` per host.** Running parallel instances with different environments would need different `container_name` and image names in compose.yaml; not supported out of the box.

---

## Design notes

**Why not a monorepo.** Projects stay where they are on the host (`~/<name>`), and `workspace/projects/<name>/` is a bind-mount target. Claude Code can't tell the difference, but your IDE, git clones, deploy scripts, and CI all keep their original paths.

**Why two compose files instead of generated config.** `compose.local.yaml` is hand-edited — three lines per project instead of a DSL that someone must define, parse, and generate YAML from. Simpler, transparent, standard tooling.

**Why dual-mount instead of symlinks or `--project-directory` on every compose call.** Symlinks break the `CLAUDE.md` chain because Node's `process.cwd()` returns the canonical path, not the symlink. Threading `--project-directory` through every command is verbose, and Claude forgets it. Dual-mount is invisible and always correct.

**Why git worktrees for sandbox.** Branches are reviewed with `git diff`, merged with `git merge`, discarded with `git worktree remove`. A tool you already know, consistent with the rest of the git workflow. Docker volumes or overlayfs would need custom review tooling.

---

## License

MIT — see [LICENSE](LICENSE).

Not affiliated with Anthropic. "Claude" is a trademark of Anthropic, used here nominatively to identify the product this tool wraps.
