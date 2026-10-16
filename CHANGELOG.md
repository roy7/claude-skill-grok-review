# Changelog

Notable changes to the grok-review plugin. Versions are pinned in `.claude-plugin/plugin.json`; installed users receive an update when that version bumps.

## 0.3.9 — 2026-10-16

An explicit request for the kernel sandbox is no longer dropped silently.

- **Asked for a sandbox, can't have one → the skill stops and asks.** On a Linux host without bubblewrap, the skill still omits `--sandbox` by default (the other three read-only layers hold) and mentions `apt install bubblewrap`. But if the user explicitly asked for a sandboxed review, it now stops, explains that the sandbox can't be enforced here, and runs without it only on the user's OK. That's the same fail-closed stance as the CLI itself, which refuses to start when `--sandbox` is passed without an enforcer (re-verified on grok 1.0.46: exit 1 in 0.4 s, "Refusing to start with denied paths unprotected"). Suggested by the plugin's author.
- README "verified against" caveat updated to `grok 1.0.46`: about 30 headless reviews (subtitle and title translations, 2026-10-13…16) ran with the current flag set, env vars and JSON fields (`text`, `sessionId`, `stopReason`), all ending `end_turn`.

## 0.3.8 — 2026-08-24

Restores the turn cap that 0.3.7 cut too far.

- **`--max-turns` is back to 30 for directed reviews (was 6), and 50 for unknown-diff reviews (was 30).** 0.3.7's directed-read brief and reviewer system prompt are kept — they measured 10 → 3 loops on one brief — but across a day of real use directed reviews ran 3–15 loops, and the 6-turn cap truncated reviews mid-investigation (a 12-turn cap tried in the field did too), returning findings with no verdict after spending the quota. Under 30, every review from Aug 23 (9–19 loops) completed; the single run ever to exhaust 30 was an unknown-diff review with 78 tool calls, hence 50 for that shape. Truncation is still reported, never silently retried with a bigger cap.

## 0.3.7 — 2026-08-23

Directed reviews: Grok reads the named files once and answers, instead of being bootstrapped as a coding agent and hunting.

- **Reviewer system prompt.** The invocation now passes `--system-prompt-override` with `skills/grok-review/reviewer-system-prompt.md` — a short read-only-reviewer prompt — in place of the CLI's default autonomous-coding-agent prompt, which also injected the target repo's `CLAUDE.md` as binding rules and an ~18k skill catalog (~43k tokens) ahead of every brief. Prompt swap only; every tool/deny/env safety layer is unchanged.
- **Directed-read brief.** The brief template now gives each pointer a line range and tells Grok to read every named path in its first tool batch, then write the verdict, with at most one extra batch for a missing cite; skill files and agent-instruction files are off-limits unless named as material.
- **`--max-turns 6` for pre-reg / plan / design / results reviews** (30 stays for unknown-diff reviews where Grok must discover the files). A 30-turn undirected review had been observed returning 518 chars and no verdict.
- **Measured, same brief before/after (one pre-reg review):** model loops 10 → 3, cost $0.089 → $0.040, wall-clock 5m03s → 4m20s, identical verdict and findings. The honest floor is the verdict loop itself (~3 min of reasoning on ~64k input) — expect 3–5 min per lock-grade review; the gain is cost, fewer slow-API exposures and robustness, not a 1-minute review. Corpus-packing/no-tools was evaluated and not adopted: it would not cut the reasoning floor and loses the cite-on-demand valve.
- `--sandbox read-only` is documented as OMIT on hosts without bubblewrap: the CLI refuses to start (exit 1, 0 s) rather than degrading.

## 0.3.6 — 2026-08-18

Keeps memory isolation working on current grok CLIs.

