# Session host: specification

Status: **DRAFT v0**, 2026-09-24. Not normative. Track: [`track.md`](track.md). Diagrams: [`../diagrams/`](../diagrams/README.md).

A long-lived orchestrator needs something to keep it running when nobody is looking. That program starts a harness in the right directory with the right goal. It lets a person attach and watch, delivers messages into the session, and says what happened. Today that program is `tb-sessiond`, the Session Manager inside task-board. It works, but it is fused into task-board, holds Claude sessions in its own address space, and only serves one board's primary session and that board's tracked runs. This specification describes the host as its own module: first extracted as it is, then extended for the main loop that starts and supervises the root orchestrator of every process on a node.

Three markers say how settled a statement is:

- *decided (2026-09-24)*: the operator confirmed it;
- *proposed*: the author's resolution, open to review;
- *fact*: a description of the current code, with its source.

Sources of facts: relux-works/skill-project-management at `origin/main` `9f594dba` (paths below are relative to that repository unless stated), relux-works/agent-session-manager-spec at `32b3f2b` (`ax` v0.7.0), relux-works/curator-spec Decisions 0013, 0019 and 0021 at `origin/main` `3d2c611`.

Related: the process-configuration specification (relux-works/curator-playbook: one root orchestrator per process, the board goal), the main-loop specification (planned repository relux-works/curator-keeper), waggle (relux-works/waggle: the doorbell and the carrier of messages between orchestrators), Agent Session Manager (`ax`, relux-works/agent-session-manager).

---

## 0. The design on one screen

- **What it is.** The session host keeps agent sessions alive in the background, starts a harness in a given directory with a given goal, lets a person attach and detach, delivers messages into a session (the doorbell), and reports events. *fact for today; the module is decided*
- **Phase 1: extract it as it is.** The provider-host plane of `tb-sessiond` moves into this repository behind a contract, with no change in behaviour. task-board keeps the goal ledger, routing and the spawn runtime and reaches the host through a thin seam. *decided (2026-09-24), priority*
- **Later: pluggable terminal backends.** `pty` (today), `tmux`, and `ax`. They are recorded here, not built in phase 1. *decided to record (2026-09-24); the design is proposed*
- **The main loop's needs.** Start a root session in any registered process directory, exactly one per process. Attach in control or read-only mode, including to sub-orchestrators. Ring the doorbell of a detached session. Accept wake, park and stop from the main loop. Emit events. *proposed*
- **Park.** `task-board session park` stops a session after it has written a park note, and binds the note into the board goal so the next start resumes from it. An emergency form, `park --now`, saves the most in the least time. *park decided (2026-09-24); the emergency form proposed*
- **Restart survival.** Today a Claude session dies with the daemon. The `tmux` and `ax` backends keep terminals alive across a host restart. *fact; the target proposed*

## 1. Terms

| Term | Meaning |
| --- | --- |
| host | the process that owns hosted sessions on a node; today `tb-sessiond` |
| hosted session | a harness process (Claude Code, Codex, later others) whose lifetime, terminal and goal the host owns |
| root session | the one hosted session that holds a board's `parent/primary` goal slot: the process's root orchestrator. task-board calls it the primary session |
| owner-run session | a hosted session that holds a `run/RUN-*` goal slot: a sub-orchestrator or another goal-bound run spawned by a root session |
| client, attachment | a terminal connected to a hosted session; a session may have none |
| doorbell | a message the host types or submits into a running session so that it takes a turn; task-board calls these in-session notices |
| provider host | the per-harness adapter that launches, restores, drives and stops one harness (`ProviderHost`) |
| terminal backend | what holds a session's terminal: the host's own PTY today, a tmux server or `ax` later (§4) |
| main loop | the node's always-on loop that routes a person's requests to processes and starts and supervises their root sessions (main-loop specification) |

## 2. Today's implementation *(fact)*

### 2.1 One control plane per board

