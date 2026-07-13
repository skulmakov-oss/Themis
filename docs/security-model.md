# Themis Security Model

**Status:** Security baseline for Themis v0  
**Scope:** Headless local MCP governance for repository-scoped engineering workflows

## 1. Security objective

Themis must reduce the authority of an AI coding agent from an open-ended instruction-following process to a bounded workflow participant.

The security objective is not to make an AI agent trustworthy by assumption. It is to make unsafe actions unavailable, detectable, or rejectable through explicit contracts, capabilities, state, evidence, and host enforcement.

```text
Untrusted or fallible request
        ↓
Protocol validation
        ↓
Verified Semantic policy decision
        ↓
Capability check
        ↓
Registered bounded effect
        ↓
Evidence
        ↓
Atomic audited transition
```

Themis v0 is a local governance component, not a complete operating-system sandbox or remote security service.

## 2. Security principles

### 2.1 Verifier first

Only admitted SemCode may enter the canonical execution path.

```text
embedded themis.smc
  ↓ verifier admission
verified program / entry token
  ↓ execution
```

A malformed, unsupported, or unverifiable contract prevents operational startup.

### 2.2 Least authority

Every component receives only the authority required for its current role.

- the MCP client receives tool schemas, not host handles;
- Semantic receives typed observations, not filesystem or process access;
- the effect broker receives an `EffectId`, not an arbitrary command string;
- state persistence receives one validated transition envelope, not general repository mutation authority;
- audit receives bounded structured events, not unrestricted environment capture.

### 2.3 Deny by default

Unknown tools, capabilities, effects, schema versions, state phases, and ABI discriminants fail closed.

No operation is permitted merely because it is not explicitly forbidden.

### 2.4 Separate decision from execution

Semantic owns the policy decision. Rust owns bounded effect execution.

Neither layer may silently absorb the other's authority:

- Rust must not hard-code workflow success policy;
- Semantic must not execute host commands directly;
- the client must not decide whether its own action was authorized.

### 2.5 Evidence before transition

A process exit code, client claim, or natural-language report is not automatically sufficient evidence.

Transitions require the evidence shape defined by the contract.

### 2.6 Atomic and auditable mutation

A state transition is accepted only when:

- expected task, phase, epoch, and contract digest match;
- evidence is accepted;
- required audit semantics can be satisfied;
- persistence completes atomically.

### 2.7 Deterministic policy

Identical validated inputs must produce equivalent decisions. Hidden environment reads, ambient network data, unstable ordering, and unbounded process output are excluded from the decision core.

## 3. Trust model

### 3.1 Trusted computing base

The v0 trusted computing base includes:

- the compiled `themis-host` binary;
- the Semantic verifier and VM used by the host;
- the embedded `themis.smc` bytes after digest and verifier admission;
- the Rust–Semantic ABI implementation;
- the capability/effect registry;
- the state adapter and atomic write implementation;
- the audit append and validation implementation;
- the operating-system primitives relied on for process isolation, file permissions, and atomic replace;
- the pinned build toolchain and dependencies used to produce the release artifact.

The goal is to keep this base small and reviewable.

### 3.2 Untrusted or fallible inputs

Treat the following as untrusted:

- all MCP requests and tool arguments;
- AI-agent reasoning and reports;
- repository contents;
- task manifests before validation;
- changed paths and symlinks;
- process stdout and stderr;
- exit codes without supporting evidence;
- environment variables;
- persisted state before schema and digest validation;
- audit files before chain validation;
- external Semantic artifacts not embedded and admitted at build/startup;
- user-provided paths, patterns, or identifiers.

### 3.3 Trusted but constrained inputs

Some inputs may be trusted for identity but remain constrained for policy:

- repository root selected by the operator;
- embedded contract digest;
- registered effect definitions;
- canonical state path;
- compiled-in server identity and ABI versions.

## 4. Trust boundaries

