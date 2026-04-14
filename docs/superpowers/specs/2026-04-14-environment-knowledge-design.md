---
title: environment-knowledge skill design
date: 2026-04-14
status: planning-ready
---

# Environment Knowledge Skill Design

## Problem

Current code analysis can misread system relationships when it lacks environment context. In this repository's target use case, systems are deployed across multiple Kubernetes clusters with different purposes and regional behaviors. Some regions are isolated, while specific clusters in specific regions can act as hubs for cross-region coordination. Without that context, the same code path can be interpreted incorrectly as generic service-to-service logic instead of environment-specific control behavior.

The goal of this skill is to provide a reusable environment knowledge layer that analysis workflows can load before reading code, so later reasoning can interpret cluster roles, regional topology, and special connectivity patterns consistently.

## Goals

1. Define a reusable, cluster-centric environment knowledge skill for Kubernetes-based deployments.
2. Capture environment architecture and foundational concepts only.
3. Make future analysis more precise by encoding interpretation rules, not just descriptive facts.
4. Keep project-specific service responsibilities and code relationships out of this skill.
5. Standardize how environment knowledge is authored so different analyses use the same concepts and vocabulary.

## Non-Goals

1. Documenting application business logic.
2. Describing project-level module boundaries or service ownership.
3. Maintaining a full dependency graph between projects.
4. Encoding deployment details for specific applications.
5. Replacing project knowledge skills or code analysis skills.

## Design Summary

The new skill will be a generic `environment-knowledge` skill that defines a fixed structure for describing Kubernetes cluster environments. The skill acts as an **environment interpretation layer**: it tells downstream analysis how to read code in the presence of cluster purpose, regional placement, topology role, and inter-region connectivity.

The skill is intentionally cluster-centric rather than project-centric. Each environment snapshot is described through a consistent set of fields focused on clusters and regions. Analysis can then use those fields to decide whether a code path should be interpreted as local behavior, hub-driven coordination, region-specific logic, or special-case cross-region interaction.

## Primary Use Case

The initial target environment model is:

- multiple Kubernetes clusters
- cluster purposes such as `production`, `staging`, `dev`, `testing`, and `iot`
- regional deployment across locations such as US West, US East, Japan, and Singapore
- most regions behave independently
- a special cluster in US East can directly connect to other regional clusters and commonly acts as the central coordination point

This design must support that model directly without baking those exact values into the core contract. Region IDs may vary by environment as long as they are declared in the environment snapshot region registry.

## Core Skill Responsibilities

The `environment-knowledge` skill must:

1. explain when environment knowledge should be loaded before analysis
2. define the required structure for an environment snapshot
3. define shared terminology for cluster roles and topology roles
4. define interpretation rules that downstream analysis should apply
5. forbid project-specific logic from being mixed into the environment layer

## Information Contract

Every environment snapshot described through this skill must use the following core sections.

| Section | Purpose |
| --- | --- |
| `cluster-id` | Provides the canonical identifier for a cluster entry within an environment snapshot. |
| `cluster-role` | Describes the purpose of a cluster, such as `production`, `staging`, `dev`, `testing`, or `iot`. |
| `region-layout` | Identifies the region where the cluster lives, including canonical region identity and optional human label. |
| `topology-class` | Describes the cluster's architectural role, such as standard regional cluster, hub cluster, or other special coordination role. |
| `inter-region-connectivity` | Explains whether and how the cluster connects to other regions. |
| `analysis-rules` | States how downstream analysis should interpret this cluster's code and interactions. |

These six sections are the stable contract. Additional narrative can exist around them, but downstream analysis should rely on these sections first.

## Why These Fields

### `cluster-id`

This field gives every cluster entry a stable interface. It allows analysis to refer to clusters unambiguously across environment snapshots, comparisons, and interpretation rules.

### `cluster-role`

This field encodes operational purpose. It prevents analysis from assuming that logic found in a `production` cluster should be treated the same as logic found in a `testing` or `iot` cluster.

### `region-layout`

