# Themis State Machine

**Status:** Normative workflow model for Themis v0  
**Scope:** Repository-local task governance from bootstrap to qualified completion

## 1. Purpose

Themis is a state-governed MCP server. Its available tools, evidence requirements, and legal next actions are derived from an explicit workflow state rather than from a long natural-language prompt.

This document defines:

- the v0 workflow phases;
- legal transitions;
- transition preconditions;
- tool exposure by phase;
- evidence requirements;
- quad verdict semantics;
- epoch and replay protection;
- persistence and recovery behavior;
- completion and blocking rules.

The state machine governs engineering workflow policy. It is distinct from the MCP transport session lifecycle.

## 2. Two independent state domains

Themis maintains two separate state machines.

### 2.1 MCP session state — Rust-owned

```text
Created
  ↓ initialize
Initialized
  ↓ shutdown / EOF
Closing
```

This state controls protocol ordering only.

Examples:

- `tools/list` before `initialize` is a session error;
- duplicate initialization is handled by the Rust host;
- EOF terminates the process cleanly.

### 2.2 Workflow state — Semantic-owned policy

```text
Boot
→ TaskLoaded
→ ScopeValidated
→ ChecksRunning
→ ChecksPassed
→ Completed
```

Failure and stop states:

```text
ChecksFailed
Blocked
```

The workflow state controls which development action is legal and what evidence is required.

The two domains must never be collapsed into one enum or one authority model.

## 3. Canonical workflow state

The persisted v0 state is conceptually:

```text
WorkflowState {
  schema_version,
  contract_digest,
  task_id,
  phase,
  epoch,
  completed_steps,
  last_evidence_digest,
  last_event_sequence
}
```

### 3.1 Fields

| Field | Meaning |
|---|---|
| `schema_version` | State encoding version |
| `contract_digest` | Digest of the admitted embedded Themis contract |
| `task_id` | Stable identifier from the current task manifest |
| `phase` | Current workflow phase |
| `epoch` | Monotonic compare-and-swap version |
| `completed_steps` | Stable ordered set of completed workflow steps |
| `last_evidence_digest` | Digest of evidence supporting the last committed transition |
| `last_event_sequence` | Last committed audit sequence |

The concrete encoding may evolve, but these invariants are required.

## 4. Workflow phases

### 4.1 `Boot`

Meaning:

- no task has been admitted for the current workflow;
- repository context may be known, but task policy is not active;
- no repository check may execute.

Permitted tool:

```text
load_current_task
```

Expected outcomes:

- `T`: task manifest is valid and bound to current repository context;
- `F`: manifest is invalid or explicitly forbidden;
- `N`: task manifest is absent or required repository metadata is unavailable;
- `S`: manifest fields or repository facts contradict each other.

Legal next phases:

```text
TaskLoaded
Blocked
```

### 4.2 `TaskLoaded`

Meaning:

- a versioned task manifest has been loaded;
- task identity, goal, base HEAD, path rules, required checks, and publication restrictions are known;
- repository scope has not yet been proven.

Permitted tool:

```text
inspect_task_scope
```

Required evidence:

- normalized repository root;
- current HEAD;
- task base HEAD;
- sorted changed-path list;
- normalized allowed and forbidden path policies.

Legal next phases:

```text
ScopeValidated
Blocked
```

### 4.3 `ScopeValidated`

Meaning:

- changed paths are within the admitted task boundary;
- no forbidden-path conflict remains unresolved;
- required checks may now execute.

Permitted tool:

```text
run_required_checks
```

Required evidence before entry:

- scope verdict `T`;
- supporting changed-path evidence digest;
- expected task ID;
- expected source phase and epoch.

Legal next phases:

```text
ChecksRunning
Blocked
```

### 4.4 `ChecksRunning`

Meaning:

- the registered check sequence has started;
- effects are executed in manifest order;
- no completion decision may be issued before all required results are available.

Permitted behavior:

- continue the current registered check sequence;
- evaluate completed check observations;
- terminate the sequence on a hard failure according to contract policy.

No unrelated MCP workflow tool is exposed while checks are running.

Legal next phases:

```text
ChecksPassed
ChecksFailed
Blocked
```

### 4.5 `ChecksFailed`

Meaning:

- at least one required registered check failed, timed out, exceeded limits, or produced invalid evidence;
- the task cannot complete in the current workflow attempt.

Permitted tools:

```text
none in v0
```

