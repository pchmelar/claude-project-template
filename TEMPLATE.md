# TEMPLATE.md

**Template version: 4** · changelog at the bottom · projects record theirs in `CLAUDE.md`.
Source: <https://github.com/pchmelar/claude-project-template> (public).

Starter for projects where Claude Code works **autonomously inside one directory**
(long sessions, big TODO lists, multiple agents) and the human only reviews
results — **human-on-the-loop**, not human-in-the-loop. Core idea: **maximum
autonomy inside the project, hard walls everywhere else.**

## Spin up a new project

Human part (~5 minutes):

1. Create a repo from this template (GitHub → "Use this template") and clone it.
2. Mint a **fine-grained PAT** at
   <https://github.com/settings/personal-access-tokens> — Only select
   repositories → this repo, Contents: Read and write, expiration at your
   discretion. Then `cp .env.example .env` and put it in as `GITHUB_TOKEN`
   (edit the file directly; never paste tokens into the conversation).
3. `devbox shell` → `claude`, and say: **"Bootstrap this project per TEMPLATE.md."**

Claude's bootstrap (first session):

4. Ask the user for the project name and a short description.
5. Replace `{{PROJECT_NAME}}` (`README.md`, `CLAUDE.md`, `devbox.json`) and
   `{{UID}}` in `.claude/settings.json` with the output of `id -u` — the
   settings edit needs the user's one-time approval. Settings reload live;
   no restart needed.
6. Check that `origin` is an HTTPS URL — `git@` remotes can't push through
   the sandbox. `.git/config` is write-protected, so if it's SSH, hand the
   user a `git remote set-url origin https://…` one-liner to run.
7. Fill in `CLAUDE.md`, run boundary checks (project read/write with no
   prompts; `~/.ssh` and sibling projects unreadable; git clean of warnings),
   then commit and push as the end-to-end verification.

Daily after that: `devbox shell` → `claude`, always in that order (toolchain +
`.env` only exist inside; a SessionStart hook warns if Claude was started
outside the shell). New tools come from `devbox add`, which only the human can
run — see [Devbox](#devbox).

## Overlay mode

For working on an existing repo you don't fully own — same autonomy and
protection, but nothing template-related ever enters that repo. Create
`<xyz>-wrapper` from this template (private), put two PATs in `.env`
(wrapper + target repo), then `devbox shell` → `claude` and say:
**"Bootstrap an overlay over \<target repo URL\> into `repo/` per
TEMPLATE.md."** As part of the bootstrap, Claude asks how you want to work
with git in the nested repo (branching, PRs, what never to push).

## The protection model (`.claude/settings.json`)

Two layers with different jobs:

- **Permissions** — *may this run without asking?* `defaultMode: "acceptEdits"` +
  allow `Bash`, `WebSearch`, `WebFetch`. In-project reads/edits are free;
  anything outside the project prompts the human.
- **Sandbox** — *what can a running command touch?* Writes: project + temp roots
  only. Reads: home directory denied except the project and four runtime paths
  (`~/.claude/shell-snapshots`, `~/.claude/session-env` — Bash breaks without
  them; `~/.gitconfig`, `~/.config/git` — git reads them). Network: toolchain
  allowlist (Bash only — WebSearch/WebFetch are unaffected).

Why the key lines exist:

- `failIfUnavailable: true` — no sandbox → Claude Code refuses to start.
  This is what makes the blanket `Bash` allow safe.
- `denyRead: ["~"]` — anti-exfiltration: an injected agent can't read `~/.ssh`,
  other projects, or browser profiles. With reads locked down, unrestricted
  `WebFetch` is an acceptable trade — the worst leak is the repo-scoped token.
  The discipline that keeps this true: **tokens are always fine-grained, one per repo.**
- `.git/config` + `.git/hooks` are hard-protected by Claude Code (code-execution
  vectors) — hence step 6 above, and Claude pushes without `-u`.
- When a permission prompt appears, fix `settings.json` deliberately — don't
  rephrase commands around it, don't accumulate one-off allow rules.

`settings.json` is committed and owned by the project — extend it freely
(network domains for new package sources, MCP servers, …) as the project evolves.

## Git

- Claude commits at natural milestones and pushes over HTTPS with the token
  from `.env` via an inline credential helper (token never in argv/logs):

```bash
set -a; source .env; set +a
git -c credential.helper= \
  -c credential.helper='!f(){ echo "username=<gh-user>"; echo "password=${GITHUB_TOKEN}"; };f' \
  push origin <branch>
```

- No SSH (doesn't traverse the sandbox proxy — and must stay that way: the
  personal key can touch every repo; unsandboxing git would be a code-exec hole).

## Devbox

- Tools come from `devbox add <pkg>[@version]`, never `brew`. `devbox.json` +
  `devbox.lock` are committed, `.devbox/` is not.

- **`devbox add` is a human action.** Multi-user nix reaches its daemon over a
  unix socket, which the sandbox blocks by design — it fails with `cannot
  connect to socket … Operation not permitted` and no config fixes that. Claude
  stops, asks the human to run `devbox add <pkg>` in another terminal, and
  **never works around it**: no `brew`, no binaries fetched into the project,
  no loosening the sandbox.

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
