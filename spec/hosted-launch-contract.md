# Hosted launch contract (`curator run --hosted`)

Status: **agreed draft r6** (2026-10-04), the contract between curator-agent-launcher, the private task-board receiver and the session host. Review history: six R141 rounds plus owner reviews (infra-exit, critical-fixes, tb-muse). Filed here so the session-host module carries it.

## Operator decisions after r6 (2026-10-05)

These supersede the matching OPEN items in the text below. A later revision (r7) folds them in.

- **Q-D1a, the operator hosted default:** accepted. `host: hosted` in the operator's curator-run-defaults is honoured as the operator's own opt-in. Machine defaults now wait for the controlled restart (park + resume) rather than a no-hangup upgrade, per the operator's direction of 2026-10-05.
- **Q-D2c, Codex parity by result:** accepted. Hosted Codex is judged by result (model, effort, permissions, MCP, profile env, network). Only the enumerated transport differences are allowed.
- **Q-D3, permissions:** the yolo/native default is configurable, and with no configuration the default is yolo, for hosted launches too. An explicit flag wins. The tracked-yolo refusal is removed for the hosted path.
- **Harness versions:** optimistic by default (capability/resume probes, provider checksums where published, a known-bad feed, the last-known-good cache with run-from-cache and automatic fallback). Strict and pinned are opt-in, and Muse is not an exception. The exact-build seal binds the executed binary, not an allow-list.
- **Resume without persisted env and the host key in a 0600 file:** see `spec/decisions/2026-10-05-resume-without-persisted-env-and-key-storage.md` (PR #9).

---


Draft r6, 2026-10-04. MUST/SHOULD specify requirements; Ivan’s D1–D4/R-xx bind. Admission gates remain explicit.

## 1. Scope, phases and defaults gate

Board-primary sessions MUST retain ownership/goals/notices/detach/resume/adoption. Boardless launches refuse `session_host_scope_unsupported`; fake ax/alias recursion/subagent changes/headless launcher are excluded.

| Phase | Admission gate / parity |
| --- | --- |
| 1: Claude | Explicit managed-host capability; null bundle; composed CLAUDE_CONFIG_DIR; exact visible-child parity except the two D2 slots below |
| 2: Muse | Landed R2, verified-plan import, generic seal API, Muse sealer and shared release list; exact parity (§5.2) |
| 3: Codex | Tagged module-owned app-server/client/thread/restart schemas and plans; Ivan’s Q-D2c approval |

Phase 1 stays disabled until ALL gates hold: TASK-261004-1kc4kk and TASK-261004-26qvc0 tagged; CF-Q1 decided and auxiliary projection TASK-261004-3u485d tagged; CF-Q2 decided and implemented; diagnostics registry published/pinned; TASK-261004-38ba6c, TASK-260927-1okvcs and D4 CR1–5 accepted; closed durable/capability schemas and parity evidence available. Ivan’s OPEN questions do not gate explicit native-permission Phase 1.

**Ivan NOTE under D2 (V-SES/W-UUID).** Claude Phase 1 declares exactly TWO module-owned typed slots: env `TASK_BOARD_MANAGED_SESSION_ID=<SES>` and primary argv `--session-id <UUID>` for NEW launches. Native `curator run` carries neither. Parity equals native except exactly these slots, finalized by the module. Restart/adoption retain the declared `--resume <UUID>` identity slot. Both new slots remain CF-Q2 OWNER-OPEN.

Unsupported phases refuse `session_host_provider_unsupported` before contact.

**Q-D1a — OPEN for Ivan.** Proposed: operator `host: hosted` counts as explicit personal opt-in; machine defaults/locks remain gated. This requires amending S/decisions/0021-sessions-enter-through-curator-run.md:138. Until Ivan rules, every selected configured hosted default MUST refuse `session_host_default_not_ready`; explicit hosted flags remain opt-in. Default enablement MUST require upgrade-without-hangup, conformance and resume fidelity.

**Q-D2c — OPEN for Ivan.** Codex’s remote client cannot equal ordinary native TUI argv (B/cmd/codex_manager.go:418). Proposed parity compares corresponding module-owned variants for model, effort, permissions, MCP, profile env and full final env. Allowed transport differences MUST be limited to app-server/listen endpoint, client `--remote ENDPOINT resume ID` and thread-identity binding. No extra policy/home/env differences are permitted. Until approved and the bundle lands, Codex hosting MUST refuse.

**Q-D3 — OPEN for Ivan.** Muse Phase 2 needs a ruling: (a) permit hosted yolo explicitly selected by operator flag/profile, with native confirmation semantics; or (b) defer. Native `curator run muse --yolo` works today (A/pkg/agentic/systems/muse/policy.go:49); tracked yolo refuses (L/internal/execution/permission.go:88). Until Ivan rules, hosted yolo MUST refuse `permission_mode_tracked_unsupported` for every provider. Approval also requires a new admitted permission schema and phase conformance.

## 2. Launcher

Syntax: `curator run <env> [--hosted|--native|--untracked] [resume [SES-HANDLE]] [--resume ID] [-- NATIVE_ARGS...]`. Host flags share one occurrence slot; repeats/combinations refuse `usage`. `--untracked` equals `--native`; permissions remain independent.

Closed `curator-run-defaults-v3` MUST add per-environment `host: native|hosted`; v1/v2 remain accepted unchanged. Example: `{"schema":"curator-run-defaults-v3","locked":false,"defaults":{"claude_code":{"host":"native"}}}`. Keep `/etc/curator-run/defaults.json` and `$XDG_CONFIG_HOME/curator-run/defaults.json` (L/internal/defaults/defaults.go:50). Validate both files, including ignored entries, under existing file-security rules; invalid/unreadable/unknown members refuse `defaults_config_invalid`.

Launcher SPEC MUST use this precedence:

| First match | Result |
| --- | --- |
| Machine locked and names environment | Ignore entire operator entry; a flag with machine host present refuses usage, even equal; otherwise flag → machine host → native |
| Flag present | Flag |
| Operator host present | Operator |
| Machine host present | Machine |
| Otherwise | Native |

Named-environment/per-member locks follow L/internal/defaults/defaults.go:178. No fragment/runtime host selector is allowed.

After normalization/config validation, resolve host, then apply this legacy ax table, then defaults gate, then D3, before permissions/build:

| Effective ax.json | Host selection | Routing |
| --- | --- | --- |
| Absent/disabled | Any | Host table |
| Enabled | Implicit native, absent flag/default | Existing ax behavior |
| Enabled | Configured native | Existing ax behavior; native default does not imply explicit bypass |
| Enabled | Explicit native/untracked | Existing untracked native path |
| Enabled | Hosted flag/default | host_configuration_conflict/2, before defaults gate |

ax remains machine-first (L/internal/axconfig/config.go:22). Without host flag/default, native/legacy ax MUST perform zero receiver lookup/daemon probe. No fallback.

D3 MUST refuse every managed hosted network selection, including direct/default/inherited, with `network_scope_unsupported` before permission/build or probes. Hosted is tracked: preserve native permission defaults and `permission_mode_tracked_unsupported` for yolo (L/internal/execution/permission.go:76,88). Parity controls MUST use the same resolved policy, explicitly native when necessary.

Before contact, MUST reject unsupported interactive/managed-host capability, execution profile, board scope, terminal and invalid plan. Interactive support alone is insufficient. Launcher MUST call `agentic.ElevateResumeIntent` (A/pkg/agentic/resume.go:49); plugins elevate Claude `--resume UUID`/`-r UUID`, Codex `resume THREAD`/`--resume THREAD`, Muse `resume ID`, and wrapper selectors into typed intent using their arity grammar; conflicting/ambiguous/picker identities refuse `session_resume_invalid`. Receiver MUST NOT parse provider selectors. Other native tail tokens MUST retain bytes/order (L/internal/cli/cli.go:193). Native mode keeps provider grammar; SES handles are hosted-only.

## 3. Private payload 1.0.0

Private Draft 2020-12, separate from ax; locally pinned references. Freeze 1.0.0; publish/pin/negotiate new shapes under new versions.

```json
{
 "$schema":"https://json-schema.org/draft/2020-12/schema",
 "$id":"urn:relux:task-board:session-launch-plan",
 "type":"object","additionalProperties":false,
 "required":["schema","schema_version","content_digest","producer","environment","process","owned_literals","env_names","managed_home","fragment","policy","session_name","remote_control","resume","restart","host_bundle","network"],
 "properties":{
  "schema":{"const":"urn:relux:task-board:session-launch-plan"},
  "schema_version":{"const":"1.0.0"},
  "content_digest":{"$ref":"#/$defs/digest"},
  "producer":{"type":"object","additionalProperties":false,"required":["module_version","module_commit"],"properties":{"module_version":{"type":"string","pattern":"^[0-9]+\\.[0-9]+\\.[0-9]+$","maxLength":32},"module_commit":{"type":"string","pattern":"^[0-9a-f]{40}$"}}},
  "environment":{"const":"claude_code"},
  "process":{"$ref":"#/$defs/process"},
  "owned_literals":{"type":"object","maxProperties":64,"propertyNames":{"$ref":"#/$defs/envName"},"additionalProperties":{"type":"string"}},
  "env_names":{"type":"array","maxItems":64,"uniqueItems":true,"items":{"$ref":"#/$defs/envName"}},
  "managed_home":{"type":"object","additionalProperties":false,"required":["variable","path"],"properties":{"variable":{"const":"CLAUDE_CONFIG_DIR"},"path":{"type":"string"}}},
  "fragment":{"type":"object","additionalProperties":false,"required":["profile_name","pin","fragment_digest","system_modules"],"properties":{"profile_name":{"$ref":"https://relux-works.github.io/curator-spec/schemas/v1/common.schema.json#/$defs/identifier"},"pin":{"$ref":"#/$defs/digest"},"fragment_digest":{"$ref":"#/$defs/digest"},"system_modules":{"type":"boolean"}}},
  "policy":{"type":"object","additionalProperties":false,"required":["permission_mode","permission_source","execution_profile","effective_native_policy"],"properties":{"permission_mode":{"const":"native"},"permission_source":{"enum":["flag","profile","global","default-headless","default-interactive"]},"execution_profile":{"enum":["standard","yolo"]},"effective_native_policy":{"anyOf":[{"type":"null"},{"$ref":"#/$defs/effectivePolicy"}]}}},
  "session_name":{"type":"object","additionalProperties":false,"required":["host","native"],"properties":{"host":{"type":"string","pattern":"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"},"native":{"type":["string","null"]}}},
  "remote_control":{"type":"object","additionalProperties":false,"required":["enabled","argv_indices"],"properties":{"enabled":{"type":"boolean"},"argv_indices":{"type":"array","uniqueItems":true,"items":{"type":"integer","minimum":0}}}},
  "resume":{"type":"object","additionalProperties":false,"required":["kind","identity"],"properties":{"kind":{"enum":["new","latest","handle","claude_uuid"]},"identity":{"type":["string","null"]}}},
  "restart":{"type":"object","additionalProperties":false,"required":["schema","schema_version","data"],"properties":{"schema":{"const":"urn:relux:agents-management:claude-restart"},"schema_version":{"const":"1.0.0"},"data":{"type":"object","additionalProperties":false,"required":["argv","identity_slot"],"properties":{"argv":{"$ref":"#/$defs/strings"},"identity_slot":{"type":"object","additionalProperties":false,"required":["index","flag"],"properties":{"index":{"type":"integer","minimum":0},"flag":{"const":"--resume"}}}}}}},
  "host_bundle":{"type":"null"},"network":{"type":"null"}
 },
 "$defs":{
  "digest":{"type":"string","pattern":"^sha256:[0-9a-f]{64}$"},
  "envName":{"type":"string","pattern":"^[A-Za-z_][A-Za-z0-9_]{0,127}$"},
  "strings":{"type":"array","items":{"type":"string"}},
  "effectivePolicy":{"type":"object","additionalProperties":false,"required":["schema","schema_version","data"],"properties":{"schema":{"const":"urn:relux:agents-management:claude-effective-policy"},"schema_version":{"const":"1.0.0"},"data":{"type":"object","additionalProperties":false,"required":["relaxations","source_paths"],"properties":{"relaxations":{"type":"array","minItems":1,"maxItems":3,"uniqueItems":true,"items":{"enum":["permissions.defaultMode","permissions.allow","permissions.additionalDirectories"]}},"source_paths":{"type":"array","minItems":1,"maxItems":64,"uniqueItems":true,"items":{"type":"string","pattern":"^/","maxLength":4096}}}}}},
  "guard":{"type":"object","additionalProperties":false,"required":["schema","schema_version","data"],"properties":{"schema":{"const":"urn:relux:agents-management:exec-guard"},"schema_version":{"const":"1.0.0"},"data":{"type":"object","additionalProperties":false,"required":["kind"],"properties":{"kind":{"const":"unsealed"}}}}},
  "process":{"type":"object","additionalProperties":false,"required":["binary","argv","env","cwd","stdin","exec_guard"],"properties":{"binary":{"type":"string"},"argv":{"$ref":"#/$defs/strings"},"env":{"$ref":"#/$defs/strings"},"cwd":{"type":"string"},"exec_guard":{"$ref":"#/$defs/guard"},"stdin":{"anyOf":[{"type":"null"},{"type":"object","additionalProperties":false,"required":["encoding","bytes"],"properties":{"encoding":{"enum":["utf-8","base64url"]},"bytes":{"type":"string"}}}]}}}
 }
}
```

Effective policy carries only sorted source-validated detected selectors/inspected absolute paths; no permission values/uninspected-clean claims. Null means no detected relaxation; unknown schemas refuse protocol support. Phase 2 MUST extend environment with muse, managed_home with null, and resume with `muse_session`; Phase 3 adds `codex_thread` and a required bundle. Network extension is gated by §7. Unsupported nested schema/version dispatch precedes structural validation (§4).

`binary`, cwd and home MUST be absolute; argv excludes argv[0]. Home MUST equal the final env entry. New/latest require null identity; handle/UUID/provider IDs require module-validated identities. Host name defaults to environment/UTC spelling snapshotted once (L/internal/execution/execution.go:98); native name/RC indices MUST match module-parsed intent, including settings-origin RC with empty indices. Native `-n NAME` MUST reach both composed paths; host/native names remain separate.

`remote_control.argv_indices` are zero-based positions in the FINAL primary `process.argv`, excluding the binary; they are never restart-template coordinates or auxiliary indices. Each position MUST be in bounds and identify exactly the module-parsed RC tokens, including any parsed prefix/name tokens; non-RC tokens MUST NOT be indexed. Settings-origin RC may be enabled with an empty array. A null-reservation input is an unbound projection: validate its derived indices against its supplied primary argv, then derive submission indices against the finalized argv under §3.3. Only finalized positions are authoritative at submission.

After selector elevation, launcher MUST use `claude.System.ExportRestartTemplate` and `(*claude.System).ValidateRestartTransformation` (A/pkg/agentic/systems/claude/restart.go:39,51). Export places the slot at index 0. Its argv excludes original selectors; the identity slot declares insertion of exactly flag+selected UUID at index ≤ argv length. Frozen 1.0.0 new argv MUST equal restart argv; adopted/restarted argv MUST equal that template with only the declared insertion. Proposed 1.1.0 NEW argv adds only §3.3’s declared new-session insertion; this requires a tagged module validator extension, never relaxing frozen restart validation. For handle/latest, daemon resolves the stored UUID and binds that slot only on stopped restart. The unsealed adapter MUST validate transformation/native boundaries without recomposition. Renew binary/env/cwd; stdin=null. Persist only validated credential-free references. Phase 2 MUST publish its equivalent Muse identity-slot schema before admission.

Process MUST be the final PlanWithEnvironment projection, composed once (A/pkg/agentic/plan.go:350). Full ordered `NAME=value` env preserves filtering and caller credentials; `[]` means empty, never inheritance. Resolve sorted unique env_names in the original caller env, preserving absent versus empty; missing requirements refuse `launch_plan_invalid`, reason `required_env_missing`. Names MUST be disjoint from owned_literals, whose values MUST equal final env. Reject duplicate names, malformed entries and NULs. Never substitute daemon env.

Overlay order MUST remain module/owned snapshot → fragment env → prompt env → native tail → future network patch (L/internal/composition/composition.go:75). Host MUST consume final snapshots without another overlay or home/MCP rewrite. This full-env/caller-lookup contract MUST receive a private-contract amendment to S/decisions/0013-execution-ownership-and-launch-plans.md:193 and its inherited-env exclusion; it is not ax conformance.

CCJ-1 (S/protocol/registry.md:8) MUST govern canonical bytes: scalar-sorted keys, no whitespace, ordered arrays, prescribed UTF-8 escapes and shortest integers. Reject duplicate keys, invalid UTF-8, lone surrogates, fractional numbers, negative zero and integers outside ±9007199254740991; no top-level sig. content_digest = SHA-256 of the canonical object excluding only content_digest; receiver and daemon recompute. It MUST remain transient integrity, never authentication/durable provenance/resume identity. Fragment digest retains separate CCJ-1 identity.

Limits MUST apply across nested plans: wire/canonical object ≤1 MiB; depth ≤16; each argv including binary ≤128 elements, each ≤64 KiB, encoded array ≤256 KiB; env ≤512 entries/256 KiB, entry ≤16 KiB; owned values ≤4,096 bytes; versioned data ≤64 KiB; decoded stdin ≤65,536 bytes. Inline `--append-system-prompt` and `--mcp-config` JSON are supported within limits. UTF-8/base64url stdin MUST decode strictly; unpadded RFC4648 base64url only. Null means terminal stdin; present empty differs. Phase 1 MUST refuse any attached stdin with `session_host_stdin_unsupported`; future support requires true pipe EOF (§4).

### 3.1 Field classification and durable record

| Class (exhaustive) | Rule |
| --- | --- |
| process.env, owned_literals values, stdin bytes, and those fields in future nested processes | Ephemeral only; credentials may occur only here. Inherited credentials permitted; owned literals/stdin retain Decision 0013 exclusions |
| exec_guard/bundle data | Secret-bearing members forbidden outside the preceding ephemeral fields; never persist raw data or its full digest |
| content_digest, reservation.token (1.1.0) | Ephemeral, secret-dependent/control credential; never persist or log |
| Remaining fields, including argv/paths/names/identities/schema metadata | Credential-free/source-validated; secret-policy validation before allocation |

V-PREC: Known-schema shape/pattern failures MUST yield launch_plan_invalid/2 first. Structurally valid metadata failing admitted-source equality or credential policy MUST yield secret_policy_violation/16, with pinned field/reason only. Module identity, permission_source and profile name MUST match admission/resolution; profile uses S/schemas/v1/launch-env-fragment-v3.schema.json:35. Diagnostics MUST omit values/request bodies.

Values MAY travel only through anonymous launcher→receiver stdin/authenticated Unix socket; MUST NOT enter disk/argv/diagnostics/manifests/logs. Launch DTOs MUST never be serialized into ProviderHostRecord.Opaque; exclude legacy whole-plan snapshots (H/claude_host.go:845,1140) from this ingress.

The new durable record MUST have a hosted discriminator and recursively closed, pinned schemas. Its only members are record schema/version, SES/provider/native identity, owner/board fingerprint/canonical project, lifecycle/timestamps, host/native names/RC, admitted module/guard-schema versions, fragment, resolved policy, safe restart-template reference, env_names/owned-literal names, home_selectors, sanitized network Record, record_revision, execution_counter, current_generations, executions, completion, provider_host, provenance_digest and keyed literal commitments. ProviderHostRecord MUST contain only provider and Opaque. Hosted Opaque MUST contain only schema/version, provider/native identity, PID/process role/daemon incarnation/launch_generation/kernel start time/process group, safe restart reference and a credential-free reconnect reference. Each nested reference requires a closed provider schema; arbitrary JSON/string blobs and credential-bearing URLs are forbidden. Missing schemas refuse admission. Only §5.1’s closed non-secret home_selectors map MAY persist env selector values. No full plan, other env values, stdin, raw guard/bundle, wire digest or control credential may enter records, snapshots, materialization copies or logs. Restart references omit original selectors/one-shot stdin.

Persist `provenance_digest = SHA256("relux.hosted.provenance.v1\0" || CCJ1(P))`, where CLOSED P contains exactly provider, board fingerprint, project, admitted producer, fragment, policy metadata, guard schema/version, safe restart-template reference and env_names/owned-literal names. P MUST exclude every env/literal value, stdin, wire digest, commitments and raw guard/bundle data. Required literal equality MUST use HMAC-SHA256 over `"relux.hosted.literal.v1\0" || name || "\0" || value`, keyed outside records/logs; unavailable keys refuse renewal. Never store bare credential/value hashes.

### 3.2 Frozen Phase 1 1.0.0 example

Synthetic identities/pins; valid frozen 1.0.0 cannot satisfy r5 operational admission without §3.3.

```json
{
 "schema":"urn:relux:task-board:session-launch-plan",
 "schema_version":"1.0.0",
 "content_digest":"sha256:2b4cabe4a8cf3d7bbfe30ebe1ebd502fe017f5e3f31c2204597891872cbe4cc5",
 "producer":{"module_version":"0.5.48","module_commit":"1111111111111111111111111111111111111111"},
 "environment":"claude_code",
 "process":{"binary":"/synthetic/bin/claude","argv":["--model","example-model","--disallowedTools=AskUserQuestion"],"env":["CLAUDE_CONFIG_DIR=/synthetic/profiles/demo/claude","HOME=/synthetic/operator","PATH=/synthetic/bin","TERM=xterm-256color"],"cwd":"/synthetic/project","stdin":null,"exec_guard":{"schema":"urn:relux:agents-management:exec-guard","schema_version":"1.0.0","data":{"kind":"unsealed"}}},
 "owned_literals":{"CLAUDE_CONFIG_DIR":"/synthetic/profiles/demo/claude"},
 "env_names":[],
 "managed_home":{"variable":"CLAUDE_CONFIG_DIR","path":"/synthetic/profiles/demo/claude"},
 "fragment":{"profile_name":"demo","pin":"sha256:2222222222222222222222222222222222222222222222222222222222222222","fragment_digest":"sha256:3333333333333333333333333333333333333333333333333333333333333333","system_modules":false},
 "policy":{"permission_mode":"native","permission_source":"flag","execution_profile":"standard","effective_native_policy":null},
 "session_name":{"host":"claude_code-20261004T040000Z","native":null},
 "remote_control":{"enabled":false,"argv_indices":[]},
 "resume":{"kind":"new","identity":null},
 "restart":{"schema":"urn:relux:agents-management:claude-restart","schema_version":"1.0.0","data":{"argv":["--model","example-model","--disallowedTools=AskUserQuestion"],"identity_slot":{"index":0,"flag":"--resume"}}},
 "host_bundle":null,
 "network":null
}
```

### 3.3 Proposed payload 1.1.0 — OWNER-OPEN (CF-Q1, CF-Q2)

Copy §3; replace schema_version; require/add auxiliary/reservation/new_session_slot below. Preserve remaining properties/$defs; no allOf frozen 1.0.0. Publish same $id, 1.1.0.

```json
{
 "schema_version":{"const":"1.1.0"},
 "auxiliary":{"type":"array","maxItems":3,"items":{"type":"object","additionalProperties":false,"required":["role","mandatory","process"],"properties":{"role":{"enum":["version-probe","goal-probe","context-assay"]},"mandatory":{"type":"boolean"},"process":{"$ref":"#/$defs/process"}}}},
 "reservation":{"anyOf":[{"type":"null"},{"type":"object","additionalProperties":false,"required":["session_id","native_uuid","launch_generation","token"],"properties":{"session_id":{"type":"string","pattern":"^SES-[A-Za-z0-9][A-Za-z0-9_-]{0,127}$"},"native_uuid":{"type":"string","pattern":"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"},"launch_generation":{"type":"integer","minimum":1,"maximum":9007199254740991},"token":{"type":"string","pattern":"^[A-Za-z0-9_-]{43}$"}}}]},
 "new_session_slot":{"type":"object","additionalProperties":false,"required":["index","flag"],"properties":{"index":{"const":0},"flag":{"const":"--session-id"}}}
}
```

**V-AUX — OWNER-OPEN (CF-Q1).** Critical-fixes owns Q1; module/infra-exit owns TASK-261004-3u485d. Module MUST project unique-role auxiliaries from primary’s admitted inputs/binary/cwd/full env/home, each with guard/late verification. Claude MUST include mandatory version-probe and goal-probe; context-assay is optional unless context authorization requires it. Missing mandatory projection/support MUST refuse session_host_capability_missing/6 before allocation. Daemon MUST never author auxiliary argv/env or reuse legacy probes (H/claude_host.go:2364,2391,2416).

**V-SES/W-RES/W-UUID — OWNER-OPEN (CF-Q2).** Critical-fixes/session owner and module/infra-exit MUST decide/implement reservation and BOTH typed slots; new-session argv support extends TASK-261004-zwslo3. Authenticated `POST /v1/hosted/reserve` MUST use closed `{board,owner,provider,capabilities,resume,selection}`. board={fingerprint,project}, owner=primary, provider=claude; capabilities are negotiated names/versions; resume is §3’s module-normalized kind+identity. Closed selection={provenance,home_selectors,native_location}: provenance is §3.1’s CLOSED P; home_selectors is §5.1’s map; native_location is null except adoption, then closed {store,project}, absolute source-validated paths from supplied home/cwd. No credentials enter reservation. Pin schemas before admission; record_revision increments on every selection-relevant mutation.

Under owner serialization, daemon MUST apply §5.1 selection/authorization, resolving historical handles/latest/native identity BEFORE returning closed `{session_id,native_uuid,launch_generation,token,expires_at}`. Fresh Claude selection generates UUID before spawn. Token is 256-bit unpadded base64url, expiry ≤60 seconds; bound to authenticated caller, board/owner/provider, intent, selection, SES/UUID, selected record_revision (null for fresh), incumbent observation, proposed/current launch_generation and daemon incarnation. Closed incumbent observation is null or {session_id,record_revision,provider,state,liveness,attachable,birth_identity}; birth_identity uses V-PROC. Latest additionally binds candidate-set recency/revisions. Bounded per-owner reservations allocate no process/terminal/lease/durable session.

| Intent / selection | Reserved outcome |
| --- | --- |
| new/plain, free/proven-dead slot | Fresh SES + fresh UUID + proposed next primary generation |
| new/plain, same attachable live incumbent | Saved SES/UUID/current generation; reuse, no Start |
| latest | §5.1 exact historical candidate, even when different from incumbent; saved SES/UUID; live reuse or stopped restart |
| handle | Exact selected saved SES/UUID; live reuse or stopped restart; missing refuses not_found |
| claude_uuid, first adoption | Validated native store/project; fresh SES + supplied UUID + next generation; `--resume` |
| claude_uuid, re-adoption | Saved identity/location/provenance; saved SES/UUID; incumbent refusals apply |

Receiver MUST ask MODULE to finalize immutable projections: exactly one reserved SES env marker per primary/auxiliary, replacing inherited markers; exclude that name from env_names/owned_literals. Primary NEW adds exactly `--session-id <reserved UUID>` at new_session_slot.index; restart/adoption insert `--resume <reserved UUID>` at restart.identity_slot, never both. Auxiliaries retain their module role argv, sharing SES env without the primary UUID slot. Reuse creates no new process. Module MUST deliver imported-process finalization/admissible guard export; current FinalizePlan rejects bindings.

When performing the declared two-token primary identity insertion at index k, the MODULE finalizer MUST update only the derived `remote_control.argv_indices` array among admitted metadata: each original position i becomes i when i<k, otherwise i+2. Preserve RC enablement, parsed RC prefix/name/settings intent, session names, policy and all other admitted metadata. Settings-origin empty indices stay empty; reuse without insertion leaves indices unchanged. This permits neither token edits/reordering nor receiver/daemon-authored metadata. Receiver and daemon MUST independently validate final positions and module-parsed intent against final primary argv, not merely trust the shifted array. Out-of-bounds, omitted/extra/non-RC positions or stale template positions refuse launch_plan_invalid/2 before Start.

Submit finalized 1.1.0/reservation/recomputed digest with admitted metadata unchanged except that derived index update. Under owner serialization daemon MUST reauthorize/revalidate every bound observation/intent/slot before atomic consumption/allocation. Changed selection/incumbent/revision/generation → session_reservation_stale/1, no allocations; receiver MAY re-reserve/finalize once, never silently retarget. Expiry/replay/token or slot mismatch → launch_plan_invalid/2. Null reservation is pre-reservation input only. Identity-delivery alternative remains unapproved.

### 3.4 Synthetic finalized 1.1.0 example

Synthetic auxiliaries/reservation/both bindings; unsealed guards require new module support.

```json
{
 "schema":"urn:relux:task-board:session-launch-plan",
 "schema_version":"1.1.0",
 "producer":{"module_version":"0.5.48","module_commit":"1111111111111111111111111111111111111111"},
 "environment":"claude_code",
 "process":{"binary":"/synthetic/bin/claude","argv":["--session-id","44444444-4444-4444-8444-444444444444","--model","example-model","--disallowedTools=AskUserQuestion"],"env":["CLAUDE_CONFIG_DIR=/synthetic/profiles/demo/claude","HOME=/synthetic/operator","PATH=/synthetic/bin","TERM=xterm-256color","TASK_BOARD_MANAGED_SESSION_ID=SES-example"],"cwd":"/synthetic/project","stdin":null,"exec_guard":{"schema":"urn:relux:agents-management:exec-guard","schema_version":"1.0.0","data":{"kind":"unsealed"}}},
 "owned_literals":{"CLAUDE_CONFIG_DIR":"/synthetic/profiles/demo/claude"},
 "env_names":[],
 "managed_home":{"variable":"CLAUDE_CONFIG_DIR","path":"/synthetic/profiles/demo/claude"},
 "fragment":{"profile_name":"demo","pin":"sha256:2222222222222222222222222222222222222222222222222222222222222222","fragment_digest":"sha256:3333333333333333333333333333333333333333333333333333333333333333","system_modules":false},
 "policy":{"permission_mode":"native","permission_source":"flag","execution_profile":"standard","effective_native_policy":null},
 "session_name":{"host":"claude_code-20261004T040000Z","native":null},
 "remote_control":{"enabled":false,"argv_indices":[]},
 "resume":{"kind":"new","identity":null},
 "restart":{"schema":"urn:relux:agents-management:claude-restart","schema_version":"1.0.0","data":{"argv":["--model","example-model","--disallowedTools=AskUserQuestion"],"identity_slot":{"index":0,"flag":"--resume"}}},
 "host_bundle":null,
 "network":null,
 "auxiliary":[{"role":"version-probe","mandatory":true,"process":{"binary":"/synthetic/bin/claude","argv":["--version"],"env":["CLAUDE_CONFIG_DIR=/synthetic/profiles/demo/claude","HOME=/synthetic/operator","PATH=/synthetic/bin","TERM=xterm-256color","TASK_BOARD_MANAGED_SESSION_ID=SES-example"],"cwd":"/synthetic/project","stdin":null,"exec_guard":{"schema":"urn:relux:agents-management:exec-guard","schema_version":"1.0.0","data":{"kind":"unsealed"}}}},{"role":"goal-probe","mandatory":true,"process":{"binary":"/synthetic/bin/claude","argv":["-p","--output-format","json","/goal"],"env":["CLAUDE_CONFIG_DIR=/synthetic/profiles/demo/claude","HOME=/synthetic/operator","PATH=/synthetic/bin","TERM=xterm-256color","TASK_BOARD_MANAGED_SESSION_ID=SES-example"],"cwd":"/synthetic/project","stdin":null,"exec_guard":{"schema":"urn:relux:agents-management:exec-guard","schema_version":"1.0.0","data":{"kind":"unsealed"}}}}],
 "reservation":{"session_id":"SES-example","token":"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA","native_uuid":"44444444-4444-4444-8444-444444444444","launch_generation":1},
 "new_session_slot":{"index":0,"flag":"--session-id"},
 "content_digest":"sha256:426c31b9a3ac55a2bcff514c7dbeb0da1d992b58de426a1db4dc4dff51c7358d"
}
```

RC coordinate examples below are partial synthetic projections, conditional on the separately negotiated RC capability; they do not enable RC or replace the complete examples. U is the reserved/saved UUID `44444444-4444-4444-8444-444444444444`. For these module-parsed fixtures, both tokens of the RC flag/name pair are RC positions; native name is `Native-demo` and remains distinct from host name.

| Case | Unbound primary/template argv and RC metadata | Final primary argv and RC metadata |
| --- | --- | --- |
| NEW, new_session_slot.index=0 | `["--model","example-model","--remote-control","RC-demo","-n","Native-demo"]`; enabled=true, argv_indices=[2,3] | `["--session-id",U,"--model","example-model","--remote-control","RC-demo","-n","Native-demo"]`; enabled=true, argv_indices=[4,5] |
| Stopped resume, restart.identity_slot.index=0 | Same selector-free restart argv; enabled=true, argv_indices=[2,3] | `["--resume",U,"--model","example-model","--remote-control","RC-demo","-n","Native-demo"]`; enabled=true, argv_indices=[4,5] |
| Settings-origin RC, remoteControlAtStartup enabled in admitted settings | `["--model","example-model"]`; enabled=true, argv_indices=[] | NEW `["--session-id",U,"--model","example-model"]` or stopped resume `["--resume",U,"--model","example-model"]`; enabled=true, argv_indices=[] |

## 4. Receiver, admission and terminal protocol

Private argv MUST be `task-board session launch-plan --plan - --terminal-fd 3 --status-fd 4`; provider/cwd/name/intent come only from JSON stdin. Launcher supplies controlling TTY read/write fd3 and receiver→launcher status pipe fd4, validates descriptor type/access/ownership before contact, and marks both close-on-exec. stdout/stderr carry live output. fd4 uses 32-bit big-endian length-prefixed JSON, ≤64 KiB; child bytes/status alone MUST NOT create refusal metadata.

Authenticated status MUST advertise closed versioned capabilities, pinning names/schemas. Operational Phase 1 requires `launch-plan/1.1.0`, `auxiliary/1`, `ses-reservation/2`, `managed-session-binding/2`, `claude-new-session-slot/1`, `hosted-diagnostics/1.0.0`, `claude-pty/1`, `exec-guard-unsealed/1`, `claude-restart/1`, `terminal-stdin/1`, `completion/2`; non-null policy requires `effective-policy/1`. Missing mandatory auxiliary support refuses capability missing/6; other missing/malformed support refuses protocol unsupported/6. Current status lacks these (H/types.go:177).

Receiver/daemon MUST independently validate: lexical bounds/one-object/EOF → nested schema/version dispatch → structure/semantics. Unknown versions → protocol unsupported/6; missing/malformed dispatch/known shapes → launch_plan_invalid/2. Receiver validates terminal/scope before Connect, negotiates, reserves/finalizes (§3.3), then submits. Under owner serialization, daemon MUST authorize plan/board/owner/context/incumbent/provenance before records/leases/terminals/auxiliaries. Transient authorization views are allowed; retain context gates (H/provider_access.go:57).

Use one ≤10-second context for Connect plus negotiation only; startup/preflight/readiness/goal binding require separate bounded budgets. Receiver MUST use Connect (H/client.go:162), never start, replace, restart, take over, invoke aliases or recompose. Allocate the next control protocol version beyond current v8 (H/types.go:31), separately from payload versions. v8 wrappers reject newer daemons; newer wrappers reject v8 and retain equal-protocol build-skew checks (H/client.go:882). Owners MUST coordinate wrapper/receiver/daemon deployment with matching capability schemas and build identity; skew safely refuses/6. An existing daemon/sessions MUST NOT be replaced by the receiver. Defaults remain gated on upgrade-without-hangup.

Receiver MUST exclusively own input reader/raw TTY, cancel/join reader and restore TTY on every catchable exit/detach/error; SIGKILL cannot guarantee restoration. Daemon MUST forbid concurrent writers without negotiated input ownership. Initial geometry and SIGWINCH use authenticated resize; errors propagate. Process-control route MUST authorize attachment and target SES/role/launch_generation, allowing only SIGINT, SIGTERM, SIGHUP, SIGTSTP and SIGCONT. Every signal, including internal auxiliary cancellation/escalation, MUST satisfy §5.1’s role-specific evidence. SIGWINCH resizes; detach releases attachment only. Receiver lifecycle signals MUST NOT implicitly Stop the provider.

Terminal stdin uses the daemon child PTY. Later attached stdin requires a separate pipe: decoded bytes, true EOF, continuing terminal output, no keyboard reader/replay; never emulate EOF with a PTY character. Phase 1 refuses all attached stdin/6. Catchable failure MUST restore TTY, close fds, release/cancel attachment/readers. Transport loss detaches surviving children; survival after receiver/daemon loss is not guaranteed. Future Codex client: duplicated terminal streams after JSON EOF, exact verified env, one input reader; never leak private fds.

Process completion MUST contain SES, role, launch_generation and `exited`+exit-code OR `signaled`+signal. Attachment outcomes separately contain attachment ID, SES/role/launch_generation and `detached|host_lost|transport_lost`, with no invented child status; they never overwrite completion. Daemon MUST capture Wait, atomically persist credential-free completion before advertising availability, then answer bounded authenticated queries after terminal EOF. After terminal EOF, the receiver MUST query/wait for generation-matched authenticated completion under one bounded overall deadline, tolerating pending persistence; return unavailable/1 on deadline expiry, confirmed persistence failure or unrecoverable transport loss. EOF alone is neither success nor proof of permanently missing completion. Current Claude drops Wait status (H/claude_host.go:980). Confirmed child exit returns its code or 128+signal; explicit detach returns 0. Phase 3 receiver owns local client Wait bound to imported plan/attachment launch_generation: confirmed client status wins concurrent server teardown; otherwise unavailable/1. Future client completion is receiver-reported.


**W-GEN.** launch_generation is a monotonic per-record execution counter, distinct from daemon incarnation. Reservation proposes the primary value; submit durably commits it before any Start. Each auxiliary Start increments/persists execution_counter and its role’s current_generations entry before spawn; primary uses its committed value. Live reuse retains current generation. Failed attempts never recycle counters; restart gets fresh generation/birth identity. Preserve each execution’s daemon incarnation; daemon restart does not rewrite it.

Pin recursively closed schemas: target={session_id,role,launch_generation}; execution={session_id,role,launch_generation,birth_identity}; birth_identity={pid,kernel_start_time,process_group,role,launch_generation,daemon_incarnation}; attachment={attachment_id,session_id,role,launch_generation}; signal={attachment_id,session_id,role,launch_generation,signal}; completion={session_id,role,launch_generation,outcome}, outcome exactly {kind:exited,exit_code} or {kind:signaled,signal}. All listed members are required, with no additional properties. Wire DTOs MUST use `launch_generation`, never `generation`; target, attachment, signal, completion and nested birth identity MUST agree on execution identity. Roles are primary/version-probe/goal-probe/context-assay; launch_generation is a positive CCJ-1 integer. Completion has exactly one outcome variant: exit_code is an integer 0–255; signal is a positive integer native signal number. No top-level state/reason/exit_code/signal completion members, null status or mixed variants are permitted. Pending/unavailable completion queries require a separately pinned closed query envelope, never an incomplete completion DTO. Durable records use these executions and completion; segregate old completion by generation. Record counters/revisions are positive CCJ-1 integers; current_generations is closed to primary/version-probe/goal-probe/context-assay keys; overflow refuses before Start. Old completion cannot stop the current execution; stale attachment/control requests refuse session_generation_stale/1, never signal a replacement.

Independent golden DTO fixtures MUST include these literal shapes (synthetic identity):

```json
{"session_id":"SES-example","role":"primary","launch_generation":7}
```

```json
{"attachment_id":"attachment-example","session_id":"SES-example","role":"primary","launch_generation":7}
```

```json
{"session_id":"SES-example","role":"primary","launch_generation":7,"outcome":{"kind":"exited","exit_code":0}}
```

```json
{"session_id":"SES-example","role":"goal-probe","launch_generation":8,"outcome":{"kind":"signaled","signal":15}}
```

Diagnostics MUST use Appendix V-ERR’s closed registry, including preserved typed profile/provider/context/goal/resume codes. Absent receiver/endpoint → session_host_missing; unreadable/malformed/stale/unreachable/unauthorized/timed-out endpoint or unrecoverable transport → session_host_unavailable; negotiated protocol/build skew → session_host_protocol_unsupported/6.

A transport/body-read failure AFTER response headers, including timeout, connection reset or io.ErrUnexpectedEOF for a truncated Content-Length body, MUST also yield the constant session_host_unavailable/1 with empty details. Partial body bytes and underlying error text MUST NOT be reflected. A successfully read malformed/invalid envelope or detected response-size overflow yields session_host_protocol_error/6; reaching the overflow bound is a protocol failure even without consuming the remainder. Failure timing before versus after headers MUST NOT change the transport classification.

fd4 refusal MUST be closed `{schema:"urn:relux:task-board:session-launch-status",schema_version:"1.0.0",type:"refusal",code,message,details}`; all members required. Both HTTP and fd4 receiving boundaries MUST normalize against V-ERR, including successfully decoded legacy PublicError. Unknown code, mismatched message, bad detail or malformed envelope MUST become fixed session_host_protocol_error/6 with empty details. Typed producers MUST generate constants, never copy legacy messages. Other status types require separately pinned closed schemas. Launcher MUST classify only fd4, preserve code/exit and amend diagnostics/SPEC (L/internal/execution/execution.go:180). MUST suppress raw HTTP/error bodies (H/client.go:834,845; H/control.go:628).

## 5. Daemon obligations and module APIs

Hosted ingress MUST validate/authorize before allocation/provider I/O; separate legacy routing. Existing insertion precedes host decoding (H/manager.go:649,663). Resolve canonical cwd through existing board selection, bind authenticated BoardKey/fingerprint and parent/primary owner, and validate requested project/board; remote-board identity MUST remain authoritative (B/cmd/tb-sessiond/main.go:82). Preserve provider grammar, execution-profile/confirmation/bypass, workspace/version/native-goal and context gates; goal binding precedes attachment.

Every provider child/client/preflight/assay MUST consume its concrete module projection; transcript/config/discovery paths derive from its final env/home/settings. Manager socket/record/log paths remain BoardKey Layout infrastructure (H/record_store.go:64). No undeclared env/argv/home edits are allowed. Board delivery/owner markers/filtering/policy MUST be module-declared/validated or refuse. Remove daemon-env/private-home rewriting. §3.3 projections/binding MUST be implemented before admission; missing mandatory projections refuse before allocation.

Delivered at A ccf19b8/v0.5.48 (paths A/pkg/agentic): `agentic.ElevateResumeIntent` (resume.go:49; Claude 2.1.288/Codex 0.159.0/Muse 1.4.1); `claude.System.ExportRestartTemplate`/`(*claude.System).ValidateRestartTransformation` (systems/claude/restart.go:39,51); `agentic.PinnedHostedSchema` (hosted_schemas.go:16); `Plan.ExportSeal`/`agentic.ImportSeal` (seal.go:162,197); `agentic.FinalizePlan(base, FinalizeOverlays, []TypedBinding, ...SealCommitmentKey)` (finalize.go:71). Finalization preserves artifact digests, never reseals disk.

Claude exports the unsealed marker; Muse refuses it (A/pkg/agentic/systems/claude/seal.go:7; A/pkg/agentic/systems/muse/seal.go:1). Nil VerifyBeforeExec is a no-op (A/pkg/agentic/exec.go:14). The Phase 1 adapter MUST import the closed Claude marker, validate the exact process/restart transformation and run shared native binary/MCP/prompt checks immediately before Start through TASK-261004-26qvc0’s public exec-free pkg/agentic API; daemon MUST NOT import launcher internals; managed argument/goal-readiness gates remain separate. Unsealed verification proves no producer identity/artifact integrity.

V-VER: Admission MUST await tagged `agentic.NewImportedProcess(seal, process)` (TASK-261004-1kc4kk), binding exact binary/ordered argv/full env/cwd/home/stdin; `ImportedProcess.VerifyBeforeExec` refuses differences immediately before EACH Start. Interim only: `verifier, err := agentic.ImportSeal(system, guard, keys...)`; require success, retain verifier beside immutable process, call `verifier.VerifyBeforeExec(exactPlan)` immediately before Start. Receiver owns clients; daemon primary/auxiliaries; receiver checks cannot replace daemon checks. `Plan.VerifyBeforeExec` covers local plans only; ImportSeal never installs it. Guards deep-copy; digests remain transient.

Current FinalizePlan rejects all typed bindings; bound/sealed exports are local-only (A/pkg/agentic/seal.go:83; A/pkg/agentic/finalize.go:240). CF-Q2 MUST deliver admissible finalization preserving checks. Never widen 1.0.0/downgrade guards; bound/sealed transport needs pinned schemas/capabilities, not assumed by 1.1.0. Bound imports require identical transient commitment key (A/pkg/agentic/seal_binding.go:13); current payload carries none.

Phase 3 MUST publish module-owned app-server/client/thread/restart/auxiliary bundle schemas, closed policy vocabulary and explicit endpoint/conversation slots. Daemon binds slots; authenticated replies bind template/generation; receiver validates substitutions and finalizes client verification. No shell templates, argv inference or host-created policy flags. Base Plan supplies no bundle (A/pkg/agentic/plan.go:35); missing schemas/adapters refuse.

Claude RC/name support MUST be negotiated with parity evidence. Extend both total allowlist and settings-origin checks for module-parsed RC/prefix; deleting deny rows alone is insufficient (H/claude_host.go:286,310). Until supported, RC MUST refuse. Native -n/--name already work (H/claude_host.go:241); preserve tokens, prefix and separate native/host names. Supplied home/settings govern launch/renewal. Live native name/RC/effective-policy provenance MUST stay stable absent negotiated module metadata-only migration.

### 5.1 Resume/adoption transitions

Ordinary reuse/restart MUST preserve provider, board fingerprint, canonical project, profile name/pin, native identity and fragment/restart provenance. Changed fragment/restart provenance MUST refuse until an explicit migration has been authorized and completed; ordinary renewal MUST NOT perform that migration. Re-resolve saved profile before live reuse, stopped restart and startup; first adoption resolves requested profile, re-adoption saved profile. Pin drift → policy_refused/environment_drift; resolution errors retain codes.

V-ENV/W-ENV: Stopped restart MUST regenerate fresh FULL caller env, retaining fresh order/module filtering. ALL independent checks MUST pass: reserved SES equality; selector values/presence equality; EVERY owned literal’s keyed commitment equality, including overlapping selectors; inherited freshness/required caller lookup. An unchanged CLAUDE_CONFIG_DIR selector never bypasses its literal HMAC; any required unavailable key refuses renewal. Other inherited entries MAY rotate/disappear/be present-empty, including outside env_names; env_names is the required lookup set. SES marker belongs to neither set. Closed home_selectors permits only CLAUDE_CONFIG_DIR, HOME, XDG_CONFIG_HOME, XDG_CONFIG_DIRS, XDG_DATA_HOME, XDG_DATA_DIRS, XDG_STATE_HOME, XDG_CACHE_HOME, XDG_RUNTIME_DIR; absent omitted, empty retained, TERM excluded. New dependencies require schema revision. No raw secrets/bare hashes; host name may refresh; no payload-digest comparison/live mutation.

Provenance/context/future N7 MUST precede four boundaries: loaded reuse, prior Restore, fresh Launch and startup restoreAndReconcileLocked (H/manager.go:1589,1625,1690,1964).

| Transition | Required action |
| --- | --- |
| new/no incumbent | Reserve SES/UUID; module-finalize both slots; persist generation; start exact finalized argv once. Identity is known before spawn; no transcript inference (H/claude_host.go:658,677,875) |
| plain/same live incumbent | Compare provenance/liveness/attachability, authorize and reconcile goal; attach without second child |
| latest | Follow ordered resolution below; decode only selected snapshot (H/resume.go:146) |
| handle | Follow ordered resolution below |
| first native adoption | Before mutation validate canonical ID, provider store/project location and free or proven-dead released slot; establish validated profile provenance after declared UUID binding verification, before attachment |
| re-adoption | Look up saved provider/native identity/location and compare provenance before allocation; apply ordered incumbent refusals below; stopped/proven-dead slots may release. Prefer SES resume for same record |
| stopped restart | Renew caller material; module restart slot inserts saved UUID exactly once; verify bound identity; omit original selectors/one-shot stdin |
| crash restore | Re-resolve safe provenance/context/N7. Codex may reconnect to verified reachable server/thread. Claude cannot recover its PTY master even with live PID: retain recovering/live-detached handle pending explicit safe stop/renewed resume (H/claude_host.go:826; H/codex_host.go:314) |

Missing volatile material/key MUST park a hosted record as renewal-required, not corrupt/quarantined or ProviderHostNotFound fall-through. Unknown liveness/live-but-unattachable MUST retain slot and distinct session_liveness_unknown/session_live_detached; never reconstruct daemon credentials or launch duplicates (H/liveness.go:216,227). Preserve all five resume errors and candidate/quarantine/location/identity semantics (H/resume.go:15,171). Competing non-stopped incumbent refuses; proven-dead slots may release; same live incumbent may attach after checks. Legacy records MUST stay on the legacy path, never silently hosted-upgrade.

V-RES: This ordered table spans module normalization and daemon resolution; relevant branches apply before allocation.

| Order | Decision |
| --- | --- |
| 1: module | Malformed handle/conflicting selectors/pickers → session_resume_invalid/2; malformed-handle invalid/2 deliberately replaces legacy not_found/1 |
| 2: location | Invalid project/board → session_resume_location_mismatch/1 |
| 3: quarantine | Relevant quarantine → session_resume_unreadable/1 BEFORE latest candidate selection (H/resume.go:63–76) |
| 4: handle | Valid missing → session_resume_not_found/1; wrong provider/owner/location → location_mismatch; missing Opaque/pending fork → unreadable |
| 5: latest | Exact provider/project/board primary, exclude missing Opaque/pending forks; none → not_found; recency ties → ambiguous (H/resume.go:78,124) |
| 6: incumbent/adoption | Reconcile/release proven-dead first; remaining incumbent: provider conflict → unknown liveness → live-detached → adoption_conflict, preserving each code (H/manager.go:572–618); same attachable live record permits plain reuse |
| 7: acknowledgement | Different/missing native identity → session_resume_identity_mismatch/1 |

V-PROC/W-PROC: Capture kernel PID/start-time/process-group plus role/launch_generation/daemon incarnation at EVERY primary/auxiliary spawn; store closed birth_identity. Use reliable kernel precision; coarse/ambiguous timestamps are unknown. Signal proof is role-specific. For the primary Claude process, the reserved UUID is known before spawn, verified against exact module-bound argv and recorded birth/cwd before goal/attachment; never infer from newest/same-home transcripts. UUID-in-cmdline safe-stop/repair helpers apply to primary hosted Claude again (H/claude_host.go:2486–2520; H/repair.go:291–319), with birth identity REQUIRED as second factor. The following table concerns the primary only.

| Evidence | Permitted action |
| --- | --- |
| Live birth match plus UUID match | Reuse if attachable; signal only this execution/group |
| Authenticated generation-bound Wait completion persisted | Mark stopped/release safely; stopped renewal permitted |
| Conclusive absence | Recorded PID absent OR PID has different kernel start time, AND UUID absent from EVERY process cmdline; persist stopped before release/renewal |
| Unknown/mismatched/ambiguous | Retain unknown/live-detached slot; never signal replacement PID or infer death |

Primary absence requires complete authorized process enumeration; lookup/permission errors are unknown. UUID found elsewhere defeats absence. PID reuse permits conclusive primary absence only with exhaustive UUID absence, never replacement signaling. Recheck primary birth/UUID before each signal, including escalation; group targeting requires verified membership. Failed persistence retains slot. Restart allocates fresh execution identity (§4).

Auxiliaries MUST retain their original spawned-process handle in volatile daemon state, bound to SES, role, launch_generation and the recorded birth identity. Only that retained handle plus a fresh matching kernel birth/role/generation check authorizes auxiliary signaling; PID, SES env marker, home/cwd or argv resemblance alone is insufficient. Group signals additionally require fresh verified membership of the original auxiliary’s group; stale/reused group IDs or unrelated members MUST NOT be targeted. Recheck handle/birth/role/current generation and group membership immediately before EACH signal and escalation. Context cancellation/timeouts MUST use this controlled path, including replacement of any default cancellation that would signal without these checks. Never add the primary conversation UUID or primary identity slot to version-probe, goal-probe or context-assay argv.

| Auxiliary evidence | Permitted action |
| --- | --- |
| Retained original spawned-process handle plus matching live birth/role/current generation; verified group when targeted | Signal/cancel only that original auxiliary or its verified group; repeat proof before escalation |
| Authenticated Wait on that handle, generation-bound completion persisted | Mark that auxiliary execution stopped; cleanup/release its resources without changing primary completion |
| Handle lost; recorded PID conclusively absent or replaced by a different precise kernel birth, with any owned group members conclusively absent | Persist absence for that auxiliary generation before cleanup/release; never signal the replacement PID/group; no primary UUID enumeration is required |
| Handle lost and same birth still live, or lookup/group evidence unknown/ambiguous | Retain unresolved auxiliary execution/resources; refuse signaling/escalation and dependent admission/retry; no reattachment of a reconstructed PID handle or duplicate auxiliary Start |

Auxiliary recovery is based on its recorded execution identity, not conversation UUID search. Handle loss across daemon restart cannot restore signal authority from durable PID data. Permission/lookup failures and incomplete group evidence remain unknown; failure to persist completion/absence retains resources. An auxiliary completion or absence never proves primary death or permits release of its live/unknown ownership slot. New auxiliary attempts require resolved prior execution and a fresh generation (§4).

### 5.2 Muse Phase 2

R2 PTY host is absent from B main; its backlog design input and owner-accepted c350d06c candidate are not landed support. Phase 2 MUST require landed R2, imported sealed LaunchModeInteractive plans only (no LaunchRequest recomposition), TASK-261004-2hdguc sealer and TASK-261004-13opwj import/release follow-up. TASK-261004-s9mfhu APIs are delivered; new hosted sealed schemas remain gated.

Sealer MUST bind probed binary path/SHA256, exact final trust/posture argv, probed release, XDG identity and MUSE_NO_AUTO_UPDATE. TASK-261004-2hdguc DoD MUST include Muse 1.4.2-R4684.1 resume-help grammar alongside 1.4.1; only 1.4.1 exists now (A/pkg/agentic/systems/muse/resume.go:17; inbox events 12–13). Both-release captures are required.

Both native/hosted plans MUST compose --trust-workspace from profile/workspace. Host releases MUST equal module verified 1.4.1/1.4.2 (A/pkg/agentic/systems/muse/policy.go:13). HOME unchanged, managed_home=null, authoritative XDG/auth-link env. MUSE_NO_AUTO_UPDATE=1 is mandatory or launch_plan_invalid; RC=false/native name=null or scope unsupported. PermissionMode owns posture; Q-D3 refusal applies. muse_session uses validated stored R2 identity. ApplyBoardGoal/notices submit text only when log+screen show idle, rechecked before submit; native goals remain opt-in; host answers DSR/CPR.

## 6. Acceptance

Fake-boundary evidence MUST precede later real-harness checks; each admitted Claude/Muse case compares complete native/hosted binary, ordered argv/full env, cwd/home/stdin without masking additions except the exactly enumerated two D2 slots, separately validated.

| Cases | Required evidence |
| --- | --- |
| Composition | Minimal/native policy; model+effort; MCP/inline JSON/system prompt; conflicting overlays; inherited/empty env; absent/empty env_names; quoted/Unicode/empty tail; Claude denial `--disallowedTools=AskUserQuestion` unchanged (A/pkg/agentic/systems/claude/toolpolicy.go:21), caller denials/re-enable refusal |
| Claude RC/name | RC alone; -n alone; both; prefix; distinct host/native names; `curator run claude --hosted -n NAME -- --remote-control NAME`; remoteControlAtStartup settings path; RC live with intended native name; unsupported capability refuses |
| Muse | Both releases; HOME/XDG/auth link/update pin/trust/posture; text goals/notices; busy/trust-/permissions-dialog refusal; idle recheck; DSR/CPR; detach/native-ID resume; narrowing mutants; M2 status mapping |
| Codex gate/variants | Refusal now; later corresponding module app-server/client/thread/restart/auxiliary inputs and only §1 transport differences; no second conversation |
| Resume | Every §5.1 transition; quarantine/ties/pending forks/unknown/live-detached/renewal-required recovery; missing/corrupt/wrong-location handles; first/re-adoption conflict; equal/drift/unresolvable pin; renewal unavailable; selector/stdin non-replay |
| Routing/refusals | Every lock/precedence/ax row; repeat aliases; v1/v2/v3; default gate; no-config zero lookup/probe; all exits; network direct/default/explicit; missing/stale/auth/protocol/build skew; unknown nested versions; malformed/duplicate/digest/bounds/secret metadata; zero allocations on admission refusal; no fallback/start/takeover/recomposition |
| Execution/I/O | Allowed bindings/forbidden edits; artifact mutation before each start; JSON/TTY separation; catchable raw restore/reader cancellation/input ownership/resize/targeted signals/receiver failure; child 0/nonzero/signal versus detach/loss; daemon loss/EOF-before-persistence/completion failure; Phase 1 attached-stdin refusal; future empty/binary input, true EOF and continuing output |
| Secret lifecycle | Distinct synthetic caller/daemon credential canaries: accepted launch, failure after admission, live attach, stopped resume, crash restore, malformed/unknown responses. Inspect records/Opaque/snapshots/manifests/host logs/diagnostics for plaintext and encoded canaries and forbidden durable digests; no daemon credential reaches child/client/preflight. Provider-generated output is evaluated separately from host request logging |

Additional required fixtures: imported verifier refusal prevents Start; all mandatory auxiliaries use supplied env/home; fresh/reuse/restart notices target reserved/saved SES; token expiry/replay/mismatch allocates nothing; rotated/removed/present-empty inherited credentials outside env_names; unchanged versus drifted selectors/literals; malformed metadata pattern→2 paired with valid-pattern/source mismatch→16; every V-RES branch; exact-finalized-argv child survives daemon loss; inconclusive PID reuse refuses kill/release; receiver EOF waits through pending persistence under one overall deadline and expires→unavailable/1; valid error envelope with canary message and malformed bodies normalize without disclosure.


r5 discriminating fixtures MUST include: two stopped handles produce their respective saved SES; latest differs from incumbent/new; valid missing handle; incumbent/candidate revision changes between reserve/submit → stale/1 without retarget/Start; first/re-adoption; fresh idle launch with reserved UUID alongside same-home/cwd native process and probe transcript; both slots verified, forbidden mutations refused; clean-exit restart; missing-PID+exhaustive-UUID-absence recovery; PID reuse/unknown enumeration; stop/restart under one daemon with delayed old completion/signals; unchanged selector/literal plus unavailable HMAC key refuses, and independent selector/literal drift; EVERY reachable typed producer has a registry entry, fork refusals retain code/exit, unknown/canary envelopes disclose nothing.

r6 seam fixtures MUST include: hung goal-probe timeout/cancellation with fresh handle/birth/role/generation/group checks before termination and escalation; version-probe failure cleanup with persisted generation-bound Wait and no primary Start; delayed auxiliary cancellation/completion after a newer generation exists; PID/group reuse between initial check and escalation, proving only the original auxiliary can be signaled. Test handle loss with same-birth live, conclusively absent/reused PID, and unknown/incomplete group lookup: no reconstructed-handle signaling, duplicate probe or primary release. Assert every probe argv remains its module projection without the conversation UUID.

RC fixtures MUST independently check §3.4’s NEW and stopped-resume nonempty index arrays, unchanged enablement/prefix/native name/host name/settings/policy, and settings-origin enabled RC with empty indices. Cover insertion before/after indexed tokens at a permitted declared restart slot, reuse without insertion, stale/unshifted/out-of-bounds/extra/non-RC/omitted positions and forbidden metadata/token changes. Recompute digests and independently parse final primary argv at both receiving boundaries; corrupt indices with a recomputed digest still refuse before Start. Unsupported RC remains refused.

CR1 contract fixtures MUST use independent literal V-ERR code/exit/message/detail expectations, never expectations generated from the implementation registry. Cover the six CR1 emitters and all registry receiving entries, preserved policy_refused/16 and session_resume_not_found/1, every D0/D1/D2 permitted subset and invalid token/key/type, mismatched punctuation and unknown/canary envelopes, through writer plus HTTP/fd4 normalizers and decoded legacy PublicError. Golden marshal/shape fixtures MUST match §4’s target/attachment/exited/signaled DTOs and reject `generation`, mixed/missing outcomes and independent top-level status fields. Fake response-body readers MUST fail after headers, before or after partial canary bytes, with timeout/reset/io.ErrUnexpectedEOF: unavailable/1 with constant message and empty details, no canary/error reflection. Complete malformed/oversized envelopes remain protocol_error/6.

Later harness checks MUST follow N/docs/harness-verification.md: isolated allowlisted homes/configs, nonforwarding loopback sink/calibrated sandbox, no real credentials/egress; authentication-blocked rows remain blocked. No daemon/harness run is authorized.

## 7. N-C1b hook

Until enforcement lands, network MUST be null; every managed selection refuses D3. Future negotiated `urn:relux:task-board:network-binding` 1.0.0 MUST be closed `{schema,schema_version,ref,origin,expected_digest}`, with identifier ref, origin explicit|inherited|runtime|project|operator, sha256 digest. Resolve/authorize locally; verify digest/adapter/confirmation; applicable local preflight, direct/plans skip probes. Reject mixed-case proxy env_names; suppress stale proxy literals; unset case-insensitively and apply fresh set last to every child/client/preflight/assay. Persist sanitized Record only; drift refuses network_profile_drift, live attach/restore requires full binding equality. Cover startup reconciliation and identified-profiled daemon-start refusal. Ambient proxies never select profiles. Muse additionally requires muse-env-v1 (TASK-261002-14shh7) and verified entrypoint/child scope. Enforcement/conformance, not mere carrier negotiation, opens the gate (N/docs/integration-contract.md:126,147).

## 8. Ownership and outstanding review

| Work | Owner |
| --- | --- |
| Launcher flags/defaults/SPEC, TASK-261004-38ba6c | infra-exit |
| Receiver/aliases | infra-exit: 1okvcs (re-scoped 2026-10-04 per D2; snapshot README predates) |
| 1kc4kk verifier, 26qvc0 public native checks, 3u485d auxiliaries, CF-Q2 finalization; selectors/Codex schemas | module / infra-exit; critical-fixes decides CF-Q1/CF-Q2 |
| R2 import/release list; Muse R3 alias | tb-muse; module owner reviews Muse sealer |
| Daemon reservation/SES/birth identity, ingress/terminal/RC/resume/N-C1b | ivan-network-profiles with critical-fixes/session owner |
| §4–§5 advisor/reviewer | critical-fixes; 68-row review applied; no runtime approval implied |
| Diagnostics registry/normalizers, schema amendments/conformance/defaults | spec owner + launcher/receiver/daemon owners; Ivan for defaults |

## Appendix: retained r1 → r2 dispositions

rev1#1–4→§§3–4/6; rev2#1,2,6/IE#2→phase/schema gates; rev2#3/IE#1/R-SEAL/R-SEAL-M→import/finalization/Muse sealer; rev2#4,7,8→I/O/completion; rev2#5/M4→resume/provenance; rev2#9→1okvcs; rev2#10/IE#5→ax; IE#3,4,6→credentials/selectors/Q-D1a; M1–3,5/R-RC/Muse minors→§5.2/RC/name.

## Appendix: r2 → r3 disposition

Retain all 42 adopted clauses; r5 supersedes corrected wording. Changed owner rows: 4-06,10→capabilities/authorization; 4-12,13→deadlines/rollout; 4-15,18,20→TTY/signals/loss; 4-22,23,25→completion/Wait; 4-27,31→diagnostics; 5-01,02,04→ingress/BoardKey/paths; 5-05,08,12→bindings/native checks/imports; 5-18,20→RC/metadata; 5.1-02,05→provenance/finalized UUID; 5.1-09,10→adoption; rejected 5.1-12/changed 13→lost Claude PTY/renewal/slots.

## Sources read

Verified local HEADs/anchors; H=B/internal/sessionmanager.

| Alias | Local source / HEAD |
| --- | --- |
| L | ref/curator-agent-launcher @ fbcdbaf07efe6fc7a8b4929e1c709028345021da |
| B | ref/skill-project-management/tools/board-cli; repository @ a6e81e2ddf2107f43d548c2431774566abe97da5 |
| A | ref/skill-agents-management @ ccf19b87dd865cd60a22447c9f4378b392657eff |
| S | ref/curator-spec @ 43bf0a2506d5c354a73bbc3ea4623d4653db10c7 |
| N | ref/curator-network-profiles @ ec68fde8a14a7a22086b5e09cad595ff9ea216cb; docs/integration-contract.md and docs/harness-verification.md |

r5 inputs: BRIEF-r5, MERGED-r4-findings, both full seam reviews and Draft r4. Retain owner dispositions; R2 candidate is owner-reported. Static validation only; no runtime/tag-publication acceptance.

r6 inputs: BRIEF-r6, REVIEW-seam-r5-1, d4/REVIEW-cr1-1 and d4/REVIEW-cr1-2, applied to Draft r5. Only their contract-text findings are reconciled; implementation/test-review findings do not establish delivery or authorize runtime work.


## Appendix V-ERR: diagnostics registry

Pinned artifact `urn:relux:task-board:hosted-diagnostics` **1.0.0**, normative closed enumeration below; publication/consumer pinning is a Phase 1 gate. Comma-separated codes each receive exactly that row’s exit/message/details set. Messages are literal constants (≤512 bytes); no interpolation. details is closed, optional field/reason only, each ≤128 bytes. D0 permits `{}` only; D1 permits field∈{producer,fragment,policy,process,resume,reservation,auxiliary}, reason∈{shape_invalid,source_mismatch,credential_metadata,required_env_missing,reservation_invalid}; D2 permits field∈{fragment,policy,managed_home}, reason∈{environment_drift,selector_drift,literal_drift}. Any subset of a row’s permitted detail keys is allowed; no other tokens.

This is the SINGLE complete registry for that pin across daemon writers, HTTP clients and fd4 receivers. Every comma-separated code is a separate exact case-sensitive entry with that row’s numeric exit, UTF-8 message byte string and detail vocabulary; table separators/padding are not message bytes. All messages below are ASCII, including exact capitalization, internal spaces and final period, with no leading/trailing whitespace or newline. Equality is checked on the decoded message string’s UTF-8 bytes, independent of JSON escape spelling. Detail keys/values are case-sensitive strings, never null; an allowed subset means `{}`, field-only, reason-only or both for D1/D2, and only `{}` for D0. No extra keys/tokens or arbitrary text are permitted. Under this pin, partial registries, alternate messages (such as `Launch plan is invalid`), and invented tokens (`request`/`invalid`) are nonconforming. Consumers MUST recognize every entry even if its producer operation is disabled; recognition never enables that operation. Unknown codes or mismatched messages/details normalize to the exact session_host_protocol_error entry with `{}`. Any incompatible registry requires a new version and negotiation, never another definition of 1.0.0.

| Code | Exit | Constant message | Details |
| --- | --- | --- | --- |
| usage | 2 | Invalid invocation. | D0 |
| launch_plan_invalid | 2 | Invalid launch plan. | D1 |
| session_resume_invalid | 2 | Invalid resume selector. | D0 |
| host_configuration_conflict | 2 | Conflicting host configuration. | D0 |
| session_host_missing | 1 | Session host is missing. | D0 |
| session_host_unavailable | 1 | Session host is unavailable. | D0 |
| session_host_protocol_error | 6 | Invalid session host response. | D0 |
| session_host_protocol_unsupported | 6 | Session host protocol is unsupported. | D0 |
| session_host_capability_missing | 6 | Required host capability is missing. | D0 |
| session_host_provider_unsupported,session_host_scope_unsupported,session_host_execution_profile_unsupported,session_host_terminal_required,session_host_stdin_unsupported | 6 | Hosted launch requirement is unsupported. | D0 |
| secret_policy_violation | 16 | Secret policy violation. | D1 |
| policy_refused | 16 | Policy refused. | D2 |
| network_scope_unsupported,network_profile_drift | 16 | Network policy refused. | D0 |
| session_host_default_not_ready | 16 | Hosted default is not ready. | D0 |
| session_resume_not_found,session_resume_ambiguous,session_resume_unreadable,session_resume_location_mismatch,session_resume_identity_mismatch | 1 | Session resume refused. | D0 |
| session_provider_conflict,session_adoption_conflict | 1 | Session ownership conflict. | D0 |
| session_liveness_unknown | 1 | Session liveness is unknown. | D0 |
| session_live_detached | 1 | Live session is detached. | D0 |
| resolve_invocation_failed,resolve_environment_unknown,resolve_profile_unknown,resolve_repair_failed,resolve_lock_unavailable,resolve_fragment_invalid,defaults_config_invalid,defaults_unresolvable | 1 | Profile resolution refused. | D0 |
| plan_refused,plan_provider_limited,env_unsupported,permission_policy_unsupported,permission_mode_tracked_unsupported,permission_mode_unsupported,exec_provider_missing,mcp_layer_missing,mcp_layer_unreadable,sysprompt_channel_unavailable,sysprompt_file_unreadable | 1 | Provider launch refused. | D0 |
| claude_goal_unavailable,claude_goal_unsupported_version,claude_goal_workspace_untrusted,claude_goal_preflight_failed,claude_session_resume_failed,claude_launch_args_unsupported,claude_session_adoption_invalid | 1 | Claude capability refused. | D0 |
| claude_context_base_not_forkable,claude_context_cwd_slug_unverified,claude_context_fork_plan_invalid,claude_context_fork_source_mutated,claude_context_fork_unverified,claude_context_materialization_failed,claude_context_transcript_format_drift,claude_context_writer_lease_unavailable | 1 | Claude context refused. | D0 |
| context_fork_preflight_failed,context_run_handle_unavailable,context_run_transcript_unavailable | 1 | Context preparation refused. | D0 |
| owner_activation_in_progress | 1 | Owner activation is in progress. | D0 |
| session_reservation_stale,session_generation_stale | 1 | Session observation is stale. | D0 |
| session_takeover_not_applicable,session_takeover_observation_changed | 1 | Session takeover refused. | D0 |
| goal_reflection_skew | 1 | Goal reflection is incompatible. | D0 |
| codex_context_base_not_forkable,codex_context_capability_unavailable,codex_context_fork_plan_invalid | 1 | Codex context refused. | D0 |
| codex_thread_adoption_invalid,codex_thread_not_found,codex_thread_resume_failed,codex_goal_budget_required | 1 | Codex capability refused. | D0 |
| context_contract_invalid,context_binding_invalid,context_lineage_invalid,context_not_found,context_already_exists,context_revision_conflict,context_state_invalid,context_not_ready,context_stale,context_publication_denied,context_secret_class_denied,context_provider_mismatch,context_payload_capability_required,context_materialization_divergent,context_registry_unsupported,context_persistence_failed,context_compacted,context_integrity_unverified,context_classification_failed,context_builder_allowlist_denied,context_builder_confinement_unavailable,context_builder_trace_unavailable,context_builder_tool_isolation_unavailable,context_assay_isolation_unavailable,context_assay_tool_use_detected,context_payload_vanished,context_payload_unverified,context_provider_incompatible,context_not_materialized,context_pointer_invalid,context_assertion_unverified,context_correctness_failed,context_contaminated,context_secret_detected,context_validation_unavailable,context_validation_budget_exceeded | 1 | Context authorization refused. | D0 |
| goal_producer_clause_unsupported,goal_contract_invalid,goal_role_incompatible,goal_scope_invalid,goal_scope_empty,goal_scope_blocked_only,goal_dependency_closure_invalid,goal_objective_too_long,goal_provider_condition_too_long,goal_revision_required,goal_revision_conflict,goal_mutation_restricted,goal_actor_unauthorized,goal_activation_failed,goal_budget_invalid,goal_manager_protocol_skew,spawned_run_goal_plane_disabled,goal_owner_invalid,goal_not_found,goal_persistence_failed,goal_native_reflection_required | 1 | Goal operation refused. | D0 |


W-ERR inventory: all 37 named PublicError producer codes appear above, including excluded-route entries; client.go:838 only decodes wire errors. Retain all 36 ContextError/21 GoalError codes (ref/skill-project-management/pkg/board/context_registry.go:214–331; goal.go:222–253; board_goal_v2.go:161–164). Never parse prose.

| Operation / producers | Phase 1 treatment |
| --- | --- |
| Provider/goal | claude_host.go:300,413,1025,1920,2370; manager.go ApplyGoalRevision; owner_activation.go:55; existing Claude/goal rows + owner_activation_in_progress |
| Context authorization/inspection/assay | context_fork_admission.go:53,93,122; claude_context.go:25–46,544,603; claude_context_fork.go:222–548; claude_context_assay.go:395,617,1251; claude_context_lease.go:254; claude_transcript.go:269,439,485; context_security.go; ContextError and eight Claude-context entries |
| Resume/adoption/liveness | resume.go:15–178; manager.go:476,606,786; liveness.go:217,228; five resume codes, ownership/liveness rows |
| Excluded operations | Codex/Muse launch → provider_unsupported before contact; takeover/daemon start-replacement, owner runs, standalone materialization/build/fork/publication/consumer observation and primary-goal mutation → session_host_scope_unsupported/6 BEFORE operation/provider I/O. No extra fork flags pass the two-slot boundary |

Context entries preserve retained authorization/inspection/assay refusals; registration never admits standalone mutation. context_run.go:232,313/context_publication.go:189 and takeover.go:127–190 are inventoried but excluded. Non-code typed errors MUST map explicitly: ClaudeGoalAckTimeoutError→claude_goal_unavailable; ManagerProtocolUpgradeError→session_host_protocol_unsupported; GoalReflectionSkewError→goal_reflection_skew; record-load failure→session_resume_unreadable; contextPayloadUnusableError preserves underlying ContextError. Other untyped operation failures→session_host_unavailable; unknown wire codes remain protocol_error. StartupCancelledError, ManagerReplacementRefusedError, consumerForkBoundaryError and codexContextAssayInterruptedError are excluded before their operations. Coverage MUST compare named producers/shared enumerations with registry and test mappings at BOTH receiving boundaries.

## Appendix: r3 → r4 disposition

No delivery/runtime acceptance: sec#1/impl#3→V-VER; sec#2/impl#5→V-ENV; sec#3→V-ERR; sec#4→V-PREC; impl#1→V-AUX/CF-Q1; impl#2→V-SES/CF-Q2/D2; impl#4→V-RES; impl#6→V-PROC (r5 supersedes transcript attribution/unconditional live-match renewal); delta#1→V-DONE/4-23; delta#2→V-MIG; delta#3→V-NIT.

## Appendix: r4 → r5 disposition

Admission remains gated.

| Ruling / findings | r5 disposition |
| --- | --- |
| W-RES: r4-1#1,delta#1 | §3.3 intent/selection, saved-target outcomes, revision/incumbent/generation-bound tokens, stale/1; §6 discriminating fixtures |
| W-UUID: r4-1#2 | Two declared slots/new_session_slot + reservation UUID; restart --resume; D2/CF-Q2 gate; both examples retained/updated |
| W-PROC: r4-1#3 | §5.1 live/completed/absent/unknown outcomes, UUID+birth proof, fresh restart identity; §6 positive/negative cases |
| W-GEN: r4-1#4 | §4 durable per-execution counter, closed record/attachment/completion/signal fields; old-event rejection |
| W-ERR: r4-1#5,delta#3 | Complete producer/shared-code inventory, constants/D0, excluded operations/mappings; coverage gate |
| W-ENV: delta#2 | §5.1 cumulative independent checks; overlapping literal/key and selector fixtures |
| W-NIT: r4-1#6 | manager.go:1690; muse/resume.go:17 verified |

## Appendix: r5 → r6 disposition

Admission remains gated; these are contract requirements and acceptance fixtures, not delivered implementation or runtime acceptance. All other r5 clauses and prior dispositions are retained.

| Review finding | r6 disposition |
| --- | --- |
| REVIEW-seam-r5-1 #1 — auxiliary signal evidence | §§4/5.1 role-specific proof: primary UUID+birth; auxiliary retained spawned-process handle+birth/role/generation/group checks before every signal/escalation; conservative handle-loss recovery, no probe UUID; §6 timeout/failure/stale-generation/PID/group-reuse fixtures |
| REVIEW-seam-r5-1 #2 — RC index insertion | §§3/3.3 final-primary-argv positions; module updates only derived indices for the declared insertion, preserving all intent/other metadata; independent validation; §3.4 NEW/stopped-resume/settings examples and §6 positive/negative fixtures |
| REVIEW-cr1-1 #1 / REVIEW-cr1-2 #1 — diagnostics pin divergence | V-ERR is one complete exact code/exit/message-byte/detail-token registry; no subset or alternate vocabulary under 1.0.0; §6 independent literals at writer and both receiving boundaries |
| REVIEW-cr1-2 #3 — target/completion DTO identity | §4 closed target/attachment/completion shapes use launch_generation and exclusive outcome union; pending/unavailable query envelope separate; literal golden fixtures and §6 rejection cases |
| REVIEW-cr1-1 #2 / REVIEW-cr1-2 #4 — response-body transport failure | §4 after-headers/partial-body timeout/reset/unexpected EOF → constant session_host_unavailable/1; complete malformed/overflow response → protocol_error/6; §6 fake-reader/canary cases |