The client receives structured failure details and may modify the repository outside the current Themis state flow only through a later explicitly designed workflow. Themis v0 does not provide patching tools.

This phase is terminal for the current attempt unless a future milestone introduces an explicit reset transition.

### 4.6 `ChecksPassed`

Meaning:

- every required check produced accepted evidence;
- scope evidence remains bound to the same task, contract, and repository epoch assumptions;
- completion may be requested.

Permitted tool:

```text
complete_task
```

Required evidence:

- ordered check-result set;
- all required checks present exactly once;
- accepted result for every check;
- aggregate evidence digest;
- audit sequence continuity.

Legal next phases:

```text
Completed
Blocked
```

### 4.7 `Completed`

Meaning:

- the admitted v0 workflow has completed;
- all required transitions and evidence have been committed and audited;
- no further workflow tool is exposed.

Permitted tools:

```text
none
```

This is a terminal state.

Completion does not imply:

- a Git commit exists;
- a branch was pushed;
- a pull request was created;
- remote CI passed;
- the implementation is production-ready.

It means only that the local task contract defined for v0 was satisfied.

### 4.8 `Blocked`

Meaning:

- continuation is unsafe, contradictory, unsupported, or impossible under the current contract;
- no automatic recovery path is available in v0.

Typical causes:

- contract digest mismatch;
- corrupted state;
- contradictory task policy;
- stale or concurrent transition conflict that cannot be safely retried;
- unknown required check;
- audit append failure before a mandatory state change;
- repository escape or symlink-policy violation;
- unsupported schema version.

Permitted tools:

```text
none in v0
```

This is a terminal fail-closed state for the current attempt.

## 5. State transition graph

```text
                         ┌─────────────┐
                         │   Blocked   │
                         └──────▲──────┘
                                │ fail closed
                                │
┌──────┐  load task  ┌──────────┴─┐  scope T  ┌────────────────┐
│ Boot │────────────→│ TaskLoaded │──────────→│ ScopeValidated │
└──────┘             └────────────┘            └───────┬────────┘
                                                       │ run checks
                                                       ▼
                                                ┌───────────────┐
                                                │ ChecksRunning │
                                                └──────┬───┬────┘
                                                       │   │
                                                    pass   fail
                                                       │   │
                                                       ▼   ▼
                                            ┌────────────┐ ┌────────────┐
                                            │ChecksPassed│ │ChecksFailed│
                                            └─────┬──────┘ └────────────┘
                                                  │ complete
                                                  ▼
                                            ┌───────────┐
                                            │ Completed │
                                            └───────────┘
```

Any phase may transition to `Blocked` when a mandatory invariant fails and no safe local rejection is sufficient.

## 6. Legal transition table

| Source | Trigger | Required decision | Target | Mutates state |
|---|---|---:|---|---:|
| `Boot` | `load_current_task` | `T` | `TaskLoaded` | yes |
| `Boot` | manifest absent | `N` | `Boot` | no |
| `Boot` | invalid manifest | `F` | `Boot` or `Blocked` by reason | conditional |
| `Boot` | contradictory manifest | `S` | `Blocked` | yes |
| `TaskLoaded` | `inspect_task_scope` | `T` | `ScopeValidated` | yes |
| `TaskLoaded` | scope violation | `F` | `TaskLoaded` | no |
| `TaskLoaded` | missing evidence | `N` | `TaskLoaded` | no |
| `TaskLoaded` | conflicting path policy | `S` | `Blocked` | yes |
| `ScopeValidated` | `run_required_checks` accepted | `T` | `ChecksRunning` | yes |
| `ChecksRunning` | all checks accepted | `T` | `ChecksPassed` | yes |
| `ChecksRunning` | any required check rejected | `F` | `ChecksFailed` | yes |
| `ChecksRunning` | incomplete result set | `N` | `ChecksRunning` | no |
| `ChecksRunning` | contradictory result set | `S` | `Blocked` | yes |
| `ChecksPassed` | `complete_task` | `T` | `Completed` | yes |
| any nonterminal | mandatory invariant failure | `F` or `S` | `Blocked` | yes |

A verdict alone does not determine mutation. `DecisionReason` and transition policy decide whether the source state remains unchanged or enters `Blocked`.

## 7. Quad verdict semantics

Themis uses Semantic four-valued logic as an explicit decision domain.