```text
┌──────────────────┐
│ AI coding agent  │  untrusted requests
└────────┬─────────┘
         │ MCP / JSON-RPC
═════════╪════════════════ Transport boundary
         ▼
┌──────────────────┐
│ Rust MCP host    │  framing, lifecycle, typed validation
└────────┬─────────┘
         │ typed ContractRequest
═════════╪════════════════ Contract ABI boundary
         ▼
┌──────────────────┐
│ Semantic policy  │  verified deterministic decision
└────────┬─────────┘
         │ EffectRequest / TransitionDecision
═════════╪════════════════ Capability boundary
         ▼
┌──────────────────┐
│ Effect broker    │  registered bounded execution
└────────┬─────────┘
         │ OS / repository observation
═════════╪════════════════ Host resource boundary
         ▼
┌──────────────────┐
│ Git/Cargo/files  │  untrusted observations and mutable state
└──────────────────┘
```

Crossing a boundary requires validation and conversion into the receiving layer's typed model.

## 5. Protected assets

Themis protects:

| Asset | Required protection |
|---|---|
| Workflow authority | Only verified Semantic policy chooses legal transitions |
| Repository scope | Effects remain inside the admitted repository root and path policy |
| Host process | Malformed requests and tool failures do not crash or hijack the process |
| Contract integrity | Embedded contract digest and verifier admission are enforced |
| State integrity | Phase/epoch transitions are atomic and replay-resistant |
| Audit integrity | Events are ordered, bounded, and linked to state/evidence |
| Protocol integrity | `stdout` contains MCP JSON-RPC only |
| Secrets | Environment and credentials are not exposed or logged |
| Build integrity | Release artifact is reproducible enough to identify source and contract inputs |
| User privacy | No telemetry or hidden external upload exists in v0 |

## 6. Adversary and failure model

The v0 model considers:

- a confused or hallucinating AI agent;
- a client intentionally calling hidden or wrong-phase tools;
- malformed or oversized JSON-RPC input;
- repository files designed to escape path checks;
- symlink and traversal attacks;
- command injection attempts through task manifests or tool arguments;
- stale or replayed transition requests;
- two clients racing the same workflow state;
- corrupted or manually edited state files;
- corrupted embedded or external SemCode;
- process output intended to exhaust memory or poison parsing;
- malicious environment variables;
- dependency or build-script compromise;
- accidental log output corrupting the MCP stream;
- audit or state write interruption;
- contradictory evidence;
- operator misconfiguration.

The v0 model does not claim protection against a fully compromised operating system, kernel, administrator account, compiler toolchain, or hardware platform.

## 7. Primary threats and controls

### 7.1 Prompt-based policy bypass

**Threat:** The client ignores or misinterprets natural-language restrictions.

**Controls:**

- policy is represented as an executable Semantic state machine;
- legal tools are derived from current state;
- every call is independently revalidated;
- forbidden effects are absent from the registry;
- environment protections remain authoritative even if the client requests otherwise.

### 7.2 Hidden-tool invocation

**Threat:** The client manually calls a tool not returned by `tools/list`.

**Controls:**

- `tools/list` is treated only as guidance;
- `tools/call` validates phase, epoch, task ID, schema, and capabilities;
- wrong-phase requests produce structured denial;
- no mutation occurs on denial.

### 7.3 Arbitrary command execution

**Threat:** A task field or tool argument becomes a shell command.

**Controls:**

- no `Shell(String)` capability exists;
- ABI carries stable `EffectId` values only;
- executable and arguments are defined in the trusted registry;
- no shell interpolation is used;
- working directory and environment are fixed by policy;
- unknown effect IDs fail closed.

### 7.4 Argument injection

**Threat:** User-controlled values become extra command-line options or paths.

**Controls:**

- fixed argument templates;
- strongly typed effect parameters;
- no concatenated command strings;
- explicit allowlists for acceptable values;
- repository-relative path normalization;
- tests for leading hyphens, separators, quoting, Unicode edge cases, and empty values.

### 7.5 Path traversal and repository escape

**Threat:** `../`, absolute paths, alternate separators, symlinks, or case rules escape the repository boundary.

**Controls:**

- canonical repository root established at startup;
- absolute paths rejected unless internally generated;
- lexical normalization before policy matching;
- traversal components rejected;
- symlink policy explicit and tested;
- resolved paths verified to remain below repository root;
- allowed and forbidden policy conflicts produce `S` or fail closed;
- changed paths sorted and represented in canonical repository-relative form.

### 7.6 Stale-state replay

