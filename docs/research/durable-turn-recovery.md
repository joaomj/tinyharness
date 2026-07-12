# Durable Turn Recovery Semantics

## Executive decision

TinyHarness should use a per-session SQLite database as the authoritative recovery journal. A turn, each provider attempt, each approval, and each side-effecting operation must have stable identifiers and explicit states. Before TinyHarness starts a hook or mutation tool, it must commit an operation intent and the evidence needed to reconcile that operation. After execution, it must commit the observed result. On restart, an operation whose intent is durable but whose result is absent is **uncertain**; TinyHarness must reconcile it or ask the user, never execute it automatically.

Provider requests are a separate boundary. Streaming bytes are not a durable assistant message, and the OpenAI Responses API documentation reviewed on 2026-07-12 does not document replay safety for a client-generated operation key. Therefore TinyHarness may retry an interrupted provider attempt only by creating a new attempt, preserving the previous attempt as interrupted, and clearly exposing that the provider could have processed the old request. It must not infer that retrying is exactly-once.

Keep transcript JSONL and event traces as human-readable, regenerable projections. They are not recovery authority because current writes are unsynchronized appends (`src/session.rs:59-68`, `src/turn_trace.rs:294-310`) and related records cannot be committed atomically.

## Scope and terminology

- **Turn**: one accepted user input through a terminal outcome: `completed`, `failed`, or `cancelled`. `needs_attention` is a durable nonterminal recovery state requiring a user decision.
- **Provider attempt**: one request to a model provider and its streamed response. A retry is a new attempt, not continuation of the old one.
- **Operation**: one hook or tool invocation with a stable `operation_id` and canonical input.
- **Intent**: a durable declaration that TinyHarness is about to cross an external side-effect boundary.
- **Result**: the durable observation returned after an attempted operation.
- **Reconciliation**: determining whether an uncertain operation happened by inspecting durable external evidence without repeating it.
- **Uncertain**: intent is committed, but no definitive result is committed. It does not mean “not run.”
- **Projection**: transcript or trace output derived from authoritative records.

This model targets process termination, host restart, I/O failure, cancellation, and provider disconnection. It cannot make arbitrary external commands exactly-once. It instead prevents silent duplication and makes ambiguity explicit.

## Current behavior and gaps