This field anchors every cluster in a geographic or logical region. It gives later analysis the basis for discussing intra-region and cross-region behavior while preserving a canonical region identifier for validation.

### `topology-class`

This field captures architectural semantics that region alone cannot express. Two clusters can both exist in named regions while playing very different roles in the overall system.

### `inter-region-connectivity`

This field captures whether cross-region interaction is normal, restricted, hub-mediated, or exceptional. It is the minimum required signal for interpreting distributed coordination paths.

### `analysis-rules`

This is the most important field. The skill must not stop at describing structure; it must actively define how later analysis should read the structure. This turns the skill from a passive environment note into a reasoning aid.

## Required Interpretation Rules

The skill must instruct downstream analysis to apply the following reasoning rules:

1. Start with environment semantics before drawing conclusions from code structure alone.
2. Treat clusters with different `cluster-role` values as potentially different execution contexts, even if the code shape looks similar.
3. Treat `hub` topology-class values as higher-probability locations for global coordination, shared control, or cross-region orchestration.
4. Treat isolated regional clusters as local-first by default; cross-region behavior should be interpreted as special-case or explicit coordination logic, not assumed baseline behavior.
5. When a behavior only appears in one topology class or one cluster role, interpret it first as an environment-specific design choice before assuming it is universally applicable.

## Vocabulary and Precedence Rules

The skill must define explicit precedence between shared rules and per-cluster rules:

1. Global interpretation rules in `SKILL.md` apply to all clusters by default.
2. Per-cluster `analysis-rules` may refine those defaults for one cluster.
3. Per-cluster rules must not contradict the core meaning of `cluster-role`, `topology-class`, or `inter-region-connectivity`.
4. If a per-cluster rule narrows the general rule for a specific cluster, the per-cluster rule wins for that cluster only.
5. Later project- or service-specific knowledge layers may add implementation detail, but they must not override environment facts established by the environment snapshot. If a later layer appears to conflict with environment semantics, the analysis workflow should flag the conflict and request clarification instead of silently choosing one interpretation.

The taxonomy must also use a controlled-but-extensible vocabulary:

1. `cluster-role` and `topology-class` should prefer terms defined in `cluster-taxonomy.md`.
2. For v1, `cluster-role`, `topology-class`, connectivity modes, and analysis directives are fixed to the starter values listed in this spec.
3. For v1, `region-id` values are not globally hardcoded; they must be declared in the environment snapshot region registry.
4. Future versions may add terms, but only through an explicit taxonomy update and a corresponding spec revision.
5. Synonyms should be avoided inside snapshots; taxonomy should pick one canonical term and list aliases only as documentation.

## Skill Structure

The repository implementation should create a skill directory with this structure:

```text
environment-knowledge/
├── SKILL.md
├── cluster-taxonomy.md
└── authoring-template.md
```

### `SKILL.md`

Responsibilities:

- define when to use the skill
- explain the environment interpretation purpose
- require the six core sections
- describe the analysis loading order
- redirect readers to taxonomy and authoring guidance

### `cluster-taxonomy.md`

Responsibilities:

- define allowed and recommended terms for `cluster-role`
- define allowed and recommended terms for `topology-class`
- explain how to talk about hub clusters, regional clusters, and other special topology roles
- keep the vocabulary stable across analyses

Minimum starter taxonomy for v1:

- `cluster-role`: `production`, `staging`, `dev`, `testing`, `iot`
- `topology-class`: `regional`, `hub`
- `region-id`: declared in the environment snapshot region registry; `use1`, `usw1`, `jp1`, `sg1` are starter examples
- `inter-region-connectivity.mode`: `none`, `upstream-hub`, `hub-spoke`, `peer`
- `analysis-rules` directives:
  - `interpret-as`: `local-service`, `coordination-entry`, `global-control`, `exceptional-cross-region`
  - `locality-bias`: `region-first`, `hub-first`, `neutral`
  - `cross-region-default`: `allow`, `deny`

Source-of-truth ownership for v1:

