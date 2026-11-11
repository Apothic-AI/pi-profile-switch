# Spec Delta

## ADDED Requirements

### Requirement: Composed reference declarations

A definition produced by the `append` composition strategy SHALL treat each resource list as one merged
declaration: the global half's raw declarations followed by the project half's raw declarations. The merged
reference list SHALL be expanded exactly once per resource kind. Expanding each half separately and combining
the results MUST NOT be used, because it invents per-scope matches that the merged declaration does not imply.

Selection SHALL deduplicate by resolved identity — skill name, MCP server name, resolved extension entry path,
and expanded tool name — using the identities the existing resolution contracts already define. Raw `tools`
references SHALL additionally be deduplicated by exact string before expansion. Reference order SHALL be global
declarations then project declarations, so the merged order is deterministic.

Every selected identity SHALL carry the set of scopes whose raw declarations contributed to it: `global`,
`project`, or both. Contribution SHALL be recorded per raw declaration that produced the identity, and
contribution sets SHALL be unioned. An identical literal declared in both halves SHALL report both scopes. A
glob in one half and a literal in the other that resolve to the same identity SHALL report both scopes. A
package alias in one half and an entry name in the other that resolve to the same identity SHALL report both
scopes. First-contributor attribution MUST NOT be used.

A resource that user-native settings or package filters exclude from discovery SHALL remain unselectable under a
composed definition; a diagnostic SHALL be produced instead of a selection. A resource that is eligible in
discovery but absent from the global half's allowlist SHALL remain selectable through the project half.

#### Scenario: Identical literal in both halves reports both scopes

- **WHEN** the global and project halves both declare skill `review`, which discovery provides
- **THEN** the skill is selected once and reports both scopes as contributors

#### Scenario: Overlapping glob and literal report both scopes

- **WHEN** the global half declares `internal-*` and the project half declares `internal-a`
- **THEN** `internal-a` is selected once and reports both scopes

#### Scenario: Overlapping extension alias and entry name report both scopes

- **WHEN** the global half declares an extension package alias and the project half declares one of that package's entry names
- **THEN** the entry is selected once and reports both scopes

#### Scenario: Each kind expands once

- **WHEN** a composed definition declares references for every supported resource kind
- **THEN** each kind is expanded once, and no selection is duplicated or attributed to a scope that did not declare it

#### Scenario: Native exclusion stays unselectable

- **WHEN** the user's own settings exclude a skill from discovery and the project half declares that skill
- **THEN** the skill is not selected and an actionable diagnostic names the reference

#### Scenario: Globally unselected but eligible resource is added

- **WHEN** an eligible skill exists in discovery and the global half's `skills` allowlist omits it while the project half declares it
- **THEN** the composed definition selects the skill

### Requirement: Composite declaration intent and unapplied references

A composed definition SHALL preserve the existing three-state declaration grammar per kind: a kind with no
effective declaration SHALL remain undeclared, never an explicitly empty selection. Declaration intent SHALL be
derived from the composed halves as follows.

- A project half that omits a list or declares it empty SHALL contribute nothing.
- A global half that declares an empty list SHALL remain a declared base, so project references widen it.
- A global half that omits a list SHALL keep that kind undeclared, so the composed definition carries native
  visibility; project references for that kind SHALL NOT be applied to the selection.

References that cannot be applied because their kind stays undeclared SHALL be reported as unapplied
references: a static, deterministic diagnostic that names the file and each unapplied reference verbatim. That
diagnostic SHALL state that the references were not validated against discovery. An unapplied reference MUST NOT
be reported as an unknown reference, MUST NOT produce a zero-match warning, and MUST NOT be attributed to any
selected resource. The composed definition MUST NOT claim that a dropped reference contributed to native
selection.

Project-instructions text SHALL be appended to the global half's instructions without replacing it, dropping a
byte-identical duplicate. Scalar settings SHALL be overridden per field, with an undeclared project scalar
inheriting the global value. The model and thinking-level pair SHALL be validated exactly once through the
existing native validation path, with no provider probing, no default-model lookup, and no reset; a partial
declaration SHALL inherit the other half's field and SHALL produce a model only when both fields are present.
Native child-role settings SHALL remain a whole-field override: the project half replaces the global
declaration and its keys are not merged.

#### Scenario: Unrestricted kind keeps native selection with an honest diagnostic

- **WHEN** the global half omits `skills` and the project half declares `skills: ["review"]`
- **THEN** the composed definition leaves `skills` undeclared, native visibility applies, and a diagnostic names `review` as unapplied and unvalidated

#### Scenario: Declared empty base is widened

- **WHEN** the global half declares `skills: []` and the project half declares `skills: ["review"]`
- **THEN** the composed definition declares `skills` and selects `review`

#### Scenario: Empty project list contributes nothing

- **WHEN** the project half declares `skills: []` and the global half declares `skills: ["review"]`
- **THEN** the composed selection equals the global selection

#### Scenario: Unrestricted rows never become empty selections

- **WHEN** neither half declares a list kind
- **THEN** that kind stays undeclared in the composed definition and is never emitted as an empty array

#### Scenario: Instructions concatenate

- **WHEN** both halves declare `instructions`
- **THEN** the composed instructions contain the global text followed by the project text, and an identical duplicate is not repeated

#### Scenario: Scalars inherit without probing

- **WHEN** the project half declares only one field of the model pair and the global half declares both
- **THEN** the composed definition takes the project field and the inherited field, and no provider or default-model lookup occurs

