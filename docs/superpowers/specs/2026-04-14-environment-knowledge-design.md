---
title: environment-knowledge skill design
date: 2026-04-14
status: approved-for-planning
---

# Environment Knowledge Skill Design

## Problem

Current code analysis can misread system relationships when it lacks environment context. In this repository's target use case, systems are deployed across multiple Kubernetes clusters with different purposes and regional behaviors. Some regions are isolated, while specific regions can act as hubs for cross-region coordination. Without that context, the same code path can be interpreted incorrectly as generic service-to-service logic instead of environment-specific control behavior.

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
- a special US East region can directly connect to other regions and commonly acts as the central coordination point

This design must support that model directly without baking those exact values into the core contract.

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
| `cluster-role` | Describes the purpose of a cluster, such as `production`, `staging`, `dev`, `testing`, or `iot`. |
| `region-layout` | Identifies the region where the cluster lives. |
| `topology-class` | Describes the cluster's architectural role, such as standard regional cluster, hub cluster, or other special coordination role. |
| `inter-region-connectivity` | Explains whether and how the cluster connects to other regions. |
| `analysis-rules` | States how downstream analysis should interpret this cluster's code and interactions. |

These five sections are the stable contract. Additional narrative can exist around them, but downstream analysis should rely on these sections first.

## Why These Fields

### `cluster-role`

This field encodes operational purpose. It prevents analysis from assuming that logic found in a `production` cluster should be treated the same as logic found in a `testing` or `iot` cluster.

### `region-layout`

This field anchors every cluster in a geographic or logical region. It gives later analysis the basis for discussing intra-region and cross-region behavior.

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
3. Treat `hub` or equivalent `topology-class` values as higher-probability locations for global coordination, shared control, or cross-region orchestration.
4. Treat isolated regional clusters as local-first by default; cross-region behavior should be interpreted as special-case or explicit coordination logic, not assumed baseline behavior.
5. When a behavior only appears in one topology class or one cluster role, interpret it first as an environment-specific design choice before assuming it is universally applicable.

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
- require the five core sections
- describe the analysis loading order
- redirect readers to taxonomy and authoring guidance

### `cluster-taxonomy.md`

Responsibilities:

- define allowed and recommended terms for `cluster-role`
- define allowed and recommended terms for `topology-class`
- explain how to talk about hub regions, standard regions, and other special regional roles
- keep the vocabulary stable across analyses

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

### Cluster: {name}
- cluster-role:
- region-layout:
- topology-class:
- inter-region-connectivity:
- analysis-rules:
```

Multiple cluster entries can be listed in one snapshot. The important constraint is that every cluster uses the same contract so downstream analysis can compare them consistently.

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

If an environment snapshot is incomplete, the skill should instruct the analysis workflow to ask for the missing fields instead of guessing. Missing `topology-class`, `inter-region-connectivity`, or `analysis-rules` must be treated as blocking gaps for accurate interpretation.

## Example Semantics

For the motivating example described during design:

- clusters may have roles such as `production`, `staging`, `dev`, `testing`, and `iot`
- most regions should be treated as independent regional units
- US East should be representable as a `hub` or equivalent topology class
- downstream analysis should treat US East-hosted cross-region behaviors as likely coordination logic rather than as generic peer-to-peer behavior

These are examples of how the contract is used, not hardcoded universal rules.

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
5. validating that the skill is understandable and internally consistent when read on its own

## Final Recommendation

Implement `environment-knowledge` as a generic, cluster-centric skill that standardizes how Kubernetes environment architecture is described and how code analysis should interpret that architecture. Keep the scope tightly limited to environment semantics and require explicit analysis rules so the skill actively improves downstream reasoning quality.