**Threat:** A previously valid request is replayed after state changes.

**Controls:**

- every state-changing call includes expected phase and epoch;
- compare-and-swap persistence;
- epoch increments once per committed transition;
- evidence is bound to observed phase and epoch;
- stale evidence cannot commit into a newer state;
- rejected replays are audited.

### 7.7 Concurrent writers

**Threat:** Two clients both attempt to advance the same state.

**Controls:**

- single logical writer model;
- process or file lock as platform mechanism;
- phase/epoch compare-and-swap as correctness mechanism;
- atomic replace;
- one commit wins; all stale competitors fail;
- no merge of competing transition envelopes.

### 7.8 State-file tampering

**Threat:** State is edited, truncated, rolled back, or copied from another contract/task.

**Controls:**

- versioned schema;
- contract digest binding;
- task ID binding;
- epoch and audit-sequence validation;
- optional state digest or envelope checksum;
- last-known-valid recovery rules;
- unsupported or inconsistent state fails closed;
- no inferred phase from partial fields.

### 7.9 Contract substitution

**Threat:** A different `.smc` is executed than the one reviewed or bound to state.

**Controls:**

- release form embeds the artifact;
- digest exposed in status, state, and audit;
- verifier admission before operational startup;
- verified token reused for requests;
- no runtime contract replacement in v0;
- state with another digest is rejected.

### 7.10 Malformed contract result

**Threat:** Semantic returns an unsupported ABI version, invalid discriminant, or malformed decision.

**Controls:**

- versioned typed ABI;
- strict decoding;
- enum exhaustiveness;
- unknown fields/discriminants handled by explicit compatibility policy;
- malformed results fail closed;
- raw JSON is not canonical at the execution boundary.

### 7.11 Resource exhaustion

**Threat:** Oversized input, unbounded output, infinite execution, or too many requests exhaust resources.

**Controls:**

- maximum JSON-RPC message size;
- bounded line/read buffers;
- Semantic VM quotas and traps;
- effect timeout;
- stdout/stderr byte limits;
- bounded audit excerpts;
- request-rate or sequential-processing policy where required;
- maximum manifest size, path count, check count, and tool argument size;
- deterministic cleanup after recoverable failures.

### 7.12 MCP stream corruption

**Threat:** Logging or panic text is written to `stdout` and breaks protocol framing.

**Controls:**

- protocol output exclusively on `stdout`;
- diagnostics and tracing exclusively on `stderr`;
- process-level golden tests;
- panic handling policy that does not emit arbitrary protocol bytes;
- no dependency permitted to initialize stdout logging implicitly.

### 7.13 Secret leakage

**Threat:** Environment variables, credentials, repository secrets, or process output enter responses or audit logs.

**Controls:**

- minimal environment allowlist for effects;
- no environment dump capability;
- output redaction where a registered effect can expose secrets;
- bounded excerpts disabled for sensitive effects;
- credentials not required for v0 because network and publication are forbidden;
- audit stores structured fields and digests, not full ambient context.

### 7.14 Audit omission or forgery

**Threat:** A transition occurs without evidence, events are reordered, or records are silently deleted.

**Controls:**

- monotonic event sequence;
- state references last audit sequence and evidence digest;
- optional previous-event hash chaining;
- required audit append participates in transition commit protocol;
- independent audit parser and validator;
- local file permissions;
- audit failure blocks mandatory mutation.

### 7.15 Dependency and build compromise

**Threat:** A dependency, build script, or toolchain changes the host or embedded contract unexpectedly.

**Controls:**

- committed lockfile for applications;
- minimal dependency set;
- review of build scripts and proc macros;
- pinned Semantic toolchain version or commit;
- contract and host build metadata;
- reproducible artifact check where feasible;
- dependency audit and license review before release;
- CI uses locked dependency resolution;
- no download-and-execute step in normal build without explicit review.

## 8. Capability model

A capability is an explicit authority token or compiled permission associated with a typed operation.

Initial v0 capabilities:

```text
ReadHarnessState
ReadRepositoryMetadata
ReadGitHead
ReadGitStatus
ReadChangedPaths
RunRegisteredCheck
ValidateChangedPaths
AdvanceStateAtomically
AppendAuditEvent
```

