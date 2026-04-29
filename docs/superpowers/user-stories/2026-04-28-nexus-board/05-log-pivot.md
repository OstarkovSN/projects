# Story 05: Log a pivot — create a branch fork

**As a** manager
**I want** to record when a project changes direction and visualize the abandoned approach as a fork
**so that** I have a permanent visual record of pivots — the approaches we tried and dropped, and the new direction we're now pursuing.

## Why this matters

Scientific research projects pivot frequently — a hypothesis fails, a method proves intractable, or a finding redirects the work. Per spec §1, this is the *defining* feature of Nexus Board: pivots are rendered as literal git-style branch forks rather than buried in changelog text, so the manager can see at a glance what the team tried, what was abandoned, and what direction the project is on now.

## Trigger

Spec §5 calls this a **special flow** — distinct from the simple `+`-at-end-of-rail used for `update`/`milestone`/`blocker` and distinct from the click-beyond-today used for `planned`. The spec does **not** specify the entry-point UI surface.

**Proposal (FLAG: ambiguity — entry-point not specified in spec):** A dedicated "Log pivot" action surfaced on each project rail — either as a small fork-glyph button next to the `+` button, or as an item in a kebab/right-click menu on the rail. Recommendation: a discoverable secondary button (e.g. a fork-glyph icon) so the manager doesn't need to discover a hidden menu for one of the most important actions.

## UI flow

1. Manager triggers the pivot flow on a specific project rail (entry-point per Trigger above).
2. A modal (or inline form) opens with three required fields: **person**, **date**, **new-direction description**. The form is distinct from the simple add-update form and is explicitly labelled "Log pivot".
3. Manager fills the form and submits.
4. Backend performs the compound write (see Data implications): one `Branch` UPDATE, one `Branch` INSERT, one `Node` INSERT.
5. Frontend re-fetches (or optimistically updates) and re-renders the timeline. The just-active rail now shows: a curved abandoned branch dropping below the main line, a rotating-diamond pivot node at `fork_date`, a vertical dashed connector linking the diamond to the abandoned branch, the "ABANDONED APPROACH" SVG label, and the main rail continuing forward in the new direction with the pivot node visually anchoring the change.
6. The pivot also appears in the Live Feed sidebar section (spec §4) as the most recent activity.

## Form fields

