# TEMPLATE.md

**Template version: 1** · changelog at the bottom · projects record theirs in `CLAUDE.md`.

Starter for projects where Claude Code works **autonomously inside one directory**
(long sessions, big TODO lists, multiple agents) and the human only reviews
results — **human-on-the-loop**, not human-in-the-loop. Core idea: **maximum
autonomy inside the project, hard walls everywhere else.**

## Spin up a new project

Human part (~5 minutes):

1. Create a repo from this template (GitHub → "Use this template") and clone it
   **via the HTTPS URL** — that sets `origin` correctly for token-based pushes
   (`git@` remotes won't work in the sandbox).
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
6. Fill in `CLAUDE.md`, run boundary checks (project read/write with no
   prompts; `~/.ssh` and sibling projects unreadable; git clean of warnings),
   then commit and push as the end-to-end verification.

Daily after that: `devbox shell` → `claude`, always in that order (toolchain +
`.env` only exist inside; a SessionStart hook warns if Claude was started
outside the shell). Add tools only via `devbox add <pkg>[@version]` — never
`brew install`; `.devbox/` stays gitignored.

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
  vectors) — hence step 5 above, and Claude pushes without `-u`.
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

## macOS/devbox quirk

- `devbox.json`'s `init_hook` exports
  `DEVELOPER_DIR=/Library/Developer/CommandLineTools` — keep that line.
- Why: nix packages otherwise hijack the Apple toolchain pointer and
  `/usr/bin/git` fails with `error: tool 'git' not found`.

## Conventions

- **Language:** conversation in any language; every artifact (docs, code,
  comments, test names, commits, UI strings) in English unless specified otherwise.
- **Autonomy:** proceed without asking while inside the project; verify with
  tests/builds before reporting done; report honestly. Durable project rules →
  `CLAUDE.md`; secrets → `.env`.
- **Declarations in git, artifacts out of git** (devbox.json + lockfiles
  committed; binaries, caches, `.env` ignored).

## Updating a project to a newer template version

Never git-merge from the template — projects diverge. In the project, tell Claude:

> Update this project to template vN — changelog is in claude-project-template/TEMPLATE.md.

Claude walks the changelog deltas between the recorded and target version with
you, applies them on top of the project's customizations, bumps the marker in
`CLAUDE.md`, and commits. Changelog entries are written as migration notes for
this purpose.

---

## Changelog

### v1 (2026-07-17)

Initial version. Two-layer protection (minimal permissions + full sandbox with
`failIfUnavailable` and home-dir read denial), git via per-repo fine-grained
PAT in `.env` over HTTPS, devbox toolchain, English artifacts,
human-on-the-loop conventions.