| Value | Workflow meaning |
|---|---|
| `T` | Required condition is proven by accepted evidence |
| `F` | Condition is disproven or action is explicitly denied |
| `N` | Evidence is incomplete, unavailable, or not yet observed |
| `S` | Evidence or policy is contradictory |

Examples:

```text
current HEAD == manifest base HEAD                  → T
changed path matches forbidden policy               → F
Git status could not be observed                     → N
path is simultaneously required and forbidden       → S
```

Authorization failures remain structured decision reasons. They do not introduce a fifth logical state.

## 8. Tool exposure matrix

| Phase | Exposed tools |
|---|---|
| `Boot` | `load_current_task` |
| `TaskLoaded` | `inspect_task_scope` |
| `ScopeValidated` | `run_required_checks` |
| `ChecksRunning` | none as a new top-level action |
| `ChecksFailed` | none |
| `ChecksPassed` | `complete_task` |
| `Completed` | none |
| `Blocked` | none |

`tools/list` is a guidance surface only.

Every `tools/call` must be independently revalidated against:

- current phase;
- expected epoch;
- task ID;
- contract digest;
- tool schema;
- effective capabilities;
- required evidence preconditions.

A tool omitted from `tools/list` is not automatically safe to call.

## 9. Transition request model

A state-changing decision is conceptually:

```text
TransitionRequest {
  task_id,
  contract_digest,
  expected_phase,
  expected_epoch,
  requested_tool,
  decision,
  evidence_digest,
  target_phase
}
```

The Rust state adapter validates:

1. state schema is supported;
2. task ID matches;
3. contract digest matches;
4. persisted phase equals `expected_phase`;
5. persisted epoch equals `expected_epoch`;
6. target transition is structurally legal;
7. required audit record can be appended;
8. atomic persistence succeeds.

Only then is the transition committed.

## 10. Epoch semantics

`epoch` is a monotonic workflow version.

Rules:

- initial state starts at a documented fixed epoch, normally `0`;
- every committed transition increments epoch exactly once;
- non-mutating decisions do not increment epoch;
- failed compare-and-swap attempts do not increment epoch;
- audit events may advance their own sequence independently;
- epoch wraparound is an explicit hard failure;
- clients must provide the epoch observed when the tool was listed or state was read.

Example:

```text
persisted: phase=TaskLoaded, epoch=4
request:   expected_phase=TaskLoaded, expected_epoch=4
result:    commit ScopeValidated, epoch=5
```

A replay of the same request with epoch `4` is rejected as stale.

## 11. Evidence model

A transition is supported by one or more evidence records.

Conceptual shape:

```text
EvidenceRecord {
  evidence_type,
  source,
  effect_id,
  stable_payload,
  payload_digest,
  observed_for_phase,
  observed_at_epoch
}
```

Evidence rules:

- evidence is bound to task ID, contract digest, phase, and epoch;
- evidence ordering is deterministic;
- repository paths are normalized and sorted;
- process output is bounded;
- full unbounded stdout/stderr is not a decision input;
- timestamps may be recorded for audit but are excluded from deterministic policy unless explicitly normalized;
- unavailable evidence yields `N` unless policy requires fail-closed `F`;
- contradictory accepted observations yield `S`.

## 12. Required-check execution

The manifest defines required check IDs, not shell strings.

Example:

```yaml
required_checks:
  - workspace-fmt-check
  - workspace-test
```

Execution sequence:

```text
manifest order
  ↓
resolve registered CheckId
  ↓
validate capability and limits
  ↓
execute bounded effect
  ↓
record structured observation
  ↓
Semantic evaluates accumulated evidence
```

Rules:

- unknown check IDs fail closed;
- duplicate required checks are rejected or normalized by explicit schema rule;
- order is stable;
- every required check appears exactly once in the final evidence set;
- optional checks cannot substitute for required checks;
- a successful process exit is not sufficient if required evidence fields are absent.

## 13. Failure policy

Themis distinguishes rejection from corruption or unsafe continuation.

### 13.1 Stay in current phase

Use when:

- user supplied invalid tool arguments;
- evidence is incomplete;
- scope check returns a clear non-contradictory violation;
- requested tool is not legal in the current phase;
- stale epoch is detected and current state remains valid.

### 13.2 Enter `ChecksFailed`

Use when:

- a known required check completes with a rejected result;
- a check times out according to registered policy;
- bounded process execution fails in a way classified as a check failure;
- required check evidence is invalid but state integrity remains intact.