| Value family | Owning file |
| --- | --- |
| `cluster-role` values | `cluster-taxonomy.md` |
| `topology-class` values | `cluster-taxonomy.md` |
| `region-id` registry | environment snapshot |
| `inter-region-connectivity.mode` values | `cluster-taxonomy.md` |
| `analysis-rules` directive names and allowed values | `cluster-taxonomy.md` |
| loading order, precedence, stop behavior | `SKILL.md` |
| snapshot syntax | `authoring-template.md` |

`cluster-role` semantics for v1:

| cluster-role | Meaning for downstream analysis |
| --- | --- |
| `production` | default to stable, live, and operationally critical behavior |
| `staging` | default to pre-release validation and production-like testing behavior |
| `dev` | default to developer-oriented or iterative behavior; avoid assuming production guarantees |
| `testing` | default to validation, synthetic execution, or non-user-facing control behavior |
| `iot` | default to edge- or device-oriented behavior, often more constrained and environment-specific |

### `authoring-template.md`

Responsibilities:

- provide the canonical authoring template for environment snapshots
- show one or more cluster entries
- remind authors not to include project logic
- require explicit `analysis-rules` for every cluster entry

## Authoring Model

The skill is a reusable standard, not a single frozen environment snapshot. Each analysis that uses this skill should supply an environment snapshot using the required structure.

The authoring template should follow this shape:

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

Multiple cluster entries can be listed in one snapshot. The important constraint is that every cluster uses the same contract so downstream analysis can compare them consistently.

## Snapshot Delivery Interface

For v1, the environment snapshot is not stored inside the reusable skill directory. Instead, it is supplied at analysis time in one of two supported forms:

1. an inline markdown block in the current analysis task input
2. one or more markdown file paths that follow `authoring-template.md` and are explicitly loaded alongside the skill

`SKILL.md` must instruct the analysis workflow to read that snapshot immediately after loading the skill and before loading project- or service-specific knowledge.

This keeps the reusable skill generic while still giving downstream analysis a concrete input interface.

For v1, "explicitly loaded alongside the skill" means the analysis prompt must either paste the snapshot inline or provide the exact markdown file path and load that file before continuing. Implicit discovery of candidate snapshot files is out of scope.

The precise rules for:

- authoritative snapshot selection
- inline vs file precedence
- region registry grammar
- strict validation rules
- failure output shapes
- later-layer conflict escalation behavior

are defined in the companion spec:

`docs/superpowers/specs/2026-04-14-environment-snapshot-protocol-design.md`

The runtime ownership boundary is:

- `SKILL.md` defines loading order and runtime stop behavior
- `cluster-taxonomy.md` defines all canonical value vocabularies
- `authoring-template.md` defines the exact snapshot syntax
- the companion protocol is a repository design artifact, not a fourth shipped skill file; it normatively defines selection, validation, and conflict rules that must be reflected in `SKILL.md` and `authoring-template.md` during implementation

Cluster-to-code binding is intentionally out of scope for this skill. Later project- or service-specific knowledge layers are responsible for stating which codebase, service, or subsystem maps to which `cluster-id` values. The environment skill only defines the cluster semantics that those later layers can reference.

## Minimal Snapshot Grammar

To keep authoring consistent, the skill implementation should define this minimum grammar:

- `cluster-id`: lowercase kebab-case unique identifier inside the snapshot, for example `use1-prod-hub`
- `cluster-role`: single canonical value from taxonomy
- `region-layout`: a structured field using:
  - `region-id`: canonical region identifier, for example `use1`
  - `region-label`: optional human-readable name; if present, it must match the label declared for that `region-id` in the `### Region Registry`
- `topology-class`: single canonical value from taxonomy
- `inter-region-connectivity`: a structured list using:
  - `mode`: one of `none`, `upstream-hub`, `hub-spoke`, `peer`
  - `targets`: a markdown bullet list of target `cluster-id` values declared in the same snapshot
  - `notes`: optional short explanation of why the connection exists
- `analysis-rules`: a markdown bullet list of fixed directives using:
  - `interpret-as`
  - `locality-bias`
  - `cross-region-default`

