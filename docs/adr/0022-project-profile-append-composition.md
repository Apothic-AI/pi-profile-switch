# Project profile append composition

## Context

A project profile replaces its same-named global profile in full. Precedence is decided by filename before any
definition is read, so the global definition is not merely overridden — it is unreachable, and a project that
wants the team-wide selection plus one project-specific skill has to copy the whole definition and keep the copy
in sync by hand. Duplicated copies drift, and a copied field is indistinguishable from a deliberate project
decision.

The rejected alternatives below show what the design must not become: a parent chain, a deep merge, or a second
approval authority. The operating constraint is narrower than any of those: let a project file *add* to what the
global file already allows, deterministically, at one resolution point, with the existing trust decision and the
existing three-state declaration grammar.

## Decision

A profile definition may declare an optional composition strategy. Accepted values defer to the authoritative
profile schema. In a **project** definition, `append` composes the definition with its same-named global
definition; every other accepted value, and an absent value, keeps complete replacement. `append` in a global
definition is a field error — a strategy value must be true in the scope that carries it — while the replacement
value in a global definition is accepted and inert.

Composition is additive only. It may add to what the global half already allows and may never introduce a
restriction the global half did not have; narrowing remains the job of a replacing profile or the runtime overlay.
The rule is applied per list kind and per MCP server key, so a global kind that is left unrestricted keeps native
visibility, and a project reference that cannot apply is reported as unapplied and unvalidated rather than being
silently dropped or attributed to the native selection.

Resource lists are concatenated global-then-project and expanded **exactly once** per kind, deduplicating by the
resolved identity the existing contracts use. Every selected identity carries the union of the contributor scopes
that produced it, so an identical literal, an overlapping glob/literal pair, and an alias/name pair all report
both halves. Extension provenance is keyed by the resolved entry path, because an identifier can be shared by
distinct entries while entry path is the identity selection deduplicates on. Tool provenance keeps two
coordinates: a tool resolved before session start, and a raw reference still pending session-start resolution —
boot expansion passes unknown literals through, so its presence proves nothing about a runtime tool.

Trust is unchanged. An untrusted project's catalog is never opened and ordinary global resolution still reads the
global definition; a replacing winner never reads the global definition; only a trusted `append` winner reads both.
An absent namesake is a standalone definition, not an error; an existing but malformed base fails closed. No
reapproval, digest, or drift gate is added, because the existing trust decision already covers project files and a
second authority would add state without adding a decision.

Three inputs fail closed: a strategy value outside the accepted set, `append` in a global definition, and a
`!`-prefixed glob in the project half of an `append` definition, where the current glob reading would invert the
selection. Nothing else tightens: global halves and every replacing definition keep today's behavior, including
the pre-existing leading-`!` reading, which stays a separate follow-up.

## Consequences

- A project can extend a team-wide selection without copying it, and the copy-drift failure disappears for that
  case. Field coverage grows by exactly one optional field; every file without it behaves as before.
- A composed profile still reports the existing project source, so the five persistence sites keyed on that value
  are untouched and no fourth source value enters the model.
- Composition happens per resolution and is never cached or materialized, so an edited global half is observed on
  the next resolution and no stale merged definition can exist.
- Provenance is presentation only. It cannot influence selection, and a disagreement between the two is a defect
  with a test, not a runtime fallback.
- Rollback stops being free. While `append` consumers exist, a build without the field warns and **replaces** —
  the destructive outcome this decision removes. Reverting therefore requires enumerating those consumer files and
  removing, renaming, or converting them first, as a separately reviewed operation. That guard is a stated fact of
  this change, not a footnote.
- The unguarded project-scope route is untouched. A trusted project's `.pi/skills/*` and `.pi/extensions/*` stay
  visible under every profile, `append` included, because exclusion entries are scoped to their own settings
  scope, generated settings are global scope, and Pi exposes no in-session API to remove discovered resources
  (ADR-0011). "A profile cannot restrict project-scope resources" remains an open capability gap owned outside
  this change.
- The generic leading-`!` glob reading stays a separate follow-up with its own scope, migration, and approval.

## Rejected alternatives

**`extends` with a parent chain.** Rejected for the reasons ADR-0003 already recorded: once users depend on a
chain, only a full resolution reveals what a profile selects, and variants silently inherit changes. This decision
adds no parent pointer and no chain; it composes exactly two files that already share a name.

**Deep or recursive merge.** Rejected because per-key recursive merging has no stable meaning for resource lists
and object-valued settings, and it makes a narrowing indistinguishable from an addition. The composition here
merges one level, per kind, with fixed rules per field, and refuses to narrow.

**Per-kind strategies.** Rejected as unnecessary surface: the additive-only rule already answers each kind's
composition question, and a per-kind strategy would multiply the matrix without adding a user need.

**`replace_base` / `replace_final` (the pi-modes vocabulary).** Rejected because those values exist to choose how
*prompt text* is assembled from base and overlay slots; this feature composes resource declarations, not prompts,
and adopting the vocabulary would falsely imply slot semantics that do not exist here.

**Degrading to the base when the project half is unusable (the pi-modes precedent).** Rejected: an unusable
`append` half is a malformed definition, and serving the global half instead would hide a broken project file.
"Unconfirmed" and "malformed" are different events; only the second one fails closed.

**A mode-style reapproval, digest, or drift gate.** Rejected: it would add a second authority over the same
project file, plus state that must itself be trusted. The existing project trust decision already decides whether
the file participates at all.

**Validating unapplied references against discovery.** Rejected for this change: the composition step is
deliberately I/O-free, and adding a discovery input to the resolver for an advisory would widen the resolver's
surface to answer a question that a static, honestly-worded diagnostic answers. The limitation is stated in the
diagnostic and in the user documentation instead.

## Related decisions

This ADR supersedes ADR-0003 only where that ADR states that a same-named project profile completely replaces the
global one. ADR-0003's rejection of `extends`, parent chains, and deep array appending remains in force and is
restated above. Behavior is owned by the `profile-catalog`, `resource-reference`, and `in-session-switch`
requirements in the change that introduces this decision.
