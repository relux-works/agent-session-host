# Session host: specification

Status: **DRAFT v0.3**, 2026-09-27. Not normative. Track: [`track.md`](track.md). Diagrams: [`../diagrams/`](../diagrams/README.md). Command-line shape: relux-works/wiki `research/cli-shaping-review.md`, adopted 2026-09-27. Change logs: §17 (v0 → v0.1), §18 (v0.1 → v0.2) and §19 (v0.2 → v0.3).

A long-lived orchestrator needs something to keep it running when nobody is looking. That program:

- starts a harness in the right directory with the right goal;
- lets a person attach and watch;
- delivers messages into the session;
- reports what happened.

Today that program is `tb-sessiond`, the Session Manager inside task-board. It works, but three things limit it:

- it is fused into task-board;
- it holds Claude sessions in its own address space;
- it only serves one board's primary session and that board's tracked runs.

This specification describes the host as its own module. The module is first extracted as it is, then extended for the main loop that starts and supervises the root orchestrator of every process on a node.

Three markers say how settled a statement is:

- *decided (date)*: the operator confirmed it;
- *proposed*: the author's resolution, open to review;
- *fact*: a description of the current code, with its source.

Sources of facts:

- relux-works/skill-project-management at `origin/main` `9f594dba`. Paths below are relative to that repository unless stated.
- relux-works/agent-session-manager-spec at `32b3f2b` (`ax` v0.7.0).
- relux-works/curator-spec Decisions 0013, 0019 and 0021 at `origin/main` `3d2c611`.
- relux-works/waggle `spec/waggle.md` §10 at `origin/main`.

Related specifications:

- the process-configuration specification (relux-works/curator-playbook): one root orchestrator per process and the board goal;
- the trust specification (relux-works/curator-trust `spec/trust.md`): who counts as a person, presence-bound confirmations, and running agents under an operating-system account of their own;
- the main-loop specification (relux-works/curator-keeper);
- waggle (relux-works/waggle): the doorbell's delivery modes and the carrier of messages between orchestrators;
- Agent Session Manager (`ax`, relux-works/agent-session-manager).

## Names used in this specification *(decided 2026-09-25)*

People read about the system in the Apiary names. Beekeeping words appear only where people type them in the apiary: the `keeper` command and its nouns (`keeper hive …`), the apiary's directories and the profile name `apiary`. Specifications, code, the host's and the kernel's command lines, flags, configuration keys and board items use technical terms; the same rule stands in the main-loop specification (Names) and in the process-configuration specification §16 *(decided 2026-09-27)*. This is the map; the rest of this document uses the technical terms.

| Apiary name | Technical term | What it is |
| --- | --- | --- |
| apiary | the system | everything one owner runs: all nodes, processes and operators; the smallest apiary is one node |
| yard | node | one machine |
| keeper | main loop | the node's master orchestrator: routes a person's requests, starts and supervises the root orchestrators |
| hive | process | one domain with its own directory, board and exactly one root orchestrator: a family calendar, a café, a hobby project |
| queen | root orchestrator | the one orchestrator started from outside a process; its hosted session is the root session |
| comb | board | the process's board; its cells are the tasks |
| bee | run | one agent run of a role; a goal-bound run is hosted as an owner-run session |

---

## 0. The design on one screen

- **What it is.** The session host keeps agent sessions alive in the background. It starts a harness in a given directory with a given goal, lets a person attach and detach, delivers messages into a session (the doorbell) and reports events. *fact for today; the module is decided*
- **Phase 1: extract it as it is.** The provider-host plane of `tb-sessiond` moves into this repository behind a contract, with no change in behaviour. *decided (2026-09-24), priority* Everything after §3 is architecture for later phases.
- **One host per node.** One daemon per OS user supervises every session: the main loop's and every root orchestrator's. Each session gets its own terminal holder, so compatible sessions can survive unplanned daemon loss. A requested controlled restart closes admission and parks its consumers (§14). Shared-artifact replacement requires those consumers to stop (§8.2). *proposed, pending the operator*
- **Access control.** Today one bearer token, which any process of the operator's user can read, controls every session. The target:
  - peer-credential checks;
  - a capability token per session: a session acts only on itself and its own children;
  - a supervision matrix;
  - operator commands backed by the operator's own authority.

  *proposed; closing access is the first refactor after the as-is extraction, decided (2026-09-25)*
- **Doorbell modes.** `next_turn` is the default. `steer` puts the message into the running turn where the harness can take it. `interrupt` is only for halt, park and emergency stop. Delivery no longer waits for an attached client. *proposed*
- **Park and emergency stop.** Park is the orderly stop: the orchestrator writes a note, the note is bound into the board goal, and the next start resumes from it. The host's `curator session park` does it, and `keeper park` fronts it. The emergency stop is a separate, unmistakable command for when there is no time: `keeper estop [<hive>]`, over the host's `curator session estop`. The kernel adds only a snapshot, `task-board board snapshot`, and no kernel command parks or stops a session or the node. *park decided (2026-09-24); the commands decided (2026-09-27); the emergency stop's mechanics proposed*
- **Its command line.** The host is a Curator provider, `curator-session`, so people and tools reach it as `curator session list|show|status|attach|watch|park|stop|estop|logs|doctor|restart`, and keeper fronts it (§12.1). `task-board session` stays an alias for one release after the node host lands, and its context-registry commands move to `task-board context`. Phase 1 changes no command. *decided (2026-09-27)*
- **A session is a cache.** The goal, the handoff note and the board carry the work across compaction and restarts. Adapters report context and limits so that a handoff is written before compaction. *proposed, not the first priority*
- **One entry point after a reboot.** `keeper up`, started at login, brings the host, the main loop and every root orchestrator back. *proposed*
- **Conversation and watch.** `keeper` alone opens the main loop's session. `keeper chat` with no target lists every session and opens a conversation with the chosen one; `keeper watch` only looks. Underneath all three are the host's `curator session attach`, in control mode, and `curator session watch`, read-only. *proposed; the command names are the operator's (2026-09-25)*
- **Later: pluggable terminal backends.** `pty`, `tmux` and `ax`, behind a transport abstraction. *decided to record (2026-09-24); the design is proposed*

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
| terminal holder | the process that owns one session's terminal: the host itself today, a per-session holder process later (§5, §13) |
| terminal backend | the kind of holder: `pty`, `tmux` or `ax` (§13) |
| capability token | a secret the host issues to one session, naming what that session may do (§6) |
| kernel | task-board's own code, the deterministic part of the board with no model involved. "The kernel writes a snapshot" means task-board writes it |
| main loop | the node's master orchestrator (main-loop specification) |
| `curator session` | the host's command line: the Curator provider `curator-session`, which `curator session …` dispatches to and `keeper` fronts (§12.1) |

## 2. Today's implementation *(fact)*

### 2.1 One control plane per board

- `tb-sessiond` is keyed by board: a `BoardKey` "identifies one Session Manager control plane" (`tools/board-cli/internal/sessionmanager/key.go`).
- Its state lives in the user state directory, never under the board tree (`layout.go`, `DefaultLayout`). The per-session notice queues are in its `NoticesDir` (`layout.go:44-48`).
- Startup reconciles durable records, restores running sessions, sweeps stale sockets and reports `startup_healthy` (`docs/codex-goal-runtime.md`, "Self-restoring startup").

### 2.2 Root (primary) sessions

- `task-board claude` and `task-board codex` start or attach the board's primary session. The wrapper validates the launch plan and runs primary-session preparation, then calls `ConnectOrStart` (`docs/claude-goal-runtime.md:32-39`).
- The `parent/primary` slot is singular and provider-independent: "one board has one authoritative primary goal owner". A second provider is refused with `session_provider_conflict`, or, on a TTY, offered a takeover (`docs/claude-goal-runtime.md:9-29`; `docs/codex-goal-runtime.md:10-31`).
- A graceful detach prints `anthropic-board resume SES-...` or `openai-board resume SES-...`. The manager then either reattaches a healthy host, or relaunches a dead one with its durable native identity: `--resume` with the UUID for Claude, `thread/resume` for Codex. Before exposing the terminal it reconciles the current primary goal (`docs/claude-goal-runtime.md:290-303`; `docs/codex-goal-runtime.md:233-247`).
- `task-board session stop SESSION-ID` stops one session "without clearing its board goal" (`tools/board-cli/cmd/session.go:94-95`). The other operator commands are `status`, `list`, `logs [-f]`, `doctor` and `restart` (`cmd/session.go:35-274`).

