# Themis Architecture

**Status:** Architecture baseline for Themis v0  
**Scope:** Headless MCP contract server governed by verified Semantic code  
**Audience:** Maintainers, contributors, reviewers, and AI coding agents working through the roadmap

## 1. Purpose

Themis converts repeated natural-language development instructions into an executable, stateful contract.

Instead of asking an AI coding agent to remember a long prompt containing scope rules, required checks, forbidden operations, and completion criteria, Themis exposes only the actions permitted by the current workflow state and validates every requested action again at execution time.

```text
Policy as prose
      ↓
Policy as an executable state machine
```

Themis is not an IDE, a general-purpose agent framework, or an unrestricted command runner. It is a narrow governance layer between an AI coding agent and a repository-local engineering workflow.

## 2. Architectural objective

The v0 system must prove this complete path:

```text
MCP request
  ↓
Rust transport and protocol validation
  ↓
Verified embedded Semantic contract
  ↓
Typed decision and quad verdict
  ↓
Optional registered capability request
  ↓
Bounded host observation
  ↓
Evidence evaluation
  ↓
Atomic workflow-state transition
  ↓
Deterministic audit record
  ↓
MCP response
```

The governing rule is:

```text
Semantic decides.
Rust executes bounded effects.
Themis governs the transition.
```

## 3. System context

```text
┌──────────────────────────────────────────────────────────────┐
│                    AI coding agent                           │
│  Codex or another MCP client                                │
└───────────────────────────┬──────────────────────────────────┘
                            │ MCP / JSON-RPC over stdio
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                       Themis host                            │
│  process lifecycle · framing · schemas · session state      │
└───────────────────────────┬──────────────────────────────────┘
                            │ typed contract request
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                  Verified Semantic contract                  │
│  workflow policy · tool exposure · evidence rules · verdict │
└───────────────────────────┬──────────────────────────────────┘
                            │ typed effect request
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                  Capability / effect broker                  │
│  registry · limits · environment · working directory        │
└───────────────────────────┬──────────────────────────────────┘
                            │ bounded operations
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                  Repository-local resources                  │
│  task manifest · Git metadata · Cargo checks · state · audit│
└──────────────────────────────────────────────────────────────┘
```

## 4. Layer model

### 4.1 MCP client

The client analyzes code and performs engineering work. It may request only the MCP tools exposed by Themis, but tool visibility is not treated as the security boundary.

Responsibilities:

- perform code analysis and edits;
- call `tools/list` and `tools/call`;
- provide arguments conforming to the published tool schema;
- consume structured verdicts and evidence summaries;
- retry or stop according to explicit results.

The client does not own:

- workflow phase transitions;
- capability authorization;
- effect execution policy;
- persisted workflow state;
- audit ordering.

### 4.2 Rust MCP host

The Rust host is the trusted transport and execution shell. It must remain small, explicit, and policy-poor.

Responsibilities:

- process startup and shutdown;
- line-delimited JSON-RPC framing over `stdio`;
- MCP session lifecycle;
- request ID preservation;
- schema validation and typed serialization;
- loading the repository context;
- verifier admission of embedded SemCode;
- invocation of the verified Semantic entry point;
- enforcement of resource limits;
- execution of registered effects;
- atomic persistence and audit append.

The host must not own:

- phase-to-tool routing policy;
- successful or failed workflow verdicts;
- evidence sufficiency rules;
- roadmap ordering;
- hidden workflow shortcuts.

### 4.3 Semantic verifier and VM

The execution layer admits and runs the compiled Themis contract.

Responsibilities:

- reject malformed or unsupported SemCode;
- produce a verified execution token or equivalent admitted representation;
- execute only admitted code;
- enforce VM quotas and trap semantics;
- keep execution deterministic for identical inputs;
- expose only the typed host boundary required by Themis.

Raw SemCode bytes are not a canonical executable input after startup admission.

### 4.4 Semantic contract

`contracts/themis.sm` is the policy owner.

Responsibilities:

