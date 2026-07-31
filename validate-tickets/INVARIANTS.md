# Ticket Invariants

The **single source of truth** for what a well-formed ticket is. Every ticket a planner publishes — whether by `/to-spec`, `/to-tickets`, or hand — must satisfy every invariant below. The list is ordered: a tracer-bullet failure (Invariant 1) is the worst; a title-prefix drift (Invariant 8) is the cheapest to fix but a sign the audit ran lazily.

Every invariant is **checkable** — the validator can ask the live tracker a single question and decide pass / gap / n/a. Every invariant is **exhaustive** — every open ticket is checked, not a sample. The pair matters: a checkable-but-not-exhaustive invariant is an audit that declares done after one ticket; an exhaustive-but-not-checkable invariant is hand-waving.

The invariants are referenced from the planner skills (`/to-spec`, `/to-tickets`) so they prevent these failures at creation time, not just catch them later.

---

## 1. Vertical slice (tracer bullet)

A ticket's `## What to build` and `## Demoable behavior` together describe a complete **end-to-end path**: schema + host + worker or UI or persistence + tests, all exercised by the same runnable assertion. The ticket does not "implement the X type", "wire up the Y call", or "add the Z endpoint" in isolation; it makes one user-visible capability work on the production code path.

**Check.** The ticket's body contains a `## Demoable behavior` section. The acceptance criteria include a line of the form "Vertical slice produces the demoable behavior end-to-end on the production code path." No acceptance criterion says "implements the X type in module M" or "wires up the Y handler". The body has no `### Phase 1` / `### Phase 2` subsection, no `step 1` / `step 2` numbering across the slice, and no horizontal-decomposition pattern (`a/01`, `a/02`, `a/03` titles in the same area).

**Why this is the first invariant.** A horizontal slice is the failure mode that forces the cleanups the validator exists to prevent — cross-area dependencies, dead edges between siblings of an unfinished horizontal decomposition, agents picking up a half-slice that lacks the rest.

---

## 2. Single area

The ticket belongs to exactly one parent sub-spec. Its live `issue_dependencies_summary.blocked_by` edges only point at **siblings of the same sub-spec**, except for tickets whose sub-spec is a rehearsal sub-spec (which legitimately cross areas). The tracker will enforce this; the body must reflect it.

**Check.** For every ticket except a root spec or rehearsal sub-spec, all live blockers share the same `sub-spec` as the ticket itself. The body's `## Part of #<n>` heading references an open issue whose body is a sub-spec for the ticket's area.

**Example.** A ticket labelled with area A that has live blockers in areas B, C, D — that ticket is a multi-area rehearsal slice and lives in the rehearsal area, not in A.

---

## 3. Sized for a single 400k-token context

The ticket's scope — every layer it touches, every file it might modify, every test it must add — fits inside one fresh agent context window. The ticket does not implement two loosely related features "while you're at it". The acceptance criteria list ≤6 items, and the body does not contain more than one `and` clause between `## What to build` and `## Demoable behavior`.

**Check.** Count the acceptance criteria. Count the `and also` / `, and ` connectors in the `What to build` paragraph. If either count is high, the slice is too large and should be split by the planner (`/to-tickets` knows how).

**Why this invariant exists.** Tickets that exceed one context window trigger premature completion: the agent runs out of attention before the tests pass and the slice lands half-done, accumulating the next round of stale edges.

---

## 4. Demoable behavior, not layer-by-layer

The `## Demoable behavior` section says **what the slice makes work for the user**, not what it implements in code. "Pressing arrow keys moves selection between features with a visible (non-color) gesture acknowledgement" passes; "implement Selection::next() and Selection::prev() with visible acknowledgement markers" fails.

**Check.** The `Demoable behavior` section describes an observable end-state (the user can do X; the test asserts Y), not a code surface (we add a class / function / module). A code surface description is a sign the ticket is a layer task wearing a slice costume.

---

## 5. Body `## Part of #<n>` points at the live sub-spec

The body's `## Part of #<n>` heading points at the ticket's actual sub-spec, not at a closed parent-area wrapper or a no-longer-current spec area. The sub-spec referenced is open.

**Check.** Read the body's `## Part of #<n>` line. Resolve the number to a live issue. The live issue's body must be a sub-spec (the `documentation` label or a sub-spec pattern in its title).

**Example.** A leaf that points at a closed parent-area wrapper instead of the live sub-spec misleads every agent that reads the body.

---

## 6. `## Blocked by` body matches the live dependency graph