### 2.3 Owner-run sessions

A goal-bound spawn runs in its own hosted session. The manager:

- "starts or resumes one PTY for that run owner";
- strips the parent's Session Manager and app-server credentials and provider session markers;
- injects the native goal and waits for acknowledgement of the exact revision (`docs/claude-goal-runtime.md:41-56`).

Sub-orchestrators spawned with `--role orchestrator` live here. A person cannot attach to them. They are observable only through `task-board spawn transcript`, `observe` and `events`.

### 2.4 Goals and continuation

- The host delivers the board goal as the harness's native goal. For Claude that is `/goal tbg-<token> [board goal <ID> rev <N> - reference only, not the condition - objective "<excerpt>"] <provider-condition>`, submitted through bracketed paste (`docs/claude-goal-runtime.md:57-66, 180-183`). Codex receives the same condition through its app-server.
- "The provider-native evaluator owns continuation". So a restarted or resumed session goes on working without being told to: the host re-applies the current goal on resume, and the harness's own evaluator keeps it going until the condition holds (`docs/claude-goal-runtime.md:180-183, 298-303`).
- The goal itself is task-board data. `BoardGoalRevision` in `pkg/board/board_goal_v2.go:173` carries the objective, the provider condition, the reason and the lineage. It has no field for a handoff note today (§8).

### 2.5 The doorbell today: in-session notices

- **Who can be rung.** Only a hosted primary session is addressable (`TASK_BOARD_MANAGED_SESSION_ID`). Owner-run sessions are not, and the marker is stripped from their children (`docs/claude-goal-runtime.md:392-403`).
- **Producer.** The tracked-spawn terminal hook (`DeliverTerminal`) writes one durable notice per terminal run into the session's inbox.
- **Consumer.** A pump in the daemon delivers only to a session that is running, whose host implements `ProviderSessionNotifier`, and that "has at least one attached client" (`internal/sessionmanager/manager_notice.go:76-77, 90-96`). A detached session keeps its notices queued. On the next attach they are replayed, coalesced into one message.
- **Delivery into Claude.** The text goes in by bracketed paste into the PTY, serialized with goal injection (`docs/claude-goal-runtime.md:424-447`; `internal/sessionmanager/claude_host.go:1090-1111`). "Claude Code does not process typed input … while a tool call is executing": injected text sits in the PTY input queue until the running call returns (`docs/claude-goal-runtime.md:213-219`).
- **Delivery into Codex.** An idle thread gets a new turn through `turn/start`. An active thread gets the text steered into its exact in-progress turn through `turn/steer` with the expected turn ID, and is refused when that turn is ambiguous (`internal/sessionmanager/codex_host.go:700-780`). The app-server also offers `turn/interrupt` (`internal/sessionmanager/codex_context.go:663`).
- **waggle today.** waggle §10 maps its delivery modes onto these mechanics:
  - `interrupt` becomes a Codex `turn/steer` or a Claude paste at a safe point;
  - `next_turn` becomes a Codex `thread/queue/add`, or a Claude paste after the turn;
  - `on_request` rings no doorbell.

  It also says that "with no client attached messages wait durably".

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

### 2.8 Access control today

- **The socket.** The control socket lives in the per-board state directory, or under a per-user `/tmp/tbsm-<hash>/` when the path is too long for Darwin (`layout.go:74-86`). It is created with `net.Listen` and then set to mode 0600 (`daemon.go:154-168`).
- **The token.** The daemon writes one random bearer token to a private file in its state directory. Every control request must carry it (`daemon.go:170-190`; `control.go:431-442`).
- **The consequence.** One token grants every operation on every session of that board: stop, attach, deliver, take over. Every agent session and every run runs as the operator's own OS user, and any of them can read the token file. So any agent can operate on any session. The protection keeps out other OS users, not the operator's own agents.

## 3. Phase 1: extract the module as it is *(decided 2026-09-24, priority)*

The first step moves code, not behaviour. Improvements come after the module stands on its own. Sections 4 to 14 describe later phases; none of them is part of phase 1.

### 3.1 What moves here

- **The provider-host plane:**
  - the `ProviderHost` and `ProviderHostFactory` contract and its optional capabilities;
  - the Claude PTY host and the Codex app-server host;
  - attachments and the terminal relay;
  - the notice inbox and pump;
  - session records and their store;
  - liveness, restore and repair;
  - the operator log.
- **The daemon skeleton:** instance lock, control socket, self-restoring startup and health reporting. It is parameterized by a key, today task-board's `BoardKey`.

### 3.2 What stays in task-board

- The board and its goal ledger (`board-goal-v2`): what the goal is, who may write it, and when it counts as achieved.
- Routing, the spawn runtime, run manifests, acceptance and handoff evidence.
- The `task-board claude|codex`, `session` and `spawn` commands. They keep their names and behaviour and reach the module through a thin seam: task-board supplies the goal revision to apply, and consumes the host's acknowledgements, events and notices. Their later names are in §12.1.

### 3.3 The contract

- **In process first.** Phase 1 keeps the adapters as in-process Go implementations behind the existing interfaces, exported from this module. *proposed*
- **The out-of-process form follows the ax provider plugin protocol** (`ax` SPEC §7.1-§7.3): *decided 2026-09-24*
  - line-delimited JSON over stdin/stdout, one plugin process per operation;
  - a manifest and a probe;
  - structured errors that fail closed.

  A plugin exits after each operation, so only stateless operations cross that boundary: probe, launch spelling, goal-command rendering and acknowledgement parsing. The long-lived terminal stays with the host or its terminal holder (§5, §13).
- **Operations:** start, resume, stop, status, attach target, apply a goal revision, clear it, acknowledge the exact revision, deliver a notice, and an event stream.
- **Adapters:** Claude (PTY) and Codex (app-server) first. agy, Pi and Qwen follow, each under its own story. Muse goes ahead of the extraction, narrowly: the `muse serve` owner host is built now, unwired, and wired after TASK-260927-1okvcs; no Muse producers until the environment fix is released and pinned (track; *decided 2026-09-28*).
  - **Muse** *(decided 2026-09-24)*: an owner through `muse serve`, in its own Curator managed home. The original binary pin is superseded by the optimistic default of §8.1 *(decided 2026-10-05)*. It is built by a separate orchestrator.
  - **agy** *(decided 2026-09-24)*: hooks in the profile's managed home, scoped by an environment marker.

### 3.4 Where it sits in the launch plane

- A person's sessions enter through `curator run`. With the `ax` integration configured, they are hosted "from birth by whatever implements the `ax start --launch-plan` contract", and "until an `ax` implementation ships, task-board's session daemon implements the contract through a bridge" (curator-spec Decision 0021 §2).
- A host "receives a composed plan and never composes" (0021 §2; Decision 0019). This module is where that bridge's hosting half lives.
- The bridge's shape, an `ax` shim or the daemon answering `ax start`, is Decision 0021 open question 1.
- Tracked non-interactive children stay with `task-board spawn` (0021 §6).
- **Until the bridge exists**, primary and goal-bound sessions are built with the same construction as children: a per-role Curator profile through agents-management. Gate A does not wait for the `curator run` → host bridge, so the bridge is a later dependency (track SH6), not a prerequisite of SH1 or SH2. *decided 2026-09-28*

### 3.5 Done when

- task-board's session, goal and notice test suites pass unchanged against the extracted module.
- The goldens for both harnesses are byte-identical.
- The operator commands behave as before.
- No behaviour difference is reported in the change log. Any difference found is either removed or listed and accepted by the operator.

## 4. Requirements from the main loop *(proposed; later phases)*

The main loop starts and supervises the root orchestrator of every process on its node. The host must support it:

1. **R1. Start in a process directory.** Start a hosted root session in any process directory the main loop has registered:
   - the session gets that directory as its working directory, the process's Curator profile and managed home, and the board's `parent/primary` goal;
   - it runs in the background with no client attached;
   - the launch is composed by `curator run` (Decision 0021) and handed to the host once the bridge exists; until then it uses the children's construction (§3.4).