#### Scenario: Native child-role settings are not merged

- **WHEN** the global half declares native child-role settings and the project half declares different ones
- **THEN** the composed definition contains the project declaration only

### Requirement: Per-server MCP tool policy in a composed definition

Per-server MCP tool policy SHALL follow the additive-only rule of a composed definition, evaluated per server
key against what the global half already allows.

A project entry for a server that the global half does not name SHALL NOT be applied, and SHALL be reported as a
diagnostic naming the server and stating that a composed definition cannot introduce a restriction. A project
entry for a server the global half does name SHALL be unioned with the global selectors, in global-then-project
order with exact-string deduplication. An empty project list for a named server SHALL contribute nothing. A
global empty list SHALL remain a deny-all base that project selectors widen. Both halves empty SHALL stay an
explicit deny-all.

A retained project entry MUST NOT create a server, enable a disabled server, alter project-owned servers, or
turn a disabled policy into an unrestricted one.

#### Scenario: Project key for an unnamed server is not applied

- **WHEN** the global half omits `mcp_tools` for server `github` and the project half declares `github: ["search"]`
- **THEN** the composed definition has no entry for `github`, its tools keep native availability, and a diagnostic names `github` as unapplied

#### Scenario: Named server policy is unioned

- **WHEN** the global half declares `github: ["a","b"]` and the project half declares `github: ["b","c"]`
- **THEN** the composed policy for `github` is `["a","b","c"]`

#### Scenario: Deny-all base is widened

- **WHEN** the global half declares `github: []` and the project half declares `github: ["search"]`
- **THEN** the composed policy for `github` is `["search"]`

#### Scenario: Both halves deny all

- **WHEN** the global half declares `github: []` and the project half declares `github: []`
- **THEN** the composed policy keeps the empty list and denies all callable tools for that server

### Requirement: Negated glob reference guard in a composed project half

A reference in the project half of a composed definition that is both a glob and prefixed with `!` SHALL be a
field error naming the file, the reference, and the reason: the current glob semantics read a leading `!` as a
negation gate, so such a reference would select every resource except its own pattern instead of adding to the
selection.

The guard SHALL apply only to the project half of a composed definition. The same reference in a global
half, and in any definition using complete replacement, SHALL keep exactly today's behavior: not rejected, not
reinterpreted, not repaired. The generic leading-`!` grammar question SHALL remain outside this change.

#### Scenario: Negated glob in an append project half fails closed

- **WHEN** the project half of an `append` definition declares `!internal-*` in `skills`
- **THEN** the profile is rejected with an error naming the file, the reference, and the selection-inverting reason

#### Scenario: Replacement definitions keep today's behavior

- **WHEN** a definition using complete replacement, or a global definition, declares the same `!`-prefixed glob
- **THEN** resolution behaves exactly as before this change, with no new rejection

### Requirement: Reference provenance in the activation plan

The activation plan SHALL expose reference provenance for a composed definition, keyed by the same resolved
identities the selection uses: skill name, MCP server name, resolved extension entry path, and tool name. Each
entry SHALL report its contributor scopes.

Extension provenance SHALL be keyed by the resolved entry path, which is the identity the existing selection
deduplicates on, and MAY carry the resolved extension identifier as an additional value. An identifier-only
key MUST NOT be used, because distinct entries can share an identifier.

Tool provenance SHALL distinguish a resolved tool from a raw reference that has not resolved yet. A tool
expanded before session start SHALL report the boot stage. A raw tool reference that matches no built-in tool
SHALL remain a pending session-start reference with its contributor scopes and MUST NOT be presented as an
existing runtime tool. When per-tool provenance is promised after session-start expansion, contributors SHALL be
propagated through the existing session-start expansion rather than through a new runtime reader.

Provenance SHALL be presentation only: it MUST NOT influence selection, and a disagreement between provenance
and selection SHALL be treated as a defect rather than a runtime fallback. A plan produced without composition
SHALL be unchanged, byte-identical to the plan the same inputs produce today.

Provenance SHALL describe the resource selection that is currently effective. An identity removed by the runtime
overlay SHALL NOT be reported as currently selected, regardless of which scope originally contributed it; an
overlay-removed identity MAY appear only as a disabled entry.

#### Scenario: Provenance cannot disagree with selection

- **WHEN** a composed definition is resolved
- **THEN** every identity in the provenance map is present in the selection, and every selected identity has provenance entry

#### Scenario: Extension provenance is collision-safe

- **WHEN** two distinct extension entries would share an identifier
- **THEN** provenance keeps them distinct by resolved entry path, and neither entry's scopes are merged into the other

#### Scenario: Unresolved raw tool reference stays pending

- **WHEN** a composed definition declares a tool reference no built-in tool provides
- **THEN** it is reported once as a pending session-start reference with its scopes, and not as an existing tool

#### Scenario: Boot and session-start tools are distinguished

- **WHEN** a composed definition declares one built-in tool and one extension-provided tool
- **THEN** the built-in tool reports the boot stage and the extension tool is resolved at session start

#### Scenario: Non-composed plan is unchanged

- **WHEN** a profile is resolved without composition
- **THEN** its plan contains no composition or provenance data and is byte-identical to the pre-change output

#### Scenario: Overlay-removed resource is not currently selected

- **WHEN** the overlay disables a resource that a composed definition selected
- **THEN** status does not report it as currently selected and reports it, if at all, as disabled