Capabilities intentionally absent from v0:

```text
ArbitraryShell
NetworkAccess
WriteRepositoryFile
GitCommit
GitPush
GitForcePush
GitMerge
DeleteBranch
CreatePullRequest
InstallPackage
LoadDynamicPlugin
```

Rules:

- capabilities are closed enums or equivalent stable identifiers;
- unknown capabilities fail closed;
- capabilities may be attenuated by phase;
- possession of one capability does not imply another;
- read and write authority are separate;
- capability checks occur in Rust immediately before effect execution;
- Semantic cannot mint host capabilities;
- the client cannot supply effective capabilities as trusted input.

## 9. Effect registry security

Every effect entry must document:

```text
EffectId
purpose
required capability
implementation/executable
fixed arguments
typed parameters
working directory
environment allowlist
timeout
stdout limit
stderr limit
result schema
mutation class
security notes
```

The registry is part of the trusted codebase.

An effect may execute only when:

1. requested by admitted Semantic policy;
2. legal for current phase and epoch;
3. present in the registry;
4. required capability is effective;
5. parameters pass validation;
6. repository containment holds;
7. resource limits are installed;
8. audit intent is established where required.

## 10. Process execution policy

For registered external processes:

- invoke the executable directly, never through a command shell;
- pass arguments as an array;
- set a known working directory;
- clear environment by default, then add allowlisted variables;
- close or control inherited handles;
- enforce timeout and termination policy;
- capture bounded stdout/stderr separately;
- record exit status structurally;
- normalize platform-specific status into a stable result schema;
- treat spawn failure separately from check failure;
- do not parse terminal control sequences as trusted content.

Where an operation can be implemented safely in-process, prefer that over spawning an external tool, provided ownership remains clear.

## 11. Filesystem policy

Themis v0 filesystem access is narrow.

Allowed categories:

- read canonical task manifest;
- read validated workflow state;
- atomically replace workflow state;
- append audit events;
- observe repository metadata through registered adapters;
- access temporary files required for atomic persistence.

Forbidden by default:

- arbitrary file read requested by the client;
- arbitrary file write;
- writes to source files;
- access outside the repository-local Themis/Harness state area;
- following unvalidated symlinks;
- user-selected absolute output paths.

File permissions should restrict state and audit mutation to the current user where the platform supports it.

## 12. Network policy

Themis v0 has no network capability.

Consequences:

- no Git fetch or push;
- no GitHub API calls;
- no telemetry;
- no update check;
- no package installation;
- no remote contract loading;
- no remote audit upload;
- no DNS or HTTP dependency in the normal runtime path.

A future network capability requires a separate threat model, endpoint allowlist, credential model, replay policy, timeout policy, and audit contract.

## 13. MCP protocol security

The host must validate:

- JSON-RPC version;
- request/notification shape;
- method names;
- request ID type and preservation;
- initialization order;
- maximum message size;
- required fields;
- tool name and argument schema;
- duplicate or ambiguous fields according to parser policy.

Protocol errors and domain denials remain distinct.

Malformed input must not:

- panic the process;
- mutate workflow state;
- execute an effect;
- create a misleading success response;
- contaminate subsequent framing.

## 14. Semantic execution security

The host must:

- verify embedded SemCode before serving operational requests;
- use only canonical verified-entry APIs;
- configure bounded runtime quotas;
- reject unexpected traps as structured internal failures;
- avoid exposing raw host pointers, file descriptors, or process handles;
- validate every returned ABI object;
- prevent contract-controlled allocation or output from becoming unbounded;
- record the contract digest in state and audit.

The Semantic contract is policy code but remains subject to verifier, VM, ABI, capability, and resource constraints.

## 15. State and audit commit security

A transition must not create these invalid outcomes:

```text
state advanced, no required audit record
state partially written
state bound to wrong contract
state epoch advanced twice
accepted evidence belongs to old epoch
```

The implementation must define and test a crash-consistent protocol.

Security acceptance requires tests that interrupt or simulate failure at each persistence stage.

## 16. Logging and privacy

Themis v0 follows a local-first privacy contract.

```text
telemetry = forbidden
hidden analytics = forbidden
silent crash upload = forbidden
user-content upload = forbidden
```

