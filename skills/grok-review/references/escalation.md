# Multi-round baton loop (ESCALATION)

Load and follow this file **only when the user asks to continue the argument** after a single-shot review. Everything here assumes the single-shot in `SKILL.md` already ran: you have its `SID`, its brief, and its findings.

1. Create a review doc at `<untracked-dir>/grok-reviews/YYYY-MM-DD-<slug>.md` — review docs never get committed. Choosing `<untracked-dir>`: use an **existing** gitignored working directory if the project already has one (e.g. `.scratch/`, `tmp/`, `.claude/work/`). If none exists, **ask the user** whether to (a) create a dir and add it to `.gitignore`, or (b) use a temp path outside the repo. Do **not** edit `.gitignore` without an explicit yes. On (b), put the **absolute path** in the user-facing report and note the trail is ephemeral. Contents:
   - Header: topic, date, `grok-session: <SID>`.
   - Grok's round-1 findings, then inline replies. Tags carry the round number they were written in: the initial single-shot review is R1 and the resume rounds are R2–R4 — never a range like `R1-3` — with finding numbers appended as needed:
     - `> **[GROK R1.3]** <third finding from round 1>`
     - `**[CLAUDE R1]** <reply>` plus a status tag: `RESOLVED / DISPUTED / USER-CALL`.
2. Resume the same Grok session with the doc — launched as a **background** Bash call (`run_in_background: true`, no Bash `timeout` parameter), same as the single-shot and for the same reason: resume rounds re-read the repo and can overrun a foreground timeout. Wait for the completion notification; don't poll, don't kill on elapsed time (grok at ~0% CPU is blocked on xAI inference, not hung), and give any retry fresh capture filenames:

   ```bash
   env GROK_CLAUDE_SKILLS_ENABLED=false GROK_CURSOR_SKILLS_ENABLED=false GROK_DISABLE_AUTOUPDATER=1 GROK_MEMORY=0 \
   grok --resume "<SID>" \
     -p "Read <absolute doc path>. Respond to each CLAUDE reply inline; concede or sharpen each DISPUTED item. Same verdict format." \
     --tools "read_file,grep,list_dir" \
     --disallowed-tools "search_tool,use_tool,Agent,run_terminal_cmd,search_replace" \
     --deny Bash --deny Edit --deny Write --deny MCPTool \
     --no-subagents --no-plan --permission-mode default \
     --disable-web-search --max-turns 30 \
     --output-format json > <scratchpad>/grok-out-r<N>.json 2><scratchpad>/grok-err-r<N>.log; echo "grok_exit=$?"
   ```

   Do NOT pass `--sandbox` on resume — the session's saved profile applies automatically, and passing a profile that differs from the initial call (including passing one when the initial call omitted it on a no-bwrap machine) is a hard error, not a warning. The env prefix follows the same session-consistency rule: if the initial call ran with `GROK_SANDBOX=off` (no bwrap), keep it on every resume; if the session started with `--sandbox read-only`, don't add it. Path deny rules (e.g. `--deny 'Read(**/.env*)'`) are per-invocation and are NOT saved with the session — repeat them on every resume call.

   `<SID>` is the LITERAL session ID string recorded in step 3 of the single-shot — substitute it into the command; never a shell variable set in an earlier Bash call (shell state does not survive between calls) and never an unchecked `$(jq ...)` substitution (if the file is missing it yields an empty/`null` `--resume`, which doesn't error: it falls back to title-matching in the cwd and can silently resume the WRONG session). Always give Grok the doc's **absolute path** in the resume prompt (and in the doc header) — a relative path fails whenever the doc lives outside the repo cwd. Every resume call gets the SAME capture and parse treatment as the single-shot: redirect stdout/stderr to fresh scratchpad files, check the echoed `grok_exit`, `jq -r '.text'` / `.stopReason` (`end_turn`, else truncated/failed per the single-shot rules), and **re-read `.sessionId` each round** — update `SID` if it changed rather than assuming it's stable. On failure, report from both the out file (`.message`) and the error log; don't improvise. If Grok reports the review doc "is ignored by .gitignore and cannot be read", the user has `respect_gitignore` enabled in their grok config — move the doc to a non-ignored path or include its content inline in `-p`.

3. Fold Grok's responses back into the doc each round. **Cap at 3 resume rounds after the initial review** (4 Grok invocations total) — if still disputed, stop and present both positions to the user. Two agents politely agreeing burns quota; converge or escalate.
4. When done, leave the doc in place as the trail, and summarize the resolution (what changed, what was disputed, what went to the user) in your report.

## Escalation guardrails

- Every resume call spends the user's Grok quota. Never loop without the user asking, and stop at the round cap.
- If `--resume` errors (session expired/missing), say so and offer a fresh single-shot with the review doc as context — don't quietly start a new thread pretending it's the old one.
- The read-only layers apply unchanged on every resume: all three tool/permission layers, every round. Claude remains the only agent that edits the working tree.
