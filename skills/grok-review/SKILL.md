---
name: grok-review
description: Get a second opinion from Grok Build on a plan, design doc, diff, or piece of code, via the local `grok` CLI (uses the user's Grok login — subscription or API key, whatever `grok` is authenticated with). Single-shot review by default; escalate to a multi-round baton loop only when the user asks to continue the argument.
argument-hint: <file | doc path | "diff" | freeform question>
---

# Grok review — second opinion via the Grok Build CLI

Drive the locally installed `grok` CLI headlessly to get an independent review from a different model family. The CLI runs in the repo cwd, so Grok reads real code itself — point it at paths instead of pasting files. Requires `grok` on PATH and already authenticated (`grok login`); if it's missing or unauthenticated, tell the user instead of substituting a different Grok access path.

## Single-shot review (DEFAULT)

1. **Frame the brief.** Write a prompt file to the scratchpad (never the repo). It must contain:
   - One paragraph of project context (what the feature/doc is, what decision is at stake).
   - **Neutral framing:** state the decision, the alternatives considered, and open risks — do NOT argue for a preferred outcome in the brief; a stacked brief buys agreement, not a second opinion. If Claude has a position, disclose it in one labeled line at the end ("Claude currently leans X because Y") rather than weaving it through the framing.
   - The specific question(s) — ask for a verdict (`pass / pass-with-fixes / fail`) plus numbered findings, most severe first.
   - Pointers to repo paths to read (Grok reads them itself — don't paste whole files; paste only diffs or excerpts that aren't on disk).
   - For diff reviews: paste `git diff` output AND state the base SHA + whether the project's quality gates (typecheck / tests / build) passed or weren't run — Grok has no shell, so it can't verify any of that itself.
   - Instruction: "Be adversarial; if you agree, say so briefly rather than inventing objections."
2. **Invoke** (from the repo root, read-only via tool allowlist, JSON output so you get the session ID back):

   ```bash
   grok --prompt-file <brief> \
     --tools "read_file,grep,list_dir" --disable-web-search --max-turns 30 \
     --output-format json > <scratchpad>/grok-out.json 2><scratchpad>/grok-err.log
   ```

   - `--tools "read_file,grep,list_dir"` keeps Grok read-only (do NOT use `--permission-mode plan` — headless plan mode stalls after one narration line). **One writer per repo: Claude.** Never grant Grok edit/shell tools.
   - Drop `--disable-web-search` only if the review genuinely needs external facts (it re-enables web tools despite the allowlist note — verify before relying on it).
   - Timeout: allow up to 10 min (`timeout: 600000`); repo-reading reviews are slow.
3. **Parse:** `jq -r '.text'` for the review, `jq -r '.sessionId'` for the resume handle (`SID`), `.stopReason` should be `end_turn` (if `max_turn_requests`, the review is truncated — say so). If `.text` is missing/empty or exit ≠ 0, report the failure (with `grok-err.log`) — don't improvise a summary from partial output.
4. **Report back:** relay Grok's verdict and numbered findings **verbatim (or faithfully condensed) first**, then add Claude's own take as a clearly separated second section (agree / disagree / user-call per finding) — don't interleave rebuttals into Grok's findings; the user should see the independent opinion before the author's defense. Never silently apply Grok's fixes — findings are input, the user decides. Record the `SID` in your report so escalation is possible.

## Multi-round baton loop (ESCALATION — only when the user says continue)

1. Create a review doc at `<untracked-dir>/grok-reviews/YYYY-MM-DD-<slug>.md` — use the project's gitignored working-files directory (add one to `.gitignore` if none exists); review docs never get committed. Contents:
   - Header: topic, date, `grok-session: <SID>`.
   - Grok's round-1 findings, then inline replies using this convention:
     - `> **[GROK R1-3]** <finding>` followed by `**[CLAUDE]** <reply>` and a status tag: `RESOLVED / DISPUTED / USER-CALL`.
2. Resume the same Grok session with the doc:

   ```bash
   grok --resume "$SID" -p "Read <doc path>. Respond to each CLAUDE reply inline; concede or sharpen each DISPUTED item. Same verdict format." \
     --tools "read_file,grep,list_dir" --disable-web-search --max-turns 30 \
     --output-format json
   ```

3. Fold Grok's responses back into the doc each round. **Cap at 3 rounds** — if still disputed, stop and present both positions to the user. Two agents politely agreeing burns quota; converge or escalate.
4. When done, leave the doc in place as the trail, and summarize the resolution (what changed, what was disputed, what went to the user) in your report.

## Guardrails

- Every call spends the user's Grok quota: default single-shot; never loop without the user asking.
- Grok is read-only, always. Claude is the only agent that edits the working tree.
- Never put secrets or `.env*` contents in the brief.
- If `grok` exits non-zero or hangs past timeout, report the failure verbatim — never silently substitute a different Grok access path (other tools may bill differently).
- If `--resume` errors (session expired/missing), say so and offer a fresh single-shot with the review doc as context — don't quietly start a new thread pretending it's the old one.
