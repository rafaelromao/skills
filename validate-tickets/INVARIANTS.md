# Ticket Invariants

The **single source of truth** for what a well-formed ticket is. Every ticket a planner publishes — whether by `/to-spec`, `/to-tickets`, or hand — must satisfy every invariant below. The list is ordered: a tracer-bullet failure (Invariant 1) is the worst; a title-classification drift (Invariant 8) is the cheapest to fix but a sign the audit ran lazily.

Every invariant is **checkable** — the validator can ask the live tracker a single question and decide pass / gap / n/a. Every invariant is **exhaustive** — every open ticket is checked, not a sample. The pair matters: a checkable-but-not-exhaustive invariant is an audit that declares done after one ticket; an exhaustive-but-not-checkable invariant is hand-waving.

The invariants are referenced from the planner skills (`/to-spec`, `/to-tickets`) so they prevent these failures at creation time, not just catch them later.

Every invariant is also **parser-compatible**: the body shapes it requires match what the consumer-side resolver recognises (dependency resolution, specification expansion, acceptance-criteria parsing). A body the validator passes is one the consumer can also read without guessing. Inline phrases and off-heading references are explicitly rejected because the consumer treats them as incidental prose, not authoritative declarations.

---

## 1. Vertical slice (tracer bullet)

A ticket's `## What to build` and `## Demoable behavior` together describe a complete **end-to-end path**: schema + host + worker or UI or persistence + tests, all exercised by the same runnable assertion. The ticket does not "implement the X type", "wire up the Y call", or "add the Z endpoint" in isolation; it makes one user-visible capability work on the production code path.

**Check.** The ticket's body contains a `## Demoable behavior` section. The acceptance criteria include a line of the form "Vertical slice produces the demoable behavior end-to-end on the production code path." No acceptance criterion says "implements the X type in module M" or "wires up the Y handler". The body has no `### Phase 1` / `### Phase 2` subsection, no `step 1` / `step 2` numbering across the slice, and no horizontal-decomposition pattern (`a/01`, `a/02`, `a/03` titles in the same area).

**Why this is the first invariant.** A horizontal slice is the failure mode that forces the cleanups the validator exists to prevent — cross-area dependencies, dead edges between siblings of an unfinished horizontal decomposition, agents picking up a half-slice that lacks the rest.

---

## 2. Single area

The ticket belongs to exactly one parent sub-spec. Its live `issue_dependencies_summary.blocked_by` edges only point at **siblings of the same sub-spec**, except for tickets whose sub-spec is a rehearsal sub-spec (which legitimately cross areas). The tracker will enforce this; the body must reflect it.

**Check.** For every ticket except a root spec or rehearsal sub-spec, all live blockers share the same `sub-spec` as the ticket itself. The body's parent reference (Invariant 5) names an open issue whose body is a sub-spec for the ticket's area.

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

## 5. Body references its live parent spec

The body references the ticket's live parent spec — the immediate sub-spec for a leaf, or the root spec for a sub-spec. The reference must use a heading the consumer-side parser recognises so a resolver can read it without inline-phrase guessing.

Accepted heading shapes (case-insensitive substring match on the heading text):

- Any H2 whose heading contains the word `parent` (`## Parent`, `## Parent area`, `## Parent spec`).
- Any H2 whose heading starts with `Part of` (`## Part of #<n>`).
- Any H2 whose heading contains the words `Sub-spec of` or `Sub-issue of`.

The heading body is `#<n>`, optionally preceded by a label, optionally with one trailing comment line. Reference forms accepted inside the heading body:

- `#<n>` shorthand.
- Full `/issues/<n>` URL.
- `[<text>](https://.../issues/<n>)` titled link.

The heading must resolve to an open issue that lies on the path from the ticket up to the root spec. Prose `see #<n>` mentions in `## Planning context` are an acceptable alternative when no heading is used; the consumer-side parent-section matcher picks them up too.

**Check.** Find every parent-reference heading or prose mention. Resolve each to a live issue. Every referenced issue must be open and must lie on the parent path. A reference to a closed parent-area wrapper, a no-longer-current spec area, or any issue off the parent path is a gap.

**Example.** A leaf whose body uses `## Part of #N` where N is a closed parent-area wrapper instead of the live sub-spec id misleads every agent that reads the body.

