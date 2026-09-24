# agent-session-host

The session host keeps long-lived agent sessions running in the background. It starts a harness (Claude Code, Codex and later others) in a given directory with a given goal. A person can attach to a session and detach from it, messages can be delivered into a running session (the doorbell), and the host reports what happens to each session.

Today this job is done by `tb-sessiond`, the Session Manager inside task-board (relux-works/skill-project-management). This repository is where the host becomes its own module: first extracted as it is, then extended for the main loop that starts and supervises the orchestrator of every process.

Status: specification draft. See [`spec/session-host.md`](spec/session-host.md) and [`spec/track.md`](spec/track.md).
