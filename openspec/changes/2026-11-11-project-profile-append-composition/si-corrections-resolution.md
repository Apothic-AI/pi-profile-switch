# SI binding corrections — resolution map

The seven binding corrections from the SI decision (`si-approved-plan-and-s0-brief.md`) and where each one is
resolved. Read this first when reviewing the proposal: each row states the hazard, the resolution, and the
artifact that makes it normative.

## 1. A raw reference cannot be mapped to a single scope

**Hazard.** `referenceOrigins` as a literal → one scope map loses the second scope for an identical literal, and
glob/literal or alias/name overlaps get misattributed.

**Resolution.** Contributor **sets** are unioned, not chosen. The composer emits
`referenceContributors: raw literal → "global" | "project" | "both"`, and the resolver unions the contributor
scope of every reference that produced an identity while performing the single expansion. An exact literal in
both halves yields `both`; a global glob overlapping a project literal yields `both`; an extension alias in one
half overlapping an entry name in the other yields `both`.

**Normative.** `resource-reference` "Composed reference declarations" (requirement + six scenarios) and
"Reference provenance in the activation plan"; matrix 5 rows 1–6. **One native expansion per kind**, asserted by
the "Each kind expands once" scenario and by the expansion-count test in `test-plan.md` §3.

## 2. Extension provenance must be collision-safe

**Hazard.** An identifier-only map merges distinct entries: identifiers are package-derived or path-derived and a
local file wins an identifier collision with a warning, while selection deduplicates by resolved entry path.

**Resolution.** Provenance is keyed by **resolved entry path** — the selection's own dedupe identity — with the
resolved identifier carried as a value field (`extensionOrigins[entryPath] = { id, contributors }`). The design
states explicitly that an identifier-only key must not be used. Selection, naming, dedupe, and collision warning
semantics are unchanged.

**Normative.** `resource-reference` "Reference provenance in the activation plan" and its "Extension provenance
is collision-safe" scenario; `design.md` export surface (`extensionOrigins`); matrix 5 row 7.

## 3. Tool raw references are not resolved tool provenance

**Hazard.** Boot expansion passes unmatched literals through (`literalMustExist: false`), so presence in the boot
`tools` list proves nothing about a runtime tool. Reporting those as provenance invents tools.

**Resolution.** Two coordinates, kept distinct: `toolOrigins[name] = { contributors, at }` where `at: "boot"`
means the identity resolved against built-in names, and a raw reference that matched nothing stays a **pending
session-start reference** with its contributors and is never relabelled `unknown`. When per-tool provenance is
promised, contributors propagate through the **existing** session-start expansion
(`src/switching/tool-references.ts` + its existing caller) via an optional contributor map; no new runtime
reader, backend, or extension is added. If that propagation is dropped, status limits its claim to raw pending
references — the alternative is documented rather than silently weakened.

**Normative.** `resource-reference` "Reference provenance in the activation plan" (three tool scenarios);
`design.md` export surface (`ToolExpansion.contributors`, `expandToolReferences` signature); matrix 5 rows 8–10;
tests in `test-plan.md` §4 covering extension-tool literal and glob, global/project overlap, empty matches,
overlay removal, and preserved activation selection.

## 4. Unrestricted kinds must not be falsely attributed

**Hazard.** Project references dropped because their kind stays undeclared were described as if they contributed,
and overlay-removed resources could linger in an origin trace and look selected.

**Resolution.** Status shows the kind as **unrestricted with native selection**, and lists the project references
as **unapplied, unvalidated** references with the file that requested them. Nothing is attributed to them, no
`unknown-reference` or zero-match claim is made, and no discovery validation is promised (the composer is
IO-free; a second resolver input for an advisory was rejected). Provenance describes the **effective** selection,
so an overlay-removed identity is absent from origins and may appear only as a disabled entry.

**Normative.** `resource-reference` "Composite declaration intent and unapplied references" (unapplied-reference
paragraph + four scenarios) and "Reference provenance in the activation plan" (overlay scenario);
`in-session-switch` "Composed activation observability" (unrestricted and overlay scenarios); matrix 1 rows 1–3,
matrix 5 rows 11–12.

