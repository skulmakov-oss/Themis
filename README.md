# Themis

> A headless MCP contract server that governs AI coding agents through verified state transitions, scoped capabilities, evidence checks, and native Semantic quad logic.

**Themis** turns development policy from repeated natural-language instructions into an executable contract.

Instead of giving an AI coding agent the same long prompt for every task—what it may change, what it must not touch, which checks it must run, and when it must stop—Themis exposes only the actions permitted by the current verified state.

```text
User intent
    ↓
AI coding agent
    ↓ MCP / JSON-RPC over stdio
Themis Rust host
    ↓ verified execution
Semantic contract
    ↓ bounded capability requests
Git / Cargo / filesystem / audit
```

## Status

Themis is in the **architecture and repository-bootstrap stage**.

The current goal is a minimal vertical slice proving that a Semantic program can govern an AI coding agent through a deterministic MCP workflow. The repository does not yet claim a production-ready implementation.

## Core idea

```text
Policy as prose
      ↓
Policy as an executable state machine
```

Themis separates transport from authority:

| Layer | Responsibility |
|---|---|
| AI agent | Analyze code and perform the requested engineering work |
| Rust MCP host | Process lifecycle, JSON-RPC transport, typed serialization, bounded host effects |
| Semantic VM | Execute admitted SemCode deterministically |
| Semantic contract | Own workflow policy, state transitions, tool exposure, evidence rules, and quad verdicts |
| Capability broker | Enforce which host operations are actually available |
| Audit layer | Record observations, decisions, transitions, and effect results |

The governing rule is:

```text
Semantic decides.
Rust executes bounded effects.
Themis governs the transition.
```

## Why MCP

MCP provides the protocol surface between an AI agent and the contract runtime.

The available tools are derived from the current state rather than exposed as one large static list.

Example:

```text
State: LocalPreflight
Available tool:
  run_local_preflight

Preflight evidence accepted
  ↓

State: RemoteDriftGuard
Available tool:
  check_remote_drift
```

`tools/list` is a guidance surface. Every `tools/call` is still validated against the current state, epoch, evidence, tool schema, and effective capabilities.

## Quad verdicts

Themis uses Semantic's native four-valued logic as an explicit decision domain:

| Value | Meaning in Themis |
|---|---|
| `T` | The required condition is proven |
| `F` | The condition is disproven or the action is explicitly denied |
| `N` | Evidence is incomplete or unavailable |
| `S` | Evidence or policy is contradictory |

Authorization failures remain structured decision reasons; they do not introduce a fifth quad state.

## Initial vertical slice

The first implementation target is intentionally narrow.

### States

```text
Boot
TaskLoaded
ScopeValidated
ChecksRunning
ChecksFailed
ChecksPassed
Completed
Blocked
```

### MCP tools

```text
load_current_task
inspect_task_scope
validate_changed_paths
run_required_checks
write_harness_report
complete_task
```

### Initial capabilities

```text
ReadHarnessState
ReadRepositoryMetadata
ReadGitHead
ReadGitStatus
RunRegisteredCheck
ValidateChangedPaths
AdvanceStateAtomically
AppendAuditEvent
```

### Explicitly excluded from v0

```text
arbitrary shell execution
force push
merge
pull-request creation
branch deletion
unrestricted filesystem access
implicit network access
GUI / desktop interface
```

## Execution model

The planned release form is a single headless executable:

```text
themis-mcp.exe
├── MCP stdio transport
├── JSON-RPC validation
├── Semantic verifier
├── Semantic VM
├── embedded verified themis.smc
├── capability broker
├── repository state adapter
└── audit writer
```

The Semantic source contract is compiled before the Rust release build:

```text
contracts/themis.sm
        ↓ smc compile
artifacts/themis.smc
        ↓ embedded into the host
     themis-mcp.exe
```

The executable contains immutable governing logic. Repository-local task state remains external and mutable under an atomic state-transition contract.

## Security model

Themis is designed around several non-negotiable rules:

1. **Verifier first** — raw, unadmitted SemCode must not enter the canonical execution path.
2. **No arbitrary shell capability** — host operations are selected from a registered, typed command vocabulary.
3. **Least authority** — the current contract state receives only the capabilities required for the next permitted action.
4. **Atomic transitions** — state updates use expected phase and epoch checks to reject stale or concurrent transitions.
5. **Evidence before transition** — successful process exit alone is not sufficient when stronger evidence is required.
6. **Deterministic decisions** — identical state, request, observations, and capabilities must produce the same verdict.
7. **Auditable effects** — every external observation and mutation must be represented in the audit trail.
8. **No GUI surface** — the MCP protocol is the interface; Themis remains a headless system component.

## Planned repository structure

```text
Themis/
├── Cargo.toml
├── crates/
│   ├── themis-host/        # MCP process, stdio, JSON-RPC, lifecycle
│   ├── themis-protocol/    # typed MCP and contract ABI models
│   ├── themis-state/       # state, epoch, evidence, atomic transitions
│   └── themis-effects/     # registered bounded host operations
├── contracts/
│   └── themis.sm           # governing Semantic source contract
├── artifacts/
│   └── themis.smc          # generated verified SemCode artifact
├── docs/
│   ├── architecture.md
│   ├── state-machine.md
│   └── security-model.md
└── tests/
    ├── fixtures/
    └── golden/
```

This structure is provisional until the first implementation slice freezes the actual ownership boundaries.

## Relationship to Semantic

Themis is intended to be the first standalone system application governed by [Semantic](https://github.com/skulmakov-oss/Semantic).

Semantic provides the language, verifier-first execution model, deterministic VM, native quad logic, and controlled host boundary. Themis applies those properties to AI-agent development workflows.

The relationship is intentionally recursive:

```text
Semantic governs Themis
Themis governs an AI coding agent
The agent helps develop Semantic and Themis
```

## Project principles

- Small, reviewable changes
- Tests before optimization
- No architecture-wide refactors during the first vertical slice
- One clear owner for every contract and state type
- Rust transport must not absorb workflow policy
- Semantic policy must not receive unrestricted operating-system access
- Documentation must distinguish implemented behavior from planned behavior

## License

Licensed under the [Apache License 2.0](LICENSE).

Copyright 2026 Said Kulmakov.
