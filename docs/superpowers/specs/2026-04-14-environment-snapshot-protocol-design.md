---
title: environment snapshot protocol design
date: 2026-04-14
status: planning-ready
---

# Environment Snapshot Protocol Design

## Purpose

This document is the companion protocol for `2026-04-14-environment-knowledge-design.md`.

The main skill spec defines the reusable environment knowledge contract. This protocol defines how an environment snapshot is selected, parsed, validated, rejected, and handed off to downstream analysis.

## Scope

This protocol covers:

1. authoritative snapshot selection
2. inline versus file snapshot handling
3. region registry grammar
4. cluster entry grammar constraints
5. validation and rejection rules
6. failure output shapes
7. conflict escalation when later knowledge layers disagree with environment facts

This protocol does **not** define:

1. project-to-cluster mapping
2. business logic semantics
3. application-specific deployment rules
4. any executable validator binary for v1

## Ownership Boundary

| Concern | Owner |
| --- | --- |
| environment field meanings | `environment-knowledge` main spec |
| canonical vocabulary values | `cluster-taxonomy.md` |
| snapshot syntax shape | `authoring-template.md` |
| snapshot selection and validation behavior | this protocol |
| project/service mapping to clusters | later project knowledge layers |

Until the shipped skill files exist, this protocol is the normative source for snapshot selection, parsing, validation, and failure behavior.

In v1, this protocol also normatively defines conflict detection and conflict error output. `SKILL.md` only enforces stop/continue behavior at runtime.

Until `cluster-taxonomy.md` exists in the shipped skill, the canonical v1 vocabulary and compatibility matrix are the ones defined in the paired main spec.

## Authoritative Snapshot Sources

For v1, an authoritative environment snapshot may come from exactly one of these sources:

1. an inline markdown block in the current analysis task input
2. one or more explicitly designated snapshot-candidate markdown file paths

Implicit file discovery is out of scope.

Definitions for deterministic selection:

- **quoted reference material**: any content inside fenced code blocks or markdown blockquotes
- **snapshot block**: content beginning at `## Environment Snapshot` and continuing until the next heading of level `##` or higher, or end of source
- **inline candidate ID**: `inline-1`, `inline-2`, and so on, assigned by encounter order within the current task input
- **file candidate ID**: the explicit file path itself
- **snapshot-candidate file path**: a markdown file path explicitly identified by the analysis prompt as an environment snapshot input; other referenced markdown files are not candidates

## Authoritative Snapshot Selection

### Inline snapshot

An inline snapshot is authoritative only if all of the following are true:

1. its first non-empty line is exactly `## Environment Snapshot`
2. it is not inside a fenced code block
3. it is not inside a markdown blockquote
4. its nearest preceding heading line does not match `^#+\s+.*\b(Example|Draft|Invalid)\b.*$`
5. it is presented as task input rather than quoted reference material

### File snapshot

A file snapshot is authoritative only if all of the following are true:

1. the file path is explicitly provided to the analysis workflow
2. the file contains exactly one block whose first non-empty line is `## Environment Snapshot`
3. that block's nearest preceding heading line does not match `^#+\s+.*\b(Example|Draft|Invalid)\b.*$`
4. any additional snapshot-like material inside fenced code blocks or markdown blockquotes is ignored and must not be counted as an authoritative candidate

### Selection decision

1. If one or more authoritative inline snapshots are found, ignore file snapshots.
2. If exactly one authoritative inline snapshot is found, use it.
3. If more than one authoritative inline snapshot is found, stop and request user selection.
4. If no authoritative inline snapshot exists, all explicitly provided file paths must load successfully and satisfy file-snapshot eligibility before file-tier selection continues.
5. If no authoritative inline snapshot exists and exactly one authoritative file snapshot exists, use it.
6. If no authoritative inline snapshot exists and more than one authoritative file snapshot exists, stop and request user selection.
7. If no authoritative inline snapshot exists and no authoritative file snapshot exists, stop.

When selection stops because multiple candidates exist, the response must list only candidates from the active precedence tier:

