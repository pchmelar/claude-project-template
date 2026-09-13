# TEMPLATE.md

**Template version: 6** · changelog at the bottom · projects record theirs in `CLAUDE.md`.
Source: <https://github.com/pchmelar/claude-project-template> (public).

Starter for projects where Claude Code works **autonomously inside one directory**
(long sessions, big TODO lists, multiple agents) and the human only reviews
results — **human-on-the-loop**, not human-in-the-loop. Core idea: **maximum
autonomy, guarded by auto mode's classifier and narrowly scoped credentials**
rather than an OS sandbox.

## Spin up a new project

Human part (~5 minutes):

1. Create a repo from this template (GitHub → "Use this template") and clone it.
2. Mint a **fine-grained PAT** at
   <https://github.com/settings/personal-access-tokens> — Only select
   repositories → this repo, Contents: Read and write, expiration at your
   discretion. Then `cp .env.example .env` and put it in as `GITHUB_TOKEN`
   (edit the file directly; never paste tokens into the conversation).
3. `claude` in the project directory, and say: **"Bootstrap this project per TEMPLATE.md."**

Claude's bootstrap (first session):

4. Ask the user for the project name and a short description.
5. Replace `{{PROJECT_NAME}}` (`README.md`, `CLAUDE.md`, `devbox.json`) and
   `{{UID}}` in `.claude/settings.json` with the output of `id -u` — the
   settings edit may need the user's approval (`.claude/` is a protected
   path). Settings reload live; no restart needed.
6. Check that `origin` is an HTTPS URL — pushes go through the repo-scoped
   PAT, never the personal SSH key. If it's `git@`, switch it with
   `git remote set-url origin https://…`.
7. Fill in `CLAUDE.md`, run checks (devbox toolchain present in Bash —
   `DEVBOX_SHELL_ENABLED=1`; `Read` of `.env` denied; git clean of warnings),
   then commit and push as the end-to-end verification.

Daily after that: just `claude` in the project directory — a SessionStart hook
loads the devbox toolchain and `.env` into every Bash command, so `devbox shell`
first is optional. See [Devbox](#devbox).

## Overlay mode

For working on an existing repo you don't fully own — same autonomy and
guardrails, but nothing template-related ever enters that repo. Create
`<xyz>-wrapper` from this template (private), put two PATs in `.env`
(wrapper + target repo), then `claude` and say:
**"Bootstrap an overlay over \<target repo URL\> into `repo/` per
TEMPLATE.md."** As part of the bootstrap, Claude asks how you want to work
with git in the nested repo (branching, PRs, what never to push).

## The protection model

**No OS sandbox — auto mode is the gate.** A classifier model reviews every
shell command and network action before it runs and blocks what goes beyond
the task, destroys things, or leaks data (`curl | bash`, force pushes, printing
tokens, sensitive files leaving the machine, …). Reads and in-project edits
skip it. Boundaries you state in the conversation ("don't push yet") are
enforced by the classifier too.

Why no sandbox: devbox/nix (daemon socket, binary caches, shell env) kept
colliding with it, and the constant exceptions cost more than the containment
bought. The accepted trade-off: commands *can* read the home directory, so the
worst case of a fooled classifier is bounded by what credentials are reachable.
The discipline that keeps that small: **tokens are always fine-grained, one
per repo; git never uses the personal SSH key.**

`.claude/settings.json` adds a few hard rules on top of the classifier:

- `allow: WebSearch` — read-only, no reason to review it.
- `allow: Read(//private/tmp/claude-{{UID}}/**)`, `Read(//tmp/claude-{{UID}}/**)`
  — Claude Code's own temp root (background task output, scratchpad); without
  it the first read outside the project prompts.
- `deny: Read(//**/.env)` — secrets never enter the transcript through `Read`;
  commands get them from the environment instead.
- `ask: git push --force / -f` (all argument orders) — rewriting remote history
  always gets a human prompt, whatever the classifier thinks.

## Git

- Claude commits at natural milestones and pushes over HTTPS with the token
  from `.env` via an inline credential helper (token never in argv/logs).
  `GITHUB_TOKEN` is already in the Bash environment (hook → devbox `init_hook`):

```bash
git -c credential.helper= \
  -c credential.helper='!f(){ echo "username=<gh-user>"; echo "password=${GITHUB_TOKEN}"; };f' \
  push origin <branch>
```

- No SSH: the personal key can touch every repo; the PAT only this one.

## Devbox

- Tools come from `devbox add <pkg>[@version]`, never `brew` and never binaries
  fetched into the project. Claude runs `devbox add` itself. `devbox.json` +
  `devbox.lock` are committed, `.devbox/` is not.

- The SessionStart hook appends
  `eval "$(devbox shellenv --init-hook --config <project>)"` and
  `export DEVBOX_SHELL_ENABLED=1` to `$CLAUDE_ENV_FILE`, which Claude Code
  sources before each Bash command. Result: toolchain + `.env` (loaded by
  `init_hook`) are present however `claude` was launched, and packages added
  mid-session are available on the next command.

- Keep `init_hook`'s `DEVELOPER_DIR=/Library/Developer/CommandLineTools`: nix
  packages otherwise hijack the Apple toolchain pointer and `/usr/bin/git`
  fails with `error: tool 'git' not found`.

## Conventions

- **Language:** conversation in any language; every artifact (docs, code,
  comments, test names, commits, UI strings) in English unless specified otherwise.
- **Autonomy:** proceed without asking while inside the project; verify with
  tests/builds before reporting done; report honestly. Durable project rules →
  `CLAUDE.md`; secrets → `.env`.
- **Declarations in git, artifacts out of git** (devbox.json + lockfiles
  committed; binaries, caches, `.env` ignored).

## Updating a project to a newer template version

In the project, tell Claude: **"Check for template updates."** Claude `WebFetch`es
`https://raw.githubusercontent.com/pchmelar/claude-project-template/main/TEMPLATE.md`
(public, no token needed) and applies the changelog entries above the project's
recorded version — they're written as migration notes. Never git-merge from the
template; projects diverge.

---

## Changelog

### v6 (2026-09-14)

**Sandbox removed, auto mode instead**; the SessionStart hook now loads the
devbox env instead of warning. Drops the v3 and v5 rules. Migration: take
`.claude/settings.json` from the template (keep project additions, re-fill
`{{UID}}`) and copy all sections except Conventions.

### v5 (2026-09-05)

Web reads must use **`WebFetch`/`WebSearch`, not `curl`/`wget`** — Bash egress
is capped by the domain allowlist, so shell fetching floods you with permission
prompts. Migration: copy the new bullet into the protection-model section; no
config changes.

### v4 (2026-08-07)

Template repo made **public**; updates now come from its raw URL instead of a
sibling checkout. Migration: replace the `## Updating…` section and the
"Built from" line in `CLAUDE.md`.

### v3 (2026-08-07)

Documented that **`devbox add` is a human action** — multi-user nix needs a
unix socket the sandbox blocks by design — with an explicit ban on working
around it. Migration: copy the `## Devbox` section over; no config changes.

### v2 (2026-07-27)

Added **overlay mode**: wrap the template around a clone of an existing repo
so autonomy, sandbox, and devbox apply while that repo stays free of template
files. Additive — existing projects need no migration.

### v1 (2026-07-17)

Initial version. Two-layer protection (minimal permissions + full sandbox with
`failIfUnavailable` and home-dir read denial), git via per-repo fine-grained
PAT in `.env` over HTTPS, devbox toolchain, English artifacts,
human-on-the-loop conventions.
