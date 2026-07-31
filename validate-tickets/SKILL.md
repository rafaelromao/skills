---
name: validate-tickets
description: Use when the user wants to verify that the tickets produced by the `/to-spec` or `/to-tickets` skills are well-formed and ready for implementation, asks whether the tracker is in a consistent state, or asks whether a planned ticket is a vertical slice. The skill audits every open ticket against the ticket invariants in `INVARIANTS.md`, produces a gap map of (current state) vs (should-be state), and asks the user before refactoring any tickets.
---

# validate-tickets

The skill that says **no, not yet** to a tracker that has grown horizontal layer cake, dead edges, stale bodies, and parent references that point at the wrong sub-spec. The single source of truth for **what a well-formed ticket is** — every other ticket-producing skill should reference the invariants in `INVARIANTS.md` and never produce tickets that violate them.

The skill is **comprehensive**: every invariant is applied to every open ticket, every ticket state is read from the live tracker, every gap is reported. A short, vague report is a sign the audit ran with **premature completion**, not that the tracker is clean.

## Steps

1. **Inventory every open ticket** in the Wayfinder map's tracker. Pull state, labels, parent spec link, body, and the live `issue_dependencies_summary` from `/dependencies/blocked_by`. Read every open ticket — every ticket, not a sample. The number of open tickets is the lower bound of work.

2. **Apply every invariant** in [`INVARIANTS.md`](INVARIANTS.md) to every open ticket. For each (ticket, invariant) pair, classify the result into one of three states:
   - **pass** — the invariant holds for this ticket.
   - **gap** — the invariant fails; record the current value and the expected value.
   - **n/a** — the invariant does not apply (e.g. the ticket is the root spec, where the "single area" invariant doesn't apply). Record why.

3. **Build a gap map** as a single Markdown table with columns: `Ticket`, `Invariant`, `Status`, `Current`, `Should-be`. Group rows by ticket for human reading. Order invariants in the same order they appear in `INVARIANTS.md`.

4. **Score the tracker** by invariant — count `pass`, `gap`, `n/a` per invariant. Surface the invariant with the most gaps as the dominant issue. If the dominant invariant is body staleness or live-edge staleness, the report must include a `Tracer bullet` note calling out which tickets will mislead an agent on takeoff.

5. **Ask before refactoring.** Present a `## Open question` to the user with three explicit options: (a) report-only — produce the gap map and stop; (b) refactor-this-invariant-only — fix the dominant issue and re-audit; (c) refactor-all-gaps — fix everything the gap map found, but ask for confirmation per ticket category (rename, parent rewire, edge rebuild, body scrub, ticket close-as-duplicate). Default is **report-only**; never mutate without explicit human approval.

6. **If the user approves a refactor**, run it branch by branch and re-audit after each branch so a fresh invariant never regresses into a worse state.

The skill ends when the gap map is delivered and the user has chosen the action. **Do not declare the tracker clean until every invariant row is pass or n/a.**

## Information hierarchy

- `SKILL.md` (this file) is the audit-then-ask procedure. It is short on purpose: every step ends on a checkable **completion criterion**, and the audit's demand axis lives in `INVARIANTS.md`.
- [`INVARIANTS.md`](INVARIANTS.md) is the single source of truth for ticket quality. Reference every other ticket-producing skill's output against this file. If a ticket fails any invariant here, the ticket is **not ready for implementation**.

## Branching

Two branches diverge at Step 5:

- **Report-only** — print the gap map, the per-invariant scores, the dominant invariant, and stop. The human reads the map and decides what to do. Most runs end here.
- **Refactor** — proceed with the human-approved fix branches. Each fix branch is its own narrow slice; one fix branch per request, then re-audit. Never batch fixes across multiple invariants in a single pass.

## Leading words

- **Tracer bullet** — the spec's vertical-slice invariant. Every ticket should be a tracer bullet through every layer; if it isn't, the ticket is wrong.
- **Stale** — the most common failure mode: a body that lists blockers the live API no longer shows, a parent reference that points at the wrong sub-spec.
- **Foundation** — the live frontier on which an agent can claim work. Stale edges and stale body blockers both inflate the apparent frontier.
- **Comprehensive** — the demand axis on Step 2. Every invariant × every ticket. A short audit is premature completion.