`tb-sessiond` is keyed by board: a `BoardKey` "identifies one Session Manager control plane" (`tools/board-cli/internal/sessionmanager/key.go`). Its state lives in the user state directory, never under the board tree (`layout.go`, `DefaultLayout`); the per-session notice queues are in its `NoticesDir` (`layout.go:44-48`). Startup reconciles durable records, restores running sessions, sweeps stale sockets and reports `startup_healthy` (`docs/codex-goal-runtime.md`, "Self-restoring startup").

### 2.2 Root (primary) sessions

- `task-board claude` and `task-board codex` start or attach the board's primary session. The wrapper validates the launch plan and runs primary-session preparation, then calls `ConnectOrStart` (`docs/claude-goal-runtime.md:32-39`).
- The `parent/primary` slot is singular and provider-independent: "one board has one authoritative primary goal owner". A second provider is refused with `session_provider_conflict`, or, on a TTY, offered a takeover (`docs/claude-goal-runtime.md:9-29`; `docs/codex-goal-runtime.md:10-31`).
- A graceful detach prints `anthropic-board resume SES-...` or `openai-board resume SES-...`. The manager reattaches a healthy host, or relaunches a dead one with the durable native identity (`--resume` UUID for Claude, `thread/resume` for Codex) and reconciles the current primary goal before exposing the terminal (`docs/claude-goal-runtime.md:290-303`; `docs/codex-goal-runtime.md:233-247`).
- `task-board session stop SESSION-ID` stops one session "without clearing its board goal" (`tools/board-cli/cmd/session.go:94-95`). The other operator commands are `status`, `list`, `logs [-f]`, `doctor` and `restart` (`cmd/session.go:35-274`).

### 2.3 Owner-run sessions

A goal-bound spawn runs in its own hosted session. The manager "starts or resumes one PTY for that run owner", strips the parent's Session Manager and app-server credentials and provider session markers, injects the native goal and waits for acknowledgement of the exact revision (`docs/claude-goal-runtime.md:41-56`). Sub-orchestrators spawned with `--role orchestrator` live here. A person cannot attach to them. They are observable only through `task-board spawn transcript`, `observe` and `events`.

### 2.4 Goals and continuation

- The host delivers the board goal as the harness's native goal. For Claude that is `/goal tbg-<token> [board goal <ID> rev <N> - reference only, not the condition - objective "<excerpt>"] <provider-condition>`, submitted through bracketed paste (`docs/claude-goal-runtime.md:57-66, 180-183`). Codex receives the same condition through its app-server.
- "The provider-native evaluator owns continuation". This is why a restarted or resumed session goes on working without being told to: the host re-applies the current goal on resume, and the harness's own evaluator keeps it going until the condition holds (`docs/claude-goal-runtime.md:180-183, 298-303`).
- The goal itself is task-board data: `BoardGoalRevision` in `pkg/board/board_goal_v2.go:173` carries the objective, the provider condition, the reason and the lineage. It has no field for a handoff note today (§6).

### 2.5 The doorbell today: in-session notices

- **Who can be rung.** Only a hosted primary session is addressable (`TASK_BOARD_MANAGED_SESSION_ID`). Owner-run sessions are not, and the marker is stripped from their children (`docs/claude-goal-runtime.md:392-403`).
- **Producer.** The tracked-spawn terminal hook (`DeliverTerminal`) writes one durable notice per terminal run into the session's inbox.
- **Consumer.** A pump in the daemon delivers only to a session that is running, whose host implements `ProviderSessionNotifier`, and that "has at least one attached client" (`internal/sessionmanager/manager_notice.go:76-77, 90-109`). A detached session keeps its notices queued; on the next attach they are replayed, coalesced into one message.
- **Delivery.** Claude receives the text through bracketed paste into its PTY, serialized with goal injection (`docs/claude-goal-runtime.md:424-447`; `internal/sessionmanager/claude_host.go:1090-1111`). Codex receives it through its app-server.

![Doorbell delivery](../diagrams/artefacts/plantuml/doorbell-delivery.svg)

### 2.6 Provider hosts

The seam the module is extracted along already exists as Go interfaces in `internal/sessionmanager/host.go:32-165`:

- `ProviderHost`: `State`, `Snapshot`, `ApplyBoardGoal`, `RollbackBoardGoal`, `Stop`;
- `ProviderHostFactory`: `Provider`, `Launch`, `Restore`;
- the optional capabilities: `ProviderAttachmentTarget`, `ProviderSessionNotifier`, `ProviderNativeIdentity`, `ProviderOwnerRunner`, `ProviderClientHome`, `ProviderClientToolPolicy` and `ProviderLaunchReporter`.

There are two implementations:

- **Claude.** The host starts the harness in a PTY it creates itself (`claude_host.go:727`, `pty.StartWithSize`).
- **Codex.** The host starts one private app-server per session and one thread per logical session (`codex_host.go:136-227`; `docs/codex-goal-runtime.md:1-8`).

### 2.7 Restart

"A Claude host's PTY master file descriptor is held in the `tb-sessiond` process itself". The daemon exiting closes that master, so "a Claude host does not survive a manager restart". A Codex host does, because its app-server is a separate process the replacement daemon reconnects to (`docs/claude-goal-runtime.md:305-314`).

Two guards follow from this (`internal/sessionmanager/hosted_takeover_guard.go`):

- A call from inside a hosted session that would replace the daemon is refused.
- From an outside shell, replacement is refused while live PTY-held sessions exist, unless the operator runs `task-board session restart --stop-hosted` (`docs/claude-goal-runtime.md:316-341`).

![Starting a hosted primary session today](../diagrams/artefacts/plantuml/session-start-today.svg)

## 3. Phase 1: extract the module as it is *(decided 2026-09-24, priority)*

The first step moves code, not behaviour. Improvements come after the module stands on its own.

### 3.1 What moves here

- The provider-host plane: the `ProviderHost` and `ProviderHostFactory` contract and its optional capabilities, the Claude PTY host, the Codex app-server host, attachments and terminal relay, the notice inbox and pump, session records and their store, liveness, restore and repair, and the operator log.
- The daemon skeleton: instance lock, control socket, self-restoring startup, health reporting. It is parameterized by a key, today task-board's `BoardKey`.

### 3.2 What stays in task-board

- The board and its goal ledger (`board-goal-v2`): what the goal is, who may write it, and when it counts as achieved.
- Routing, the spawn runtime, run manifests, acceptance and handoff evidence.
- The `task-board claude|codex`, `session` and `spawn` commands. They keep their names and behaviour and reach the module through a thin seam: task-board supplies the goal revision to apply and consumes the host's acknowledgements, events and notices.

### 3.3 The contract

- **In process first.** Phase 1 keeps the adapters as in-process Go implementations behind the existing interfaces, exported from this module. *proposed*
- **The out-of-process form follows the ax provider plugin protocol:** line-delimited JSON over stdin/stdout, one plugin process per operation, a manifest and a probe, and structured errors that fail closed (`ax` SPEC §7.1-§7.3). A plugin exits after each operation, so only stateless operations cross that boundary: probe, launch spelling, goal-command rendering, acknowledgement parsing. The long-lived terminal stays with the host or its terminal backend (§4). *proposed*
- **Operations:** start, resume, stop, status, attach target, apply a goal revision, clear it, acknowledge the exact revision, deliver a notice, and an event stream.
- **Adapters:** Claude (PTY) and Codex (app-server) first. Muse, agy, Pi and Qwen follow, each under its own story.

### 3.4 Where it sits in the launch plane

A person's sessions enter through `curator run`. With the `ax` integration configured, they are hosted "from birth by whatever implements the `ax start --launch-plan` contract", and "until an `ax` implementation ships, task-board's session daemon implements the contract through a bridge" (curator-spec Decision 0021 §2). A host "receives a composed plan and never composes" (0021 §2; Decision 0019). This module is where that bridge's hosting half lives. The bridge's shape, an `ax` shim or the daemon answering `ax start`, is Decision 0021 open question 1. Tracked non-interactive children stay with `task-board spawn` (0021 §6).

### 3.5 Done when

- task-board's session, goal and notice test suites pass unchanged against the extracted module.
- The goldens for both harnesses are byte-identical.
- The operator commands behave as before.
- No behaviour difference is reported in the change log. Any difference found is either removed or listed and accepted by the operator.