Local diagnostics:

- go to `stderr`;
- avoid secrets and full repository contents;
- use structured error codes where feasible;
- may include local paths only when necessary and documented;
- remain separate from the deterministic audit model.

Local audit:

- stays on device;
- is inspectable and deletable by the user;
- stores bounded structured evidence;
- is not product analytics.

## 17. Release and supply-chain controls

Before a v0 release:

- build with a committed lockfile;
- run formatter, lints, unit, golden, negative, and end-to-end tests;
- record Rust and Semantic toolchain versions;
- record embedded contract digest;
- confirm the release binary needs no external `.smc`;
- inspect dependency licenses;
- audit dependencies for known vulnerabilities;
- confirm no unexpected network or shell dependency;
- confirm release help/version output contains no misleading security claims;
- produce a qualification report with limitations.

Code signing and reproducible-build guarantees may be added later, but the artifact must at minimum identify its source version and embedded contract digest.

## 18. Security test matrix

### Protocol

- malformed JSON;
- oversized input;
- invalid request ID;
- method before initialization;
- unknown method;
- unknown tool;
- invalid tool arguments;
- stdout contamination.

### Contract and ABI

- corrupted SemCode;
- unsupported SemCode version;
- missing entry point;
- wrong ABI schema version;
- invalid quad discriminant;
- malformed tool descriptor;
- excessive contract output;
- VM quota exhaustion.

### Scope and filesystem

- absolute path;
- `../` traversal;
- mixed separators;
- case-collision scenarios where relevant;
- symlink escape;
- forbidden-path match;
- allowed/forbidden contradiction;
- repository root changed during operation;
- state path replaced by symlink.

### Effects

- unknown `EffectId`;
- missing capability;
- injected argument;
- spawn failure;
- timeout;
- stdout/stderr overflow;
- hostile control characters;
- non-zero exit;
- environment secret absent from child process.

### State and replay

- stale epoch;
- concurrent transition;
- task mismatch;
- contract digest mismatch;
- truncated state;
- interrupted replace;
- audit append failure;
- audit sequence rollback;
- evidence from previous epoch;
- transition from terminal state.

### Privacy

- no network calls during end-to-end workflow;
- no telemetry endpoint or analytics dependency;
- no environment dump in logs;
- audit excerpts obey limits;
- secrets are redacted or excluded.

## 19. Security non-goals for v0

Themis v0 does not claim:

- protection from a compromised OS or administrator;
- hardware-backed isolation;
- multi-user hostile tenancy;
- remote-agent authentication;
- cryptographic non-repudiation;
- formally verified Rust host code;
- complete protection from malicious compiler or dependency supply chains;
- safe arbitrary shell execution;
- secure network or GitHub-write automation;
- production certification for high-assurance environments.

These are not excuses to weaken v0 controls. They define the boundary of the current claim.

## 20. Security review checklist

Every PR that changes a tool, state, ABI, effect, or persistence path must answer:

- What new authority is introduced?
- Which component owns the decision?
- Can client-controlled data reach a command, path, or environment variable?
- What is the failure mode for unknown input?
- Is the operation available in the wrong phase?
- Can it be replayed?
- Can two clients race it?
- Is repository containment proven?
- Are outputs bounded?
- Can secrets enter logs, responses, or audit?
- Does the change add network access?
- Does it widen the trusted computing base?
- Are negative tests included?
- Does the README or documentation overstate the resulting security posture?

## 21. Vulnerability handling

Until a dedicated security policy is added, suspected vulnerabilities should be reported privately to the repository owner rather than disclosed with an immediately exploitable public proof before coordination.

A future `SECURITY.md` should define:

- supported versions;
- private reporting channel;
- acknowledgement expectations;
- disclosure coordination;
- severity and remediation process.

## 22. Evolution rule

Any future mutating or networked capability requires a separate security decision record.

The record must define:

- threat model delta;
- credentials and identity model;
- scope and attenuation;
- rollback and recovery;
- replay protection;
- audit requirements;
- operator confirmation requirements;
- negative tests;
- release-claim impact.

No capability becomes safe merely because the Semantic contract can name it. Host enforcement remains mandatory.