---

## 6. `## Blocked by` body matches the live dependency graph

The body lists every blocker the consumer-side resolver needs, under a heading it can recognise. The heading must be one of `## Blocked by`, `## Depends on`, or `## Blocked-by` (case-insensitive, leading whitespace tolerated). Body entries under the heading are one of:

- `- #<n>` bare bullet.
- `- [#<n>](https://.../issues/<n>)` link bullet.
- `- [<text>](https://.../issues/<n>)` titled link, with optional trailing annotation. The bullet prefix is required when the titled link carries a trailing annotation.
- `[<text>](https://.../issues/<n>)` titled link on a line by itself (no bullet, no annotation).
- Table-row equivalents inside a markdown table when the row carries `- [ ]`-style entries.

The list must equal the live `/dependencies/blocked_by` endpoint response, with `(open)` or `(closed)` state shown for each entry. **Inline phrases** such as `Blocked by #123`, `Depends on #123`, or `Child Issues: #123` outside an explicit heading are NOT recognised by the resolver and constitute a gap; they frequently appear in prose mentions inside child-list annotations like `- #10 (blocked by #2319)` and treating them as authoritative makes unrelated parents appear blocked.

**Check.** Confirm the blocker heading uses one of the three accepted H2 names. Parse the heading body using the accepted entry shapes; extract every `#<n>` mention. Compare against the live blockers list. Any mismatch is a gap. Inline prose mentions of blockers outside the heading are gaps.

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

## 10. Reachable from the root spec

Every ticket except the Wayfinder map and the root spec is reachable by walking the parent-reference chain (Invariant 5) from the root spec through open sub-specs. Orphan tickets (no parent reference, or the parent reference is closed) are gaps.

**Check.** Build the open sub-spec graph from every parent reference. For every open ticket, walk from the ticket's parent to the root spec. Tickets with no path are gaps. The fix is either adding a parent reference or closing the orphan as a duplicate.

---

## 11. Body references its live children (parent back-reference)

A spec, sub-spec, or any ticket that has children must list every live child via one of the accepted forms the consumer-side parser recognises. Each form must use the canonical heading vocabulary and entry shapes; inline prose mentions of children outside an explicit heading are NOT recognised.

Accepted parent-to-child forms:

- **Body heading.** Any H2 whose heading contains the word `child` or `children` (case-insensitive substring), e.g. `## Children`, `## Child Issues`, `## Leaf children`, `## Children in this area`, `## Sub-children`. The heading body is either a markdown table or a bullet list of entries using the canonical shapes from Invariant 6 (`- #<n>`, link bullet, titled link with trailing annotation after bullet prefix).
- **GitHub sub-issue link.** The tracker's `/repos/{o}/{r}/issues/{n}/sub_issues` endpoint, when enabled.
- **Comment with `#<n>` references.** The tracker's `/repos/{o}/{r}/issues/{n}/comments` endpoint, when neither body nor sub-issues work. Each child must appear as a `#<n>` reference in at least one comment.

For a sub-spec the children are its leaf slices; for a Wayfinder map the children are its planning tickets; for a root spec the children are its area sub-specs. Whatever the parent's role in the hierarchy, the listed children must equal the live children of this parent. Any child reference in the body that falls under a `## Blocked by`/`## Depends on`/`## Blocked-by` heading, or under a parent-reference heading (`## Parent`/`## Part of`), is NOT counted as a child — the consumer skips blocker and parent sections when harvesting children.

**Check.** Determine the parent's expected children. Compare against the three forms above. If the body table, the sub-issue field, and the comment harvest are all empty, the parent fails this invariant. If either form lists a child that is closed or off the parent path, the parent fails. If a live child is missing from every form, the parent fails.

**Why this invariant exists.** A parent that doesn't list its children breaks the agent takeoff: a leaf can be correctly referenced from the root, but the root can't reconstruct its child tree. The three accepted forms match what the consumer's children harvest recognises; the validator accepts any of them, so the planner can pick the convention that fits the tracker.

**Example.** A root spec that lists `#N1..#Nk` in its `## Children` table but forgets to mention a live sub-spec `#Nx` ships a child the planner thinks doesn't exist. The validator catches it.

---

## 12. No off-heading issue references in body