2. **R2. One root session per process.** Exactly one root session is started from outside for each process. Today's singular `parent/primary` slot already enforces this per board (§2.2). Sub-orchestrators and runs are spawned by the root session inside its own board; the main loop never starts them.
3. **R3. Attach.** A person can attach to root sessions and to owner-run sub-orchestrators, in control mode (`keeper chat`) or read-only mode (`keeper watch`) (§12). Attaching to short-lived worker runs is optional; their transcripts suffice.
4. **R4. Doorbell for detached sessions.** Notices and main-loop messages reach a running session that has no client attached, through the delivery modes of §7.
5. **R5. Supervision.** The main loop can wake, park and stop root sessions within the matrix of §6.3. The operator can always stop a session.
6. **R6. Events.** The host emits a stream that the main loop and waggle consume:
   - session state changes;
   - goal acknowledgements;
   - turn starts and ends;
   - context and limit readings (§10);
   - failure markers;
   - park and emergency-stop completion.
7. **R7. Many processes on one node.** One host serves all of them (§5).
8. **R8. Recovery.** After a reboot every session that was running comes back through one entry point (§11).
9. **R9. Orchestrator sharding across harnesses and subscriptions** *(decided 2026-09-24: carried by this module)*. The module carries the sharding of orchestrators across harnesses and across subscriptions. What a shard is, and which phase builds it, are open.

## 5. One host per node *(proposed; pending the operator)*

Today each board has its own daemon (§2.1). On a node with a main loop and several processes, that means several daemons, several tokens, several upgrade paths, and several places holding terminals. The proposal replaces this with one host per OS user and one terminal holder per session.

- **One supervisor.** One host process per OS user on the node supervises every hosted session: the main loop's session, each process's root session, and their owner-run sessions.
- **Per-board state inside.** Each board keeps its own namespace inside that host. The singular `parent/primary` slot stays per board, so R2 holds as today.
- **One holder per session.** Each session gets its own terminal holder: a small `pty` holder process, a tmux pane, or later `ax` (§13). The holder owns the terminal and the harness process. The host owns the registry, the goals' delivery, the doorbell router, access control and the event stream.
- **Survival after daemon loss.** After unplanned daemon loss, compatible holders can keep running. The new host re-adopts them from its durable records and the holders' sockets, subject to admission and reconciliation (§14), as today's startup already does for Codex (§2.7). A requested controlled restart parks and stops its intended consumers first. Shared-artifact replacement cannot use holder survival to skip that stop (§8.2).
- **One registry.** One list of sessions serves attach (§12), the doorbell (§7), supervision (§6) and recovery (§11).

Why this is more stable than a daemon per board:

- A harness that crashes takes down its holder, not the host.
- A host crash need not take down its harnesses. A controlled restart stops them resumably before restoration.
- There is one upgrade path, one startup reconciliation and one event stream, instead of one per board.
- The main loop talks to one endpoint, not to a daemon it must find for each process.

Why it is safer:

- Access control, the supervision matrix and the audit log live in one place, instead of one bearer token file per board (§2.8).
- A holder's socket accepts only the host. The host authenticates to it with a secret passed through an inherited file descriptor when it spawns the holder, and that secret is never written to disk.
- The limits stay the same as in §6.4: processes of the same OS user can still signal each other, and can trace each other where the operating system allows it.

Phase 1 keeps one daemon per board. The node host is a later milestone (track SH3).

![One host per node with per-session holders](../diagrams/artefacts/plantuml/one-host-per-node.svg)

## 6. Access security *(proposed)*

Today one token controls everything, and every agent can read it (§2.8). The target closes the socket, identifies callers and gives each session only the powers it needs.

*Order (decided 2026-09-25).* Phase 1 extracts the host as it is, token and all. Closing access, as described in this section, is the first refactor after that extraction.

### 6.1 The socket

- The control socket is created inside a directory of mode 0700 that the host creates before it listens. The socket itself is mode 0600. This closes the window between `listen` and `chmod` of §2.8.
- Every connection's peer credentials are checked: `getpeereid` or `LOCAL_PEERCRED` on macOS, `SO_PEERCRED` on Linux. A peer of another UID is refused.
- The Darwin fallback path under `/tmp` gets the same private directory.

### 6.2 Capability tokens

Every agent runs as the operator's UID, so a UID check cannot tell sessions apart. Authorization therefore uses capability tokens.

- **Per session.** The host issues each hosted session its own token when it starts it. The token says what that session may act on: itself and the sessions it spawned. It is passed into the session's environment and is valid only while the session lives.
- **The main loop's token** covers the root sessions of the processes registered with it. It covers nothing inside them.
- **Operator commands** need the operator's authority: attaching with control, stopping any session, parking any session, and emergency stop.
  - Until the trust component exists, the operator's authority is a CLI act outside any hosted session or run. The host checks that the caller carries no session token and that its process is not a descendant of a hosted session. This is a guardrail, as the process-configuration specification calls unsigned human acts.
  - From the trust component on, it is a presence-bound confirmation: the companion app, or the operating system's user authentication (relux-works/curator-trust `spec/trust.md`).
- **Every accepted request is logged** with the token's session, the act and its target.

### 6.3 The supervision matrix

| Actor | Wake | Park | Stop | Read events and transcripts | Attach | Emergency stop |
| --- | --- | --- | --- | --- | --- | --- |
| the operator (a person) | every session | every session | every session | every session | every session, control or read-only | yes |
| the main loop's session | root sessions of registered processes | root sessions of registered processes | only with the operator's confirmation or a delegation that covers it | registered processes' root sessions | never | only under a delegation that names the trigger |
| a root session | its own subtree | its own subtree | its own subtree | its own subtree | never | no |
| an owner-run session | its own spawned runs | its own spawned runs | its own spawned runs | its own spawned runs | never | no |
| a worker run | nothing | nothing | nothing | itself | never | no |

- Attach is for people. Agents read events and transcripts instead.
- From waggle milestone CM2, wake, park and stop between sessions travel as signed waggle `cmd` messages in the `halts` category, and the host verifies them against the roster before acting. Until then, the capability token is the check.
- The main-loop specification states the same matrix for the main loop's side.

### 6.4 What this does not stop

Tokens in a session's environment can be read by other processes of the same user on systems that expose process environments. On Linux, `/proc/<pid>/environ` is readable by the same UID. So capability tokens give least privilege and attribution among well-behaved agents; they are not a boundary against a hostile one. The boundary is either of these:

- running agents under an operating-system account separate from the operator's;
- the presence-bound confirmations of the trust component, for the acts that need a person.

Both belong to the trust specification (open question 11).

## 7. Doorbell delivery modes *(proposed)*

A message into a session must be both autonomous and stable. The session must act on it without anyone attached, and it must not break the work in progress. The host delivers every message in one of three modes.

| Mode | When | Meaning |
| --- | --- | --- |
| `next_turn` | the default for every message | delivered at the next turn boundary. A session that is idle gets a new turn at once |
| `steer` | a message that should influence the running turn, where the harness supports it | put into the running turn without stopping it |
| `interrupt` | only halt, park and emergency stop | the running turn is stopped first, then the message is delivered |

Per adapter:

| Adapter | `next_turn` | `steer` | `interrupt` |
| --- | --- | --- | --- |
| Codex (app-server) | `turn/start` when idle (fact, `codex_host.go`); queued for after the turn otherwise (`thread/queue/add`, per waggle §10) | `turn/steer` with the expected turn ID into the exact in-progress turn; refused when that turn is ambiguous (fact, `codex_host.go:736-772`) | `turn/interrupt` (fact, `codex_context.go:663`), then `turn/start` |
| Claude Code (PTY) | submitted by bracketed paste when the prompt is free; typed input waits in the input queue while a tool call runs (fact, `docs/claude-goal-runtime.md:213-219`) | a paste at a safe point between tool calls. When no safe point can be observed, it degrades to `next_turn` *proposed; to verify per harness version* | the harness's interrupt key (Escape), then the paste *proposed; to verify* |

- **Delivery does not depend on an attached client.** This removes the condition in `manager_notice.go:94`. Coalescing stays: several waiting messages arrive as one.
- **Every delivered turn emits an event** (R6), so turns that nobody watches are visible to the main loop and to people.
- **Mapping to waggle:**
  - waggle's `next_turn` is the host's `next_turn`.
  - waggle's `interrupt`, which waggle §10 implements as a Codex `turn/steer` or a Claude safe-point paste, is the host's `steer`.
  - The host's own `interrupt`, which stops the turn, is reserved for `cmd.halt`, park and emergency stop.
  - waggle's `on_request` rings no doorbell; the message waits to be pulled.
