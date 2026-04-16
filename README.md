# WACP — Workspace Agent Coordination Protocol

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)

**A formal protocol specification for coordinating autonomous agents in distributed, fault-tolerant systems.**

WACP defines the rules by which autonomous agents — particularly AI agents — coordinate work. It specifies what agents say, how they communicate, and the structures they operate within. It does not specify how the underlying system schedules, allocates, or manages resources; that is the domain of the operating system layer.

This repository contains **only the protocol specification**. For the reference implementation (Rust runtime, middleware, CLI, and seven ecosystem verticals), see [AAkil98/wacp](https://github.com/AAkil98/wacp).

---

## Table of Contents

- [Overview](#overview)
- [The Five Questions](#the-five-questions)
- [Design Principles](#design-principles)
- [Core Primitives](#core-primitives)
- [Roles and Permissions](#roles-and-permissions)
- [Mechanisms](#mechanisms)
- [Topology](#topology)
- [Taxonomy](#taxonomy-extension-registry)
- [Repository Structure](#repository-structure)
- [Reading Guide](#reading-guide)
- [Reference Implementation](#reference-implementation)
- [Status](#status)
- [Authors](#authors)
- [License](#license)

---

## Overview

Modern AI systems increasingly require multiple agents working together — decomposing problems, producing artifacts in parallel, reviewing each other's output, and integrating results. WACP provides the coordination surface for this: the contracts between agents, between agents and the coordinator, and between all parties and the audit record.

WACP is a **protocol specification**, not an implementation. It is deliberately transport-agnostic — implementable over files, HTTP, pipes, message queues, or anything else. The coordination model is independent of how messages physically move.

Key properties:

- **Isolation through structure.** Workspaces are hard boundaries; communication is explicit.
- **Immutability by default.** Checkpoints, trails, and closed workspaces cannot be modified.
- **Capability-based security.** The runtime enforces permissions; agents cannot grant themselves rights.
- **Human oversight is architectural.** Humans can observe, approve, inject, and escalate — without stopping the protocol.
- **Full auditability.** Every event produces exactly one trail entry. No gaps. No inference required.

---

## The Five Questions

WACP is organized around five fundamental coordination questions:

| Question | Answer | Primitive |
|---|---|---|
| **Where** does an agent work? | Workspaces — isolated, bounded execution contexts | `workspace` |
| **How** do agents communicate? | Envelopes and signals — structured messages and typed notifications | `envelope`, `signal` |
| **How** is progress recorded? | Checkpoints — immutable snapshots of work | `checkpoint` |
| **How** is work organized? | Tasks — units of work forming dependency graphs | `task` |
| **What** happened? | The trail — append-only audit log | `trail` |

---

## Design Principles

Every design decision in WACP traces back to at least one of these principles:

1. **Messages over mutations.** Agents never modify shared state directly. All coordination happens through explicit, typed messages.
2. **Roles are structural, not suggested.** Permissions are walls, not guidelines. Enforced by the runtime.
3. **Explicit lifecycle, no inference.** Every state transition is declared. If the runtime does not know an agent's state, the agent has not declared it.
4. **Context is scoped, not shared.** Each workspace has a defined boundary of visibility. Least privilege, applied to attention.
5. **History is first-class.** The trail is not a log to be parsed — it is a structured record with typed entries.
6. **Protocol over tooling.** WACP defines a protocol, not an application. Transport-agnostic by design.
7. **Human access is architectural.** Autonomy is a spectrum configured per workflow, not a binary switch.
8. **Ordering requires a clock.** Trail integrity, signal ordering, timeouts, and replay all depend on a well-defined logical clock.

---

## Core Primitives

| Primitive | Role | Spec |
|---|---|---|
| **Workspace** | The unit of isolation — bounded context assigned to one agent, linear 9-state lifecycle, immutable once closed | [`primitives/workspace.md`](primitives/workspace.md) |
| **Envelope** | The unit of communication — structured messages (`directive`, `feedback`, `query`) with priority and reply-to threading | [`primitives/envelope.md`](primitives/envelope.md) |
| **Signal** | The unit of notification — 11 closed-set types that drive the state machine | [`primitives/signal.md`](primitives/signal.md) |
| **Checkpoint** | The unit of progress — immutable work-product snapshots (`artifact`, `observation`) with intent + confidence | [`primitives/checkpoint.md`](primitives/checkpoint.md) |
| **Task** | The unit of work — dependency-graph structure supporting goal decomposition | [`primitives/task.md`](primitives/task.md) |
| **Trail** | The unit of history — hash-chained append-only audit log; single source of truth | [`primitives/trail.md`](primitives/trail.md) |
| **Identity** | The unit of uniqueness — opaque, globally unique, never-reused identifiers | [`primitives/identity.md`](primitives/identity.md) |
| **User** | The unit of human identity — hierarchy, ownership, originator relationships | [`primitives/user.md`](primitives/user.md) |

---

## Roles and Permissions

WACP defines three base roles:

| Role | Purpose | Key capabilities |
|---|---|---|
| **Coordinator** | Orchestrates work | Creates workspaces, dispatches directives, evaluates results, integrates output |
| **Worker** | Produces output | Receives directives, creates checkpoints, emits signals, sends queries |
| **Observer** | Monitors activity | Reads trails and checkpoints, cannot send envelopes or create checkpoints |

Roles are extensible through **single-level inheritance**. A derived role (e.g., `reviewer`) extends exactly one base role, inheriting its permissions and applying overrides. Derived roles are registered in the taxonomy before use.

Permissions are enforced across five dimensions: **send**, **receive**, **emit**, **create**, and **access**. The permission matrix is capability-based — enforced at runtime, not advisory. Full specification in [`foundations/roles.md`](foundations/roles.md).

---

## Mechanisms

| Mechanism | Purpose | Spec |
|---|---|---|
| **Integration** | Merging completed workspace output into the parent — three strategies (`direct`, `layered`, `evaluated`) | [`mechanisms/integration.md`](mechanisms/integration.md) |
| **Recovery** | Fault tolerance through trail replay | [`mechanisms/recovery.md`](mechanisms/recovery.md) |
| **Human Highway** | Protocol path for human oversight — injection, gates, escalations, approvals | [`mechanisms/human-highway.md`](mechanisms/human-highway.md) |
| **Security** | Cryptographic guarantees — hash-chained trails, capability-based access, boundary enforcement | [`mechanisms/security.md`](mechanisms/security.md) |

---

## Topology

The topology layer defines structural relationships between protocol objects:

| Structure | Spec |
|---|---|
| Workspace tree — parent-child hierarchy of workspaces | [`topology/tree.md`](topology/tree.md) |
| Task graph — DAG of task dependencies | [`topology/graph.md`](topology/graph.md) |
| Causal ordering — happens-before relationships | [`topology/causation.md`](topology/causation.md) |
| Channels — message-passing pathways | [`topology/channels.md`](topology/channels.md) |
| Ownership — which humans own which workspaces | [`topology/ownership.md`](topology/ownership.md) |
| Visibility — what each workspace can read | [`topology/visibility.md`](topology/visibility.md) |

---

## Taxonomy (Extension Registry)

The [`TAXONOMY.md`](TAXONOMY.md) spec is the protocol's extension mechanism. It registers:

- **Derived roles** — application-specific roles inheriting from base roles (e.g., `reviewer` extends `worker`)
- **Custom envelope types** — domain-specific message types beyond the three base types (e.g., `report`, `review`)
- **Custom checkpoint types** — domain-specific output types beyond `artifact` and `observation` (e.g., `decision`, `analysis`)

Three rules: **open where safe, closed where critical** (envelopes and checkpoints are extensible; signals are not); **registration before use** (unregistered types are rejected at runtime); and **registry, not schema** (it registers names and permissions, not payload formats).

---

## Repository Structure

```
wacp-protocol/
├── PROTOCOL.md                  # Authoritative protocol specification (~976 lines)
├── TAXONOMY.md                  # Extension registry for derived types (~385 lines)
├── LICENSE                      # CC BY-SA 4.0
├── README.md                    # This file
│
├── primitives/                  # Core data structures
│   ├── workspace.md             #   Execution containers and isolation
│   ├── envelope.md              #   Structured messages
│   ├── signal.md                #   Lightweight state notifications
│   ├── checkpoint.md            #   Immutable work products
│   ├── task.md                  #   Work units and DAG structure
│   ├── trail.md                 #   Audit log and recovery log
│   ├── identity.md              #   Identifier uniqueness and opaqueness
│   └── user.md                  #   Human identity and user hierarchy
│
├── foundations/                 # Baseline concepts
│   ├── clock.md                 #   Monotonic logical time
│   └── roles.md                 #   Authorization and permission matrix
│
├── mechanisms/                  # Operations
│   ├── integration.md           #   Assembly and merge operations
│   ├── recovery.md              #   Fault tolerance and repair
│   ├── human-highway.md         #   Human oversight integration
│   └── security.md              #   Cryptographic guarantees
│
└── topology/                    # Structural relationships
    ├── tree.md                  #   Workspace hierarchy
    ├── graph.md                 #   Task graph structure
    ├── causation.md             #   Causal ordering
    ├── channels.md              #   Message passing
    ├── ownership.md             #   Workspace ownership
    └── visibility.md            #   Data visibility model
```

---

## Reading Guide

**If you want the full picture**, start with [`PROTOCOL.md`](PROTOCOL.md). It is the authoritative specification — approximately 70 KB covering all primitives, roles, lifecycle states, and integration procedures in a single document.

**If you want to understand a specific concept**, go directly to the relevant constituent spec. Each spec is self-contained with its own rules, examples, and conformance requirements.

**Recommended reading order for newcomers:**

1. `PROTOCOL.md` §1–3 — Scope, vocabulary, and design principles
2. [`primitives/workspace.md`](primitives/workspace.md) — The foundational abstraction
3. [`primitives/envelope.md`](primitives/envelope.md) — How agents communicate
4. [`primitives/signal.md`](primitives/signal.md) — How state changes propagate
5. [`primitives/checkpoint.md`](primitives/checkpoint.md) — How progress is recorded
6. [`primitives/task.md`](primitives/task.md) — How work is organized
7. [`primitives/trail.md`](primitives/trail.md) — How history is preserved
8. [`foundations/roles.md`](foundations/roles.md) — Who can do what
9. [`mechanisms/integration.md`](mechanisms/integration.md) — How results are assembled
10. [`mechanisms/human-highway.md`](mechanisms/human-highway.md) — How humans participate
11. [`TAXONOMY.md`](TAXONOMY.md) — How the protocol is extended

---

## Reference Implementation

WACP has a Rust reference implementation in a separate repository:

**[github.com/AAkil98/wacp](https://github.com/AAkil98/wacp)** — licensed under Apache-2.0.

The reference implementation includes:

- 16 Rust crates (types, runtime, transport, coordinator, workspace, trail, permissions, taxonomy, tools, LLM adapters, security, SDKs, test support)
- gRPC services (AgentService, HighwayService, CoordinatorService) + REST gateway + WebSocket binding
- TypeScript CLI agent (`@wacp/cli`) with multi-vertical workflow execution
- Python agent SDK (`sdk-python`)
- Seven ecosystem verticals: SWE, DevOps, MLOps, Finance, Healthcare, Data Analytics, Data Science
- Extensive test coverage across Rust, TypeScript, and Python

The reference implementation implements the protocol; it does not define it. Other implementations are welcome and should conform to the specs in this repository.

---

## Status

WACP v0.1 is **complete**. All 20 constituent specs plus `PROTOCOL.md` and `TAXONOMY.md` are published.

| Component | Count | Status |
|---|---|---|
| Protocol specification | 1 (`PROTOCOL.md`) | Complete |
| Taxonomy | 1 (`TAXONOMY.md`) | Complete |
| Primitives | 8 specs | Complete |
| Foundations | 2 specs | Complete |
| Mechanisms | 4 specs | Complete |
| Topology | 6 specs | Complete |

Breaking changes will bump the minor version (`0.2`, `0.3`, …) until the protocol stabilizes at `1.0`.

---

## Authors

- **Akil Abderrahim** — Lead
- **Claude Opus 4.6** — Co-author

---

## License

This work is licensed under [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/).

You are free to share and adapt this material for any purpose, including commercially, provided you give appropriate credit and distribute contributions under the same license.

The **reference implementation** at [github.com/AAkil98/wacp](https://github.com/AAkil98/wacp) is separately licensed under **Apache-2.0**. Code that implements this protocol is not a derivative work of the specification in the CC BY-SA 4.0 share-alike sense; implementers may license their implementations under any compatible license.
