# Product Requirements Document

## Status

Draft for review.

## Product Identity

A remote coding agent with a terminal interface and a Telegram mobile
companion.

The agent runs on a private server. The server owns all workspaces, sessions,
and active work. Terminal and Telegram provide different views of the same
server sessions.

The product is for personal use. It favors speed, low resource use, simple
setup, clear limits, and user control instead of a broad feature set.

## Problem

Current coding agents often assume that the user works from one computer.
Terminal access from an iPhone is difficult, but a full browser application can
add unwanted setup and resource use.

Existing agent platforms can also become slow or expensive on a low-cost
virtual private server. Startup work, background services, and stored session
data can grow without clear limits. This makes long-term operation difficult to
trust.

The user needs one coding agent that:

- Works fully from a terminal at home.
- Remains available through Telegram when the user is away.
- Continues work when an interface disconnects.
- Runs well on a low-cost private server.
- Keeps detailed evidence of agent behavior.
- Applies clear limits to memory, storage, output, and background work.
- Supports optional features without adding them to the core product.

## Users And Context

### Primary User

The primary user is one software developer who operates the agent on a private
server.

The user works in two main contexts:

- At home, the user works through a terminal on a computer.
- Away from the computer, the user works through Telegram on an iPhone.

The first release does not serve teams, public bots, or large organizations.

### Operating Context

The agent uses remote model providers through application programming
interfaces or an approved subscription login.

The target host is a low-cost virtual private server. The product must remain
useful under limited memory and storage conditions.

## Desired Outcome

The user can start coding work from either interface and continue the same
server session from the other interface.

The terminal provides the complete working view. Telegram provides a concise
mobile view for prompts, important updates, questions, approvals, corrections,
and results.

The server remains responsive over long periods. Stored data and inactive
features do not cause unbounded resource growth.

The product follows these principles:

- **Fast:** Connecting to the agent does not start unnecessary work.
- **Bounded:** Every stored or queued resource has a default limit.
- **Persistent:** Work continues when an interface disconnects.
- **Reachable:** The user can work from a computer or iPhone.
- **Inspectable:** The user can review what the agent did and why.
- **Modular:** Optional features remain outside the core.
- **Private:** Access is restricted to the owner by default.

## Product Model

### Server Sessions

The server owns each session. A session is not a terminal session or a
Telegram session.

Each server session has:

- One stable identity.
- One workspace.
- One ordered conversation.
- One current status.
- One append-only raw trace.
- One usage and cost record.

Terminal and Telegram can connect to the same server session at the same time.
The server orders all input and events.

Only one agent turn runs in a session at a time. New user input must have a
clear result. It can correct active work, wait as a follow-up, or stop the
active turn.

### Telegram Attachment

Telegram does not receive session updates by default.

The user must explicitly attach a Telegram bot conversation to a server
session. Attachment creates a mobile view of that session. It does not move,
copy, or replace the session.

After attachment:

- Telegram messages can add input to the server session.
- Telegram receives important updates from that session.
- Terminal clients can continue to view and use the same session.
- All input remains in one ordered conversation.
- The raw trace records the source interface for each input.

The user can detach Telegram at any time. After detachment, Telegram stops
receiving updates from that session. Detachment does not stop or delete the
server session.

## User Flows

### Work From Home

1. The user opens the terminal interface.
2. The user selects a workspace.
3. The user starts or resumes a server session.
4. The user sends a coding task.
5. The terminal shows detailed progress, tool activity, changes, and results.
6. The user reviews and approves sensitive actions when required.

### Continue From An iPhone

1. The user opens the private Telegram bot.
2. The user lists available server sessions.
3. The user attaches Telegram to one server session.
4. Telegram shows a concise current-state summary.
5. The user sends prompts, corrections, approvals, or stop requests.
6. Telegram shows important progress and the final result.

The terminal does not need to close before this flow starts.

### Return To The Terminal

1. The user opens the terminal interface.
2. The user resumes the same server session.
3. The terminal shows all Telegram input in the correct order.
4. The user continues with the complete working view.