## 4. Terminal backends *(recorded for later; proposed)*

A terminal backend owns one terminal instance per hosted session: hosting the harness process, presenting its terminal, attaching and reconnecting, backend-local IPC, and observing the process. The terms follow `ax` §4.A, where a backend "owns exactly one host-local Terminal Instance" and never owns identity, ownership or goals.

| Backend | How it works | Survives a host restart | Attach | Platforms |
| --- | --- | --- | --- | --- |
| `pty` (today) | the host creates the PTY and holds its master | Codex yes, Claude no (§2.7) | through the host's relay | macOS, Linux |
| `tmux` | the host starts the harness inside a tmux server on a dedicated per-user socket (as `ax` does with `tmux -S <runtime>/tmux/ax.sock`, §3.2). Input goes in through `load-buffer` and `paste-buffer -p`, which keeps bracketed paste. Output comes out through control mode (`tmux -C`) or `pipe-pane` | yes: the tmux server holds the PTY, and a restarted host re-adopts the panes | `tmux attach` for control, `tmux attach -r` for read-only | macOS, Linux, WSL2 |
| `ax` | the host delegates the session to Agent Session Manager | yes | local, remote and web attach; multi-attach, where extra clients get no input unless authorized; attach never changes ownership (`ax` §1.2 item 5, §4.C-§4.D) | Unix through tmux; Windows through ConPTY (`ax` §1.3) |

- The backend is chosen per node. `pty` stays the default until another backend passes a conformance suite built from task-board's session tests.
- `ax` is the long-term backend. Its specification (v0.7.0) already defines the Unix terminal backend, attach and continuation. Its implementation runs through milestones M0 (contract foundation, done), M1 (single-host durability), M2 (multi-host preview), M3 (daily-driver tmux) and M4 (cloning, directory and platforms), per relux-works/agent-session-manager README. We do not wait for it: the contract of §3.3 is shaped so that switching to `ax` is configuration, not a rewrite.
- `ax` is explicitly "not a replacement for task-board or `tb-sessiond`" (`ax` §1.2). It provides the terminal and ownership plane; the goal plane stays with task-board and this module.

![Surviving a host restart](../diagrams/artefacts/plantuml/restart-survival.svg)

## 5. Requirements from the main loop *(proposed)*

The main loop starts and supervises the root orchestrator of every process on its node. The host must support it:

1. **R1. Start in a process directory.** Start a hosted root session in any process directory the main loop has registered. The session gets that directory as its working directory, the process's Curator profile and managed home, and the board's `parent/primary` goal. It runs in the background with no client attached. The launch is composed by `curator run` (Decision 0021) and handed to the host.
2. **R2. One root session per process.** Exactly one root session is started from outside for each process. Today's singular `parent/primary` slot already enforces this per board (§2.2). Sub-orchestrators and runs are spawned by the root session inside its own board; the main loop never starts them.
3. **R3. Attach.** A person can attach to root sessions and to owner-run sub-orchestrators, in control mode or read-only mode. Attaching to short-lived worker runs is optional; their transcripts suffice.
4. **R4. Doorbell for detached sessions.** Notices and main-loop messages reach a running session that has no client attached. This lifts the condition of `manager_notice.go:94`. Coalescing stays. A delivery policy chooses between the next turn and an interrupt, in line with waggle's delivery modes.
5. **R5. Supervision.** The main loop can wake a session (deliver a message and start a turn), park it (§6) and stop it. The host accepts these only from the main loop's authorized principal: from waggle milestone CM2, a signed `cmd` message. The operator can always stop a session locally.
6. **R6. Events.** The host emits session state changes, goal acknowledgements, turn starts and ends, limit and failure markers, and park completion as a stream that the main loop and waggle consume.
7. **R7. Many processes on one node.** Today each board has its own control plane (§2.1). Either the node runs one control plane per process, as today, with the main loop keeping the list, or one node-level host keeps per-board state. Open question 1.

## 6. Park