- inline candidates if any authoritative inline snapshot exists
- otherwise file candidates

Unreadable or unloadable file paths are handled as follows:

1. if an authoritative inline snapshot exists, file load failures are ignored because file candidates are outside the active precedence tier
2. if no authoritative inline snapshot exists, each unreadable or unloadable explicit file path is a fatal file-candidate failure and must be reported
3. if no authoritative inline snapshot exists, any readable file that fails authoritative file-snapshot eligibility is also a fatal file-candidate failure and must be reported

## Selection Failure Output

When no authoritative snapshot is found:

```md
Environment snapshot selection required

- no authoritative snapshot was found
- checked sources: inline, file
```

When more than one authoritative snapshot is found:

```md
Environment snapshot selection required

- multiple authoritative snapshots were found
- candidates:
  - [inline:inline-1]
  - [inline:inline-2]
- specify which snapshot is authoritative before analysis continues
```

If the active precedence tier is file-based, use file paths instead:

```md
Environment snapshot selection required

- multiple authoritative snapshots were found
- candidates:
  - [file:/path/to/snapshot-a.md]
  - [file:/path/to/snapshot-b.md]
- specify which snapshot is authoritative before analysis continues
```

When a provided file path cannot be loaded:

```md
Environment snapshot selection required

- [file:/path/to/snapshot.md] could not be loaded
- provide a readable snapshot file or inline snapshot before analysis continues
```

When a provided file path is readable but does not contain exactly one authoritative snapshot block:

```md
Environment snapshot selection required

- [file:/path/to/snapshot.md] is not a valid authoritative snapshot source
- expected exactly one authoritative `## Environment Snapshot` block
```

When file-tier selection is blocked by multiple file-candidate failures:

```md
Environment snapshot selection required

- [file:/path/to/a.md] could not be loaded
- [file:/path/to/b.md] is not a valid authoritative snapshot source
- fix file-source errors before analysis continues
```

## Snapshot Grammar

The authoritative snapshot must follow this shape:

```md
## Environment Snapshot

### Region Registry
- {region-id}: {region-label}

### Cluster: {cluster-id}
- cluster-id:
- cluster-role:
- region-layout:
  - region-id:
  - region-label:
- topology-class:
- inter-region-connectivity:
  - mode:
  - targets:
    - {cluster-id}
  - notes:
- analysis-rules:
  - interpret-as:
  - locality-bias:
  - cross-region-default:
```

Narrative text may appear between `## Environment Snapshot` and `### Region Registry`, and between the region registry and the first cluster, but once a `### Cluster:` block starts, only defined schema fields are allowed until the next `### Cluster:` heading or the end of the snapshot.

Exact parsing rules for v1:

1. `targets` must be written as nested markdown bullets, one target per line
2. `analysis-rules` must be written as nested markdown bullets, one directive per line
3. comma-separated target lists are invalid
4. paragraph-form `analysis-rules` are invalid
5. unknown directives or directive values are invalid
6. `### Region Registry` must appear exactly once
7. `### Region Registry` must appear before the first `### Cluster:`
8. inside `## Environment Snapshot`, the only allowed level-3 headings are `### Region Registry` and `### Cluster: <cluster-id>`
9. nested fields under `region-layout:`, `inter-region-connectivity:`, and `analysis-rules:` must be indented exactly two spaces below the parent bullet
10. target entries under `- targets:` must be indented exactly four spaces below the `targets` line

## Region Registry Grammar

The `### Region Registry` block is mandatory.

Rules:

1. every line must use `- {region-id}: {region-label}`
2. `region-id` must match `^[a-z][a-z0-9]*$`
3. `region-id` must be unique within the snapshot
4. `region-label` must be non-empty
5. every `region-layout.region-id` used by a cluster must appear in this registry
6. if `region-layout.region-label` is present, it must exactly match the registry label for the same `region-id`

`region-label` is optional inside a cluster entry and mandatory inside the region registry.

## Cluster Entry Grammar

Each cluster entry must satisfy these rules:

1. `### Cluster: <value>` must exactly match `cluster-id`
2. `cluster-id` must be lowercase kebab-case
3. `cluster-role` must be one of the values defined in `cluster-taxonomy.md`
4. `topology-class` must be one of the values defined in `cluster-taxonomy.md`
5. `inter-region-connectivity.mode` must be one of the values defined in `cluster-taxonomy.md`
6. `analysis-rules` must include exactly these three directives, each exactly once:
   - `interpret-as`
   - `locality-bias`
   - `cross-region-default`
7. unknown fields are invalid
8. duplicate fields are invalid

## Connectivity Mode Semantics

Mode semantics are fixed in v1:

| mode | required target count | source topology | target topology | reciprocity |
| --- | --- | --- | --- | --- |
| `none` | 0 | any allowed by main spec | none | none |
| `upstream-hub` | exactly 1 | `regional` | `hub` | optional |
| `hub-spoke` | 1 or more | `hub` | non-`hub` | optional |
| `peer` | 1 or more | same `topology-class` as target | same `topology-class` as source | never implied |

Additional rules:

1. `none` must omit `targets`
2. self-targeting is invalid
3. duplicate targets are invalid
4. every target must reference a declared `cluster-id` in the same snapshot
5. if both ends declare the same relationship, only these reciprocal combinations are valid:
   - `upstream-hub` on one side + `hub-spoke` on the other side
   - `peer` on both sides

## Analysis Rule Compatibility

The protocol reuses the compatibility matrix defined by the main spec. Validation must reject any directive combination not allowed by that matrix.

## Validation Rules

The authoritative snapshot must be rejected if any of the following occur:

1. zero cluster entries
2. missing `### Region Registry`
3. duplicate `cluster-id` values
4. missing `cluster-id`
5. missing `cluster-role`
6. missing `region-layout`
7. missing `region-layout.region-id`
8. missing `topology-class`
9. missing `inter-region-connectivity`
10. missing `inter-region-connectivity.mode`
11. missing `analysis-rules`
12. missing required analysis directives
13. unknown vocabulary values
14. unknown fields
15. duplicate fields
16. malformed region registry entries
17. undeclared `region-id` references
18. mismatched `region-layout.region-label` values
19. undeclared target `cluster-id` references
20. self-targeting entries
21. duplicate targets
22. invalid target counts for the selected connectivity mode
23. invalid topology pairing for the selected connectivity mode
24. invalid reciprocal declarations
25. invalid analysis directive combinations

## Validation Failure Output

```md
Environment snapshot invalid

- [snapshot] <snapshot-level error>
- [cluster-id: <value>] <cluster-level error>
```

If `cluster-id` is missing, use:

```md
- [snapshot] missing cluster-id for one cluster entry
```

After emitting this error block, analysis must stop and request a corrected snapshot.

Validation should accumulate all detectable errors in the authoritative snapshot before stopping.

## Later-Layer Conflict Escalation

Later knowledge layers are any project, service, or code-analysis context loaded after the authoritative environment snapshot. They may add implementation detail, but they may not override environment facts.

A conflict exists if a later layer contradicts any of:

1. `cluster-role`
2. `topology-class`
3. declared cluster-to-cluster connectivity
4. region identity for a cluster
5. `analysis-rules`

When such a conflict is detected, analysis must accumulate all detected conflicts in the current analysis pass, then stop and emit:

```md
Environment conflict detected

- [cluster-id: <value>] <later-layer statement> conflicts with <environment fact>
- clarify which source is authoritative before analysis continues
```

## v1 Implementation Shape

This protocol is procedural in v1:

1. `SKILL.md` references this protocol before analysis continues
2. `authoring-template.md` reflects the required snapshot shape
3. validation is performed by following the protocol rules, not by a dedicated validator binary

In v1, `SKILL.md` owns stop/continue behavior at runtime, while this protocol owns the normative rules that `SKILL.md` must embed or reference.

If later planning decides these rules are too heavy to enforce procedurally, a future version may introduce a separate validator artifact. That is out of scope for v1.