This grammar is still markdown-friendly, but it is constrained enough for repeatable authoring and future validation.

Exact syntax conventions for v1:

- `targets` must be written as nested markdown bullets, one target per line
- `analysis-rules` must be written as nested markdown bullets, one directive per line
- comma-separated target lists are invalid
- paragraph-form `analysis-rules` are invalid
- unknown directives or directive values are invalid

## Companion Protocol Boundary

This design intentionally separates the reusable skill contract from the snapshot protocol:

1. This document defines **what environment knowledge means** and what fields the skill requires.
2. The companion protocol defines **how an environment snapshot is selected, parsed, validated, and rejected**.
3. The reusable skill files should reference the protocol, but the protocol itself is a separate design artifact so validation complexity does not bloat the core skill specification.

## Analysis Directive Semantics

`analysis-rules` must use the following meanings:

| Directive | Allowed values | Meaning for downstream analysis |
| --- | --- | --- |
| `interpret-as` | `local-service`, `coordination-entry`, `global-control`, `exceptional-cross-region` | Primary interpretation lens for the cluster's code and network interactions. |
| `locality-bias` | `region-first`, `hub-first`, `neutral` | Tells analysis whether to prefer local-region explanations, hub-mediated explanations, or neither when multiple readings are possible. |
| `cross-region-default` | `allow`, `deny` | Tells analysis whether cross-region behavior should be considered expected baseline behavior or treated as an exception requiring explicit evidence. |

Interpretation details:

- `local-service`: default to region-local service behavior
- `coordination-entry`: default to control ingress or orchestration entry behavior
- `global-control`: default to system-wide coordination responsibility
- `exceptional-cross-region`: treat cross-region behavior as rare and special-case
- `region-first`: prefer explanations that stay inside the current region
- `hub-first`: prefer explanations that route through a hub cluster
- `neutral`: do not prefer regional or hub-mediated explanations without more evidence
- `allow`: cross-region behavior may be baseline for this cluster
- `deny`: cross-region behavior must be justified by explicit references or topology evidence

These directives refine how analysis interprets behavior. They do not override topology facts, cluster roles, or connectivity declarations.

Compatibility rules for directive values:

| topology-class | connectivity mode | allowed `interpret-as` | allowed `locality-bias` | allowed `cross-region-default` |
| --- | --- | --- | --- | --- |
| `regional` | `none` | `local-service` | `region-first`, `neutral` | `deny` |
| `regional` | `upstream-hub` | `local-service`, `exceptional-cross-region` | `region-first`, `neutral` | `deny` |
| `regional` | `peer` | `local-service`, `exceptional-cross-region` | `region-first`, `neutral` | `deny`, `allow` |
| `hub` | `none` | `global-control`, `coordination-entry` | `hub-first`, `neutral` | `deny` |
| `hub` | `hub-spoke` | `coordination-entry`, `global-control` | `hub-first`, `neutral` | `allow` |
| `hub` | `peer` | `coordination-entry`, `global-control` | `hub-first`, `neutral` | `allow`, `deny` |

## Analysis Workflow

Analysis should consume this skill in the following order:

1. load `environment-knowledge`
2. establish cluster purposes, regional layout, and topology roles
3. read project- or service-specific knowledge layers
4. analyze source code using the environment rules as interpretation context

This ordering ensures that code analysis does not flatten regional and topology-specific behaviors into generic relationships too early.

## Boundary Rules

To keep the skill focused, it must explicitly reject the following content:

- per-project module descriptions
- API endpoint catalogs
- business workflow descriptions
- service ownership maps
- implementation details of specific applications

If an analysis needs those details, it should load another knowledge layer after this skill.

## Handling Missing Information

If an environment snapshot is incomplete, the skill should instruct the analysis workflow to ask for the missing fields instead of guessing. Missing `cluster-role`, `topology-class`, `inter-region-connectivity`, or `analysis-rules` must be treated as blocking gaps for accurate interpretation.

Missing `cluster-id`, `region-layout`, or the snapshot `### Region Registry` must also be treated as blocking because analysis cannot anchor cluster references consistently without them.

