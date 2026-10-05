# Track: session host

Status: **DRAFT v0.3**, 2026-09-27. Specification: [`session-host.md`](session-host.md). Board: relux-works/skill-project-management, epic EPIC-260924-3q1f1v (session-host-module), story STORY-260924-2jwvbm (sh-contract-and-claude-codex-extraction). Priority: the session-host module comes first in the operators' priority order of 2026-09-24 (relux-works/wiki `decisions.md`).

Names follow the map at the top of the specification: a process (hive) has one root orchestrator (queen), and the main loop (keeper) supervises them on a node (yard).

## Why

Today every long-lived orchestrator runs inside `tb-sessiond`. That daemon:

- is fused into task-board;
- holds Claude sessions in its own address space;
- rings only attached primary sessions;
- serves one board at a time;
- lets any agent of the operator's user control any session with one shared token.

The main loop needs to start and supervise the root orchestrator of every process on a node. People need to watch those orchestrators, and a reboot or an emergency must not lose the work. The first step is to make the host a module without changing what it does (the retained as-is alternative, pending A-Q1 in `session-host.md`). After that it can grow.

## Milestones

| # | Delivers | Done when |
| --- | --- | --- |
| SH0 | this specification, its diagrams and review | merged as a draft; the operator has answered the open questions that block SH1 |
| SH1 | **extraction as it is** (*decided, priority; under review by Architecture v1, disposition open, see A-Q1 in spec/session-host.md*). The provider-host plane and daemon skeleton move here behind the contract of spec §3.3; the Claude PTY and Codex app-server adapters come along; task-board reaches the module through a thin seam. The unchanged-suite obligation stands until the operator rescinds it | task-board's session, goal and notice suites pass unchanged against the module; goldens are identical; operator commands behave as before |
| SH2 | **what the main loop needs first:**<ul><li>access control first, the first refactor after the extraction (*decided 2026-09-25*): private socket directory, peer credentials, per-session capability tokens, the supervision matrix (spec §6)</li><li>the doorbell modes and delivery to detached sessions (§7)</li><li>park, `curator session park`, with `handoff_ref` on the goal revision, and the run directive kind `park` (§8)</li><li>the emergency stop, `curator session estop` behind `keeper estop`, with the kernel's `task-board board snapshot` (§9)</li><li>starting a root session in a registered process directory (R1)</li><li>the event stream (R6)</li></ul> | the main loop starts a root session in a second process directory, rings it while nobody is attached, parks it, and a new start resumes from the park note. An agent session cannot stop another process's session. `keeper estop` stops everything and `keeper up` brings it back |
| SH3 | **one host per node** with per-session `pty` holders (§5), one registry, the `keeper chat` picker and `keeper watch` (§12) with read-only views of sub-orchestrators (R3), and reboot restore through `keeper up` (§11); the command line becomes `curator session …`, `task-board session` an alias for one release, and the context-registry commands move to `task-board context` (§12.1) | compatible sessions survive unplanned daemon loss and are re-adopted, while requested restarts use controlled park/stop/restore with confirmed stops; one picker lists every session on the node; a reboot restores the main loop and every root orchestrator |
| SH4 | ~~the `tmux` terminal backend (§13)~~ (superseded 2026-10-05: never tmux; kept as dated history) | superseded; no tmux conformance is sought |
| SH5 | context and limit telemetry per adapter; the handoff before compaction (§10) | a session that compacts continues from its handoff; the watchdog receives context and limit events |
| SH6 | the `ax` backend, once `ax` ships its Unix terminal backend and task-board bridge; the `curator run` → host bridge (spec §3.4) | the host delegates a session to `ax` by configuration; attach and continuation come from `ax` |
| SH7 | further adapters: agy, Pi, Qwen | each passes the adapter conformance suite |

**Muse goes ahead of the extraction, narrowly** (*decided 2026-09-28*, EPIC-260927-5c4nxe), outside SH7:

- now: the `muse serve` owner host with its MSP client, built unwired;
- after TASK-260927-1okvcs (primary and goal sessions without infra): its wiring into task-board, the Muse PTY primary and the as-is extraction;
- no Muse producers until the environment fix is released and pinned, because a Muse child currently inherits the whole parent environment.

## Dependencies

- **task-board.** The seam; the goal ledger and a `handoff_ref` field on `BoardGoalRevision`; the `session` commands until SH3, then an alias; `task-board board snapshot` for the emergency stop; the run directive kind `park`; `task-board context` for the context registry; a resumable suspension of runs (spec open question 5).
- **Curator launcher.** `curator run`, and the `ax start --launch-plan` contract and its bridge (curator-spec Decisions 0019 and 0021). The bridge is a later dependency (SH6), not a prerequisite of SH1 or SH2: until it exists, primary and goal-bound sessions are built with the children's construction, a per-role Curator profile through agents-management (Gate A, wiki decisions.md 2026-09-28; spec §3.4).
- **Curator.** Provider installs, so that `curator-session` reaches a trust root: first-party providers beside `curator`, and `curator provider install` (curator-spec issue #100, curator issue #96).
- **waggle.** The delivery modes and their mapping (spec §7); signed `cmd` messages for supervision from CM2. waggle §10 needs an amendment, because it still lets messages wait while no client is attached.
- **The main loop** (relux-works/curator-keeper): the registry of processes, the same supervision matrix, `keeper up`, `keeper down`, `keeper chat`, `keeper watch` and `keeper estop`, and the watchdog.
- **The trust component** (relux-works/curator-trust `spec/trust.md`): presence-bound confirmation for operator commands, and running agents under a separate operating-system account.
- **Agent Session Manager.** `ax`, relux-works/agent-session-manager; spec v0.7.0 at relux-works/agent-session-manager-spec.

## Risks

- **Extraction drift.** A move that silently changes behaviour. Keep SH1 a pure move, with the existing suites as the gate.
- **Claude hosts end on replacement.** A Claude host dies when the daemon is replaced (spec §2.7). Until SH3, installs and upgrades happen between sessions, from an outside shell, or after a park.
- **Detached delivery.** Ringing a detached session starts turns nobody watches. The event stream must make those turns visible.
- **Same-user limits.** Capability tokens give least privilege among well-behaved agents, not a boundary against a hostile one (spec §6.4). The boundary comes with the trust component.
- **Coupling with `ax` timing.** Mitigated by keeping the contract shaped for `ax` while not waiting for it.

**Next (under review by Architecture v1).** Review SH0, then resolve the SH1 disposition (open question A-Q1 in `session-host.md`): hybrid (c), or an as-is compatibility capsule first. Do not assume the operator has rescinded the unchanged-suite obligation.
