# Matrices

Decision tables the implementation must satisfy row for row. Each table names the spec requirement whose
scenarios pin it; the tables are implementation truth, the requirements are the contract. Notation: `G` = the
same-named global definition, `P` = the project definition declaring `append`, `—` = field absent.

## 1. Omitted and empty declarations (`skills`, `extensions`, `mcps`, `tools`)

Pinned by `profile-catalog` "Profile composition strategy" and `resource-reference` "Composite declaration intent
and unapplied references".

| `G` list | `P` list | Composite field | Effect | Diagnostic |
| --- | --- | --- | --- | --- |
| `—` | `—` | `—` | kind undeclared; native visibility | none |
| `—` | `[]` | `—` | kind undeclared; native visibility | none (nothing was requested) |
| `—` | refs | `—` | kind undeclared; native visibility; `P` refs **not** applied | `append-unapplied-reference`, naming file and each ref, stating "not validated against discovery" |
| `[]` | `—` | `[]` | declared empty selection; nothing selected | none |
| `[]` | `[]` | `[]` | declared empty selection; nothing selected | none |
| `[]` | refs | refs | refs expand; nothing inherited from `G` | none |
| refs | `—` | `G` refs | `P` contributes nothing | none |
| refs | `[]` | `G` refs | `P` contributes nothing | none |
| refs | refs | `G` refs ++ `P` refs | one expansion, dedupe by resolved identity | none |

Rules that fall out of the table, and must not be re-derived per kind: a composite never emits `[]` where neither
half declared the field; a project `[]` is never a narrowing of a declared global list; an unapplied reference is
never reported as `unknown-reference` and never produces a zero-match warning.

## 2. Per-server `mcp_tools`

Pinned by `resource-reference` "Per-server MCP tool policy in a composed definition".

| `G` has key for `s` | `P` has key for `s` | Composite | Diagnostic |
| --- | --- | --- | --- |
| no key | no key | no key | none |
| no key | refs | **no key** — base leaves `s` unrestricted | `append-cannot-narrow`, naming `s`, pointing at the replacement strategy or the overlay |
| `[a]` | no key / `[]` | `[a]` | none |
| `[]` | `[a]` | `[a]` | none |
| `[]` | `[]` | `[]` (deny-all preserved) | none |
| `[a,b]` | `[b,c]` | `[a,b,c]` (global order first, exact-string dedupe) | none |

A key present in `G` but empty for `s` counts as *named* (it is a declared deny-all base), so `P` may widen it.
`append` never creates a server, never enables a disabled server, never touches a project-owned server.

## 3. Scalar and non-list fields

Pinned by `resource-reference` "Composite declaration intent and unapplied references".

| Field | `G` | `P` | Composite | Notes |
| --- | --- | --- | --- | --- |
| `defaultProvider` | `openai` | `—` | `openai` | inherit |
| `defaultProvider` | `—` | `anthropic` | `anthropic` | override |
| `defaultProvider` | `openai` | `anthropic` | `anthropic` | per-field override |
| `defaultModel` | `gpt-5.4` | `—` | `gpt-5.4` | inherit; pair may become partial |
| provider only in `G`, model only in `P` | partial | partial | both present ⇒ model resolved from the pair | exactly today's path; no probe, no default lookup, no reset |
| `defaultThinkingLevel` | `high` | `—` | `high` | validated once by the native predicate |
| `defaultThinkingLevel` | `high` | `medium` | `medium` | project wins; invalid value warns and drops, model retained |
| `instructions` | `A` | `B` | `A\n\nB` | concatenated, global first |
| `instructions` | `A` | `A` | `A` | byte-identical duplicate dropped |
| `instructions` | `A` | `—` | `A` | inherit |
| `subagents` | map | map | `P` map only | whole-field override, no `agentOverrides` key merge |
| `subagents` | map | `—` | `—` | replacement semantics, not inheritance of the global declaration |
| `label` / `description` | any | `—` | `G` value | pure presentation inheritance |

## 4. Trust

Pinned by `profile-catalog` "Profile composition strategy" and "Catalog file format validation"; instrumented in
both directions by the attempted-read test.

