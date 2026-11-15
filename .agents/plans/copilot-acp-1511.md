# Deliver Copilot CLI ACP sessions (#1511)

This living ExecPlan follows `.agents/PLANS.md`. Maintain all sections as investigation and implementation proceed.

## Purpose / Big Picture

Users should configure a Copilot profile, discover the options available to their account, and run, cancel, and recover sessions through Mjolnir. ACP is the JSON request/notification protocol between the worker and coding harness. Readiness must distinguish absent runtime, incompatible runtime, and absent authentication. Repository checkpoints must preserve native conversation state if supported, or explicitly report its absence.

## Progress

- [x] (2026-11-15) Read issue and inspected harness identity, runtime installation, ACP launch, configuration discovery, and checkpoint paths.
- [x] (2026-11-15) Installed isolated `@github/copilot@1.0.112`; user authorized account `mgasca` through device OAuth.
- [x] (2026-11-15) Real runtime initialize, session creation, prompt completion, and cancellation observed directly over ACP.
- [x] (2026-11-15) Verified session/load after process restart and session/set_config_option for allow_all=off.
- [ ] Verify restore from copied native state and authentication failures.
- [ ] Implement harness identity, credential staging, runtime availability, capability handling, and role restrictions.
- [ ] Add deterministic protocol integration tests and human setup documentation.
- [ ] Run isolated Mjolnir end-to-end tests, touched-crate suites, workspace clippy, and platform CI; commit and push validated changes.

## Surprises & Discoveries

Copilot 1.0.112 advertises `loadSession: true`, HTTP/SSE MCP, image and embedded-context prompts, and session close/list. It does not advertise session resume or additional directories. Existing `mj-worker/src/acp/session.rs` prefers advertised resume but otherwise calls load without checking load capability; review that decision point when adding support.

The free-tier account advertises only `auto`, twice, in both model lists. Configuration options are `mode` (agent, plan, autopilot using ACP URI values), `model`, and `allow_all` (on/off). Do not invent model or reasoning-effort choices. The user explicitly agreed that live discovery is authoritative.

The direct prompt returned text `ACP_OK` and `stopReason: end_turn`. Cancellation returned `stopReason: cancelled`, but chunks continued after an immediately submitted cancellation; investigate ordering with a pipe-based test that cancels after observing turn activity.

This Linux host has no system keychain. After user-authorized OAuth, Copilot stored its login in the isolated profile config. The directory is mode 0700 and config is mode 0600. Never include its contents in output or commits. GitHub CLI account `mgasca` cannot self-assign this issue or add its progress label; both operations were attempted and rejected by GitHub.

## Decision Log

Use live ACP configuration as the source of selectable options. This follows the issue requirements and explicit user agreement (2026-11-15).

Use version 1.0.112 for investigation, with `COPILOT_AUTO_UPDATE=false`. A fixed tested runtime is necessary because ACP remains preview. Whether to make it the shipped pin depends on recovery and authentication investigation.

Keep real-runtime investigation outside the repository and use an empty temporary workspace. Do not point a test Mjolnir build at the default instance.

## Outcomes & Retrospective

Investigation has demonstrated direct authenticated protocol operation, but Mjolnir integration is not implemented and acceptance is not complete. No claim of Mjolnir end-to-end validation is warranted yet.

## Context and Orientation

`mj-core/src/config/harness.rs` owns HarnessKind, home variables, authentication markers, execution enforcement, launch arguments, and native state directories. `mj-core/src/harness_runtime.rs` owns pins and fallback launch commands. `mj-core/src/harness_install.rs` resolves and installs runtimes. Controller profile staging and login live in `mj-controller/src/controller/profile_config.rs` and `mj-controller/src/profile_login.rs`. Worker ACP handshake and recovery live in `mj-worker/src/acp/session.rs`; request construction and additional-directory negotiation live in `mj-worker/src/acp/launch.rs`. Session configuration application lives in `mj-worker/src/acp/session_config.rs`. Native checkpoint scanning and restore live in `mj-checkpoint/src/checkpoint/`. Existing real-harness integration infrastructure is `tests/e2e/acp_real_harness.py`.

## Plan of Work