- **waggle needs an amendment.** Its §10 still lets messages wait while no client is attached. The detached delivery of this section needs that sentence to change (track dependency).

![Doorbell delivery modes](../diagrams/artefacts/plantuml/doorbell-delivery.svg)

### 7.1 Harness-specific mechanisms inside adapters *(proposed)*

An adapter may use a harness's own extension mechanism where it makes the contract more reliable. That stays an implementation detail of the adapter; the contract of §3.3 and the modes above do not change.

- **Example: Claude Mods.** Claude Code plugins with function hooks: a TypeScript module (`hooks/register.ts` in an ordinary plugin directory) registers handlers that wrap engine events and interface rendering, like middleware. In the local CLI they are enabled by `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1`. They are in early access, and their API may change between releases without notice.
- **What the Claude adapter could use them for:**
  - reliable message delivery, instead of typing into the terminal;
  - context telemetry for §10;
  - panels that show the board and the goal inside the harness.
- **Never for authority.** Authorization must not depend on a mod that can be removed or disabled, and their own guidance says so. Supervision, capability tokens and confirmations stay in the host and the trust component (§6).
- **Not a dependency now.** It is early access and Claude-only. The PTY paths of the table above remain the implementation.

## 8. Park *(decided 2026-09-24; the mechanics proposed)*

Park is the orderly stop. It leaves a clear entry point for the next start.

`curator session park <session>` starts it; `keeper park` and `keeper down` front it for the person and the main loop (§12.1). *decided 2026-09-27; `task-board session park` is withdrawn*

1. The host rings the session's doorbell with a park directive, in `interrupt` mode.
2. The orchestrator writes a park note as a resource of the goal's element. The note lists:
   - runs in flight;
   - Change Requests awaiting review or integration;
   - decisions waiting for a person;
   - the next steps;
   - pointers to anything else needed.
3. The host asks the kernel, and task-board appends a goal revision with the same objective and condition and a new `handoff_ref` field naming the note. The field is an addition to `BoardGoalRevision` (§2.4). Nothing else about the goal changes.
4. The host stops the session through the resumable stop. The goal is not cleared (§2.2).
5. On the next start the version gate (§8.1) passes before the harness is resumed or replaced. The native goal payload then carries the reference, for example `[board goal <ID> rev <N+1> ... handoff "<note>"]`. The start may be `curator run`, or its alias `task-board claude|codex`, the main loop's wake, or `keeper up` (§11). The session reads the note first, and the native evaluator continues from there (§2.4). *proposed*

If the orchestrator does not write the note within a bound, the kernel snapshot of §9 becomes the note.

The same mechanism writes a handoff without stopping, before compaction (§10).

**Runs park by directive.** A run, whether a sub-orchestrator in its owner-run session or a worker, is a board record the kernel tracks, so it parks through the kernel: `task-board spawn directive <RUN> park`, a new directive kind beside `cancel` and `pause_after_checkpoint`. For an owner-run session the host carries it out as in steps 1–4. A root session parks its sub-orchestrators this way, and the person does through `keeper park <hive>/<RUN-ID>`. *decided 2026-09-27; the directive's semantics proposed*

### 8.1 The optimistic version gate

A version change must be caught for every session on resume. This covers the main loop, root sessions and owner-run sessions. It covers manual starts, wakes, spawns, login restore and host restarts. *decided (2026-10-05)*

The default is optimistic. A new harness may run when it passes authenticity checks where the vendor publishes checksums, capability and resume probes, and the known-bad check. Strict verification and pinned harness versions are opt-in. Muse follows the same default. *decided (2026-10-05)*

- **Launch identity.** Capture the executable path, SHA-256 and probed build id at launch. Keep this identity in the durable session record when the session parks. Re-probing a replaced path must not rewrite the identity of the running process. A path or version string alone is insufficient. *proposed*
- **Resume record.** Record the native resume identity, goal revision, `handoff_ref`, launch-plan identity and seal, and module and adapter versions. Record the cached artifact identity and protocol probe results. The host does not persist the launch environment (§11). *proposed; no persisted environment decided (2026-10-05)*
- **Run from cache.** Copy the executable and its required artifact root on first use into a content-addressed cache outside launcher pruning. Resolve the cached executable by SHA-256. Execute that copy rather than the launcher's current default or a replacement found on `PATH`. Protect the whole artifact root while referenced by live or parked sessions, surviving holders, admitted launch or retry attempts, unresolved updates or active exact pins. Release a live reference only after confirmed consumer exit. Keep two or three last-known-good builds by hash in otherwise unreferenced history. That target never overrides protected references. Report extra storage needs instead of pruning artifacts in use. A missing or changed cached artifact is reported. *decided in direction (2026-10-05); copy-on-first-use, artifact layout and retention rules proposed*
- **Adapter contract.** Each adapter describes its artifact root, build probe, capability probe and native-resume probe. It may provide a checksum source and an updater-off hook. Managed sessions disable the vendor updater where one exists. An optional soak delay may defer adoption. *proposed; managed adoption decided in direction (2026-10-05)*
- **Checks before exec.** Match vendor-published checksums where available. Absence of vendor checksums does not itself reject an optimistic launch. Probe the capabilities used by the session and native resume. Consult the known-bad feed by exact build identity. Our sessions never execute a bad-listed build, including a cached fallback. Strict release-list membership is not required by the optimistic default. *decided in direction (2026-10-05); check placement proposed*
- **Probe isolation.** Capability and native-resume probes use fixtures or isolated copies of protected state. They must not own a production goal, mutate the original native-session state or create an unregistered production consumer. Any probe that starts a component consumer obeys admission, including the update lock and controlled-restart closure, and confirms teardown. Check authenticity and known-bad status before executing the probed artifact. Bind the result to the exact selected cached artifact and applicable adapter and entrypoint. The real resume is a separate admitted operation. *proposed*
- **Plan and admission.** The launch plane recomposes and finalizes the plan for the selected cached executable (§3.4, §11). A changed module, adapter, package or profile also requires that launch path, even when harness bytes are unchanged. Preserve the native resume identity, goal, `handoff_ref` and the plan's full permission posture (§11). Verify the plan immediately before exec. Check the node-wide admission lock (§8.2) and controlled-restart closure (§14). The host receives the plan and never composes it. *proposed*
- **Automatic fallback.** If the candidate fails a probe, launch or resume, automatically make one retry with an eligible last-known-good cached build. First close the failed attempt and confirm that its harness, holder or app-server and relevant child consumers can no longer act on the conversation. A timeout is not evidence of exit. Uncertain cleanup or ownership leaves the session parked with the cause reported; no second attempt starts. Do not select the just-failed identity as its own fallback in that failure context. The retained build must safely use the current state (§8.2). Disable the vendor updater for the retry. Reject checksum failures and bad-listed candidates before exec and select an eligible last-known-good build instead. Preserve the native identity, goal, `handoff_ref` and permission posture. The fallback still passes identity, known-bad, plan, admission and restore-readiness checks (§11). An opt-in strict or pinned policy also applies. It must not rewrite artifacts used by another live session. Record the failed candidate, cause and fallback outcome. Successful probes and a ready launch or resume mark the selected build last-known-good. *decided in direction (2026-10-05); serial retry and record details proposed*
- **Refusals.** Use `harness_update_unverified` only when probes fail and no eligible last-known-good copy exists, or when the opt-in strict policy refuses admission. Report authenticity, known-bad, missing-artifact, plan and launch/resume failures with their specific causes. If fallback fails, keep the session parked and report both attempts. A fresh session from the goal and park note passes the same checks. *decided in direction (2026-10-05); cause reporting proposed*
- **Strict and pinned.** An operator may choose strict verification against qualified release evidence or pin an exact cached identity. These policies remain subject to the known-bad check. Qualification work supplies known-good signals and probe fixtures for the optimistic default. It does not make every unlisted build a refusal. *decided in direction (2026-10-05); policy encoding proposed*
- **Muse's probes.** The MSP probe checks schema compatibility for everything the host consumes. Consumed names, types and enums must be unchanged. Server request types must be known. Additions are allowed. Record the schema fingerprint as probe evidence rather than an exact default pin. The PTY probe replays the grammar of a short real Echo turn. Unknown records or screens defer admission. At runtime the host remains fail-closed: unknown or mistyped MSP deliveries taint the dispatch. Probe failures use the same fallback rule as other harnesses. *decided in direction (2026-10-05); probe fixtures proposed*
- **Report every outcome.** A successful version change records both identities and sends an `updated X→Y` notice to the session and its supervising root session. A root session's notice goes to the main loop. Reports include successful resumes, fresh starts, fallbacks and every refusal. *proposed*

