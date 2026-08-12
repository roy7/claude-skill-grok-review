# grok-review — maintainer notes

Single-skill Claude Code plugin: `skills/grok-review/SKILL.md` drives the local `grok` CLI headlessly for read-only second-opinion reviews. The SKILL.md is the product — changes to it are behavior changes.

## Releasing a new version

Installed users only receive updates when the version bumps (the community marketplace pins by version too), so every substantive change ships as a release:

1. Bump `version` in BOTH `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` (keep them identical).
2. Add a human-readable entry at the top of `CHANGELOG.md` — write it for a user deciding whether to update, not a developer reading diffs: lead with what now works better or differently, bold the headline items, credit external reviewers/prior art where due.
3. Run `claude plugin validate ./` — must pass clean.
4. If the grok CLI invocation changed, smoke-test it live before pushing (a tiny brief that makes Grok read one file proves the flag set works end-to-end).
5. Commit and push. Doc-only tweaks (README wording, this file) don't need a version bump.

## Conventions that took seven review rounds to learn — don't quietly undo them

- Grok is read-only via four layers (tool allowlist, `--disallowed-tools`, `--deny` rules, optional kernel sandbox). The redundancy is deliberate: the tool lists fail open on ID drift.
- Every claim about grok CLI behavior in SKILL.md was verified against the installed CLI or its docs. When editing, re-verify rather than trusting memory — flags have already vanished between versions (`--no-auto-update`).
- The README "verified against" caveat lists exactly the things that drift; keep it in sync when flags change.