### 13.3 Enter `Blocked`

Use when:

- state or audit integrity cannot be trusted;
- policy is contradictory;
- contract digest differs from persisted state;
- an unknown required effect is requested;
- an atomic write cannot establish a known valid result;
- repository containment cannot be proven;
- supported schema cannot interpret persisted state safely.

## 14. Persistence contract

The state file is repository-local and must be written atomically.

Required sequence:

```text
read current state
  ↓
validate schema and digest
  ↓
compare phase + epoch
  ↓
prepare next state
  ↓
prepare required audit event
  ↓
write temporary state
  ↓
flush according to platform policy
  ↓
atomic replace
  ↓
confirm committed state
```

The implementation must define ordering between audit append and state replace so that a committed transition is never silently unaudited.

Acceptable designs include:

- write-ahead audit intent followed by state commit and final audit confirmation;
- transactional envelope containing state and audit sequence;
- another documented scheme that passes crash-recovery tests.

## 15. Restart and recovery

At process startup:

1. verify embedded contract;
2. compute contract digest;
3. locate repository-local state;
4. validate state schema;
5. validate contract digest binding;
6. validate audit sequence continuity where present;
7. recover only from a documented last-known-valid representation;
8. otherwise fail closed into an operationally blocked startup condition.

Themis must not guess a phase from partial files or process output.

## 16. Concurrency model

The v0 workflow supports one logical state writer per repository.

Concurrency controls must ensure:

- two clients cannot both commit from the same epoch;
- stale requests are rejected;
- read-only state inspection may occur concurrently if it cannot observe partial writes;
- effect execution is associated with one expected phase and epoch;
- evidence collected for a stale epoch cannot be committed into a newer state;
- process restart does not create a second valid writer silently.

The exact lock mechanism is platform-specific, but correctness is defined by compare-and-swap state semantics rather than by advisory convention alone.

## 17. Task manifest relationship

The task manifest is immutable input for one admitted task attempt.

Minimum v0 fields:

```yaml
task_id: THEMIS-0001
goal: "Describe the intended engineering outcome"
base_head: "full commit digest"
allowed_paths:
  - "crates/themis-host/**"
forbidden_paths:
  - ".github/workflows/**"
required_checks:
  - "workspace-test"
publication:
  push: forbidden
  pull_request: forbidden
```

Manifest changes after `TaskLoaded` require a new admitted task attempt. Themis must not silently apply an edited manifest to an existing state epoch.

## 18. State invariants

The following invariants are mandatory:

1. One persisted state has one active contract digest.
2. Epoch increases only on committed transitions.
3. A transition has one source phase and one target phase.
4. A committed transition references evidence or an explicit no-evidence policy.
5. `Completed`, `ChecksFailed`, and `Blocked` are terminal in v0.
6. `ChecksPassed` is impossible without accepted evidence for every required check.
7. `ScopeValidated` is impossible without accepted changed-path evidence.
8. `tools/call` is revalidated independently of `tools/list`.
9. A stale request cannot mutate state.
10. A state transition cannot bypass required audit policy.
11. Unknown phase values fail closed.
12. The state machine never grants arbitrary shell, network, or publication authority.

## 19. Test matrix

Minimum tests:

### Positive

- `Boot → TaskLoaded` with a valid manifest;
- `TaskLoaded → ScopeValidated` with allowed paths;
- `ScopeValidated → ChecksRunning`;
- `ChecksRunning → ChecksPassed` with all accepted checks;
- `ChecksPassed → Completed`;
- restart at every nonterminal persisted phase.

### Negative

- wrong-phase tool call;
- stale epoch replay;
- task ID mismatch;
- contract digest mismatch;
- invalid state schema;
- missing task manifest;
- absolute or traversal path in manifest;
- forbidden changed path;
- conflicting allowed and forbidden policy;
- unknown required check;
- duplicate or missing check result;
- check timeout;
- contradictory evidence;
- audit append failure;
- interrupted atomic write;
- concurrent commit attempt;
- request against terminal state.

## 20. Evolution rule

A new phase or transition requires:

- explicit source and target phases;
- triggering tool or internal event;
- required capability set;
- evidence contract;
- quad-verdict mapping;
- failure and recovery behavior;
- epoch semantics;
- audit event definition;
- positive and negative tests;
- versioning impact.

No transition may be introduced solely by adding an `if` branch in Rust. Workflow authority remains in the Semantic contract.