| Situation | Project catalog file | Global definition | Result |
| --- | --- | --- | --- |
| untrusted project, `.pi/profiles/leader.json` present (any strategy) | **never opened** (not indexed) | **read** by ordinary resolution | ordinary global leader; no strategy is consulted |
| trusted project, winner `replace` (or absent strategy) | read | **never read** | replacement, as today |
| trusted project, winner `append`, namesake exists and is valid | read | read once as base | composed, project source |
| trusted project, winner `append`, no namesake | read | nothing to read | standalone + informational diagnostic; not an error |
| trusted project, winner `append`, namesake exists but malformed | read | read and **fails closed** | activation error naming the global file; no half applied |
| trusted project, winner invalid (`append`, malformed project file) | read and rejected | **not read** | error naming the project file (existing rule) |

Two prohibitions with their own tests: no code path in an untrusted directory opens `<project>/.pi/profiles/*`,
and no code path for a `replace` winner opens the global profile path.

## 5. Provenance and contributors

Pinned by `resource-reference` "Composed reference declarations" and "Reference provenance in the activation
plan"; rendered by `in-session-switch` "Composed activation observability".

| Case | Selected identity | Reported contributors |
| --- | --- | --- |
| literal in `G` only | once | `global` |
| literal in `P` only | once | `project` |
| identical literal in both | once | `both` |
| `G` glob `internal-*`, `P` literal `internal-a` | `internal-a` once | `both` |
| `G` package alias, `P` one of that package's entry names | that entry once | `both` |
| `G` glob matches nothing, `P` declares nothing | none | zero-match diagnostic (unchanged), no origin entry |
| both halves name an extension whose entries share an identifier | each entry distinct | keyed by resolved entry path; no cross-merge |
| tool provided by a package extension, `P` only | resolved at session start | `project`, `at: "session-start"` |
| built-in tool named in both halves | once | `both`, `at: "boot"` |
| raw tool reference no tool provides | not selected | pending `at: "session-start"` with its scope, never `unknown` and never "existing tool" |
| resource selected by the composite, then disabled by the overlay | not currently selected | absent from origins; at most a disabled entry |
| resource selected by the composite, then removed by an overlay glob | not currently selected | absent; overlay zero-match accounting unchanged |

Invariants: every provenance identity is present in the selection; every selected identity has provenance;
disagreement is a test failure, never a runtime fallback. A non-composed plan carries no provenance and is
byte-identical to today's output.

## 6. Adoption, reload and rollback

Pinned as behavior by `in-session-switch` "Composed activation observability" (status truth) and recorded as
deployment truth in `design.md`; release-note obligation in `tasks.md`.

| Operation | What changes | Effect on a lane with an `append` consumer |
| --- | --- | --- |
| package install / source pin movement | installed tree | none while the lane runs; **not** loaded-runtime adoption |
| new launch | a new process resolves the installed code | composes, provided the launch source resolves to that installation |
| native Pi reload | re-reads settings/context, replays the saved launch plan | no re-resolution of definitions; a changed catalog file is not picked up |
| `/profile reload` | re-resolves definitions, rewrites managed runtime files, then reloads | picks up an edited `append` file |
| `/profile use` | existing switch path, writes plan/state, reloads | picks up the new composed result |
| extension module re-execution | loaded extension code | required before a lane's in-process commands know `promptStrategy` |
| revert to a build without the field, `append` consumers present | installed tree | **BLOCKED**: that build warns and replaces |
| revert after consumers were removed/converted | installed tree | safe; runtime-state needs no repair |

No mandatory restart, no fleet reload, no silent model/thinking/profile/source switch is part of this change.
Rollback guard: enumerate consumer files with `"promptStrategy": "append"`, then either remove/rename them or
convert them to replacement with their effective content materialized — an exact, separately reviewed operation,
never automatic file deletion.

## 7. Fail-closed inputs and unchanged inputs

| Input | Result | Scope of the rule |
| --- | --- | --- |
| `promptStrategy` outside the accepted set | field error: file, profile, accepted values | any definition |
| `promptStrategy: "append"` in a **global** definition | field error: a global definition has no base | global catalog |
| `promptStrategy` replacement value in a global definition | accepted, inert (`docs/prd.md`); no behavior change | global catalog |
| `!`-prefixed **glob** in the project half of `append` | field error: inverting selection | the append project half only |
| `!`-prefixed glob in a global half or any replacement definition | unchanged behavior, no new rejection | everywhere else |
| bare `!name` (no glob characters) anywhere | unchanged: literal miss, existing error | everywhere |
| any catalog file without `promptStrategy` | unchanged in both catalogs | everywhere |

Recorded follow-ups, not rows of this change: generic leading-`!` grammar, project-scope resource enforcement,
safe live-consumer conversion procedure.
