# Native Compatibility Baseline

**Baseline:** SmallHarness `b8b6a6d` (`v1.2.0`, 2026-07-08)
**Decision status:** accepted research baseline for TinyHarness
**Purpose:** decide which native user-visible behaviors are compatibility commitments before adding another client.

## Executive decision

TinyHarness must preserve the native harness's **engine contract**: a user can run a persisted multi-turn coding-agent session against a selected backend; configured tools are selected, approval-gated, executed, and reported; results, traces, checkpoints, and configured integrations retain their stated semantics. A new client may present that contract differently, but it must not silently weaken documented approval, tool, session, or persistence outcomes.

The native terminal application's exact rendering, keystrokes, ANSI styling, banner, spinner, and completion menu are **terminal-only presentation**, not cross-client requirements. Native slash commands remain supported by the terminal client. A browser client must expose equivalent core outcomes where applicable, not emulate every slash command or line of terminal output.

This note is implementation-neutral. It states externally observable outcomes, not the internal protocol or storage design that TinyHarness must use.

## Scope and evidence rules

This baseline covers behavior present at commit `b8b6a6d`; it does not require later upstream features. “Covered” means a test or eval at that commit exercises an outcome. It does not mean a behavior is proven portable or browser-ready.

- Primary source repository: [GetSmallAI/SmallHarness](https://github.com/GetSmallAI/SmallHarness), pinned at [`b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/tree/b8b6a6d).
- Public contract source: [`README.md`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/README.md), especially tools and commands at lines 278-387 and configuration/integration sections at lines 632-876.
- Implementation evidence is cited as `b8b6a6d:path:line-range`.
- A capability is not classified as a shared engine commitment merely because it appears in command registration. Registration proves terminal reachability; behavior and tests determine the commitment.

## Compatibility classes

| Class | Meaning | Client requirement |
| --- | --- | --- |
| **Core engine** | Shared session, agent, safety, persistence, and configured-integration semantics. | Every client that exposes the capability must preserve its outcome. |
| **Core terminal** | A supported native coding workflow whose semantic result matters, but whose command entry point is terminal-specific. | Keep in the terminal client; another client may provide an equivalent workflow rather than the command text. |
| **Client-specific terminal-only** | Terminal interaction or display behavior with no independent engine outcome. | Preserve only for terminal compatibility. Do not block another client. |
| **Optional configured integration** | Disabled by default or requires explicit local configuration, credentials, or a supporting service. | Preserve its contract only when enabled; do not enable it implicitly in a new client. |
| **Experimental** | Upstream explicitly labels the behavior experimental or beta. | Preserve only behind its existing opt-in boundary. No baseline capability is placed here without source evidence. |

## Capability inventory

| Capability and required observable behavior | Class | Evidence at `b8b6a6d` | Coverage and gap |
| --- | --- | --- | --- |
| **Conversation and provider continuity.** Start a session, submit multi-turn prompts, stream responses, retain conversation history, switch backend/model, and resume a saved transcript. On process restart, the current configuration supplies backend, model, tools, and approval policy; these are not session-persisted native state. | Core engine | README lines 32-40, 230-252; `src/main.rs:662-991,784-807,854-876`; `src/session.rs:11-34,59-117`; `src/session_turn.rs:388-792` | Documented session behavior; unit tests cover save/load (`src/session.rs:327-344`) and a streamed user turn (`src/session_turn.rs:1054-1107`). No end-to-end resume test across a separate process. |
| **Tool-call agent loop.** A configured model can request an enabled tool, receive its result, and continue to a final response. A client must serialize mutation-capable operations. Native versus inline call formats, input-validation mechanics, step limits, and read-only scheduling are implementation-observed terminal behavior, not a cross-client provider-wire-format commitment. | Core engine | `src/agent.rs:432-804`; read-only classification `src/tools/mod.rs:186-203` | Mock-SSE agent integration tests exist in `src/agent_integration_test.rs`; tool-selection and concurrency classifiers have unit tests in `src/tools/mod.rs:308-430`. No full CLI test proves an actual tool round trip. |
| **Workspace tools and path boundary.** The terminal client retains documented tool names and schemas. Every client that exposes their outcomes must preserve approval and workspace-bound behavior; it need not expose the model-facing tool protocol or terminal command text. | Core engine | Public table: README lines 282-293; construction and policy wiring: `src/tools/mod.rs:205-300` | Documented public surface; individual tool tests are uneven. Built-in live evals exercise only read, edit, test, and refactor outcomes; they do not cover shell, patch, batch edit, ship status, subagent, critic, or outside-workspace policy. |
| **Approval and mutation preview.** A tool that requires approval presents its preview/risk when available; the user can allow once, deny, allow a tool for the session, or allow the native cache key for the session. Denial prevents execution. Native exact-call allowance is keyed only by tool plus `command` or `path`, so it is not an argument-exact safety guarantee. The default policy is `always`. | Core engine | README lines 295-304; `src/approval.rs:18-114`; `src/config.rs:697-726`; agent approval phase `src/agent.rs:481-789` | Documented approval behavior plus an implementation-observed cache-scope caveat. Unit coverage exists around approval/agent behavior, but not a black-box terminal approval transcript. Current approval cache is in-memory (`src/approval.rs:18-27`); persistence across restart is not a native commitment. |
| **Session artifacts and exports.** Persist transcript JSONL, metadata/title, session listing/search/resume/export, and event-log sidecar; the event log redacts recognized secret keys/patterns and supports export/summarization. | Core engine | README lines 310-320, 1036-1045; `src/session.rs:59-280`; `src/turn_trace.rs:218-315,351-414` | Session and trace round-trip/redaction tests exist (`src/session.rs:327-344`, `src/turn_trace.rs:457-620`). No corruption/restart recovery acceptance test. |
| **Undo checkpoints.** For recognized mutation tools, capture pre-turn file baselines; `/undo` restores last-turn files including created files, subject to documented limits and partial snapshots. | Core terminal | README lines 54-56, 200-211; `src/turn_checkpoint.rs:138-180,291-326`; `src/session_turn.rs:668-852` | Strong file-level unit coverage, including created files and two-turn undo (`src/turn_checkpoint.rs:429-688`). It is in-memory stack behavior, not durable restart recovery. |
| **Parallel session paths.** Fork/switch/diff/pick/drop alternate session/workspace paths and restore the selected path when resuming. | Core terminal | README lines 57-59, 319-320; `src/commands/session.rs`; initialization/resume wiring `src/main.rs:854-905` | Command-level tests exist, but no multi-process user workflow test. Browser implementation may defer path UI, but must not claim equivalent branching until it preserves workspace and transcript selection semantics. |
| **Context management.** Show context state, compact earlier turns, and reset into a fresh session seeded from a continuation artifact; do not silently discard the user's usable continuation. | Core engine | README lines 574-583; `src/commands/context_cmds.rs:31-315` | Reset behavior has local backend tests (`src/commands/context_cmds.rs:409-487`). Compaction and reset effects have no CLI acceptance suite. |
| **Project memory.** Build a metadata-only repo map, honor exclusions, inject relevant memory, and allow durable project notes to be saved/removed. | Core engine | README lines 812-819; commands registered in `src/commands/mod.rs:226-231` | Existing tests are primarily module/unit coverage. No eval asserts that injected memory improves a task or excludes secret content. |
| **Operator presets and normal coding workflows.** `explore`, `edit`, `ship`, and `review` alter tool/policy/verification defaults; planning, test, fix, iterate, auto, refactor/batch, and isolated play flows produce their documented workspace artifacts/outcomes. `/auto` requires an explicit terminal invocation and is never an implicit client default. | Core terminal | README lines 323-353, 514-623; preset wiring `src/config.rs:792-860`; workflow entry points `src/commands/workflow.rs`, `src/fix_loop.rs`, `src/iterate_loop.rs`, `src/auto_loop.rs`, `src/planner.rs` | Parsers and local flow tests are substantial, particularly `/auto` guards (`src/auto_loop.rs:936-1220`). No deterministic full workflow suite covers all modes. |
| **Planning and routed execution.** Create a spec with fallback sections on model failure; create/store a routed task graph; execute ready tasks sequentially with configured models; save status after each task. | Core terminal | README lines 516-550; `src/planner.rs:34-131,255-555`; `src/commands/mod.rs:618-625,1068-1185` | Parser/fallback tests exist (`src/commands/mod.rs:1612-1756`). No live eval checks routing, dependencies, model restoration, or partial failure. |
| **Ship readiness, guarded git/PR operations, and scorecard.** Inspect local readiness, optionally run tests, make guarded commit/push/PR operations, retain ship records, score close-time evidence, and distinguish numeric score from a PR that counts as quality-shipped. | Core terminal | README lines 212-220, 465-508; `src/shipcheck.rs:206-652`; `src/commands/ship.rs`; `src/scorecard.rs:319-465,1612-1705` | Extensive unit coverage, including manual closes with URL and tests counting (`src/scorecard.rs:2483-2507,2689-2705`). No real GitHub CLI acceptance test; remote verification relies on `gh`. |
| **Backend, model, credential, and capability management.** Support documented local/cloud backends, model selection, API key and Codex OAuth flows, and doctor probes/recommendations without changing session semantics merely because the backend changes. | Core terminal | README lines 230-274, 394-410; `src/backends.rs`; `src/commands/config_cmds.rs:118-303,502-1007`; `src/commands/doctor.rs` | Configuration/parser tests exist. Provider/OAuth flows need real-service or contract-server acceptance coverage; never treat stored credential-file layout as a cross-client API. |
| **Hooks.** Trusted project hooks and launcher-managed hooks run for documented events, receive payloads, can warn/block/stop/allow or supply bounded context, and project config cannot grant its own trust. Gating failures fail closed. | Optional configured integration | README lines 652-766; `src/hooks/config.rs:10-251`; `src/hooks/runtime.rs:39-397`; `src/hooks/registry.rs:54-101` | Documented integration; strong unit and local-process coverage (`src/hooks/mod.rs:42-1477`), including trust, environment, timeout, rewrite, and fail-closed paths. No user-facing cross-client event-contract suite. |
| **MCP.** Configured stdio JSON-RPC servers are launched, failures are reported, and discovered tools are surfaced to the model as `mcp__<server>__<tool>`. Unknown MCP tools are not assumed read-only. | Optional configured integration | README lines 67-68, 632-650; `src/mcp.rs:19-24,272-284`; startup `src/main.rs:839-852` | Documented integration; naming/config unit tests exist (`src/mcp.rs:371-382`). No live fake-server integration test for discovery, invocation, failure, or cancellation. |
| **Images and web fetch.** An image attachment is sent as a next-turn multipart user message; optional `web_fetch` fetches a URL, strips HTML to text, and is approval-gated. | Optional configured integration | README lines 797-810; `/image` command handler `src/commands/config_cmds.rs:348-401`; optional tool construction `src/tools/mod.rs:272-274` | Documented surface with no built-in eval. Treat remote fetching as externally observable and policy-sensitive. |
| **Subagent orchestration.** `task` runs a nested, curated read-only subagent. When the event log is enabled, event records retain nested activity; trace-panel visibility is separately configurable. | Core engine | Tool construction/comments `src/tools/mod.rs:257-289`; trace controls README lines 363-366, 1038-1045 | Partly implementation-observed; unit tests cover classifiers and traces. No eval verifies nested agent result quality, context isolation, cancellation, or trace ordering. |
| **Critic orchestration.** `critique` runs a separate critic when enabled in the configured tool pool. | Optional configured integration | Tool construction/comments `src/tools/mod.rs:257-289` | Implementation-observed; no eval verifies critic result quality, context isolation, or cancellation. |
| **Fusion, compare, route selection, and Fable usage.** Preserve only when their provider/configuration prerequisites are present: OpenRouter comparison/Fusion, configured multi-model selection, and local Fable ledger reporting. | Optional configured integration | README lines 821-876; `src/commands/config_cmds.rs:713-1007`; `src/commands/route.rs`; `src/fable_usage.rs` | Documented advanced surface; parser and ledger tests exist, but no live provider acceptance tests. They must remain unavailable with a clear prerequisite error rather than becoming implicit defaults. |
| **Interactive terminal ergonomics.** TUI renderer, colors/ASCII/NO_COLOR, reasoning/verbose/trace panels, input history, Ctrl-J newline, clear screen, banner/warmup/status footer, shell completions, setup wizard, and terminal confirmation prompts. | Client-specific terminal-only | README lines 1032-1065; `src/main.rs:198-296,662-991`; `src/renderer.rs`; `src/input.rs` | Mostly unit/snapshot-like coverage. Exact formatting is not a TinyHarness browser contract. |

### Native command surface

The terminal client must retain the documented command families at the baseline: session (`/new`, `/session`, `/sessions`, `/resume`, `/export`, `/undo`, `/path`, `/paths`); workflow (`/mode`, `/plan`, `/shipcheck`, `/ship`, `/scorecard`, `/fable`, `/handoff`, `/test`, `/fix`, `/iterate`, `/auto`, `/batch`, `/refactor`, `/play`); backend/configuration (`/backend`, `/model`, `/tools`, `/auth`, `/login`, `/logout`, `/image`, `/reasoning`, `/verbose`, `/trace`, `/hooks`, `/compare`, `/fusion`, `/route`); and memory/context/diagnostics (`/index`, `/map`, `/memory`, `/remember`, `/forget`, `/context`, `/compact`, `/reset`, `/doctor`, `/checkpoints`). README lines 306-387 are the authoritative public listing; command registration is `b8b6a6d:src/commands/mod.rs:113-259`.

Aliases, redirected legacy commands, help ordering, and exact prose are terminal UX details unless a scriptable CLI contract explicitly depends on them.

### Documentation boundary

The following are public compatibility commitments because README documentation describes their outcomes: terminal commands and tool names, approval modes, sessions/exports, checkpoints, paths, hooks, MCP, images, web fetch, configuration/backends, and integrations. The inventory marks implementation-observed behavior separately where the source does not establish a public contract: provider call formats, input-validation mechanics, step limits, read-only concurrency and auto-selection heuristics; approval-cache key granularity; current JSONL/checkpoint durability limits; trace enablement; and subagent trace ordering. TinyHarness adds one safety commitment based on the native serial loop: mutation-capable operations are serialized. It must not represent other implementation-observed behavior as a stable promise until it gains an explicit product decision and regression evidence.

## Current evaluation baseline and gaps

The native project has four built-in live-agent fixtures:

| Fixture | Asserted outcome | Evidence |
| --- | --- | --- |
| `read-and-explain` | `file_read` is called and the response mentions `add`. | `src/agent_eval.rs:81-96` |
| `fix-failing-test` | The fixture's tests pass after the agent run. | `src/agent_eval.rs:97-103` |
| `small-refactor` | A renamed symbol appears and tests pass. | `src/agent_eval.rs:104-118` |
| `add-feature` | A new `mul` function appears and tests pass. | `src/agent_eval.rs:119-132` |

These fixtures create real temporary workspaces and invoke the agent, but use unconditional approval (`src/agent_eval.rs:67-79`) and do not validate session resume, approvals, hooks, MCP, traces, checkpoints, provider switching, or destructive failure paths. Native CI builds/tests on Ubuntu and macOS, while its live eval job is optional, `continue-on-error`, and invokes only `read-and-explain` and `fix-failing-test` (`.github/workflows/ci.yml:10-51`). It is useful signal, not a compatibility gate.

The following behavior is materially user-visible but underdocumented or unevaluated and must receive explicit TinyHarness coverage before being claimed compatible:

- Read-only calls may run concurrently, but mutations, `shell`, `run_tests`, and unknown/MCP calls stay serial (`src/agent.rs:481-489`; `src/tools/mod.rs:186-203`).
- Auto tool selection withholds tools only for a narrow small-talk set; all real prompts receive the configured pool (`src/tools/mod.rs:88-150`).
- The default enabled tool pool is deliberately small; web fetch, test runner, patch/batch, repo search, critic, and other tools are opt-in (`src/config.rs:711-726`).
- Checkpoints are limited, in-memory file baselines. They are not rollback for shell, hooks, network/MCP effects, or process restart (`src/turn_checkpoint.rs:88-120,138-180`).
- Transcripts and traces are append-only JSONL. A successful append is not a crash-recovery guarantee (`src/session.rs:59-68`; `src/turn_trace.rs:294-310`).
- Hook payload is intentionally raw for trusted hooks, while visible trace/context values are redacted and bounded; a new client must not expose raw hook payload to untrusted UI code (README lines 706-754; `src/turn_trace.rs:218-248`).
- A score and a counting quality ship are distinct. Manual `/scorecard close --url --tests` can count when the evidence warrants it (`src/scorecard.rs:1612-1705,2689-2705`).

## Compact TinyHarness regression baseline

Run these in risk order. Prefer black-box CLI/client tests using real temporary repositories and local contract servers. Do not substitute mocked internal collaborators for the observable behavior.

1. **Core turn and safety gate:** against a local OpenAI-compatible streaming server and real temporary repository, prove text, a tool call, its result, and final response. Request a file edit, patch, and shell command; assert approval under `always`, denial prevents the effect, allow-once runs one effect, and mutations stay serial. Terminal compatibility tests additionally cover native/inline forms and step-limit termination.
2. **Core session and recovery gate:** submit multiple turns, launch a new process, resume, and verify transcript/history, title, and exported markdown/JSON. Include malformed final JSONL and assert prior history is readable or the failure is precise.
3. **Core mutation artifact gate:** mutate existing and newly created files, invoke undo, and verify exact restoration. Prove shell and external effects are not advertised as undoable.
4. **Representative task-eval gate:** retain one read/understand fixture, one repair fixture, and one small-change fixture against a deterministic local model or contract server; report model variability separately from contract failure.

These four are the release gates for a client exposing core turns and workspace mutation. Optional integrations receive a narrow contract test only before that integration is exposed: hooks (trust, block/rewrite, failure-closed, redaction), MCP (discovery/invocation/failure and serial classification), web/image (local HTTP or provider contract), and critic orchestration (result, isolation, cancellation). Core-terminal specialized workflows, including planning/routed execution and ship/scorecard, remain release gates for the terminal distribution and for any client that claims an equivalent workflow; their tests validate the documented persisted artifacts and prerequisites.

## Client rollout rules

- A new client may ship read-only conversation viewing before workspace mutation, but it must label itself read-only and must not expose an approval bypass.
- Before a new client enables workspace mutation, it must pass the four core gates and use equivalent approval, workspace-boundary, tool-result, and transcript/trace-redaction outcomes.
- Before it enables an optional integration, it must pass that integration's contract test. Issue #8 decides trust boundaries for external effects; this note does not prescribe the enforcement mechanism.
- Client UX may use controls and panels instead of slash commands. It must provide equivalent visibility for approvals, tool state, blocked hooks, errors, and saved-session identity where those capabilities are exposed.
- Do not claim parity for terminal-only features: shell completion, terminal keybindings, ANSI display toggles, local setup wizard, or exact terminal streaming layout.

## Upstream attribution and selective-sync record

On the first compatibility-relevant upstream sync, create `docs/upstream-sync/README.md` and one record at `docs/upstream-sync/<upstream-sha>-<capability>.md`; later records are linked from that index and from the TinyHarness PR or commit. This prevents an accidental upgrade from silently changing TinyHarness's contract.

```markdown
## Upstream sync: <short capability name>

- Upstream repository: GetSmallAI/SmallHarness
- Upstream commit(s): <full SHA(s)>
- Public surface affected: <commands, tools, config keys, artifacts>
- Compatibility class: <core engine | core terminal | terminal-only | optional configured integration | experimental>
- TinyHarness disposition: <adopt | adapt | defer | reject>
- Behavioral contract retained: <observable outcomes>
- Intentional differences: <none or explicit list>
- Evidence added/updated: <test or eval names>
- Security/safety review: <approval, secrets, local process, network effects>
- Migration/user communication: <needed or not needed>
- Owner and review date: <name/date>
```

Selective sync rule: adopt a change only after recording the upstream commit, classifying the changed public surface, and adding an outcome-level regression test. A terminal rendering-only change may be noted without a TinyHarness engine change. A tool, approval, session, hook, MCP, trace, or artifact change requires an explicit disposition.

## Consequences for follow-on work

Issue #4 may assume a stable client-neutral seam around turns, tool calls, approvals, events, session identity, and persisted artifacts. It must not assume terminal strings, renderer types, or input-loop ownership are part of that seam.

Issue #6 must specify worktree identity, mutation evidence, undo, and cross-client completion semantics. The present baseline only requires the native visible outcome; it does not make current in-memory checkpoints durable.

Issue #3 remains the authority for durable-turn recovery. This note does not reinterpret append-only JSONL as crash-safe transaction state.

Future upstream synchronization uses this document's record format. Issue #7 may rely on the configuration/model compatibility classifications here; issue #8 must classify every new external side-effect boundary before exposing it through another client.

## Acceptance criteria for this research issue

1. The compatibility inventory distinguishes core engine, core terminal, terminal-only, and experimental/opt-in behavior with source evidence.
2. The inventory identifies existing tests/evals and meaningful gaps rather than treating registration or README presence as coverage.
3. The minimum regression suite orders safety and persistence before convenience workflows.
4. A new client has clear rollout conditions and may not claim full native parity by reproducing terminal presentation alone.
5. Future upstream changes have an attributable, reviewable selective-sync record.

## Sources

- [SmallHarness README at `b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/README.md)
- [SmallHarness CHANGELOG at `b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/CHANGELOG.md)
- [Command registry at `b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/src/commands/mod.rs)
- [Tool registry at `b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/src/tools/mod.rs)
- [Agent eval fixtures at `b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/src/agent_eval.rs)
- [CI workflow at `b8b6a6d`](https://github.com/GetSmallAI/SmallHarness/blob/b8b6a6d/.github/workflows/ci.yml)