| Field | Type | Maps to |
|---|---|---|
| person | select (from project's `people[]`) | `Node.person_id` on the new pivot node |
| date | date picker | `Branch.fork_date` (new branch) AND `Node.date` (pivot node) — these are equal by the §2 invariant |
| new-direction description | text | `Branch.label` (new branch) AND `Node.text` (pivot node, summarises the change) |

Single submit produces all three writes atomically.

## Acceptance criteria

- [ ] The currently-active `Branch` for the project is UPDATED: `is_active = false`, `abandoned = true`. Its existing `forked_from_branch_id` and `fork_date` are unchanged.
- [ ] A new `Branch` row is INSERTED with: `is_active = true`, `abandoned = false`, `forked_from_branch_id = <old branch id>`, `fork_date = <date from form>`, `label = <description from form>`.
- [ ] A new `Node` row is INSERTED with: `type = 'pivot'`, `branch_id = <new branch id>` (per spec §2 invariant — the pivot node lives on the **new** branch, not the old one), `date = <fork_date>`, `person_id = <selected person>`, `text = <description>`. `resolved` is irrelevant for non-blocker nodes.
- [ ] The three writes succeed or fail together (transactional). A partial failure must not leave the project with two active branches or with a pivot node on the wrong branch.
- [ ] The timeline re-renders to match spec §3.4: main rail continues forward from the pivot point in the new direction; secondary branch curves away via SVG cubic bezier; secondary branch runs horizontally to its existing terminus and dead-ends with a circle cap; secondary branch uses dashed stroke at opacity 0.55–0.7; "ABANDONED APPROACH" label is rendered as SVG `<text>`; vertical dashed connector links the pivot diamond to the branch fork point on the abandoned branch.
- [ ] The pivot node renders per spec §3.3: rotating diamond with amber→red gradient — distinct from circles used for other node types.
- [ ] The pivot is exactly one node per spec §2 invariant — there is no second "pivot" node on the old branch and no duplicate marker at the fork point.
- [ ] The pivot appears in the sidebar Live Feed (spec §4).
- [ ] If the project has people-filter chips disabled for the selected person (spec §4 — Filter people), the pivot diamond is hidden from the rail just like any other node from that person; the abandoned branch line itself remains visible (it's a structural rail element, not a per-person node).
- [ ] The "Pivots" type-filter chip (spec §5) shows the pivot when active and hides it when filtered out.

## Visual result (cite spec §3.4 line by line)

- **Main rail continues forward from the pivot point in the new direction** (§3.4 step 1) — the project rail does *not* end at the pivot; the new active branch carries it onward.
- **Secondary branch curves away (SVG cubic bezier), runs horizontally to the fork date, dead-ends with a circle cap** (§3.4 step 2) — the abandoned branch's tail is an explicit terminal cap, not a fade-out.
- **Secondary branch is drawn with a dashed stroke, reduced opacity (0.55–0.7)** (§3.4 step 3) — visually de-emphasised but still legible.
- **"ABANDONED APPROACH" label rendered as SVG `<text>` below the dead branch** (§3.4 step 4) — uppercase, anchored under the branch.
- **A vertical dashed connector links the pivot diamond to the branch fork point** (§3.4 step 5) — this is the visual signature that says "this diamond *is* the fork point of these two branches".
- **Pivot node itself: rotating diamond, amber→red gradient** (§3.3) — animated rotation distinguishes it from static node types.

## Data implications

A pivot is a **compound logical operation** with three database writes:

1. UPDATE on the previously-active `Branch`: set `is_active = false`, `abandoned = true`.
2. INSERT a new `Branch`: `is_active = true`, `abandoned = false`, `forked_from_branch_id = <old branch id>`, `fork_date = <form date>`, `label = <form description>`.
3. INSERT a new `Node`: `type = 'pivot'`, `branch_id = <new branch id>` (NOT the old one — see spec §2 invariant), `date = <fork_date>`, `person_id`, `text`.

These three writes must be transactional. The spec §2 invariant ("every Node of type `pivot` is the fork point between two Branch records — the old (abandoned) and the new (active). The node lives on the new branch at `fork_date`.") is the load-bearing constraint: any implementation must guarantee it.

**FLAG: ambiguity — pivot atomicity in spec §6.** Spec §6 lists `POST /api/v1/projects/{project_id}/branches` with the comment "created automatically on pivot", which strongly implies the client should not call this endpoint directly during a pivot flow. The spec does not explicitly state that `POST /nodes` with `type=pivot` performs the compound operation, but this is the only consistent reading. **Proposal:** the backend treats a `POST /nodes` request with `type = 'pivot'` as the pivot operation: it performs all three writes in a single transaction and returns the new node (with the new branch id embedded so the client can refresh its branch list). Clients SHOULD NOT call `POST /branches` directly to simulate a pivot.

## API touchpoints

- `POST /api/v1/projects/{project_id}/nodes` (spec §6) with body `{ type: 'pivot', date, person_id, text }`. Backend performs the compound operation per Data implications and returns the created pivot node together with sufficient information to refresh the branch list (e.g. the new branch object embedded, or the client follows up with `GET /api/v1/projects/{project_id}/branches`).
- `GET /api/v1/projects/{project_id}/branches` (spec §6) — the client refreshes the branch list after a pivot so the abandoned branch and new active branch render correctly.
- `POST /api/v1/projects/{project_id}/branches` (spec §6) — **not** invoked from the client during the pivot flow; spec §6 explicitly notes branches are created automatically on pivot.

## Edge cases

- **Pivoting a project with no prior nodes / no prior active branch.** Should not happen in practice — every project has at least an implicit main branch (`forked_from_branch_id = null`) per the data model. The pivot flow simply abandons that main branch and creates a new active one.
- **Pivoting a branch that's already abandoned.** Should not be allowed — only the currently-active branch of a project can be pivoted. The form's project-rail entry-point should only be available when an active branch exists. **Proposal:** backend validates and returns 409 / 400 if the only active branch is missing or already marked abandoned.
- **Future-dated pivot.** Spec doesn't address this. **Proposal (FLAG: ambiguity):** pivots must have `date <= today` — a pivot is a record of a decision that has been made, not a planned future event. Future-dated pivots would conflict with the visual semantics of spec §3.2 (past = solid rail, future = dashed marching-ant rail) since a pivot diamond on a dashed future segment would be visually nonsensical. If the manager wants to plan a possible future direction change, they should use a `planned` node, not a pivot.
- **Future-dated nodes existing on the old branch at the time of pivot.** Spec doesn't address this. **Proposal (FLAG: ambiguity):** these nodes remain on the old (now abandoned) branch in the database; visually the abandoned branch only renders historical rail up to its dead-end (per spec §3.4 step 2 — the secondary branch "runs horizontally to the fork date, dead-ends with a circle cap"), so future-dated nodes on the abandoned branch are not rendered by default. If the manager wants their previously-planned work to continue on the new direction, they must explicitly re-create those nodes on the new branch. (The simpler alternative — auto-moving future nodes to the new branch — is plausible but introduces silent data movement and complicates the §2 invariant; recommend deferring to a clarification with the manager.)
- **Multiple pivots in a row (chained pivots).** Fully supported by the data model — each new pivot abandons the then-active branch and creates a fresh active branch with `forked_from_branch_id` pointing to the just-abandoned branch. Visually, this produces a stacked series of curved abandoned branches dropping below the main rail. The current rendering rules in spec §3.4 must scale to N abandoned branches without overlap — **FLAG: ambiguity — vertical layout of multiple abandoned branches is not specified**, but this is a rendering concern, not a data-model concern, and is out of scope for this story.
- **Pivot node text rendering on hover.** Per spec §5 ("Edit node — Click node → tooltip expands to form"), clicking the pivot diamond opens the same edit-tooltip as other nodes. Editing the `text` field is safe and does not affect branches. Editing `date` or `branch_id` would break the §2 invariant (see Out of scope).

## Out of scope

- **Un-pivoting / merging branches back together.** Once a branch is abandoned it stays abandoned. There is no "merge" operation.
- **Editing a pivot node's `date` or `branch_id` after creation.** Doing so would invalidate the §2 invariant and require restructuring the branch records. The edit-node flow (§5) should treat these fields as read-only on pivot nodes; only `text` and `person_id` are editable.
- **Deleting a pivot node.** Out of scope here — would require either deleting the new branch (and orphaning its subsequent nodes) or reactivating the old branch. Defer.
- **Drag-to-extend the abandoned branch's terminus.** Not interactive (spec §3.5, §8 — drag-to-extend is a future feature for *active* rails; abandoned branches are static).
- **Visualising N stacked abandoned branches without overlap** (rendering concern, not story scope).
- **Self-reporting pivots by team members** (spec §8 — out of scope for v1).
