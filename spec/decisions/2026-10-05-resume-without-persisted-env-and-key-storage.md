# Decision: resume without persisted environment; host key in a file (2026-10-05)

Status: **decided** by the operator on 2026-10-05, for the controlled restart (§8 Park, §11 Reboot restore, §14 Restart survival) and for the hosted-launch record commitments. This record exists because the operator asked for the reasons and the accepted risks to be written down.

## 1. No persisted launch environment

**Decision.** The host does not store the launch environment of a hosted session. A parked or interrupted session is resumed by the resume bridge. The bridge is a login agent today (launchd on macOS, a systemd user unit on Linux, a logon task on Windows later), and `keeper up` once keeper exists. It runs `curator run <env> --hosted resume <SES>` through a login shell. The launcher then:

1. **recomposes the plan** from the operator's profile and login environment;
2. **checks the profile pin and drift** as on every hosted launch;
3. **parks the session if a required variable is missing**, with `required_env_missing`, and lists it in the restore report.

**Why.**

- **No secret at rest.** Nothing that could hold a credential is written by the host for later replay.
- **No keystore dependency for restore.** A keystore prompt that nobody sees cannot block an unattended restore. This also keeps Muse's rule that Muse sessions never depend on Keychain automation.
- **Fresh credentials.** Rotated credentials are picked up on resume instead of stale copies being replayed.
- **Same trust path as a new launch.** The restored plan goes through the same admission and verification as any new hosted launch.

**Accepted cost.** Variables exported by hand in one terminal, and not set by the login shell's profile, do not come back after a restart. The session reports that instead of starting without them.

## 2. Host key in a file

**Decision.** The host's own secret keys sit in a file of mode 0600 in a separate directory excluded from backups (for example, `tmutil addexclusion` on macOS). Today these are the keyed commitments of owned literals in hosted records. The key is read through a pluggable key provider, so a keystore can replace the file without changing callers. No agent, test or tool reads the operator's real keystore. Tests use an injected provider.

**Why a file and not the macOS Keychain now.**

- **Prompts after every update.** Host binaries are ad-hoc signed today, so their code identity changes on every build. A Keychain item restricted to the host binary would prompt for the login password after every update, which is exactly when an unattended controlled restart runs. Allowing `/usr/bin/security` instead removes the prompt, but then any process of the same user can read the item, which is no stronger than a 0600 file.
- **Locked after a reboot.** The login keychain is locked until the user logs in to the GUI, and is unavailable over SSH-only sessions.
- **No common store.** Linux servers often have no Secret Service, and Windows needs DPAPI. A single file provider behaves the same everywhere.

**Accepted risks.** Any process running as the same user can read the key: an agent, a package install script or an MCP server. The same is already true of the harness credential files and of the environments of running processes, so the marginal exposure is small. Copies of the home directory can expose it if the exclusion is lost. With FileVault, the disk is encrypted at rest when the machine is off.

**Follow-up design work (not decided).**

- **Code signing and notarization.** A stable Developer ID signature and notarization for macOS builds, so that a keystore item can trust the host binary across updates. Signing for Linux builds as well.
- **Keystore providers.** The macOS Keychain once signing is stable; Linux Secret Service, or TPM-bound credentials (`systemd-creds`) on servers; Windows DPAPI. The file provider stays as the fallback.