The transcript serializer opens the session JSONL in append mode, writes JSON and a newline, and returns without `sync_all` (`src/session.rs:59-68`). The trace follows the same pattern (`src/turn_trace.rs:294-310`). A successful Rust `write_all` is not a durable-commit guarantee; `File::sync_all` is the API that attempts to synchronize file content and metadata to storage ([Rust `File::sync_all`](https://doc.rust-lang.org/std/fs/struct.File.html#method.sync_all)). POSIX likewise defines `fsync()` as requesting transfer of file data and associated status to the storage device ([POSIX `fsync`](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fsync.html)).

Hook tracing records `HookStart` before awaiting dispatch and `HookEnd` afterward (`src/hooks/runtime.rs:39-75`), but these records are best-effort trace appends, not a transactional journal. A crash between process spawn and `HookEnd` is indistinguishable from several other failures. The trace schema also identifies hooks by event/key rather than a unique invocation (`src/turn_trace.rs:168-187`).

Mutation tools execute directly in the serial loop (`src/agent.rs:785-793`). Their returned output is processed only after execution (`src/agent.rs:795-804`). There is no durable operation intent around that call. Checkpoints capture selected file baselines immediately before tools (`src/turn_checkpoint.rs:138-155`), but the checkpoint stack itself is an in-memory vector (`src/turn_checkpoint.rs:88-120`), and snapshots cover only recognized workspace file arguments (`src/turn_checkpoint.rs:163-180`), not shell, network, subprocess, or hook effects.

Approvals are also process-local. `ApprovalCache` contains only an in-memory `HashSet` (`src/approval.rs:18-27`) and checks it during approval (`src/approval.rs:60-72`). Restart therefore loses both grants and the exact approval associated with a pending operation.

OpenAI documents streaming as server-sent events that deliver incremental response events ([OpenAI streaming responses](https://platform.openai.com/docs/guides/streaming-responses)). Background responses can be polled by response ID, but background mode is a distinct API mode and is not a general guarantee that an interrupted foreground stream can be resumed ([OpenAI background mode](https://platform.openai.com/docs/guides/background)). The API overview does not document an idempotency guarantee for replaying model generation requests ([OpenAI API overview](https://platform.openai.com/docs/api-reference/introduction)). TinyHarness must not manufacture such a guarantee.

## Invariants

1. Every accepted turn has a stable `turn_id` before provider or local work starts.
2. Every provider request, hook, and tool call has a stable unique ID; a retry is a new provider attempt or operation linked to the original.
3. No side-effecting operation starts before its intent transaction commits.
4. Approval is bound to the operation ID plus a hash of canonical effective input. Rewritten input invalidates prior approval.
5. A committed operation result is terminal and is never executed again automatically.
6. Intent without a terminal result is uncertain and is never interpreted as “did not run.”
7. Recovery never appends a second assistant/tool message for the same durable message or operation ID.
8. The journal stores enough evidence to explain a recovery decision, but excludes secrets and unbounded output.
9. Transcript and trace projections may lag the journal and must be repairable idempotently.
10. A turn cannot become `completed` until all referenced operations and the final assistant message are terminal and durable.

## State model

### Turn state machine

```text
accepted
   |
   v
running <-------------------+
   |                         | approval or recovery resolution
   +--> awaiting_approval ---+
   +--> needs_attention -----+
   +--> completed
   +--> failed
   +--> cancelled
```

`accepted` is committed with the user message. `running` means orchestration may continue. `awaiting_approval` names the exact operation awaiting a decision. `needs_attention` means automatic recovery cannot safely decide, commonly because an operation is uncertain. Terminal states are immutable; “retry turn” creates a successor turn or attempt with linkage rather than reopening history.

### Provider attempt state machine

```text
prepared -> in_flight -> response_complete -> committed
                  |              |
                  +-> interrupted+
                  +-> failed
```

- `prepared`: request envelope, input-message IDs, provider/model, and request hash committed.
- `in_flight`: dispatch intent committed; request transmission may have started. This state is uncertain after a crash.
- `response_complete`: the full provider response and finish metadata are validated and committed in the same transaction.
- `committed`: response messages/tool calls have stable IDs and are attached to the turn.
- `interrupted`: stream ended without a complete validated response. Partial text may be retained for diagnostics but never enters conversation history as an assistant message.

After recovery, a `prepared` attempt with no dispatch intent may continue under the same attempt ID. If a provider supplies a durable response ID and an officially documented retrieve/resume mechanism, TinyHarness may reconcile an `in_flight` attempt with that mechanism. Otherwise recovery marks the `in_flight` attempt interrupted and requires an explicit retry policy. A replacement request gets a new attempt ID.

### Operation state machine

```text
proposed -> awaiting_approval -> ready -> intent_committed -> running
    |              |              |              |              |
    +-> denied     +-> denied     +-> cancelled  |              +-> succeeded
                                                  |              +-> failed
                                                  +-------------> uncertain

uncertain -> reconciled_succeeded
          -> retry_permitted ------> new proposed operation (user-authorized only)
          -> needs_attention
```

`intent_committed` is the side-effect boundary. For read-only operations, recovery may use a simpler replayable path, but only tools explicitly classified as read-only are eligible. Hooks are side-effecting by default because TinyHarness cannot inspect arbitrary commands.

## Durable information

The authoritative journal must relate stable identities and states for sessions, turns, messages, provider attempts, operations, approvals, reconciliation evidence, and projection progress. It must retain parent/retry relationships, canonical input and content hashes, terminal reasons, provider identifiers when supplied, and bounded result metadata. Original and hook-rewritten input hashes must remain distinguishable so audit history identifies what the model proposed and what actually ran. Exact schema, blob storage, and migration design are deferred.

## Side-effect boundary protocol

For every mutation tool and every hook:

1. Construct a stable `operation_id`, canonicalize effective input, classify the operation, and collect safe pre-execution evidence.
2. Obtain approval when required. Commit the approval against `operation_id + input_hash`.
3. Atomically commit the operation as `intent_committed` with its pre-evidence, then commit.
4. Execute exactly once in the current process. Transitioning an in-memory object to `running` does not weaken the rule: after restart, both `intent_committed` and `running` are uncertain.
5. In one transaction, store bounded/redacted output, post-evidence, error/exit metadata, and terminal operation state.
6. Attach the tool result to model history only from the committed terminal record.

This is an intent/result protocol, not a claim of exactly-once execution. SQLite transactions are atomic even when interrupted, subject to documented hardware and filesystem assumptions ([SQLite atomic commit](https://www.sqlite.org/atomiccommit.html)). The implementation specification must select and verify a SQLite journal and synchronization mode whose documented durability and concurrency guarantees meet this model. WAL has companion-file and version-specific operational requirements, so this decision does not mandate it ([SQLite WAL](https://www.sqlite.org/wal.html), [SQLite `synchronous`](https://www.sqlite.org/pragma.html#pragma_synchronous)).

Do not hold a SQLite transaction open while awaiting approval, provider I/O, hooks, or tools. Commit intent first, perform external work, then use a second short transaction for the result.

## Recovery matrix

| Interruption point | Durable observation | Recovery action | Automatic replay? |
| --- | --- | --- | --- |
| Before turn acceptance commit | No turn | Treat input as unaccepted; user may submit it again | No internal replay |
| After accepted, before provider preparation | Attempt absent | Create a prepared attempt | Yes, orchestration only |
| After `prepared`, before dispatch intent | Complete request envelope, no dispatch intent | Continue the same attempt to dispatch intent | Yes, orchestration only |
| After dispatch intent commits, before or during provider stream | Attempt `in_flight`, no complete response | Retrieve by provider response ID only if officially supported; otherwise mark interrupted | Never claim the request is replay-safe |
| During provider stream | Attempt `in_flight`, no complete response | Discard partial response from conversation; retrieve by provider response ID only if officially supported, otherwise mark interrupted | Never claim same request is replay-safe |
| After complete response, before tool-call projection | `response_complete` with full content/tool calls | Commit/repair messages and operation proposals idempotently by stable IDs | Yes, projection only |
| Before approval decision | Operation `awaiting_approval` | Re-display exact canonical input and request approval | No execution |
| After approval commit, before intent commit | Approved operation without intent | Verify input hash; continue to intent commit | Yes, orchestration only |
| After intent commit, before process spawn | Operation `intent_committed` | State is uncertain because crash timing cannot prove spawn did not occur; reconcile | No |
| During hook/tool execution | Intent/running, no result | Reconcile using operation-specific evidence; otherwise `needs_attention` | No |
| Cancellation before operation intent | Turn cancellation requested, no operation intent | Cancel pending orchestration and preserve partial diagnostics | No execution |
| Cancellation after operation intent | Intent/running, cancellation requested | Request process/provider cancellation, then reconcile exactly as a crash; cancellation is not evidence that execution stopped | No |
| After side effect, before result commit | Intent/running, external post-state may exist | Reconcile and commit `reconciled_succeeded` if evidence is conclusive | No |
| After result commit, before transcript/trace append | Terminal operation in DB, projection cursor behind | Regenerate missing projection line by stable ID | Yes, projection only |
| During projection append | Valid lines plus possibly torn final line | Truncate/ignore invalid final line and regenerate from cursor | Yes |
| After final message commit, before turn completion | Complete message and terminal operations | Validate invariants and mark turn completed | Yes, journal transition only |

## Duplicate prevention and reconciliation

Stable IDs prevent duplicate journal rows, but not duplicate external effects. Each operation class therefore needs an explicit reconciliation strategy:

| Operation | Pre/post evidence | Safe recovery decision |
| --- | --- | --- |
| `file_write` / `file_edit` / `apply_patch` | Path, existence, metadata, and content hash before; intended post-content hash where computable | If current hash equals intended post-hash, mark reconciled success. A pre-hash match does not prove the operation never ran; permit a new user-authorized operation only when the tool contract establishes that repeating the intended mutation is safe. Otherwise require attention. |
| Shell command | Command hash, cwd, selected environment hash, child PID/start marker when available; command-specific postcondition only if declared | Generic shell is not safely replayable. Without a declared, verifiable postcondition, require attention. A missing PID is not proof the command did not run. |
| Hook command | Event, hook source/key, canonical payload hash, command hash; optional hook-declared idempotency key/postcondition | Treat as arbitrary side effect. Never auto-replay an uncertain hook unless its contract explicitly declares replay safety and reconciliation. |
| Network/API tool | Remote idempotency key and remote object/request ID, but only where the remote API documents their semantics | Query remote state using documented identifier; otherwise require attention. |
| Read-only tool | Input hash and classification | May rerun if the classification is trustworthy; record a new attempt because output may have changed. |

For workspace files, `git status --porcelain` and `git diff` can provide useful supporting evidence, but not exclusive proof: untracked/ignored files and external effects may not be represented. Git documents `status --porcelain` as a stable, script-oriented format and `diff` as comparison of paths/content ([git-status](https://git-scm.com/docs/git-status), [git-diff](https://git-scm.com/docs/git-diff)). Prefer direct content hashes scoped to the files named by the operation, with git evidence as additional context.

Operations may expose optional contracts:

- `replay_safe`: backed by a concrete tool/hook guarantee, never guessed.
- `idempotency_key`: forwarded only when the external system documents its behavior.
- `reconcile`: a read-only procedure returning `applied`, `not_applied`, or `unknown` with evidence.
- `postcondition`: a bounded predicate established before execution.

Default all mutation tools and hooks to `unknown` replay safety.

## JSONL versus SQLite

JSONL remains useful for transcripts and diagnostics: it is inspectable, streamable, and a damaged final line can be isolated. It is a poor authority for this protocol because a turn transition spans multiple entities, uniqueness is not enforced, querying incomplete operations requires scanning, and separate files cannot atomically commit intent, approval, result, and projection cursor. Adding `sync_all` after every JSONL record would improve durability but not provide multi-record transactions or constraints.

SQLite provides atomic transactions, uniqueness constraints, schema migration, indexed recovery queries, and a single commit boundary. Its transaction semantics are explicit: changes remain atomic across commit/rollback boundaries ([SQLite transactions](https://www.sqlite.org/lang_transaction.html)). The operational cost is managing schema versions, SQLite errors, WAL companion files, and backups correctly. Those costs are justified because ambiguity at side-effect boundaries is the central problem.

Decision: SQLite is authoritative; JSONL is an idempotently regenerated projection. Migration treatment for old transcripts is deferred to an implementation specification; historical JSONL cannot establish recoverable operation boundaries.

## Redaction and minimization

The current trace recursively redacts object keys containing `api_key`, `apikey`, `token`, `secret`, or `password` and applies string patterns (`src/turn_trace.rs:218-248`). Retain this as defense in depth, but do not rely on key-name matching as the journal policy.

- Store allowlisted request metadata by default, not full environment variables, headers, hook payloads, stdout, or stderr.
- Hash canonical arguments for identity; store executable arguments only when resume requires them and apply field-aware redaction first.
- Bound stdout/stderr and partial provider data by bytes and retention time; record truncation explicitly.
- Never store approval prompt renderings as authority; store decision, actor, scope, operation ID, and input hash.
- Separate secret-bearing blobs from indexed metadata and support deletion without corrupting state transitions.
- Avoid persisting provider credentials, authorization headers, raw process environments, and secrets discovered in tool output.
- Include schema version and redaction-policy version so recovery can reject records it cannot interpret safely.

## Observable acceptance criteria

1. Killing TinyHarness at every boundary in the recovery matrix and restarting never automatically repeats an uncertain mutation tool or hook.
2. A turn accepted before a crash reappears with the same `turn_id`, exact state, and a user-visible recovery explanation.
3. A crash during provider streaming never adds partial assistant text or incomplete tool-call arguments to conversation history.
4. Retrying an interrupted provider call creates a new `attempt_id`; UI and journal retain the interrupted attempt and do not claim exactly-once delivery.
5. Approval survives restart only for the same operation ID and effective-input hash. Any hook rewrite or argument change requires a new approval.
6. A crash after file mutation but before result commit is reconciled as success when the intended post-content hash matches; a pre-hash match does not claim the operation never ran, and divergent content moves the turn to `needs_attention` without modification.
7. Generic shell and hook operations with intent but no result are not replayed unless an explicit tested replay/reconciliation contract applies.
8. A crash after an operation result commits but before JSONL append produces exactly one projected tool result after restart.
9. A torn final transcript/trace line is ignored or repaired, and all prior valid lines remain readable.
10. Database initialization verifies the implementation's specified transaction durability guarantees; failure to establish them prevents mutation execution with a clear error.
11. Fault-injection tests at transaction commit, process spawn, side-effect completion, result commit, and projection append assert the journal state and next recovery action.
12. Recovery output lists uncertain operations with tool/hook name, bounded redacted input summary, approval status, available evidence, and explicit choices; it never labels uncertainty as failure or success without evidence.
13. Journal records and projections contain no configured canary secrets after provider, approval, hook, and tool paths are exercised.
14. Recovery is idempotent: restarting repeatedly without user action does not change external state or create duplicate messages, attempts, approvals, or operations.
15. Cancellation before intent prevents execution; cancellation after intent leaves the operation terminal or uncertain and never implies that it did not run.

## Deferred dependencies

- Issue #6 must decide how worktree identity, pre-turn baselines, undo, and automatic completion commits supply reconciliation evidence. This note requires durable evidence but does not choose those semantics.
- Issue #8 must classify external mutation boundaries, define which tools require the intent/result protocol, and decide whether stronger confinement can narrow uncertainty. This note defaults unclassified tools and all hooks to side-effecting.
- Provider-specific retrieval or idempotency may be added only after its official contract is verified. The baseline state model assumes neither.
- Schema layout, migration mechanics, retention periods, encryption, and projection format belong to later implementation specifications.

## Primary sources

- [SQLite atomic commit](https://www.sqlite.org/atomiccommit.html)
- [SQLite transactions](https://www.sqlite.org/lang_transaction.html)
- [SQLite WAL](https://www.sqlite.org/wal.html)
- [SQLite `synchronous` pragma](https://www.sqlite.org/pragma.html#pragma_synchronous)
- [OpenAI streaming responses](https://platform.openai.com/docs/guides/streaming-responses)
- [OpenAI background mode](https://platform.openai.com/docs/guides/background)
- [OpenAI API overview](https://platform.openai.com/docs/api-reference/introduction)
- [Rust `File::sync_all`](https://doc.rust-lang.org/std/fs/struct.File.html#method.sync_all)
- [POSIX `fsync`](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fsync.html)
- [Git status](https://git-scm.com/docs/git-status)
- [Git diff](https://git-scm.com/docs/git-diff)