Claude Code's self-updater can replace the executable between stop and start. Muse's launcher can update itself and prune old binaries. The cache keeps the executable used by our sessions outside that pruning. *design inputs (2026-10-05); cache mechanics proposed*

### 8.2 The slim update window

Detect available updates and propose them to the operator. Pending updates may be applied during a planned restart window. Start and reboot restore install nothing. `curator runtimes` records the install method. *decided in direction (2026-10-05)*

The safety pieces are a durable node-wide admission lock, park first, a minimal update record and rollback. *decided (2026-10-05); sequence below proposed*

1. **Close admission.** Set one durable node-wide lock before parking for replacement. Every automatic starter checks it, including wakes, spawns and login restore. Serialize admission with lock acquisition. The lock survives daemon exit, replacement and restart. It stays set until the operator clears it. A successful update or rollback does not clear it automatically. *proposed*
2. **Park first.** The daemon broadcasts a park request to every session. It waits for park notes and the goal revisions that bind them through `handoff_ref`. Account for holders and spawned or detached tool and integration consumers. Drain in-flight landings and integrations and obtain adapter safe-point evidence before stopping. Muse must have no in-flight turn, pending approval or userInput, or unsettled usage revision; its MSP END-WAIT and usage-ledger latches must be settled. Confirm the relevant consumers have stopped. At the bound, force remaining resumable stops and record them distinctly. Missing notes use the kernel snapshot (§8, §9). Neither a forced stop nor a snapshot proves quiescence. Unconfirmed stops or unresolved safe-point state defer mutation and keep admission closed for recovery. Daemon exit must not turn that result into successful drain. The normal sequence exits after confirmed stops and settled safe points. A forced stop is not an emergency stop. *proposed*
3. **Record and apply.** Before mutation, write a durable record with `component`, `from`, `to`, `source` and `method`. `from` and `to` identify the exact versions or cached artifacts. `source` is the update URL or registry reference. `method` is the installer or command without credentials. Known irreversible installer or state changes are outside this slim window until their policy is decided (§15). Apply the recorded update only after the drain and stop conditions above hold for affected consumers. The supervisor restarts the daemon after replacement. *proposed*
4. **Rollback and report.** A failed harness update selects an eligible last-known-good cached build through the serial fallback rules (§8.1). A failed host or module update uses the pinned previous version only if it can safely use the current state. This state-compatibility condition also applies to harness fallback after a failed first resume. If compatibility is unknown or false, report rollback unavailable and recovery required. Keep affected sessions parked and any update lock held. Restoring a binary does not establish that state was restored. Preserve the park records and report the update or rollback outcome. Resume still waits for operator clearance of the lock. An unresolved failure keeps admission locked. Replacement of shared artifacts, including rollback, requires the same confirmed drain and stops. *proposed*
5. **Restore after clearance.** The restarted daemon enforces the lock before any automatic restore. The operator clears it before eligible sessions resume through §8.1 and §11. An ordinary controlled restart without an update restores eligible sessions automatically after the reconciliation of §14. An explicit operator stop remains a stop. *proposed*

Host, module and Curator package updates use their audited, pinned distribution path. `keeper down` leaves the sessions parked and the host running. A controlled daemon restart is the path that exits the daemon and lets its supervisor restart it. An emergency stop or unplanned reboot does not install pending updates. *proposed*

Acceptance must prove isolated multi-session restore with the same native conversation, goal and RC name/session identity. Include an unlisted build that passes probes, a probe failure with successful last-known-good fallback, and a probe failure without a fallback. Include a bad-listed cached build and a Muse schema addition versus an incompatible change. Race wakes, spawns and login restore against the lock. Restart the daemon with the lock still set. Report every negative case. These are future acceptance cases. *proposed*

Restore and update acceptance use isolated homes and state and injected key and auth providers. They never access the operator's real keystore or production credentials. A failing injected provider must report a parked outcome without falling back to a real provider. *proposed; injected key provider required by the design inputs*

Additional acceptance covers the safety boundaries below. These are future implementation checks. *proposed*

- An integration outlives its parent, a holder does not acknowledge stop, and Muse has pending approval, userInput or unsettled usage at the deadline. Report forced stops distinctly. No shared replacement or rollback write begins on assumed drain.
- A candidate starts but its resume acknowledgement times out while it survives. No fallback executes until cleanup is confirmed. Cleanup failure leaves it parked. Session B's fallback leaves live session A's artifacts intact.
- A held lock blocks consumer-starting probes and real resume. A failing isolated probe leaves original state unchanged and no surviving consumer. A build that passes the short Muse probe later emits an unknown or mistyped MSP delivery; runtime dispatch still fails closed.
- A remains live while later builds exceed the history target. Cleanup preserves A's whole artifact root, and park/resume still resolves A by hash. An older active exact pin also survives cleanup.
- An installer or first resume changes state to a format the retained version cannot read, then fails. The old binary is not resumed blindly. Report rollback unavailable separately from successful rollback.
- Race a wake and spawn against an ordinary controlled restart with no update. No consumer escapes the park set. Successful reconciliation restores eligible sessions automatically at any uptime. A real update lock still needs the operator.

![Park and resume](../diagrams/artefacts/plantuml/park-and-resume.svg)

## 9. Emergency stop *(proposed)*

For a stop that cannot wait, for example power about to fail, there is a separate command. It is not a flag on park, so that nobody runs it by accident and everybody who reads it knows what it is.

- **One command for people:** `keeper estop [<hive>]`, for the whole node or for one process. Underneath is the host's `curator session estop [<process>]`. No kernel command spells an emergency stop, because it acts on sessions and on the node, which the kernel does not own; the kernel contributes the snapshot, `task-board board snapshot`. *decided 2026-09-27; `task-board emergency-stop` and its alias `estop` are withdrawn*
- **A loud confirmation.** The command prints what it is about to stop: every hosted session and every run on the node, or of one process. It then asks for the phrase `EMERGENCY STOP` and the operator's presence: the operating system's user authentication on the node, or the companion app's presence-bound signature (main-loop specification §10.3; relux-works/curator-trust). No flag replaces the phrase, and the kernel's `--confirm` stays a plain acknowledgement of a destructive act.
- **Operator, or the main loop under a delegation.** The operator may always run it. The main loop may trigger it only under a delegation that names the trigger (for example exhausted limits or a detected runaway); root orchestrators, sub-orchestrators and runs are refused (§6.3). *decided 2026-09-25*

What it does, in order:

1. **The kernel writes a snapshot for every affected board:** the host runs `task-board board snapshot` on each. The kernel is task-board's own code, not a model. It records:
   - the runs in flight and their states;
   - Change Requests awaiting review or integration;
   - the output of `task-board worktree obligations`;
   - unrouted notices.
2. **Each orchestrator may add a short note** within a bound of 10 to 30 seconds (default 20). A missing note does not delay anything.
3. **Runs are stopped** so that they can resume. Today a run is cancelled, not suspended, so a resumable suspension is open question 5.
4. **Sessions are stopped** through the resumable stop. Each goal gets a revision whose `handoff_ref` names the snapshot and the note, appended by the kernel as in §8.
5. **The command reports** what it stopped and what it could not stop.

`keeper up` (§11) brings everything back from those notes.

![Emergency stop](../diagrams/artefacts/plantuml/emergency-stop.svg)

## 10. Compaction and context *(proposed; designed now, not the first priority)*

- **A session is a cache.** The authoritative state is the board, its goal and the handoff note, never the harness's context. A session must survive compaction or replacement by rereading them.
- **Adapters report context telemetry** as events (R6):
  - Claude Code's `PreCompact` hook and the context usage its status line receives;
  - Codex's token-usage events from the app-server.

  The exact fields are verified per harness version.
- **Before compaction** the host rings the orchestrator in `next_turn` mode, or `steer` when compaction is imminent, to write a handoff. task-board appends a goal revision with `handoff_ref`, as in §8, and the session keeps running. After compaction the native goal payload carries the reference.
- **Limits.** The same events carry provider limit readings that the launch plane already tracks.
- **Budgets for Muse and agy** *(decided 2026-09-24)*. The host enforces the budgets of the Muse and agy adapters: it keeps the accounting, rings a wrap-up notice at 90 % of the budget and pauses the session at 100 %. Whether Claude and Codex get the same, and which delivery mode of §7 the wrap-up notice uses, are open.
- **The watchdog** of the main-loop specification consumes both kinds of events. It asks for a handoff early, parks a session that cannot continue, and tells the person.

