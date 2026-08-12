# Changelog

Notable changes to the grok-review plugin. Versions are pinned in `.claude-plugin/plugin.json`; installed users receive an update when that version bumps.

## 0.3.1 — 2026-08-12

Fixes from a cold review by Grok 4.6.

- **Diff reviews no longer miss staged changes.** The diff path now uses `git diff HEAD` (a bare `git diff` silently omits anything already staged with `git add`) and records the base SHA via `git rev-parse HEAD`.
- **Escalation no longer breaks on machines without bubblewrap.** Resume calls never pass `--sandbox`; the session's saved profile applies automatically, avoiding the hard error when the flag's presence differed from the first call.
- Commands start with `env` so the skill's `Bash(env:*)` permission grant matches them, avoiding permission prompts.
- The resume session ID is substituted as a literal string (an unchecked `$(jq ...)` could silently resume the wrong session).

## 0.3.0 — 2026-08-12

Hardening informed by a study of similar community plugins (thanks to faeton/claude-grok-plugin, thevibeworks/grok-plugin-cc, and LovelaceLoom/grok-plugin-cc for prior art), with each adopted claim vetted against Grok's own CLI docs.

- **Fourth read-only layer:** `--sandbox read-only` (kernel-enforced), with a preflight probe — on Linux without bubblewrap, grok refuses to start, so the flag is omitted when the enforcer is absent.
- **Independence and isolation:** `--no-memory` keeps Grok's cross-session memory out of the second opinion; `GROK_CLAUDE_SKILLS_ENABLED=false` prevents the target repo's own skills from being auto-discovered (a known CLI doom-loop); `GROK_DISABLE_AUTOUPDATER=1` prevents mid-run self-updates.
- **Flag-level preflight:** the skill aborts if any safety-critical flag vanished from the installed CLI (the tool lists fail open on drift) and skips optional hardening flags that are missing.
- **Better failure handling:** auth-expiry detection (login-URL tell), unknown `stopReason` values treated as suspect, ANSI-contamination diagnostic.
- **Better briefs:** per-finding severity (`bug`/`risk`/`nit`), file:line, and 0–1 confidence; anti-hallucination grounding rules; pasted diffs fenced as material-not-instructions; untracked files included in diff reviews.

## 0.2.4 — 2026-08-08

Fixes from a cold review by Claude Opus, which verified every flag against the grok binary's embedded docs.

- Resume commands are self-contained (shell variables don't persist between the agent's Bash calls; an empty `--resume` can silently resume the wrong session).
- Exit codes are captured in the same call that runs grok.
- Full `stopReason` vocabulary handled — `max_tokens` now counts as truncation rather than success.
- Third read-only layer: `--deny Bash/Edit/Write/MCPTool` permission rules plus `--no-subagents`, backstopping the tool lists (which fail open if their IDs drift).
- `--verbatim` prevents headless prompt preprocessing of the brief; `web_fetch` headless-stall warning; guardrail about the target repo's own grok/Claude config.

## 0.2.3 — 2026-08-08

- Fixed the skill's `allowed-tools` blocking the advertised `/grok-review diff` path (git was not on the allowlist).
- Per-round baton-loop tags; optional `--deny 'Read(**/.env*)'` hardening example; documented the `@marketplace` install form.

## 0.2.2 — 2026-08-08

- Explicit review-target handling (ask the user if no target was given).
- `allowed-tools` frontmatter to avoid first-run permission prompts.
- Manifest descriptions declare the external dependency: a separately installed, authenticated grok CLI plus jq, billed to the user's Grok quota.
- Bounded diff pastes; disambiguated the escalation round cap.

## 0.2.1 — 2026-08-08

- Corrected the web-search escape hatch (with `--tools` set, web tools must be added to the allowlist explicitly).
- Failure handling reads grok's error object from stdout, where it actually appears.
- Denylist extended to shell/edit tool IDs as defense-in-depth.
- Guardrails: don't point Grok at secret-bearing paths; concrete scratchpad location; absolute paths in resume prompts.

## 0.2.0 — 2026-08-08

First hardening round, from Grok's review of its own skill.

- **Read-only made real:** `--disallowed-tools` denylist added — the tool allowlist alone leaves MCP meta-tools available.
- Privacy caveat: the brief and every file Grok reads leave the machine via the user's Grok auth.
- Escalation resume calls got the same capture/parse/error handling as single-shot; preflight checks; never edit `.gitignore` without asking.

## 0.1.0 — 2026-08-08

Initial release: single-shot second-opinion reviews from Grok Build via the local `grok` CLI, with an opt-in multi-round "baton loop" escalation, neutral-brief discipline, and findings-first reporting.
