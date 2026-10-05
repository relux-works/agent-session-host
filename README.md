# agent-session-host

The session host keeps long-lived agent sessions running in the background. It starts a harness (Claude Code, Codex and later others) in a given directory with a given goal. A person can attach to a session and detach from it, messages can be delivered into a running session (the doorbell), and the host reports what happens to each session.

Today this job is done by `tb-sessiond`, the Session Manager inside task-board (relux-works/skill-project-management). This repository is where the host becomes its own module. Architecture v1 proposes hybrid (c) instead of the as-is extraction; the SH1 disposition is open (A-Q1 in `spec/session-host.md`). The as-is text stays as reference.

Status: specification draft.

| Document | Holds |
| --- | --- |
| [`spec/session-host.md`](spec/session-host.md) | the specification: Architecture v1 (2026-10-05, for review) with the module map, assurance modes, injection, journal, key lifecycle, park/restart, version policy, adopted ideas and open questions; then today's implementation, phase 1 extraction, the main loop's requirements, one host per node, access security, doorbell modes, park, emergency stop, compaction, reboot restore, the conversation and watch picker, terminal backends, restart survival |
| [`spec/track.md`](spec/track.md) | milestones SH0-SH7, dependencies, risks |
| [`spec/hosted-launch-contract.md`](spec/hosted-launch-contract.md) | the `curator run --hosted` contract: the launch-plan payload, the receiver, admission, terminal and completion protocol, daemon obligations, acceptance (agreed draft r6 plus the operator decisions of 2026-10-05) |
| [`docs/overview.ru.md`](docs/overview.ru.md) | a one-page overview in Russian: how the new session host works and where each topic lives |
| [`diagrams/`](diagrams/README.md) | diagrams as code (PlantUML) and their renders, including the four Architecture v1 diagrams: module map, signed ingress, controlled restart, preserve versus rebuild |
| [PR #8 amendment text](https://github.com/relux-works/agent-session-host/blob/4d70825ba230f8f9063881efb2cf210bbfa8ee11/spec/session-host.md) (separate branch) | relux-works/agent-session-host, branch `amend/version-gate-update-window`, commit `4d70825ba230f8f9063881efb2cf210bbfa8ee11`, `spec/session-host.md` §§8.1, 8.2, 11, 14 and §15 questions 14–21: optimistic version gate, slim update window, resume bridge, controlled restart |
| [PR #9 decision record](https://github.com/relux-works/agent-session-host/blob/a6fd2ba639507796642a97d217878c0c6c5a4b90/spec/decisions/2026-10-05-resume-without-persisted-env-and-key-storage.md) (separate branch) | relux-works/agent-session-host, branch `decision/resume-env-and-key-storage`, commit `a6fd2ba639507796642a97d217878c0c6c5a4b90`, `spec/decisions/2026-10-05-resume-without-persisted-env-and-key-storage.md`: resume without persisted env, 0600 host key |