The body must not contain recognisable issue references (`#<n>` shorthand or `/issues/<n>` URL) outside an accepted heading — specifically, outside the parent-reference headings (Invariant 5), the children-list headings (Invariant 11), and the blocker headings (Invariant 6).

**Check.** Parse the body into H2 sections using the same walker the consumer uses. For every reference found outside a recognised parent/children/blocker heading, mark it. The reference is allowed only inside one of those headings or inside `## Planning context` (which the consumer treats as prose, not as an authoritative declaration).

**Why this invariant exists.** The consumer deliberately ignores inline phrases such as `Blocked by #123` or `Children: #123` outside a heading — they are incidental prose mentions across the tracker. Treating them as authoritative made unrelated parents appear blocked or made child counts balloon. A ticket body that puts references outside the heading vocabulary is one the consumer cannot read; the validator flags it so the planner can move the reference under a heading.

---

## 13. Canonical Specification body (parents only)

A ticket that has children and is intended to be a Specification carries the canonical body shape the consumer's `IsSpecification` recognises: both `## Problem Statement` and `## Solution` H2 headings (case-insensitive, leading whitespace tolerated). Either heading alone is not enough — a lone `## Solution` heading in an ordinary issue must not be mistaken for a Specification. `## User Stories` is presentation and does not contribute to the canonical-shape signal.

**Check.** For any ticket that lists children in its body (Invariant 11), confirm both `## Problem Statement` and `## Solution` headings are present. The check is `n/a` for tickets without children.

**Why this invariant exists.** The consumer's Specification detector scans the canonical shape **after** the children-list probe. A parent that lists children but lacks the canonical shape is still detected (via the children probe) and still expands correctly. The canonical shape exists so that an agent reading the parent body sees the contract the consumer will treat it as. A parent that has children AND carries the canonical shape is detectable by both probes; a parent that has children but lacks the canonical shape is detectable only by the children probe — both are valid, but the former is the convention the consumer's `## Parent` → children path expects.

---

## 14. Acceptance criteria use the canonical H2 + checkbox shape

When the ticket declares acceptance criteria, the body carries them under `## Acceptance criteria` H2 with `- [ ]` (or `- [x]`) checkbox bullets. Each bullet line is a runnable assertion the consumer's T1 oracle parses for `go test -run` (or analogous test invocation) shape.

**Check.** If the body contains a `## Acceptance criteria` heading, every entry under it is a `- [ ]` or `- [x]` bullet. If the body does not contain the heading but the ticket declares any acceptance criteria anywhere, the validator reports a gap. The check is `n/a` if the ticket declares no acceptance criteria.

**Why this invariant exists.** The consumer parses the AC section with a heading-only contract; off-heading ACs are not seen, so the T1 oracle cannot run them. Forcing the canonical shape keeps tickets consistent across planners.

---

## Invariant catalog summary

| # | Invariant | Catches |
|---|---|---|
| 1 | Vertical slice (tracer bullet) | Horizontal layer cake; "implement X in module M" |
| 2 | Single area | Cross-area dependencies; orphan-area tickets |
| 3 | Sized for a single 400k-token context | Tickets that exceed one context window |
| 4 | Demoable behavior, not layer-by-layer | Code-surface descriptions |
| 5 | Body references live parent spec | Stale or wrong parent references; non-parser headings |
| 6 | `## Blocked by` body matches live graph | Stale blocker lists; inline-phrase false positives |
| 7 | Live edges are foundation-correct | Dead edges; cross-area edges on non-rehearsal |
| 8 | Title classification matches parent sub-spec | Stale title markers after a rename |
| 9 | No horizontal-decomposition pattern | "Phase 1/2/3" or "01a/01b/01c" titles or bodies |
| 10 | Reachable from the root spec | Orphan tickets with no parent sub-spec |
| 11 | Body references live children | Parents with stale or missing child lists; non-parser headings |
| 12 | No off-heading issue references | Prose mentions of `#<n>` outside canonical headings |
| 13 | Canonical Specification body (parents only) | Parents missing `## Problem Statement` + `## Solution` |
| 14 | Acceptance criteria use canonical H2 + checkbox | ACs declared off-heading or in non-canonical shapes |

When the validator reports gaps, the table above is the legend: each row tells you the failure mode, the check, and the fix.