Detailed validation rules, selection rules, and error output shapes are intentionally moved to the companion protocol spec so the core skill spec remains focused on environment semantics instead of parser behavior.

## Example Semantics

For the motivating example described during design:

- clusters may have roles such as `production`, `staging`, `dev`, `testing`, and `iot`
- most regions should be treated as independent regional units
- a designated US East cluster should be representable as `topology-class: hub`
- downstream analysis should treat US East-hosted cross-region behaviors as likely coordination logic rather than as generic peer-to-peer behavior

These are examples of how the contract is used, not hardcoded universal rules.

## Worked Example

```md
## Environment Snapshot

### Region Registry
- use1: US East
- jp1: Japan
- usw1: US West
- sg1: Singapore

### Cluster: use1-prod-hub
- cluster-id: use1-prod-hub
- cluster-role: production
- region-layout:
  - region-id: use1
  - region-label: US East
- topology-class: hub
- inter-region-connectivity:
  - mode: hub-spoke
  - targets:
    - usw1-prod
    - jp1-prod
    - sg1-prod
  - notes: central coordination entry point for cross-region control traffic
- analysis-rules:
  - interpret-as: coordination-entry
  - locality-bias: hub-first
  - cross-region-default: allow

### Cluster: jp1-prod
- cluster-id: jp1-prod
- cluster-role: production
- region-layout:
  - region-id: jp1
  - region-label: Japan
- topology-class: regional
- inter-region-connectivity:
  - mode: upstream-hub
  - targets:
    - use1-prod-hub
  - notes: only approved path is to the hub region
- analysis-rules:
  - interpret-as: local-service
  - locality-bias: region-first
  - cross-region-default: deny

### Cluster: usw1-prod
- cluster-id: usw1-prod
- cluster-role: production
- region-layout:
  - region-id: usw1
  - region-label: US West
- topology-class: regional
- inter-region-connectivity:
  - mode: upstream-hub
  - targets:
    - use1-prod-hub
  - notes: only approved path is to the hub region
- analysis-rules:
  - interpret-as: local-service
  - locality-bias: region-first
  - cross-region-default: deny

### Cluster: sg1-prod
- cluster-id: sg1-prod
- cluster-role: production
- region-layout:
  - region-id: sg1
  - region-label: Singapore
- topology-class: regional
- inter-region-connectivity:
  - mode: upstream-hub
  - targets:
    - use1-prod-hub
  - notes: only approved path is to the hub region
- analysis-rules:
  - interpret-as: exceptional-cross-region
  - locality-bias: region-first
  - cross-region-default: deny
```

This example shows how a hub region and a standard regional cluster can be expressed without introducing any project-specific deployment detail.

## Rationale for a Generic Skill

The chosen approach is a structured contract rather than a purely narrative document or a multi-profile skill hierarchy.

This approach was selected because it balances:

- consistency across analyses
- enough structure for repeated use
- enough flexibility for different environment snapshots
- low maintenance compared with a large profile catalog

## Success Criteria

The design is successful if:

1. two different analyses can describe different Kubernetes environments with the same field structure
2. downstream analysis can explain why a code path is interpreted differently because of cluster role or topology class
3. environment descriptions stay free of project-specific business logic
4. hub-region behavior can be represented without inventing project details
5. missing environment information is surfaced explicitly instead of silently assumed

## Implementation Notes for the Next Phase

The implementation plan should cover:

1. creating the `environment-knowledge/` directory and three files
2. writing the reusable skill instructions in `SKILL.md`
3. defining the vocabulary contract in `cluster-taxonomy.md`
4. writing the snapshot template in `authoring-template.md`
5. aligning the skill files with the companion snapshot protocol spec
6. validating that the skill is understandable and internally consistent when read together with the companion protocol

## Final Recommendation

Implement `environment-knowledge` as a generic, cluster-centric skill that standardizes how Kubernetes environment architecture is described and how code analysis should interpret that architecture. Keep the scope tightly limited to environment semantics and require explicit analysis rules so the skill actively improves downstream reasoning quality.