First complete a disposable protocol probe against the actual CLI. Restart it and load the recorded session, then repeat against copied session state in another isolated profile. Inspect configuration file structure without exposing token values. Determine credential export/staging behavior on keychain platforms before supporting those paths. Record supported package platforms separately from tested platforms.

Next add Copilot as a HarnessKind and implement its isolated home, CLI launch, version preflight, authentication readiness and staging. Choose a managed installation or explicit user-provided installation consistently across supported targets. If a pin is introduced, update the agent-dev image and package manifests in the same commit. Disable runtime auto-update so the tested bytes remain authoritative. Audit all exhaustive harness matches, serialized consumers, review/utility roles, and subagent selection. Omit or refuse unverified roles before launch.

Then extend ACP capability checks and native checkpoint interpretation using the observed protocol and file layout. Preserve live option lists, apply accepted settings before prompting, and never send unadvertised optional capabilities. Add protocol integration regressions using recorded runtime messages for startup failures, configuration application, completion, cancellation ordering, and load behavior. Publish human-facing setup and limitations under the existing docs site.

Finally run the real binary and worker in a named isolated instance and demonstrate prompt, cancellation, and recovery. Run touched-crate cargo test suites and workspace clippy in the dev profile, with cargo test outside the restricted sandbox. Commit only changed files on the current master branch, then push to origin/master. If upstream moved, fetch and merge rather than rebase.

## Concrete Steps

The installed runtime is `/tmp/mj-copilot-1511/node_modules/.bin/copilot`. Its isolated login is `/tmp/mj-copilot-1511/profile` and workspace is `/tmp/mj-copilot-1511/workspace`. Launch with COPILOT_HOME pointing to that profile, XDG_CACHE_HOME=/tmp/mj-copilot-1511/cache, and COPILOT_AUTO_UPDATE=false. Unset GH_TOKEN, GITHUB_TOKEN, and COPILOT_GITHUB_TOKEN in the child so the explicitly authorized login is used. Invoke `copilot --acp --stdio`. Initialize protocol version 1, then use session/new with the absolute temporary workspace and an empty mcpServers list.

The native session from the initial probe is `2bf6d9dd-9b0f-4066-846c-80993b15297b`. Its process has been stopped with SIGINT. The pipe-based `/tmp/mj-copilot-1511/recovery.py` probe successfully loaded that session after restart and applied allow_all=off. Still verify replay and continuity after restoring copied native state. Native files observed: session-state/<id>/events.jsonl, workspace.yaml, and checkpoints/index.md, plus a home-wide session-store.db. The config.json file is not strict JSON; do not assume serde_json can directly parse it. Use concurrent output draining and cancellation after observed activity; do not rely on a PTY for deterministic tests.

## Validation and Acceptance

A profile must report ready only with a usable runtime and login. Missing runtime, incompatible version, and missing login must have distinct actionable diagnostics. Advertised options must match the live account. A named instance such as `--instance copilot-1511` must create a real worker, stream a prompt response, cancel a running turn, and recover its native session after worker replacement. Run `cargo test -p <touched-crate>` for each touched crate and `cargo clippy --all-targets -- -D warnings`; dispatch nightly platform validation after platform-sensitive changes land. The direct protocol probe is investigation, not a substitute for this end-to-end acceptance.

## Idempotence and Recovery

Keep credentials and probe state outside version control. Stop owning processes before deleting their files. Temporary login removal must wait until tests no longer need it. Never revoke the user's account-wide authentication as test cleanup. Preserve unrelated working-tree changes and never switch branches.

## Artifacts and Notes

Runtime version output: GitHub Copilot CLI 1.0.112. npm package metadata identifies commit 24d885babba and native packages for Linux glibc/musl, macOS, and Windows on x64/arm64. These are package availability observations, not platform validation results.

## Interfaces and Dependencies

Reuse the existing ACP client and shared subprocess helpers. Do not add a workspace crate. Credential handling must stay inside the existing profile/worker staging boundary. Native data paths and runtime metadata must have one shared interpretation consumed by launch, readiness, and checkpoint code.

Revision 2026-11-15: recorded initial authenticated probe and implementation milestones; integration and recovery validation remain outstanding.
