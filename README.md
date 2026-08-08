# grok-review — a Claude Code skill

Get a second opinion from **Grok Build** without leaving Claude Code. Claude writes a neutral review brief, drives the local `grok` CLI headlessly (read-only), and reports Grok's verdict back — findings first, Claude's own take clearly separated. One shot by default; an explicit multi-round "baton loop" (Claude and Grok arguing through inline comments in a shared markdown doc) when you ask for it.

Why bother: cross-model review catches things same-model self-review doesn't, and the CLI runs in your repo's working directory, so Grok reads the actual code instead of whatever got pasted into a prompt.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- The Grok Build CLI (`grok`) installed and authenticated (`grok login`). Calls are billed to whatever your `grok` CLI is authenticated with (subscription OAuth or API key).

## Install

**As a plugin (recommended):**

```
/plugin marketplace add <your-github-user>/claude-skill-grok-review
/plugin install grok-review
```

**Manual copy:** copy `skills/grok-review/` into `~/.claude/skills/` (all projects) or `<repo>/.claude/skills/` (one project).

## Usage

```
/grok-review docs/my-design-doc.md
/grok-review diff
/grok-review should the cache layer use TTL eviction or LRU?
```

Or just ask in plain language: "get Grok's take on this plan."

After the single-shot review, say "continue the argument with Grok" to escalate to the multi-round loop (capped at 3 rounds; unresolved disputes come back to you with both positions).

## How it works

- Single-shot: `grok --prompt-file <brief> --tools "read_file,grep,list_dir" --output-format json` — a fresh Grok session per review, session ID captured for optional follow-up.
- Escalation: a shared review doc with `[GROK]` / `[CLAUDE]` inline comments and `RESOLVED / DISPUTED / USER-CALL` status tags; Grok's side resumes via `--resume <sessionId>` so it keeps its own context.
- Guardrails: Grok never gets edit or shell tools (one writer per repo: Claude), briefs must be neutrally framed (no arguing for Claude's preferred outcome), secrets stay out of briefs, failures are reported rather than papered over.

## Caveats

- Verified against `grok 0.2.118`. Two things may drift with CLI versions: the read-only tool IDs (`read_file,grep,list_dir`) and the workaround for headless `--permission-mode plan` stalling (the skill uses a tool allowlist instead).
- Every review spends your Grok quota. The skill defaults to the cheapest useful shape (one shot) for that reason.