## 5. Launch, reload, adoption and rollback truth

**Hazard.** "Restart/reload" treated as one operation, and rollback described as free while `append` files exist.

**Resolution.** Four distinct operations are recorded: package install ≠ loaded runtime; a new launch reads
installed code only when the real launch source resolves to that installation; a native Pi reload replays the
saved launch plan while `/profile reload` re-resolves definitions and rewrites managed runtime files; `/profile
use` runs its existing path. Extension code already imported cannot gain a new command implementation before
re-import, so a lane served by old code warns and **replaces** — the hazard this change removes. No mandatory
restart, and no silent model/thinking/profile/source switch. **Rollback is BLOCKED** while affected `append`
consumers would be served by old replace-with-warning code; the fixed consumer conversion (remove/rename, or
convert to replacement with effective content materialized) is a separately reviewed operation, never automatic
destructive cleanup.

**Normative.** `design.md` "Adoption and rollback truth" (deployment truth); `in-session-switch` "Composed
activation observability" (status must not imply adoption); matrix 6; a release-note task in `tasks.md` §5 with a
mechanical check.

## 6. Package ancestry is not a divergence

**Hazard.** A pin difference read as rollback/divergence, then escalated into an operator publication decision.

**Resolution.** Recorded fact: package commit `cee2d72b595ab966339472f3f7d897d92d2b1dc5` (pinned by yellowstone
root `4a47e457`) is the reviewed forward child of `369481704a52bdd22a7fe7cfe57aec392861737d` in the isolated
combined publication repo. Absence of those objects from another Git object store proves only local absence — not
absence of a clone, and not divergence. This change binds the then-current verified base (`4a47e457` /
tree `83fe68ec64db8ffbedd979e9b553685a5e08c43d`), preserves FT39 docs and every unrelated pin/source change, and
performs no fast-forward of an older base, no repin, no fetch, no push. Publication stays SI's reviewed forward
commit.

**Normative.** `result.md` base/pin record; `tasks.md` base-verification task; this file. No spec change: the
package pin is not behavior.

## 7. Project-scope enforcement and `!`-glob grammar stay owned follow-ups

**Hazard.** Framing the composition feature as closing the unguarded project-resource route, or shipping an
unrelated grammar tightening under this change's name.

**Resolution.** Both are recorded and neither is implemented. A trusted project's `.pi/skills/*` and
`.pi/extensions/*` stay visible under every profile, `append` included (ADR-0011: exclusion entries are scoped to
their own settings scope, generated settings are global scope, no in-session removal API), and the proposal says
so as an explicit non-goal. The only new rejection is a `!`-prefixed **glob** in the append project half, where
the reading would invert the selection; global halves and every replacement definition keep today's behavior,
pinned by a test that asserts we did not tighten them. The empty project-resource gap and the generic grammar bug
are named as follow-ups with owners (SI/overlord, and a separate change), not as worker approvals.

**Normative.** `proposal.md` "Non-Goals"; `resource-reference` "Negated glob reference guard in a composed
project half" (two scenarios); matrix 7 rows 4–6; `docs/prd.md` non-goal on project-resource narrowing left
untouched.

## Additional SI decisions applied without a numbered row

- **Explicit parser retention, not schema-only.** The strategy is parsed, validated, and assigned field by field;
  the schema entry is documentation, editor, and example support. A schema-only change makes the field
  known-but-unassigned, so a project file saying `append` would silently replace. Pinned by a parse-retention
  test that a schema-only change fails.
- **No mode-style authorisation or digest gate.** The existing trust decision covers project files; adding a
  reapproval surface would introduce a second authority and drift state for the same decision.
- **Global `replace` accepted and inert**, global `append` rejected, justified by "a strategy value must be true
  in the scope that carries it" rather than by symmetry.
- **Source value unchanged.** A composed definition reports the existing project source; no fourth source value,
  because five persistence sites key off the current three.
