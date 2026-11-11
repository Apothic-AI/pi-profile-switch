# Spec Delta

## ADDED Requirements

### Requirement: Composed activation observability

The status surface SHALL report the active plan's composition explicitly for a composed definition: the
strategy, the path of the global base that was read, the path of the project file, and the declaration intent of
each resource kind in the composite. It SHALL report, per selected resource, which scopes contributed it.
A resource contributed by both halves SHALL be reported as such.

For a kind the composite leaves undeclared, status SHALL describe the kind as unrestricted with native
selection, SHALL NOT attribute any resource of that kind to a project reference, and SHALL list the unapplied
references with their file. Those references SHALL be listed as unapplied and unvalidated; status MUST NOT
report them as unknown, as zero-match, or as contributing to the effective selection.

A profile list entry whose winning definition composes with a global namesake SHALL be identifiable as such in
the list output, without reading unrelated definitions.

Status SHALL keep distinguishing a declaration from the runtime mapping: a declared composition is described by
the plan that the runtime was activated with, not by re-resolving files on status display. Reporting MUST NOT
perform additional definition reads, writes, or reapproval steps.

#### Scenario: Status names the composition

- **WHEN** the active profile's project definition declares `append` and a global base exists
- **THEN** status reports the strategy, both paths, and the per-kind intent of the composite

#### Scenario: Status reports contributors

- **WHEN** a selected skill was contributed by both halves
- **THEN** status reports it as contributed by both, and no identity is attributed to a scope that did not declare it

#### Scenario: Unrestricted kind is described honestly

- **WHEN** the composite leaves `skills` undeclared and the project half declared skill references
- **THEN** status describes skills as unrestricted with native selection and lists the project references as unapplied and unvalidated

#### Scenario: Standalone append is visible

- **WHEN** the active definition declares `append` and no global namesake exists
- **THEN** status reports the missing base as informational rather than as an activation failure

#### Scenario: Status does not contradict the overlay

- **WHEN** the overlay disables a resource selected by the composed definition
- **THEN** status does not list that resource as currently selected

#### Scenario: Status reads no additional definitions

- **WHEN** status is requested for a composed profile
- **THEN** it reports from the active plan without re-resolving catalog files or writing any state