## 11. Reboot restore *(proposed)*

- **One entry point.** A user login agent invokes `keeper up` automatically only after a fresh boot. Check boot identity and uptime and durably suppress subsequent login restores in the same boot. Serialize concurrent login and manual invocations with registry reconciliation. Skip and report already-live sessions. Manual `keeper up` and the explicit controlled-restart path work at any uptime. macOS uses a launchd user agent. Linux uses a systemd user unit. *proposed*
- **The order.**
  1. The host reads its registry and reconciles any ordinary controlled-restart closure (§14). It enforces the durable node-wide update lock before automatic restore or holder re-adoption (§5, §8.2). A remaining update lock blocks automatic starts until the operator clears it. *proposed*
  2. The main loop's session passes admission, the optimistic version gate (§8.1) and restore readiness below. It resumes through the bridge. A refusal leaves it parked and appears directly in the operator's report. *proposed*
  3. Each registered process's root session passes the same admission, gate and readiness checks. It resumes natively when possible. Otherwise it starts fresh with its goal and latest handoff or park note (§8, §9). Explicit operator stops remain stops. *proposed*
- **Resume without persisted environment.** The host stores no launch environment for replay. The login bridge runs `curator run <env> --hosted resume <SES>` through a login shell. `keeper up` uses this bridge once available. The launcher recomposes the plan from the operator's profile and login environment. It checks the profile pin and drift as on every hosted launch. A missing required variable leaves the session parked with `required_env_missing` and names the variable in the restore report. Variables exported only in an earlier terminal are not restored. *decided (2026-10-05)*
- **Restore readiness.** In the selected managed home and login environment, wait for the harness's positive auth-readiness result before starting hosted resume or a fresh fallback. Bound the wait; unsuccessful authentication leaves the session parked with an explicit auth-readiness reason. Unavailable credentials are not repaired by switching binaries. Recheck authentication on every retry. Authentication failure never clears or replaces the board goal or `handoff_ref`. Preserve the verified launch plan's full permission posture through native adoption and every retry, including `--dangerously-skip-permissions` and any deny lists. Use the plan and restart template's exact argv; do not reconstruct selected permission flags. Where RC is requested, observe actual registration of the expected RC name and session identity before reporting restore success or delivering queued work. A native resume without that registration is not ready and remains parked. Report RC-registration failure distinctly and preserve the goal and `handoff_ref`. Close any failed attempt under §8.1 before a retry. These checks also govern fresh starts from the goal and park note. *proposed; requirements from the real restore (2026-10-04)*
- **A complete report.** Report every resume, fresh start, already-live skip, version change, fallback, lock block and refusal. Include `harness_update_unverified`, `required_env_missing`, auth-readiness failures, RC-registration failures and recovery-required outcomes where applicable. Include a host-health section with observed service readiness and port-binding conflicts relevant to the restored node. Mark unavailable health checks unknown. These checks are diagnostic; restore does not change network configuration or service ownership. Report health and all session outcomes directly even if the main loop cannot start. No session disappears from the results. *proposed*
- **Updates available.** At start detect updates without installing anything. Post a durable `updates available` notice to each affected process's root session. Node-component notices go to the main loop. Deliver a `next_turn` doorbell when the recipient is ready. The root orchestrator proposes the update to the operator through the main loop. A parked recipient keeps its notice queued. The restore report lists the pending update directly. *proposed*
- **`keeper down`.** Park every session before a planned shutdown. Root sessions park first and the main loop last. The command leaves the host running and sessions parked. Pending updates use the separate planned window (§8.2). *proposed*

Restore acceptance covers credentials that become usable after login and credentials that never become usable. Verify the actual adopted and retry argv with `--dangerously-skip-permissions` and deny lists, unchanged goal and `handoff_ref`, and observed RC registration. Include native resume without RC registration and fallback encountering the same auth failure. Inject a port conflict and an unavailable health probe while the main loop cannot start; both appear alongside every session outcome. Cover first login after boot, a second login with a parked session, simultaneous login/manual invocation and explicit restart long after boot. Only the first login triggers automatic restore, and live sessions are never duplicated. Use the isolated providers and state required by §8.2. *proposed*

## 12. Conversation, watch and the picker *(proposed)*

The person uses three commands of the main loop; underneath each is the host's attach operation, `curator session attach` in control mode or `curator session watch` read-only.

- **`keeper`** with no arguments opens the main loop's session, starting the node first when it is down.
- **`keeper chat` with no target** opens a small terminal UI over the host's registry. It lists every hosted session with:
  - its process and role: main loop, root orchestrator or sub-orchestrator;
  - its harness;
  - its state and attached clients;
  - its last activity.

  The person picks one and opens a conversation with it (control mode) or a read-only view.
- **`keeper chat <hive>`** goes straight to that process's root session.
- **`keeper session list`**, alias `keeper ls`, prints the same registry without opening anything (`curator session list`).
- **`keeper watch <target>`** opens a read-only view of any session: the main loop, a root session or a sub-orchestrator.
- **Harness differences stay in the adapters.** Claude reattaches to its PTY or resumes by UUID; Codex re-adopts its thread. The person sees one behaviour.
- **Attach is for people only** (§6.3). A read-only view sends no input to the session. Control needs the operator's authority.

### 12.1 The command line *(decided 2026-09-27; the table's details proposed)*

