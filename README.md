# agent-session-host

The session host keeps long-lived agent sessions running in the background. It starts a harness (Claude Code, Codex and later others) in a given directory with a given goal. A person can attach to a session and detach from it, messages can be delivered into a running session (the doorbell), and the host reports what happens to each session.

Today this job is done by `tb-sessiond`, the Session Manager inside task-board (relux-works/skill-project-management). This repository is where the host becomes its own module: first extracted as it is, then extended for the main loop that starts and supervises the orchestrator of every process.

Status: specification draft.

| Document | Holds |
| --- | --- |
| [`spec/session-host.md`](spec/session-host.md) | the specification: today's implementation, phase 1 extraction, the main loop's requirements, one host per node, access security, doorbell modes, park, emergency stop, compaction, reboot restore, the conversation and watch picker, terminal backends, restart survival |
| [`spec/track.md`](spec/track.md) | milestones SH0-SH7, dependencies, risks |
| [`spec/hosted-launch-contract.md`](spec/hosted-launch-contract.md) | the `curator run --hosted` contract: the launch-plan payload, the receiver, admission, terminal and completion protocol, daemon obligations, acceptance (agreed draft r6 plus the operator decisions of 2026-10-05) |
| [`docs/overview.ru.md`](docs/overview.ru.md) | a one-page overview in Russian: how the new session host works and where each topic lives |
| [`diagrams/`](diagrams/README.md) | diagrams as code (PlantUML) and their renders |