- **Memory isolation survives the CLI's flag removal.** grok 1.0.5 removed the `--no-memory` flag (memory control moved to the `GROK_MEMORY` env var, per the CLI's bundled `13-memory.md`); under the skill's probe-and-skip rule the flag was being silently omitted, leaving the second opinion unprotected against a user-enabled cross-session memory. `GROK_MEMORY=0` now rides the env prefix on every call (single-shot and resume) — stronger than the old flag: no probing needed, and it force-disables memory even when the user's TOML or remote settings enable it. Caught live when a smoke-test preflight flagged the missing flag.
- README "verified against" caveat updated to `grok 1.0.5`, with `GROK_MEMORY` added to the list of things that may drift.

## 0.3.5 — 2026-08-18

Fixes a real field incident: a 23-loop review ran 9m54s against the invocation's 10-minute timeout — the call was killed as grok finished, and the blind retry burned a second helping of Grok quota reviewing the same thing.

- **Reviews no longer die at 10 minutes.** The grok invocation now launches as a background Bash call instead of a foreground call capped at the Bash tool's hard 600-second maximum. Slow xAI days (25–50 s per inference loop was observed live) can push an ordinary repo-reading review past 10 minutes of near-zero-CPU waiting; that's now fine — and Claude can keep working on other parts of your task while the review runs.
- **A slow review is no longer mistaken for a hung one.** The skill now states explicitly: grok at ~0% CPU is blocked on xAI inference, not stuck — never kill or relaunch on elapsed time alone, and never relaunch into the same capture files (the incident's retry truncated the first run's logs while it was still finishing; only a surviving file descriptor saved the output). Retries, when genuinely needed, get fresh filenames and a check of whether the previous attempt's output is already complete.
- The same background treatment applies to escalation resume rounds in `references/escalation.md`.

## 0.3.4 — 2026-08-15

A context-cost round: same reviews, noticeably cheaper on the Claude side.

- **Diffs are no longer pasted into the brief.** The diff now goes to a scratchpad file (`git diff HEAD > .../review.diff`) and Grok reads it from disk — verified live that `read_file` reaches absolute paths outside the repo. Previously the diff landed in Claude's context twice (once as command output, once as the brief it wrote), which on a big change was the single largest cost of a review. The ">400 lines, paste only the hot hunks" workaround is gone with it: Grok now sees the whole diff regardless of size, and you still get told how big it is before it ships to xAI.
- **The escalation path only loads when you escalate.** The multi-round baton loop moved to `references/escalation.md`, read on demand instead of on every single-shot review; SKILL.md is ~18% smaller in the always-loaded path.

## 0.3.3 — 2026-08-14

- **Cursor-compatible skill dirs are now blocked too:** `GROK_CURSOR_SKILLS_ENABLED=false` joins the env prefix, closing the last documented skill-discovery path from the target repo (the var is documented in the grok CLI's bundled user guide rather than `--help`; flagged by Grok in the round-8 review).

## 0.3.2 — 2026-08-14

Pre-marketplace-submission round: fixes from a cold review by Grok 4.6 (round 8) plus a fresh Claude review.

- **The sandbox-on-resume contradiction is gone.** The invoke notes still carried the old "pass `--sandbox` on both calls or neither" rule that 0.3.1 had already reversed in the resume section; the file now teaches one rule — never pass `--sandbox` on resume, the session's saved profile applies automatically.
- **The review is much harder for the target repo's config to steer.** `--no-plan --permission-mode default` pin the session policy (a repo `.claude/settings.json` with `defaultMode: plan` could reproduce the headless stall), `GROK_SANDBOX=off` accompanies any run that omits `--sandbox`, and the brief now tells Grok that the repo's own `CLAUDE.md`/agent-instruction files don't bind its verdict.
- **Branch/PR reviews:** `/grok-review diff <base>` reviews a branch against its merge-base (`git diff <base>...HEAD`), not just the working tree.
- Honest read-reach wording: the `read-only` sandbox restricts writes, not reads — Grok can read outside the repo, and path `--deny 'Read(...)'` rules must be repeated on every resume (they aren't saved with the session, unlike the sandbox profile).
- Fewer first-run permission prompts: the auth preflight and exit-code echo now use allowlisted commands (`ls`, `test`, `echo` added to `allowed-tools`).
- README: documented that plugin updates require `/plugin update` (they are not automatic).

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
