# Story 04: Resolve a blocker

**As a** manager
**I want** to mark a blocker resolved when the issue is unblocked
**so that** my sidebar and topbar reflect what's actually still in the way today.

## Why this matters
Blockers are the highest-signal items on the board — the topbar stat (§3.1) and the sidebar Blockers section (§4) exist so the manager can scan "what's currently in the way" in seconds. That signal only stays useful if resolved blockers stop counting. Without a fast resolve action, stale blockers accumulate and the count loses meaning.

## Trigger
- **SPEC-STATED:** Click on the blocker node on a rail → tooltip expands to edit form (§5: "Edit node — Click node → tooltip expands to form").
- **PROPOSED — flag ambiguity:** From the sidebar Blockers card (§4), a "Mark resolved" affordance directly on the card. The spec only says the section contains "Cards for all unresolved `blocker` nodes" and does not specify whether cards are interactive. Default to "yes, the card has a resolve button" since the alternative (manager has to find the node on the rail every time) is friction the topbar/sidebar design seems built to avoid — but call this out for confirmation.

## UI flow
1. Manager clicks the blocker node on its project rail (or the card in the sidebar Blockers section, per the proposed-ambiguity above).
2. Tooltip expands into the inline edit form (§5).
3. Form exposes a `resolved` toggle/checkbox, currently false.
4. Manager flips the toggle to true and confirms (Save / Enter).
5. PATCH `/api/v1/nodes/{id}` is sent with `{resolved: true}` (§6).
6. On success:
   - Topbar blocker count decrements by 1 (§3.1).
   - The card disappears from the sidebar Blockers section (§4 — "unresolved" only).
   - The node remains on the rail at its original date.
   - Visual treatment on the rail changes (see "Visual treatment of resolved blockers" below).

## Form fields
- `resolved` (bool toggle) — the only field this flow changes. Other fields (`text`, `date`, `person_id`) are visible in the same edit form per §5 but editing them is the scope of a different story.
- **Not adding** a resolution note — the spec's Node model (§2) only has `text`; do not propose new fields here.

## Acceptance criteria
- [ ] Toggling resolved=true on a blocker Node persists via PATCH `/api/v1/nodes/{id}`.
- [ ] Topbar blocker stat decrements by 1 (§3.1).
- [ ] Sidebar Blockers section no longer renders a card for this Node (§4).
- [ ] The Node remains on its project rail at its original `date`.
- [ ] Visual treatment changes to indicate resolved state (PROPOSED — see below; flag as ambiguity).
- [ ] Re-opening (toggling resolved=false) restores the card to the sidebar and increments the topbar stat.
- [ ] Action is idempotent — saving resolved=true on an already-resolved blocker is a no-op (or a no-op at the UI level; backend may still PATCH, that's fine).

## Data implications
- Updates exactly one `Node` row, sets `resolved=true` (§2 — `resolved: bool` field, "for blockers").
- No new rows, no deletions, no Branch changes, no Project changes.
- The `resolved` field is documented in §2 as living on Node specifically for blockers; toggling it on a non-blocker node is undefined by the spec — out of scope here.

## API touchpoints
- **PATCH** `/api/v1/nodes/{id}` (§6) with body `{resolved: true}` (or `{resolved: false}` to re-open).

## Edge cases
- **Re-opening a resolved blocker** (resolved=true → false): supported via the same edit form. Card returns to sidebar, topbar count goes back up.
- **Resolving an already-resolved blocker:** idempotent. UI should treat as a no-op.
- **Resolution timestamp:** the spec's `Node` model (§2) does NOT include a `resolved_at` field. The system therefore does not capture WHEN a blocker was resolved. **Open question for the manager / spec author:** is this acceptable, or should the spec be amended to add `resolved_at: datetime | null`? Do not assume one way or the other; flag for confirmation.
- **Multiple blockers on the same project:** independent. Resolving one does not affect others. Sidebar shows N cards; resolving one leaves N-1.
- **Concurrency:** single-user system per §1, so no contention.

## Visual treatment of resolved blockers (PROPOSED — flag for confirmation)
The spec (§3.3) prescribes the unresolved-blocker visual: "Solid circle + ⚠ badge + amber ring." It does NOT prescribe how a resolved blocker should look on the rail. Two reasonable options:

1. **Keep the ⚠ badge, drop the amber ring.** Reads as "this was a blocker, now it's not." Preferred — preserves history while clearly de-emphasizing.
2. **Render as a normal `update` node**, with the ⚠ badge replaced by a checkmark ✓ or strikethrough.

Neither option is in the spec. Confirm with the spec author before implementing. If forced to pick, default to option (1) — it stays closest to the existing visual vocabulary in §3.3.

Note also that if a per-type filter chip is "Blockers" (§5: "Filter by type — Chips: All / Pivots / Milestones / Blockers / Planned"), the spec does not say whether resolved blockers are included or excluded under that chip. Flag this as a related ambiguity — handled by Story 08 (filter by node type), not here, but note the dependency.

## Out of scope
- Deleting the blocker node entirely (DELETE `/api/v1/nodes/{id}`) — semantically distinct ("this never should have been logged" vs "this is no longer blocking").
- Bulk-resolving multiple blockers from the sidebar.
- Auto-resolution heuristics (e.g. "after N days").
- Resolution notes / reason text (would require a new field on Node, not in §2).
- Notifications when a blocker is resolved (§8 explicitly excludes notifications).
- Tracking who resolved it / when — single-user system (§1), and `resolved_at` is not in §2.
