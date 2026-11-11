# Spec Delta

## ADDED Requirements

### Requirement: Profile composition strategy

A profile definition MAY declare a composition strategy. The accepted strategy values SHALL defer to the
authoritative profile schema in `schemas/profiles.schema.json`. This requirement defines the behavior of the
`append` value; every other accepted value, and an absent value, SHALL keep complete replacement.

Reading a definition SHALL retain a declared strategy as a parsed field of that definition. An unsupported
strategy value SHALL be a field error naming the file, the profile, and the accepted values; it MUST NOT be
treated as an unknown key that warns and continues.

`append` SHALL be valid only in a project catalog definition. A global definition that declares `append` SHALL
be a field error, because no definition composes a global one. A global definition that declares the
replacement value SHALL remain valid and inert: a strategy value SHALL be true in the scope that carries it.

A project definition declaring `append` SHALL compose with its same-named global definition when one exists,
and SHALL stand alone when none exists. Absence of a same-named global definition is not an error and MUST NOT
produce a fallback to another definition. An existing but invalid same-named global definition SHALL fail
closed, naming that file; it MUST NOT be partially applied, skipped, or replaced by another definition.

Composition SHALL be additive only: the composed definition MAY add to what the global half already allows and
MUST NOT introduce a restriction the global half did not have. Narrowing stays the responsibility of the
replacement strategy and of the runtime overlay.

Every accepted definition SHALL keep reporting the existing project source whenever a project file participates.
Composition MUST NOT be cached, materialized, or written back: each resolution SHALL freshly read the
participating files and freshly compose them.

#### Scenario: Project append composes with the global namesake

- **WHEN** a trusted project definition declares `append` and a valid same-named global definition exists
- **THEN** resolution reads both files, composes them additively, and reports the project source

#### Scenario: Absent strategy keeps complete replacement

- **WHEN** a project definition declares no strategy and a same-named global definition exists
- **THEN** the global definition is not read and the project definition replaces it in full

#### Scenario: Append without a namesake stands alone

- **WHEN** a trusted project definition declares `append` and no same-named global definition exists
- **THEN** resolution succeeds with the project definition alone and reports an informational diagnostic naming the absent base

#### Scenario: Invalid global base fails closed

- **WHEN** a trusted project definition declares `append` and the same-named global file exists but is malformed
- **THEN** activation fails with an error naming the malformed global file, without applying either half

#### Scenario: Unsupported strategy value is a field error

- **WHEN** a definition declares a strategy value outside the accepted set
- **THEN** the profile is rejected with an error naming the file, the profile, and the accepted values

#### Scenario: Append in a global definition is rejected

- **WHEN** a global catalog definition declares `append`
- **THEN** the profile is rejected with an error explaining that a global definition has no base to compose with

#### Scenario: Replacement value in a global definition is inert

- **WHEN** a global catalog definition declares the replacement strategy
- **THEN** it is accepted, and behavior is identical to declaring no strategy

#### Scenario: Composition is never cached

- **WHEN** a global definition is edited while an `append` project definition composes with it
- **THEN** the next resolution observes the edited content, and no merged definition is stored in a catalog or state file

## MODIFIED Requirements

### Requirement: Catalog file format validation

Activation SHALL validate only the selected profile's winning definition, after source precedence and project
trust have been determined. It MUST NOT read unrelated definition contents, and it MUST NOT read a shadowed
global definition except as the composition base of a winning project definition that declares the `append`
composition strategy. Selecting `default` SHALL require no catalog definition read.

Each selected file SHALL contain legal JSON with an object as its top-level value. Invalid JSON, object shape,
or supported-field types SHALL reject that profile with an actionable error naming its file and, for field
errors, profile and field. A present but invalid winning definition MUST NOT fall back to the shadowed
definition or to `default`. An invalid composition base SHALL reject the composed profile by the same rule.

Listing SHALL isolate definition errors per winning entry instead of failing the whole catalog. Directory
enumeration failures other than a missing directory SHALL remain errors.

#### Scenario: Illegal catalog content
- **WHEN** the selected winning file is not legal JSON or its top-level value is not an object
- **THEN** activation fails with its file path and does not substitute another profile

#### Scenario: Unrelated malformed file
- **WHEN** a valid profile is selected and another profile file contains malformed JSON or invalid field types
- **THEN** the valid profile activates without reading the other file's contents

#### Scenario: Default with corrupt catalog entries
- **WHEN** `default` is selected and catalog files are malformed
- **THEN** activation does not read those definitions and continues

#### Scenario: Project winner shadows corrupt global definition
- **WHEN** a trusted project provides a valid selected definition that replaces a malformed same-named global file
- **THEN** the project definition activates without reading the global file's contents

#### Scenario: Composition base is read only for append
- **WHEN** a trusted project's selected definition declares `append` and a same-named global definition exists
- **THEN** that global file is read exactly once as the base, and no other definition file is read

#### Scenario: Invalid project winner does not fall back
- **WHEN** the trusted project's selected definition is malformed and a valid same-named global definition exists
- **THEN** activation fails naming the project file rather than using the global definition

#### Scenario: Invalid composition base does not fall back
- **WHEN** a trusted project's `append` definition is valid and its same-named global base is malformed
- **THEN** activation fails naming the global base file rather than serving the project half alone

#### Scenario: Invalid entry does not break listing
- **WHEN** a visible winning definition is malformed
- **THEN** listing retains that entry as unavailable with a path-bearing diagnostic and still lists valid entries

### Requirement: Profile sources and project override

Every resolvable profile SHALL report its source according to the existing source contract. Precedence SHALL be
decided before reading definition contents.

A trusted project's same-named definition SHALL completely replace the global definition without merging or
appending fields, unless that project definition declares the `append` composition strategy, in which case the
two definitions SHALL compose according to "Profile composition strategy" and the merged result SHALL report the
project source. Replacement SHALL remain the default for a definition that declares no strategy.

Deleting the project file SHALL reveal the global definition on the next resolution. An invalid project winner
SHALL remain the winner and SHALL fail according to "Catalog file format validation" rather than revealing the
global definition.

#### Scenario: Project entry completely replaces global entry
- **WHEN** global and trusted-project definitions share a name, declare different fields, and the project definition declares no strategy
- **THEN** resolution uses only the project definition and reports project source

#### Scenario: Project append composes instead of replacing
- **WHEN** global and trusted-project definitions share a name and the project definition declares `append`
- **THEN** resolution composes both halves and reports project source

#### Scenario: Global entry restored after project override deleted
- **WHEN** a same-named project override is deleted
- **THEN** the next resolution reads the global definition and reports global source