### 6.1 `task-board session park` *(decided 2026-09-24; the mechanics proposed)*

Park is a planned stop that leaves a clear entry point for the next start.

1. The host rings the session's doorbell with a park directive.
2. The orchestrator writes a park note as a resource of the goal's element. The note lists runs in flight, Change Requests awaiting review or integration, decisions waiting for a person, the next steps, and pointers to anything else needed.
3. task-board appends a goal revision with the same objective and condition and a new `handoff_ref` field naming the note. The field is an addition to `BoardGoalRevision` (§2.4). Nothing else about the goal changes.
4. The host stops the session through the resumable stop. The goal is not cleared (§2.2).
5. On the next start, whether `task-board claude|codex` or the main loop's wake, the native goal payload carries the reference, for example `[board goal <ID> rev <N+1> ... handoff "<note>"]`. The session reads the note first, and the native evaluator continues from there (§2.4).

If the orchestrator does not write the note within a bound, the kernel snapshot of §6.2 is used instead.

### 6.2 Emergency park: `park --now` *(proposed)*

For a stop that cannot wait, for example power about to fail:

1. The kernel writes a snapshot with no model involved: runs in flight and their states, Change Requests awaiting review or integration, the output of `task-board worktree obligations`, and any unrouted notices. The snapshot becomes the park note.
2. The orchestrator may add a short note within a fixed bound of 30 seconds.
3. Runs are stopped so that they can resume. Today a run is cancelled, not suspended, so a resumable suspension is open question 5.
4. The session stops, and the goal revision with `handoff_ref` is written as in §6.1.

### 6.3 Automatic park *(proposed)*

The host parks its sessions before it is replaced, in place of `restart --stop-hosted` (§2.7) while the backend is `pty`, and when a session is about to run out of context. Detecting the latter is open question 9.

![Park and resume](../diagrams/artefacts/plantuml/park-and-resume.svg)

## 7. Restart survival

- **Today** *(fact)*: a Claude session ends when `tb-sessiond` exits. It resumes later from its durable identity, `--resume` with the session UUID. A Codex session is re-adopted through `thread/resume` (§2.2, §2.7).
- **Target** *(proposed)*: with the `tmux` or `ax` backend the terminals outlive the host, and a restarted host re-adopts them, as today's startup does for Codex (§2.1). This answers Decision 0021 open question 2, upgrading without hanging up. With the `pty` backend, park before replacement (§6.3) is the fallback.

## 8. Open questions

1. **Control planes.** One control plane per board, as today, or one node-level host that keeps per-board state (R7).
2. **Bridge shape.** How the `ax start --launch-plan` bridge is built: an `ax` shim, or the daemon answering `ax start` (Decision 0021 open question 1), and how it retires when `ax` ships.
3. **Doorbell policy for detached sessions.** Whether a message starts a turn at once, waits for the next turn, or interrupts, and how this maps to waggle's delivery modes.
4. **Supervision authority.** Which principal may send wake, park and stop to which sessions, the main loop's certificate and the waggle `cmd` classes, and what the host does before CM2.
5. **Resumable suspension of runs.** Needed for emergency park.
6. **`handoff_ref`.** A field on the goal revision or text in the objective; who consumes it and when it is cleared.
7. **Backend selection.** How a node chooses its terminal backend, the conformance suite, and Windows.
8. **Attach rights.** Who may attach to which session, whether sub-orchestrators are read-only by default, how input is authorized, tmux socket permissions, and remote attach through the `ax` mesh.
9. **Context exhaustion.** How to detect it early enough to park automatically.
10. **Placement of the extracted code.** Which task-board packages move and which stay, and the repository's module path.

## 9. Decisions

| Date | Decision |
| --- | --- |
| 2026-09-24 | The session host becomes its own module in this private repository. Phase 1 extracts it as it is, with no behaviour change, and is a priority. |
| 2026-09-24 | Pluggable terminal backends (`pty`, `tmux`, `ax`) are recorded as the architecture; none is built in phase 1. |
| 2026-09-24 | `task-board session park` is adopted; an emergency park is to be considered. |
