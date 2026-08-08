---
name: grok-review
description: Get a second opinion from Grok Build on a plan, design doc, diff, or piece of code, via the local `grok` CLI (uses the user's Grok login — subscription or API key, whatever `grok` is authenticated with). Single-shot review by default; escalate to a multi-round baton loop only when the user asks to continue the argument.
argument-hint: <file | doc path | "diff" | freeform question>
allowed-tools: Read, Write, Edit, Grep, Glob, Bash(grok:*), Bash(jq:*), Bash(command:*)
---

# Grok review — second opinion via the Grok Build CLI

Drive the locally installed `grok` CLI headlessly to get an independent review from a different model family. The CLI runs in the repo cwd, so Grok reads real code itself — point it at paths instead of pasting files.

**Review target:** the invocation arguments name what to review (a file/doc path, `diff`, or a freeform question) — they arrive as an `ARGUMENTS:` line or via `$ARGUMENTS` substitution depending on the harness. If no target or question was provided, ask the user what to review before doing anything else.

**Preflight (before the first call):** check `command -v grok` and `command -v jq` — both are required (`jq` parses the JSON output). If `grok` is missing, or a call fails with an authentication error, tell the user to install it / run `grok login` instead of substituting a different Grok access path.

## Single-shot review (DEFAULT)

1. **Frame the brief.** Write a prompt file to the scratchpad — the harness's session temp directory if it provides one (Claude Code does); otherwise use `${TMPDIR:-/tmp}/grok-review-<slug>/`. Never write it inside the repo. Reuse the same directory for the `grok-out*.json` / `grok-err*.log` capture files below. The brief must contain:
   - One paragraph of project context (what the feature/doc is, what decision is at stake).
   - **Neutral framing:** state the decision, the alternatives considered, and open risks — do NOT argue for a preferred outcome in the brief; a stacked brief buys agreement, not a second opinion. If Claude has a position, disclose it in one labeled line at the end ("Claude currently leans X because Y") rather than weaving it through the framing.
   - The specific question(s) — ask for a verdict (`pass / pass-with-fixes / fail`) plus numbered findings, most severe first.
   - Pointers to repo paths to read (Grok reads them itself — don't paste whole files; paste only diffs or excerpts that aren't on disk).
   - For diff reviews: paste `git diff` output AND state the base SHA + whether the project's quality gates (typecheck / tests / build) passed or weren't run — Grok has no shell, so it can't verify any of that itself. Keep pastes bounded: if the diff is large (roughly >400 lines), paste `git diff --stat` plus only the hot hunks and point at the changed paths for Grok to read on disk; ask the user before pasting anything huge (it all goes to xAI).
   - Instruction: "Be adversarial; if you agree, say so briefly rather than inventing objections."
2. **Invoke** (from the repo root, read-only via tool allowlist, JSON output so you get the session ID back):

   ```bash
   grok --prompt-file <brief> \
     --tools "read_file,grep,list_dir" \
     --disallowed-tools "search_tool,use_tool,Agent,run_terminal_cmd,search_replace" \
     --disable-web-search --max-turns 30 \
     --output-format json > <scratchpad>/grok-out.json 2><scratchpad>/grok-err.log
   ```

   - The allowlist alone is NOT read-only: MCP meta-tools (`search_tool`, `use_tool`) remain available unless denied, and could drive whatever MCP servers the user has connected. The `--disallowed-tools` denylist is what actually makes the call read-only — always pass both. Denying `run_terminal_cmd` and `search_replace` is defense-in-depth: if `--tools` is ever ignored or its IDs drift, shell and edit stay blocked anyway. (Do NOT use `--permission-mode plan` as a substitute — headless plan mode stalls after one narration line.) **One writer per repo: Claude.** Never grant Grok edit/shell tools.
   - If the review genuinely needs external facts, add `web_search` (and `web_fetch` if needed) to `--tools` AND drop `--disable-web-search` — with `--tools` set, only listed tools exist, so dropping the flag alone re-enables nothing.
   - Timeout: set the **Bash tool's** `timeout` parameter to `600000` (10 minutes) — that is a Claude Code tool option, not a `grok` flag; do not pass any timeout flag to `grok`. Repo-reading reviews are slow.
3. **Parse:** `jq -r '.text'` for the review, `jq -r '.sessionId'` for the resume handle (`SID`), `.stopReason` should be `end_turn` (if `max_turn_requests`, the review is truncated — say so). If `.text` is missing/empty or exit ≠ 0, report the failure from BOTH streams: on errors grok emits `{"type":"error","message":...}` on **stdout** (so `jq -r '.message // .'` the out file), while stderr is mostly log noise — check `grok-err.log` too, but the actionable message is usually in the out file. Don't improvise a summary from partial output.
4. **Report back:** relay Grok's verdict and numbered findings **verbatim (or faithfully condensed) first**, then add Claude's own take as a clearly separated second section (agree / disagree / user-call per finding) — don't interleave rebuttals into Grok's findings; the user should see the independent opinion before the author's defense. Never silently apply Grok's fixes — findings are input, the user decides. Record the `SID` in your report so escalation is possible.

## Multi-round baton loop (ESCALATION — only when the user says continue)

1. Create a review doc at `<untracked-dir>/grok-reviews/YYYY-MM-DD-<slug>.md` — review docs never get committed. Choosing `<untracked-dir>`: use an **existing** gitignored working directory if the project already has one (e.g. `.scratch/`, `tmp/`, `.claude/work/`). If none exists, **ask the user** whether to (a) create a dir and add it to `.gitignore`, or (b) use a temp path outside the repo. Do **not** edit `.gitignore` without an explicit yes. On (b), put the **absolute path** in the user-facing report and note the trail is ephemeral. Contents:
   - Header: topic, date, `grok-session: <SID>`.
   - Grok's round-1 findings, then inline replies using this convention:
     - `> **[GROK R1-3]** <finding>` followed by `**[CLAUDE]** <reply>` and a status tag: `RESOLVED / DISPUTED / USER-CALL`.
2. Resume the same Grok session with the doc:

   ```bash
   grok --resume "$SID" -p "Read <absolute doc path>. Respond to each CLAUDE reply inline; concede or sharpen each DISPUTED item. Same verdict format." \
     --tools "read_file,grep,list_dir" \
     --disallowed-tools "search_tool,use_tool,Agent,run_terminal_cmd,search_replace" \
     --disable-web-search --max-turns 30 \
     --output-format json > <scratchpad>/grok-out-r<N>.json 2><scratchpad>/grok-err-r<N>.log
   ```

   Always give Grok the doc's **absolute path** in the resume prompt (and in the doc header) — a relative path fails whenever the doc lives outside the repo cwd. Every resume call gets the SAME capture and parse treatment as the single-shot: redirect stdout/stderr to fresh scratchpad files, check exit code, `jq -r '.text'` / `.stopReason` (`end_turn`, else report truncation), and **re-read `.sessionId` each round** — update `SID` if it changed rather than assuming it's stable. On failure, report from both the out file (`.message`) and the error log; don't improvise.

3. Fold Grok's responses back into the doc each round. **Cap at 3 resume rounds after the initial review** (4 Grok invocations total) — if still disputed, stop and present both positions to the user. Two agents politely agreeing burns quota; converge or escalate.
4. When done, leave the doc in place as the trail, and summarize the resolution (what changed, what was disputed, what went to the user) in your report.

## Guardrails

- Every call spends the user's Grok quota: default single-shot; never loop without the user asking.
- Grok is read-only, always (allowlist + `--disallowed-tools` denylist together — see the invoke notes). Claude is the only agent that edits the working tree.
- Everything in the brief, plus any files Grok reads, leaves the machine via the user's Grok authentication to xAI. Never put secrets or `.env*` contents in the brief, and don't paste diffs containing PII or customer data without flagging it to the user first.
- The brief ban isn't enough on its own: Grok can `read_file` any path in the cwd tree itself. Don't point Grok at secret-bearing paths, and if the repo likely contains secrets, PII, or customer data anywhere Grok might read, confirm with the user before invoking.
- If `grok` exits non-zero or hangs past timeout, report the failure verbatim — never silently substitute a different Grok access path (other tools may bill differently).
- If `--resume` errors (session expired/missing), say so and offer a fresh single-shot with the review doc as context — don't quietly start a new thread pretending it's the old one.