`curator <noun>` dispatches to an executable named `curator-<noun>` on one of Curator's trust roots. The host's command line is therefore the provider `curator-session`, which ships beside `curator` as a first-party provider, or is installed with `curator provider install session` (curator-spec issue #100, curator issue #96).

| Command | Does | The person's front |
| --- | --- | --- |
| `curator session list`, `curator session show <session>` | the registry (§5): every hosted session, or one, with its process, role, runtime, state, attached clients and last activity | `keeper session list` (alias `keeper ls`); the `keeper chat` picker |
| `curator session status` | the host's own health: its instance, socket, holders and startup reconciliation | `keeper host status` |
| `curator session attach <session>` | attach in control mode; needs the operator's authority (§6.2) | `keeper`, `keeper chat` |
| `curator session watch <session>` | a read-only view | `keeper watch` |
| `curator session park <session>` | park (§8) | `keeper park`, `keeper down` |
| `curator session stop <session>` | the resumable stop; no goal is cleared | `keeper stop` |
| `curator session estop [<process>]` | the emergency stop (§9) | `keeper estop [<hive>]` |
| `curator session logs [-f]`, `doctor`, `restart [--stop-hosted]` | the host's log, its checks and its replacement (§2.7) | `keeper host logs\|doctor\|restart` |

- **The kernel keeps what writes board records:** the goal revisions with `handoff_ref` (§8), the snapshot `task-board board snapshot` (§9) and the run directive kind `park` (§8). No kernel command parks or stops a hosted session or the node.
- **Migration.** Phase 1 changes no command (§3.5). From the node host (track SH3) on:
  - `curator session …` is the host's command line, and `task-board session …` stays an alias of it for one release;
  - the four context-registry commands under `task-board session` (`context-completeness`, `context-economics`, `reclaim-contexts`, `sweep-contexts`) are not host operations, so they move to `task-board context` (`list`, `show`, `economics`, `gc`) instead of leaving with the host;
  - `task-board claude|codex` become aliases of `curator run claude_code|codex_cli` (aliases decided 2026-09-27; curator-spec Decision 0021 §5, adoption pending; environment operands clarified in Revision 2); `--resume <uuid>`, which adopts a foreign native session, becomes `--adopt <uuid>`; the detach hint prints the canonical command, and the `anthropic-board` and `openai-board` launchers retire.
- **Runtime ids.** Every command of the host names a harness by its runtime id (`claude`, `codex`, `muse`, `pi`, or a binding id), the same values `task-board spawn --runtime` takes. `curator run` takes the Curator environment id instead (`claude_code`, `codex_cli`, `pi`); both namespaces are canonical, and curator-spec Decision 0013 maps one to the other (the launcher's §4.2 table: `claude_code` → `claude`, `codex_cli` → `codex`, `pi` → `pi`). Every cross-repository contract names which namespace it uses.

## 13. Terminal backends and transport *(recorded for later; proposed)*

A terminal backend owns one terminal instance per hosted session:

- hosting the harness process;
- presenting its terminal;
- attaching and reconnecting;
- backend-local IPC;
- observing the process.

The terms follow `ax` §4.A, where a backend "owns exactly one host-local Terminal Instance" and never owns identity, ownership or goals.

| Backend | How it works | Survives a host restart | Attach | Platforms |
| --- | --- | --- | --- | --- |
| `pty` (today) | today the host creates the PTY and holds its master. In the node host (§5) a small per-session holder process holds it | today Codex yes, Claude no (§2.7); with per-session holders, yes | through the host's relay | macOS, Linux |
| `tmux` | the host starts the harness inside a tmux server on a dedicated per-user socket, as `ax` does with `tmux -S <runtime>/tmux/ax.sock` (§3.2). Input goes in through `load-buffer` and `paste-buffer -p`, which keeps bracketed paste. Output comes out through control mode (`tmux -C`) or `pipe-pane` | yes: the tmux server holds the PTY, and a restarted host re-adopts the panes | `tmux attach` for control, `tmux attach -r` for read-only | macOS, Linux, WSL2 |
| `ax` | the host delegates the session to Agent Session Manager | yes | local, remote and web attach; multi-attach, where extra clients get no input unless authorized; attach never changes ownership (`ax` §1.2 item 5, §4.C-§4.D) | Unix through tmux; Windows through ConPTY (`ax` §1.3) |

- The backend is chosen per node. `pty` stays the default until another backend passes a conformance suite built from task-board's session tests.
- `ax` is the long-term backend. Its specification (v0.7.0) already defines the Unix terminal backend, attach and continuation. Its implementation runs through these milestones (relux-works/agent-session-manager README):
  - M0, contract foundation (done);
  - M1, single-host durability;
  - M2, multi-host preview;
  - M3, daily-driver tmux;
  - M4, cloning, directory and platforms.

  We do not wait for it: the contract of §3.3 is shaped so that switching to `ax` is configuration, not a rewrite.
- `ax` is explicitly "not a replacement for task-board or `tb-sessiond`" (`ax` §1.2). It provides the terminal and ownership plane; the goal plane stays with task-board and this module.
- **Transport is an abstraction.** Remote attach and remote supervision run over a transport interface. A tailnet is one transport; another private network, or a relay with end-to-end encryption, can replace it. A transport identifies a machine. It never decides what that machine may do: that is the capability and trust layer (§6).

## 14. Restart survival

- **Today** *(fact)*. A Claude session ends when `tb-sessiond` exits, and resumes later from its durable identity: `--resume` with the session UUID. A Codex session is re-adopted through `thread/resume` (§2.2, §2.7).
- **Target** *(proposed)*. With per-session holders (§5), or the `tmux` or `ax` backend, compatible terminals can outlive unplanned daemon loss and be re-adopted, as today's startup does for Codex (§2.1). This supplies the survival mechanism sought by Decision 0021 open question 2. Requested controlled restarts use park and stop below. Replacement of shared artifacts requires the drain and stop conditions of §8.2, even with surviving holders.

- **Every resume is gated.** A new harness process passes the optimistic gate (§8.1). A surviving holder continues with its recorded live executable. Re-adoption checks that identity and protocol compatibility without invoking the current launcher. An unverifiable holder is reported and parked through the controlled-stop path. Enforce the durable update lock (§8.2) and any ordinary restart closure below before automatic re-adoption or a new start. A surviving holder is a live consumer and must stop before its shared artifacts are replaced. *proposed*
- **Controlled restart.** Before the park broadcast, close new admission and serialize that closure with starts, wakes, spawns, retries and consumer-starting probes. Durably retain this ordinary restart closure across daemon exit so the supervisor's replacement cannot admit a session before startup reconciliation. The daemon waits for park notes and `handoff_ref` revisions and satisfies the complete drain and confirmed-stop rules of §8.2. At the bound it records forced resumable stops and snapshot fallbacks. Unconfirmed stops or unresolved safe points keep the closure in place for recovery. The normal sequence then exits and the supervisor restarts it. Startup reconciles the intended consumers and park records. Once that reconciliation is complete, the ordinary closure ends automatically and eligible sessions restore through the login-shell bridge (§11), at any uptime. This closure is separate from the durable update lock: it never clears or bypasses that lock, which still needs the operator. An explicit operator stop remains a stop. With `keeper up`, each root orchestrator decides whether to restore its owner-run sessions. *decided in direction (2026-10-05); lifecycle details proposed*
- **Keep the rollback executable.** Retain the executable and whole required artifact root outside launcher pruning under §8.1's reference rules. Protect live and parked sessions, surviving holders, admitted launch or retry attempts, unresolved updates and active exact pins. Release live references only after confirmed consumer exit. The two-or-three-build target applies to otherwise unreferenced history. Report extra storage needs rather than deleting protected artifacts. Host and module replacement retains the pinned previous version; using it still requires safe current-state compatibility (§8.2). Retention beyond protected references is open (§15). *decided in direction (2026-10-05); cleanup rule proposed*

![Surviving a host restart](../diagrams/artefacts/plantuml/restart-survival.svg)

## 15. Open questions

1. **One host per node** (§5): does the operator adopt it, and in which milestone does the per-board daemon retire?
2. **Bridge shape.** How the `ax start --launch-plan` bridge is built: an `ax` shim, or the daemon answering `ax start` (Decision 0021 open question 1). How it retires when `ax` ships.
3. **Claude's `steer` and `interrupt`.** Verify the safe points and the interrupt key per harness version (§7). Decide which message classes may use `steer`. Decide whether the Claude adapter uses Claude Mods once their API is stable (§7.1).
4. **Signed supervision.** The main loop's certificate and the waggle `cmd` classes from CM2 (§6.3).
5. **Resumable suspension of runs.** Needed by emergency stop (§9).
6. **`handoff_ref`.** A field on the goal revision, or text in the objective. Who consumes it, and when it is cleared.
7. **Backend selection.** How a node chooses its terminal backend, the conformance suite, and Windows.
8. **Attach rights in detail.** How input is authorized on a shared terminal, tmux socket permissions, and remote attach through the `ax` mesh (§12, §13).
9. **Context telemetry.** Which fields each harness version exposes, and the thresholds for an early handoff (§10).
10. **Placement of the extracted code.** Which task-board packages move and which stay, and the repository's module path.
11. **The real boundary.** Running agents under an operating-system account separate from the operator's, and how the host launches them there. This is shared with the trust specification (§6.4).
12. **Emergency stop scope.** Whether a process is enough as the scope of `curator session estop`, or a board-level scope is also needed. How the companion app confirms an emergency stop from the phone.
13. **An emergency stop under a delegation.** How the main loop presents a delegation that names the trigger to `curator session estop` in place of the phrase and the person's presence (§9, §6.3).

14. **Probe and feed ownership.** Who publishes the known-bad feed and probe fixtures. How feed freshness and revocation are handled. Where strict and pinned harness policies are recorded (§8.1).
15. **Lock, record and retention.** The durable node-wide lock's storage and operator-clear command. The minimal update record's encoding and location. Cache retention beyond the protected references in §8.1 and §14.
16. **Update proposals.** How the main loop groups proposals from several root orchestrators. How the operator configures automatic updates during a planned restart window (§8.2, §11).
17. **Whole-cohort commit/rollback.** Whether host, module, package and profile updates need a shared transaction. This is deferred beyond the slim window (§8.2).
18. **Independent executor.** Whether replacement needs an executor that survives the host. Its ownership and recovery contract remain open (§8.2).
19. **Irreversible migrations.** Whether and how to admit installer or first-resume migrations that the pinned previous version cannot undo. The acknowledgement, state protection and recovery protocol remain open (§8.2).
20. **Evidence tables.** Whether to introduce component-specific evidence tables for harness, host, adapter/module, package and profile. Their encoding and publisher governance remain open. They are not admission requirements in the optimistic default (§8.1).
21. **`auto-verified`.** Whether a separate verified-only automatic update policy is useful as an opt-in. The optimistic default supersedes the earlier proposal to make verified targets the automatic path (§8.1, §8.2).

## 16. Decisions

| Date | Decision |
| --- | --- |
| 2026-09-24 | The session host becomes its own module in this private repository. Phase 1 extracts it as it is, with no behaviour change, and is a priority. |
| 2026-09-24 | Pluggable terminal backends (`pty`, `tmux`, `ax`) are recorded as the architecture; none is built in phase 1. |
| 2026-09-24 | `task-board session park` is adopted; an emergency park is to be considered. *Its name is superseded on 2026-09-27: `curator session park`.* |
| 2026-09-25 | The Apiary names map to technical terms as in "Names used in this specification"; specifications open with that map. |
| 2026-09-25 | Park writes its note into the board goal, so the next start picks it up with the goal. The emergency stop must be unmistakable. Its form, `task-board emergency-stop` / `estop` (§9), is the author's proposal. *The form is superseded on 2026-09-27: `keeper estop [<hive>]` over `curator session estop`.* |
| 2026-09-25 | Delivery must be autonomous and stable; the three modes of §7 are the author's proposal. Sessions must survive compaction; the handoff before compaction and the watchdog (§10) are designed now and built later. |
| 2026-09-25 | Closing access to the host (§6) is the first refactor after the as-is extraction. |
| 2026-09-25 | The person's commands are `keeper`, `keeper chat` and `keeper watch`; `attach` remains the host's operation underneath. |
| 2026-09-27 | The command-line shape is adopted (relux-works/wiki `research/cli-shaping-review.md`). The host's command line is the Curator provider `curator session …`, fronted by keeper; `task-board session` stays an alias for one release after the node host lands, and its context-registry commands move to `task-board context`. |
| 2026-09-27 | No kernel command parks or stops sessions or the node. `task-board session park` and `task-board emergency-stop` / `estop` are withdrawn. Park is `curator session park`, and a run parks by the kernel's directive kind `park`. The one emergency stop is `keeper estop [<hive>]` over `curator session estop`, which runs `task-board board snapshot` on each board. |
| 2026-09-27 | Beekeeping words appear only in `keeper`'s nouns, the apiary's directories and the profile name `apiary`; the host and the kernel use technical terms. |
| 2026-09-28 | The agents-infra exit campaign runs in two gates. Gate A (nothing needs agents-infra; children run in per-role Curator profiles) does not wait for the `curator run` → host bridge: primary and goal-bound sessions use the same construction as children. Gate B brings the first machine-local runtime binding and the M5a inference extraction that Gate A needs. |
| 2026-09-28 | Muse goes ahead of the host extraction, narrowly: now, a curated child environment and an update pin in the Muse plugin, and an unwired `muse serve` owner host with its MSP client; after primary and goal sessions run without infra, the wiring into task-board, the Muse PTY primary and the as-is extraction; no Muse producers until the environment fix is released and pinned, because a Muse child inherits the whole parent environment today. |
| 2026-10-05 | Harness versions use optimistic admission with probes, available vendor checksums, a known-bad feed and last-known-good fallback. Strict and pinned are opt-in. Muse follows the same default with compatibility probes and a fail-closed runtime (§8.1). |
| 2026-10-05 | The slim update window keeps a node-wide admission lock, park first, a minimal update record and rollback (§8.2). |
| 2026-10-05 | Resume does not persist the launch environment. A login-shell bridge runs `curator run <env> --hosted resume <SES>`. Missing required variables park with `required_env_missing` (§11). |

Recorded as a cross-reference, not as part of this specification *(the operators' evening call of 2026-09-29)*: remote workers start with an empty HOME; HOME and credentials move to Curator after the prototype; dynamic tool definitions stay stable over a worker's history; the orchestrator is model-agnostic.

## 17. Change log: v0 → v0.1

- **Added:**
  - the map of names;
  - §2.8, access control today;
  - Codex `turn/start`, `turn/steer` and `turn/interrupt`, and Claude's input queue during tool calls, in §2.5;
  - §5, one host per node with per-session holders;
  - §6, access security with capability tokens and the supervision matrix;
  - §7, the three doorbell modes and delivery to detached sessions;
  - §9, emergency stop as its own command;
  - §10, compaction and context;
  - §11, reboot restore through `keeper up`;
  - §12, the attach picker;
  - the transport abstraction in §13.
- **Changed:**
  - the requirements moved to §4 and gained R8 (recovery);
  - R7 now points to the node host;
  - park lost its emergency form, which became §9;
  - the doorbell policy question is answered by §7;
  - the open questions were renumbered and extended.
- **Diagrams:**
  - new: "One host per node with per-session holders" and "Emergency stop";
  - updated: "Doorbell delivery modes", "Park and resume" (the emergency part moved out) and "Surviving a host restart" (per-session holders).

## 18. Change log: v0.1 → v0.2

- **Added:**
  - §7.1, harness-specific mechanisms inside adapters, with Claude Mods as the example: reliable delivery, context telemetry and panels in the Claude adapter, never for authority, not a dependency now;
  - the order of work in §6: closing access is the first refactor after the as-is extraction (decided);
  - the trust specification's new home, relux-works/curator-trust.
- **Changed:**
  - the person's commands: `keeper`, `keeper chat` and `keeper watch` replace `keeper attach` in §0, R3 and §12; `attach` stays the host's operation;
  - §16 gains the two decisions of 2026-09-25;
  - open question 3 covers Claude Mods.
- **Diagrams:** "One host per node" names the new commands.

## 19. Change log: v0.2 → v0.3

Applied from the operator's adoption of the command-line shape on 2026-09-27:

- **Added:**
  - §12.1, the host's command line as the Curator provider `curator session …`, with keeper's front for each command and the migration from `task-board session`;
  - the run directive kind `park` (§8);
  - `curator session` in the terms (§1);
  - open question 13, an emergency stop under a delegation.
- **Changed:**
  - the naming rule: beekeeping words only in `keeper`'s nouns and the apiary's directories (Names);
  - §0: park and the emergency stop name their commands, and a new bullet names the command line;
  - §8: park starts with `curator session park`; `task-board session park` is withdrawn; the start after a park may be `curator run`;
  - §9: the one emergency stop is `keeper estop [<hive>]` over `curator session estop`, which asks for `EMERGENCY STOP` and the person's presence; `task-board emergency-stop` / `estop`, `--process` and `--confirm=STOP` are withdrawn; the snapshot is the kernel's `task-board board snapshot`;
  - §12: `keeper chat <hive>` and `keeper session list`;
  - §16 gains the decisions of 2026-09-27 and marks the superseded names;
  - open question 12 speaks of the scope of `curator session estop`.
- **Diagrams:** "Park and resume" and "Emergency stop" show the host's commands and the kernel's part; "One host per node" names the command line.

## 20. Change log: amendment — optimistic gate and slim update window

Re-cut from the operator's decisions and dispatcher review of 2026-10-05. Accepted direction is marked decided. Implementation mechanics remain proposed.

- **Changed:** §8.1 uses the optimistic default, vendor checksums where published, capability/resume probes and the known-bad feed. Strict and pinned harness policies are opt-in.
- **Changed:** run-from-cache preserves exact launch identity and last-known-good builds. Probe, launch and resume failures trigger automatic fallback with the vendor updater off. `harness_update_unverified` is limited to probe failure without a fallback or strict-policy refusal.
- **Changed:** Muse uses MSP schema compatibility and PTY grammar replay probes. The runtime remains fail-closed. Qualification supplies known-good signals and fixtures.
- **Changed:** §8.2 keeps a durable node-wide admission lock, park before replacement, a minimal update record and component rollback. The supervisor restarts the daemon. The operator clears the lock.
- **Preserved:** start only detects available updates. The login-shell bridge resumes without persisted environment (§11). Missing required variables are reported.
- **Revision 2:** isolated probes, serial fallback after confirmed cleanup, live artifact retention and state-compatible rollback bound the optimistic path (§8.1–§8.2). Unsupported state stays parked for recovery. Complete drain includes holders, detached consumers and Muse's safe points.
- **Revision 2:** restore waits for authentication, preserves the full plan permission posture and goal, and observes requested RC registration. Reports include host health. Login restore runs only after a fresh boot. Ordinary controlled restart closes admission through startup reconciliation (§11, §14). Acceptance keeps isolated homes and injected key/auth providers (§8.2).
- **Moved to open questions:** whole-cohort commit/rollback, an independent executor, irreversible migrations, component evidence tables and `auto-verified` (§15, items 17–21).
- **Removed:** network-calibration requirements. They belong to curator-network-profiles.
