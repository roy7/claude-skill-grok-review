# grok-review — a Claude Code skill

Get a second opinion from **Grok Build** without leaving Claude Code. Claude writes a neutral review brief, drives the local `grok` CLI headlessly (read-only), and reports Grok's verdict back — findings first, Claude's own take clearly separated. One shot by default; an explicit multi-round "baton loop" (Claude and Grok arguing through inline comments in a shared markdown doc) when you ask for it.

Why bother: cross-model review catches things same-model self-review doesn't, and the CLI runs in your repo's working directory, so Grok reads the actual code instead of whatever got pasted into a prompt.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- The Grok Build CLI (`grok`) installed and authenticated (`grok login`). Calls are billed to whatever your `grok` CLI is authenticated with (subscription OAuth or API key).
- `jq` (used to parse the CLI's JSON output).

## Install

**As a plugin (recommended):**

```
/plugin marketplace add roy7/claude-skill-grok-review
/plugin install grok-review@grok-review-marketplace
```

(`/plugin install grok-review` also works when the name is unambiguous.)

**Manual copy:** copy `skills/grok-review/` into `~/.claude/skills/` (all projects) or `<repo>/.claude/skills/` (one project).

## Usage

```
/grok-review docs/my-design-doc.md
/grok-review diff
/grok-review should the cache layer use TTL eviction or LRU?
```

Or just ask in plain language: "get Grok's take on this plan."

After the single-shot review, say "continue the argument with Grok" to escalate to the multi-round loop (capped at 3 resume rounds after the initial review; unresolved disputes come back to you with both positions).

## How it works

- Single-shot: `env GROK_CLAUDE_SKILLS_ENABLED=false GROK_DISABLE_AUTOUPDATER=1 grok --prompt-file <brief> --verbatim --tools "read_file,grep,list_dir" --disallowed-tools "search_tool,use_tool,Agent,run_terminal_cmd,search_replace" --deny Bash --deny Edit --deny Write --deny MCPTool --no-subagents --no-memory --sandbox read-only --disable-web-search --max-turns 30 --output-format json` — a fresh Grok session per review, session ID captured for optional follow-up. Four read-only layers on purpose: the tool allowlist and denylist fail open if their IDs ever drift, permission `--deny` rules (a stable namespace where deny always wins) back them up, and the kernel sandbox backstops all three where the OS supports it. `--no-memory` keeps the second opinion independent of Grok's cross-session memory.
- Escalation: a shared review doc with round-and-finding-numbered `[GROK R1.3]` / `[CLAUDE R1]` inline comments (never ranges like `R1-3`) and `RESOLVED / DISPUTED / USER-CALL` status tags; Grok's side resumes via `--resume <sessionId>` so it keeps its own context. Capped at 3 resume rounds after the initial review (4 Grok invocations total).
- Guardrails: Grok never gets edit or shell tools (one writer per repo: Claude), briefs must be neutrally framed (no arguing for Claude's preferred outcome), secrets stay out of briefs, failures are reported rather than papered over.

## Caveats

- **Your code leaves the machine.** The review brief, any pasted diffs, and every file Grok reads are sent to xAI through your authenticated `grok` CLI — same as using Grok Build directly. The skill bans secrets and `.env` contents in briefs, but you are the judge of whether proprietary source or sensitive data should be reviewed this way at all.
- Verified against `grok 1.0.3` (and previously `0.2.118`) — the skill doesn't pin a model, so it uses whatever your CLI defaults to (Grok 4.6 as of this writing). Things that may drift with CLI versions: the tool IDs in the allowlist (`read_file,grep,list_dir`) and denylist (`search_tool,use_tool,Agent,run_terminal_cmd,search_replace`), the permission-rule prefixes used with `--deny` (`Bash`, `Edit`, `Write`, `MCPTool`), the `--disable-web-search` / `--verbatim` / `--no-subagents` / `--no-memory` / `--sandbox` flags and the `GROK_CLAUDE_SKILLS_ENABLED` / `GROK_DISABLE_AUTOUPDATER` env vars (the skill's preflight probes for the safety-critical flags and aborts if any vanished; optional hardening flags are skipped when absent), the JSON output fields (`text`, `sessionId`, `stopReason` with snake_case values `end_turn` / `max_tokens` / `max_turn_requests` / `refusal` / `cancelled`), and the workaround for headless `--permission-mode plan` stalling (the skill uses tool lists + deny rules instead).
- Every review spends your Grok quota. The skill defaults to the cheapest useful shape (one shot) for that reason.
