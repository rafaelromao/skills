# ~/projects/skills

Personal collection of agent skills. Each skill lives in its own folder with a `SKILL.md` (the entry point the agent runtime reads) and any disclosed reference files (`GLOSSARY.md`, `INVARIANTS.md`, …) that the skill loads on demand via context pointers.

A symlink at `~/.agents/skills/<skill-name> → ~/projects/skills/<skill-name>` makes each skill discoverable to the agent runtime. Edits to the real folder take effect immediately; no re-linking needed.

## Skills

### [`validate-tickets/`](validate-tickets/)

The single source of truth for *what a well-formed ticket is*. Audits every open ticket in a Wayfinder tracker against the invariants in `INVARIANTS.md`, produces a gap map of (current state) vs (should-be state), and asks the user before refactoring any tickets. Default branch is **report-only** — never mutates without explicit approval. User-invoked.