- map workflow state to exposed tools;
- validate requested tools against phase and epoch;
- interpret evidence;
- produce `T`, `F`, `N`, or `S` verdicts;
- choose the next legal state;
- request only typed, registered effects;
- return structured reasons and evidence requirements.

The contract must not:

- parse MCP JSON directly;
- execute arbitrary command strings;
- access the filesystem directly;
- mutate persisted state directly;
- bypass the capability broker;
- emit arbitrary JSON as its canonical ABI.

### 4.5 Capability and effect broker

The broker converts a typed effect identifier into a bounded host operation.

Each registered effect defines:

- stable `EffectId`;
- required capability;
- exact executable or internal implementation;
- fixed argument template;
- working-directory policy;
- environment allowlist;
- timeout;
- stdout and stderr byte limits;
- output schema;
- observational or mutating classification.

The v0 broker permits observational and validation effects only.

### 4.6 State adapter

The state adapter persists repository-local workflow state.

Responsibilities:

- decode and validate the state schema;
- bind state to the active contract digest;
- perform compare-and-swap transitions using expected phase and epoch;
- write atomically;
- reject stale, replayed, concurrent, or wrong-contract transitions;
- preserve a recoverable last-known-valid state.

Semantic decides the desired transition. Rust validates the transition envelope and performs persistence.

### 4.7 Audit layer

The audit layer records the causal chain of the workflow.

Responsibilities:

- assign monotonic event sequence numbers;
- record requests, observations, decisions, effects, and transitions;
- retain bounded excerpts and digests instead of unlimited process output;
- prevent sensitive environment values from being recorded;
- fail closed when a required transition cannot be audited;
- support independent parsing and validation.

Audit is local by default. Themis v0 has no telemetry upload.

## 5. Planned repository ownership

```text
Themis/
├── Cargo.toml
├── crates/
│   ├── themis-core/
│   │   └── protocol-neutral domain types shared across crates
│   └── themis-host/
│       └── executable process, MCP transport, startup, and adapters
├── contracts/
│   └── themis.sm
├── artifacts/
│   └── themis.smc
├── docs/
│   ├── architecture.md
│   ├── state-machine.md
│   └── security-model.md
└── tests/
    ├── golden/
    └── e2e/
```

Additional crates such as `themis-state` or `themis-effects` are created only when real ownership pressure justifies them. Empty speculative crates are prohibited.

## 6. Dependency direction

The intended dependency direction is:

```text
themis-host
    ↓
themis-core
```

The host may depend on Semantic execution crates and protocol libraries. `themis-core` must remain independent of process, filesystem, Git, Cargo, MCP transport, and OS concerns.

Rules:

- protocol-neutral domain types live in `themis-core`;
- MCP-specific representations stay in `themis-host` or a later justified protocol crate;
- repository I/O stays in host adapters;
- Semantic policy remains in `contracts/themis.sm`;
- no crate may depend back on the executable host;
- no policy table may be duplicated in both Rust and Semantic.

## 7. Contract build and embedding

The planned release artifact is a single headless executable.

```text
contracts/themis.sm
    ↓ smc check
    ↓ smc compile
artifacts/themis.smc
    ↓ smc verify
    ↓ embedded at Rust build time
themis-mcp.exe
```

Startup sequence:

```text
load embedded bytes
  ↓
verify SemCode
  ↓
resolve required entry point
  ↓
cache verified token
  ↓
load repository-local state
  ↓
start MCP session loop
```

The contract is never compiled on each request. Verification occurs before operational readiness is advertised.

## 8. Typed Rust–Semantic ABI

The host/contract boundary must be narrow, versioned, and deterministic.

Minimum v0 concepts:

```text
ContractRequest
ContractDecision
ContractMetadata
WorkflowStateView
ToolDescriptor
EffectRequest
EvidenceRecord
QuadVerdict
DecisionReason
```

Rules:

- Semantic returns meaning; Rust serializes MCP representation;
- arbitrary `serde_json::Value` is not the canonical execution ABI;
- unknown schema versions or discriminants fail closed;
- ordering is stable;
- authorization denial is a reason attached to a quad verdict, not a fifth quad state;
- raw VM registers, frames, or implementation-specific handles never cross the boundary.

