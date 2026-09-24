# Track: session host

Status: **DRAFT v0**, 2026-09-24. Specification: [`session-host.md`](session-host.md). Board: relux-works/skill-project-management, epic EPIC-260924-3q1f1v (session-host-module) with story STORY-260924-2jwvbm (sh-contract-and-claude-codex-extraction). Priority: the session-host module is first in the operators' priority order of 2026-09-24 (relux-works/wiki `decisions.md`).

## Why

Every long-lived orchestrator runs inside `tb-sessiond` today. That daemon is fused into task-board, holds Claude sessions in its own address space, rings only attached primary sessions, and serves one board at a time. The main loop needs to start and supervise the root orchestrator of every process on a node, and people need to watch those orchestrators. The first step is to make the host a module without changing what it does. After that it can grow.

## Milestones

| # | Delivers | Done when |
| --- | --- | --- |
| SH0 | this specification, its diagrams, review | merged as a draft; the operator has answered the open questions that block SH1 |
| SH1 | **extraction as it is** (*decided, priority*): the provider-host plane and daemon skeleton move here behind the contract of spec §3.3; the Claude PTY and Codex app-server adapters come along; task-board reaches the module through a thin seam | task-board's session, goal and notice suites pass unchanged against the module; goldens identical; operator commands behave as before |
| SH2 | the main loop's requirements R1, R4, R5, R6 (spec §5): start in a registered process directory, doorbell for detached sessions, wake, park and stop from the main loop, an event stream; `task-board session park` and `park --now` (spec §6) with `handoff_ref` on the goal revision | the main loop starts a root session in a second process directory, rings it while nobody is attached, parks it, and a new start resumes from the park note |
| SH3 | attach to owner-run sub-orchestrators in control and read-only modes (R3) | a person watches a sub-orchestrator live without disturbing it |
| SH4 | the `tmux` terminal backend; restart survival; upgrade without hang-up | a host restart leaves Claude and Codex sessions running and re-adopted; `tmux attach -r` shows a session read-only |
| SH5 | the `ax` backend, once `ax` ships its Unix terminal backend and task-board bridge | the host delegates a session to `ax` by configuration; attach and continuation come from `ax` |
| SH6 | further adapters: Muse, agy, Pi, Qwen | each passes the adapter conformance suite |

## Dependencies

- **task-board.** The seam, the goal ledger and a `handoff_ref` field on `BoardGoalRevision` for park; the `session` commands.
- **Curator launcher.** `curator run`, and the `ax start --launch-plan` contract and its bridge (curator-spec Decisions 0019 and 0021).
- **waggle.** The doorbell's message classes and delivery modes; signed `cmd` messages for supervision from CM2.
- **The main loop.** The main-loop specification (planned repository relux-works/curator-keeper): which processes exist, and who may wake, park and stop them.
- **Agent Session Manager.** `ax`, relux-works/agent-session-manager; spec v0.7.0 at relux-works/agent-session-manager-spec.

## Risks

- **Extraction drift.** A move that silently changes behaviour. Keep SH1 a pure move, with the existing suites as the gate.
- **Claude hosts end on replacement.** A Claude host dies when the daemon is replaced (spec §2.7). Until SH4, installs and upgrades happen between sessions, from an outside shell, or after a park.
- **Detached delivery.** Ringing a detached session (R4) starts turns nobody watches. The delivery policy and the events stream must make those turns visible.
- **Coupling with `ax` timing.** Mitigated by keeping the contract shaped for `ax` while not waiting for it.

**Next.** Review SH0, then start SH1 on the board story above.