Telegram remains attached until the user detaches it.

### Detach Telegram

1. The user requests detachment from Telegram or the terminal.
2. The product confirms which server session is detached.
3. Telegram stops receiving session updates.
4. The server session continues without interruption.

### Recover After A Disconnect

1. An interface disconnects during active work.
2. The server continues the accepted work when safe.
3. The user reconnects through terminal or attached Telegram.
4. The interface shows the current session state.
5. The user does not need to restart the task.

## Interface Requirements

### Terminal

The terminal is the complete interface. It must support:

- Workspace selection.
- Session creation, listing, resume, and closure.
- Prompt and follow-up input.
- Streamed agent responses.
- Detailed tool activity.
- Command output.
- Complete change review.
- Approval and rejection.
- Correction and cancellation.
- Token, cost, and cache-use reporting.
- Raw-trace inspection and export.
- Storage and retention controls.
- Telegram attachment and detachment.

### Telegram

Telegram is a concise mobile companion. It must support:

- Secure owner access.
- Server health and connection status.
- Session listing.
- Explicit session attachment and detachment.
- Prompt and follow-up input for an attached session.
- Correction and cancellation.
- Approval and rejection.
- Questions that require user input.
- Concise progress and result summaries.
- On-demand session status.
- Delivery of large details as files when necessary.

Telegram must avoid routine tool noise. It must give priority to:

- Approval requests.
- Questions for the user.
- Blocked work.
- Failures.
- Completion results.

## Core Product Requirements

The core product must provide:

- A long-running remote agent.
- Multiple workspaces.
- Durable server sessions.
- Terminal and Telegram access.
- Explicit Telegram attachment.
- Provider streaming.
- Core coding tools for reading, searching, editing, and running commands.
- User approval for sensitive actions.
- Reliable cancellation and process cleanup.
- Workspace access boundaries.
- Token, cost, and provider-cache reporting when available.
- Detailed append-only raw traces.
- Default storage, memory, output, and work limits.
- A supported way to add optional capabilities.

The product must support OpenAI-compatible providers, Anthropic-compatible
providers, and OpenAI subscription login. The first-release provider order
remains an open decision.

## Raw Traces

Raw traces are a core product feature. They provide evidence for debugging,
evaluation, and agent improvement.

A raw trace must record, when available:

- Session and turn life-cycle events.
- User input and its source interface.
- Agent responses.
- Model context and model output.
- Provider and model identity.
- Tool requests, arguments, results, and errors.
- File changes.
- Approval requests and decisions.
- Corrections, cancellations, and retries.
- Token and cost data.
- Provider-cache reads and writes.
- Model and tool timing.
- Disconnect, reconnect, and recovery events.

Events must have a stable order and timestamp. New events append to the trace.
The product must not change earlier trace events.

Credentials and authentication tokens must not enter traces.

Raw traces are evidence. The agent must not use old traces as memory unless the
user enables a separate optional capability.

### Trace Limits

Trace storage must be bounded by default. The product must provide:

- A maximum trace age.
- A maximum total trace size.
- A maximum size for one event.
- Clear truncation records for oversized content.
- Automatic removal of the oldest eligible closed sessions.
- Protection for sessions that the user wants to keep.
- Trace export before manual removal.
- Current storage use and limit visibility.
- Clear notice when content is truncated or removed.

Active work must not cause unlimited storage growth. Exact default values need
measurement on the target server before release.

## Modularity Requirements

Skills, memory systems, decision notes, subagents, and workflow systems are not
part of the core product.

The core must let the user add these features as optional capabilities. When an
optional capability is absent or disabled, it must:

- Add no user-interface complexity.
- Start no background work.
- Use no persistent memory beyond its stored configuration.
- Have no effect on normal agent behavior.

The product must not download or run optional capabilities without an explicit
user action.

## Security And Trust

The first release assumes one owner.

The product must:

- Restrict Telegram access to the owner by default.
- Keep provider credentials out of prompts, traces, and tool output.
- Ask for approval before sensitive actions.
- Limit agent access to configured workspaces.
- Make Telegram attachment visible in the terminal and Telegram.
- Show which server session receives each Telegram message.
- Fail safely when approval expires or a mobile connection fails.
- Avoid automatic installation of executable optional features.

The default installation must not require Node.js or the npm package
ecosystem.

## Resource Requirements

The product must operate reliably on a low-cost virtual private server.

Every queue, cache, session store, trace store, attachment store, background
process, and tool output must have a limit or retention rule.

The product must:

- Avoid expensive work when a client connects.
- Load inactive features and workspaces only when required.
- Release inactive in-memory state.
- Limit concurrent agent work.
- Limit model and tool output retained in memory.
- Report current storage use.
- Continue to start and connect promptly as stored history grows within limits.

Numeric targets for startup time, idle memory, active memory, and storage need
measurement on the reference server before release.

## Success Measures

The MVP succeeds when:

- The user completes a real coding task through the terminal.
- The user attaches Telegram to that server session and continues from an
  iPhone.
- Terminal and Telegram show one ordered conversation.
- Telegram receives no session updates before explicit attachment.
- Detachment stops Telegram updates without stopping the server session.
- Work survives an interface disconnect and reconnect.
- The user can approve, correct, and stop work from Telegram.
- Telegram remains concise during a tool-heavy task.
- The user can inspect and export the complete available raw trace.
- Trace limits prevent unbounded storage growth.
- Disabled optional capabilities consume no ongoing resources.
- The product operates within the measured limits of the reference server.

Before release, the project must define measurable targets for:

- Time from terminal launch to ready input.
- Idle memory use.
- Memory use during one active session.
- Maximum default trace storage.
- Default trace retention age.
- Recovery behavior after a service restart.

## Constraints

- Personal use comes before team features.
- Performance and low memory use have the highest priority.
- Portability has the second priority.
- Setup must require little user work.
- The product uses remote model providers first.
- Terminal and Telegram are the only MVP interfaces.
- The default installation avoids Node.js and npm.
- Optional features must not make the core larger or less reliable when they
  are disabled.
- Stored data must remain inspectable and removable by the user.

## Risks And Assumptions

### Telegram Limits

Telegram has a small display area and platform message limits. Concise summaries
and file delivery can reduce this risk.

### Remote Agent Access

A Telegram bot that can control coding tools creates a security risk. Private
owner access, approvals, workspace limits, and clear attachment state reduce
this risk.

### Trace Value And Size

Detailed traces can grow quickly and can contain sensitive project data.
Default limits, credential removal, protection, and export controls reduce this
risk.

### Optional Capability Growth

An extension system can become a second platform. A small capability boundary
and explicit user installation reduce this risk.

### Provider Differences

Providers report streaming, token use, cost, and cache data differently. The
product can only show data that the provider makes available.

### Low-Cost Server Targets

The product assumes that a low-cost server can support one owner and a small
number of active sessions. Measurement must confirm this assumption.

## Out Of Scope

The MVP does not include:

- A browser interface.
- A desktop application.
- A general personal assistant.
- Multiple users or team collaboration.
- Public Telegram bot access.
- Telegram group use.
- Other messaging platforms.
- Built-in skills.
- Built-in long-term memory.
- Built-in decision management.
- Built-in subagents or agent graphs.
- Autonomous self-improvement.
- A capability marketplace.
- Machine fleet management.
- Scheduled autonomous work.
- A broad local-model management system.
- Unlimited session or trace retention.

## Open Decisions

- Select the first provider and authentication method for the MVP.
- Decide whether the MVP includes one provider or all required provider types.
- Define how Telegram presents and selects server sessions.
- Decide whether one Telegram conversation can attach to more than one server
  session through Telegram topics.
- Define the default behavior for new input while an agent turn is active.
- Define the default trace age and storage limits.
- Define the reference server and measurable performance targets.
- Decide which optional capability interface is necessary in the MVP.
- Decide whether Model Context Protocol support is part of the MVP.
- Select the product name.