The body's `## Blocked by` block lists exactly the issues returned by the live `/dependencies/blocked_by` endpoint, with `(open)` or `(closed)` state shown. None of those entries point at closed superseded tickets.

**Check.** Parse the `## Blocked by` block, extract every `#<n>` mention, compare against the live blockers list. Any mismatch is a gap.

**Why this invariant exists.** Stale blocker lists are the most common way an agent misreads the foundation frontier. The body must be regenerated whenever the live graph changes; the planner skills should never produce a body whose `## Blocked by` is hand-written.

---

## 7. Live `issue_dependencies_summary.blocked_by` is foundation-correct

The live edges on the ticket only point at open blockers, and those blockers either share the ticket's sub-spec (within-area edges) or, for rehearsal-area tickets only, point at the cross-area slices the rehearsal consumes. No dead edges, no cross-area edges on non-rehearsal tickets.

**Check.** Fetch the live `/dependencies/blocked_by` response. For every blocker, fetch its state. Drop any closed (dead) edge — those are gaps. For every remaining open edge, check the blocker's sub-spec. If it doesn't match this ticket's sub-spec and this ticket isn't a rehearsal-area ticket, it's a cross-area gap.

**Why this invariant exists.** Dead edges and cross-area edges are the failure mode this invariant exists to catch. The planner must never add a dead edge and must never add a cross-area edge to a non-rehearsal ticket.

---

## 8. Title classification matches the parent sub-spec

If the tracker uses a title-classification convention (a band prefix, an area code, or any other leading marker), the ticket's marker must match the parent sub-spec's marker. A title with a stale or missing marker is a stale-ticket signal — the planner renamed the parent but not the leaf.

**Check.** Extract the marker from the ticket title and the parent sub-spec title. They match. If the tracker has no title convention, this invariant is `n/a` for every ticket.

---

## 9. No horizontal-decomposition pattern

The ticket body and title do not present a horizontal layer-cake slice. The forbidden patterns are:
- Title or body containing `01a`, `01b`, `01c` style decomposition.
- `### Phase 1` / `### Phase 2` / `### Phase 3` subsections inside a single body.
- "Step 1, step 2, step 3" sequencing inside one ticket's acceptance criteria.
- A `## What to build` paragraph that names two distinct features glued together ("X and Y").

**Check.** Regex search for the forbidden patterns. If any matches, the ticket is a horizontal slice.

---

## 10. No closed superseded ids referenced in body

The body must not mention `#NN` where `NN` belongs to the closed-superseded set for this tracker. The validator computes the set live from `state=closed` + `duplicate` label (or whatever convention the tracker uses to mark tickets closed-as-superseded).

**Check.** Compute the closed-superseded set. Regex-find every `#\d{2,4}` in the body. Any hit is a gap. The fix is a swap (`#<superseded>` → `#<sub-spec>`) or a delete, never a re-open.

---

## 11. Reachable from the root spec

Every ticket except the Wayfinder map and the root spec is reachable by walking `## Part of #<n>` from the root spec through a chain of open sub-specs. Orphan tickets (no parent sub-spec, or the parent sub-spec is closed) are gaps.

**Check.** Build the open sub-spec graph from every `## Part of #<n>` heading. For every open ticket, walk from the ticket's parent to the root spec. Tickets with no path are gaps. The fix is either adding a `## Part of` heading or closing the orphan as a duplicate.

---

## Invariant catalog summary

| # | Invariant | Catches |
|---|---|---|
| 1 | Vertical slice (tracer bullet) | Horizontal layer cake; "implement X in module M" |
| 2 | Single area | Cross-area dependencies; orphan-area tickets |
| 3 | Sized for a single 400k-token context | Tickets that exceed one context window |
| 4 | Demoable behavior, not layer-by-layer | Code-surface descriptions |
| 5 | `## Part of` points at live sub-spec | Stale parent references |
| 6 | `## Blocked by` body matches live graph | Stale blocker lists; dead-blocker illusion |
| 7 | Live edges are foundation-correct | Dead edges; cross-area edges on non-rehearsal |
| 8 | Title classification matches parent sub-spec | Stale title markers after a rename |
| 9 | No horizontal-decomposition pattern | "Phase 1/2/3" or "01a/01b/01c" titles or bodies |
| 10 | No closed superseded ids referenced | Body mentions of closed duplicates |
| 11 | Reachable from the root spec | Orphan tickets with no parent sub-spec |

When the validator reports gaps, the table above is the legend: each row tells you the failure mode, the check, and the fix.