## 9. Request lifecycle

### 9.1 Read-only discovery

```text
tools/list
  ↓
Rust validates MCP session
  ↓
Rust reads validated workflow-state view
  ↓
Semantic derives permitted tool descriptors
  ↓
Rust validates descriptors
  ↓
Rust serializes MCP response
```

### 9.2 Tool invocation without an effect

```text
tools/call
  ↓
request schema validation
  ↓
Semantic validates phase, epoch, and arguments
  ↓
Semantic returns final decision
  ↓
Rust serializes result
```

### 9.3 Tool invocation requiring evidence

```text
tools/call
  ↓
Semantic returns typed EffectRequest
  ↓
capability broker validates EffectId and limits
  ↓
registered effect executes
  ↓
bounded observation becomes EvidenceRecord
  ↓
Semantic evaluates evidence
  ↓
atomic transition + audit
  ↓
MCP result
```

## 10. Determinism contract

For the same:

```text
contract digest
+ ABI version
+ workflow state
+ request
+ ordered observations
+ effective capabilities
```

The Semantic decision must be equivalent.

Determinism requires:

- stable tool ordering;
- stable map and set serialization;
- normalized repository-relative paths;
- deterministic check ordering;
- explicit timestamps outside the decision core;
- no hidden environment reads;
- no ambient network data;
- bounded output normalization;
- explicit handling of unavailable or contradictory evidence.

## 11. Error domains

Errors remain separated by owner.

| Domain | Examples | Owner |
|---|---|---|
| Transport | malformed JSON, invalid request, unknown method | Rust MCP host |
| Session | method before initialization, duplicate initialize | Rust MCP host |
| Admission | malformed SemCode, missing entry point | verifier/startup |
| Contract ABI | wrong schema version, invalid discriminant | host/contract boundary |
| Policy | wrong phase, denied capability, incomplete evidence | Semantic contract |
| Effect | timeout, output limit, process failure | effect broker |
| State | stale epoch, digest mismatch, atomic write failure | state adapter |
| Audit | append failure, invalid chain | audit layer |

A recoverable request error must not crash the server process. Startup admission failures must fail closed.

## 12. v0 vertical slice

The first governed workflow is intentionally narrow:

```text
initialize
→ tools/list
→ load_current_task
→ inspect_task_scope
→ run_required_checks
→ complete_task
```

Expected positive state path:

```text
Boot
→ TaskLoaded
→ ScopeValidated
→ ChecksRunning
→ ChecksPassed
→ Completed
```

The v0 slice validates scope and registered checks. It does not edit code, commit, push, merge, create pull requests, or contact remote services.

## 13. Explicit v0 exclusions

The following are outside the v0 architecture:

- GUI or desktop interface;
- arbitrary shell execution;
- unrestricted filesystem access;
- implicit network access;
- Git commit, push, force-push, merge, or branch deletion;
- GitHub issue or pull-request mutation;
- package installation;
- dynamic plugins;
- remote orchestration;
- native Semantic AOT compilation;
- distributed state or audit storage;
- cloud telemetry.

## 14. Architectural review checklist

Every implementation PR must answer:

- Which layer owns the new behavior?
- Does the change duplicate policy between Rust and Semantic?
- Is the capability narrower than the requested effect?
- Does the path preserve verifier-first execution?
- Can malformed input panic the process?
- Is output ordering deterministic?
- Does `stdout` remain protocol-only?
- Are state transitions atomic and auditable?
- Does the change widen v0 scope?
- Is a new crate justified by real ownership rather than anticipation?

## 15. Evolution rule

The architecture may evolve only through explicit, reviewable changes.

A new capability, state, tool, effect, or ABI field must define:

- owner;
- schema/version impact;
- security impact;
- deterministic behavior;
- tests;
- migration or compatibility rule;
- whether the change belongs to v0 or a later milestone.

No implementation convenience may silently move authority from Semantic policy into Rust transport code.