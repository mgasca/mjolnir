# Add GitHub Copilot CLI as a managed ACP harness

This ExecPlan is a living document. The sections `Progress`, `Surprises & Discoveries`, `Decision Log`, and `Outcomes & Retrospective` must be kept up to date as work proceeds.

This document must be maintained in accordance with `.agents/PLANS.md`.

## Purpose / Big Picture

After this change a person can add a profile with `kind = "copilot"`, sign in with their GitHub account, and run Mjolnir sessions on GitHub Copilot CLI, GitHub's terminal coding agent. Copilot CLI speaks the Agent Client Protocol (ACP) itself when started as `copilot --acp`: ACP is JSON-RPC over the agent's standard input and output, through which Mjolnir's worker sends prompts, mode and option changes, permission answers and cancellation, and receives streamed text, thoughts, tool calls and token usage. On a bare target the worker installs the exact pinned Copilot CLI release binary, verified by SHA-256, into Mjolnir's managed harness cache; no Node.js is needed. Sessions stream through the TUI and web viewer, the Plan toggle works, writes and commands ask for approval on raw targets and run freely inside containers, Mjolnir's sub-agent tools are available, the session survives suspend and resume, and per-turn token usage is recorded.

To see it working: run `mj setup` and pick Copilot CLI under "Set up a managed agent", or add `[profiles.copilot] kind = "copilot" home = "~/.copilot"` to the instance config, run `mj login --profile copilot` (or export `COPILOT_GITHUB_TOKEN`), start a session on `localhost`, and prompt it.

The source project is https://github.com/github/copilot-cli. Version 1.0.105 (2026-10-28) is pinned. Its ACP support has been in public preview since 0.0.397 (2026-01-28).

## Progress

- [x] (2026-11-01 15:00Z) Probe Copilot CLI 1.0.105 over ACP in an isolated `COPILOT_HOME`: initialize, session/new, prompt, permission request, set_mode, set_config_option, session/load, MCP server handling, instruction sources, login storage.
- [x] (2026-11-01 15:20Z) Decide the design with the user: login home plus optional token, verified release binary, no guardian, sub-agents and setup wizard in scope.
- [x] (2026-11-01 15:35Z) Write this plan and `mj-core/assets/copilot/runtime.json`.
- [x] (2026-11-01 16:05Z) mj-core: `HarnessKind::Copilot` identity, authentication, enforcement, pin, installer, plan-mode surface, usage scope, credential parsing.
- [x] (2026-11-01 16:20Z) mj-controller: staging allowlist, migration 103, quota, utility model, import, SessionWiki, setup authentication check, TUI and settings lists.
- [x] (2026-11-01 16:20Z) mj-worker: launch fallback, Node-free launch, sub-agent MCP registration through `mcp-config.json`.
- [x] (2026-11-01 16:20Z) mj-checkpoint: capture and restore of `session-state/<id>/`.
- [x] (2026-11-01 16:40Z) Documentation, goldens, container notes, `AGENTS.md` pins section.
- [x] (2026-11-01 16:50Z) The workspace type-checks with every target (the machine had no Rust toolchain; rustup with the pinned 1.99.1 was installed into the user's home first).
- [x] (2026-11-01 17:15Z) Targeted tests pass in mj-core, mj-checkpoint, mj-worker, mj-controller, mj-tui and the CLI crate (fixtures that enumerate harnesses gained a Copilot entry: the macOS login-home test, the setup detection probe statuses, the worker plan probe's approval selector).
- [x] (2026-11-01 17:45Z) Live check in the isolated `copilot-live` instance on a local bare target, with the user's GitHub OAuth token stored as a plaintext Copilot login in the profile home: see Outcomes.
- [x] (2026-11-01 18:00Z) Full validation: workspace `cargo clippy --all-targets -- -D warnings` clean; full suites pass for mj-core (551), mj-checkpoint (169), mj-client (43), mj-worker (738), mj-controller (2226), mj-tui (563) and the CLI crate (230 plus its integration tests).
- [x] (2026-11-01 18:05Z) Committed locally on `claude/copilot-cli-mj-integration-1a8c02`; not pushed, at the user's request.

## Surprises & Discoveries

- Observation: Copilot CLI resolves credentials in this order: `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`, a stored login (OS keychain under service `copilot-cli`, or plaintext `COPILOT_HOME/config.json` when no keychain exists), and finally the output of `gh auth token`. A probe with no token and an empty `COPILOT_HOME` still answered, because it ran the user's `gh`.
  Evidence: `[rust:copilot_runtime::auth::gh] \`gh auth token\` process started` in the debug log of the no-token probe.
  Consequence: inside a Mjolnir session `gh` is Mjolnir's wrapper serving the session's GitHub App installation token, which Copilot cannot use, and the worker strips `GH_TOKEN`/`GITHUB_TOKEN`. Copilot therefore needs its own credential channel: the staged `config.json` login or `COPILOT_GITHUB_TOKEN`.

- Observation: the plaintext login file is `COPILOT_HOME/config.json`, which starts with two `//` comment lines ("User settings belong in settings.json." and "This file is managed automatically.") before the JSON object, and exists (with only `firstLaunchAt`) before any login. Logged in, it holds `authTokens` keyed by `https://github.com:<login>` and `loggedInUsers`.
  Evidence: `copilot login --with-token` into an isolated home after accepting plaintext storage.
  Consequence: the file's existence does not prove a login and `serde_json::from_slice` rejects it; Copilot-specific parsing strips the comment lines and checks `authTokens`.

- Observation: `session/new` exposes config options `mode` (values are URLs `https://agentclientprotocol.com/protocol/session-modes#agent`, `#plan`, `#autopilot`) and `allow_all` (`on`/`off`), the same three as `modes`. On the user's plan no `model` option appears: the model catalog has 43 entries but the plan allows one (`gpt-6-luna`), selected by Auto, and `--model`/`COPILOT_MODEL` with other ids were silently ignored. GitHub's own issue tracker (github/copilot-cli#4880) confirms a `category: "model"` select appears on plans with several models.
  Evidence: `session.auto_mode_resolved` event with `"availableModels":["gpt-6-luna"]`; probe transcripts 2026-11-01.

- Observation: Copilot rejects stdio MCP servers passed in `session/new` ("Rejecting non-http/sse MCP server"), while its `initialize` advertises only `http` and `sse` MCP capabilities. Stdio servers are accepted from `COPILOT_HOME/mcp-config.json` as `{"mcpServers": {"<name>": {"type": "local", "command", "args", "env", "tools": ["*"]}}}`.
  Evidence: WARNING line in the debug log; `copilot mcp add` output.

- Observation: Copilot reads a personal instruction file at `COPILOT_HOME/copilot-instructions.md` and personal skills at `COPILOT_HOME/skills/`, and reads the workspace `AGENTS.md` itself. `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` can add directories but is not needed.
  Evidence: `copilot instruction list` and `copilot skill --help`.

- Observation: a session's native state is `COPILOT_HOME/session-state/<uuid>/` with `workspace.yaml`, `events.jsonl`, `files/`, `checkpoints/`, `research/`, created at `session/new` before any prompt. While the process runs it also holds `inuse.<pid>.hold`, `inuse.<pid>.lock` and `.workspace-fork.lock`, and `session-state/.session-operation-locks/<uuid>.lock`. `session/load` from a different `cwd` succeeds and replays the history.
  Evidence: directory listings and a load from `ws2` of a session created in `ws`.

- Observation: `session/prompt` returns `usage` whose numbers sum the turn's model calls (a turn with two tool calls reported 38,079 input tokens with 32,923 cached reads), as ACP defines, and emits a non-standard `usage_update` notification with context-window `used`/`size`.

- Observation: GitHub releases carry `copilot-<platform>.tar.gz` per platform (each holding one file, `copilot`) and a `SHA256SUMS.txt`; the npm package `@github/copilot` is a thin loader over the same binaries. Copilot auto-updates by default on standalone installs; `COPILOT_AUTO_UPDATE=false` stops it.

- Observation: Copilot's `config.json` carries no timestamp, so two copies of a Copilot login cannot be ordered and `reconcile` answers `None` when they differ: a re-login reaches new sessions through staging, and running sessions keep the copy they started with. The rotated-login staging test records this for Copilot instead of expecting a push.
  Evidence: `reconcile` in `mj-core/src/credentials.rs` returns `SyncAction::None` for `(None, None)` freshness.

- Observation: three existing test fixtures enumerate harnesses implicitly and silently misbehaved with a ninth kind: the setup detection tests' fake executor returns success past the end of its status list, so Copilot's `copilot --version` probe "succeeded" and a Copilot home was offered; the worker's plan probe advertises each harness's own approval selector and timed out waiting for the `allow_all` setting it never offered.
  Evidence: `an_installed_dsh_is_offered_before_its_first_login` and six `plan_tests` failures before the fixture entries were added.

- Observation: the first live model-discovery probe timed out after the 300-second discovery budget while the managed install was still downloading the 129 MB release archive at about 650 KB/s (3 minutes 17 seconds for a direct `curl` of the same URL on this connection, and slower while two downloads competed). The daemon retried the install with backoff and a later probe succeeded from the cache. The same budget covers Grok's and MiMo's archives, so this is a property of slow links rather than of this harness; Kimi, Grok and MiMo install the same way.
  Evidence: `profile discovery failed error=timed out after 300 s` in the `copilot-live` daemon log; staging directory with a partial `copilot-linux-x64.tar.gz`.

- Observation: Copilot has an optional command sandbox (`--sandbox`, Microsoft Execution Containers over bubblewrap on Linux, Seatbelt on macOS). On Linux it needs `bwrap`, `slirp4netns`, iptables and `/dev/net/tun`, and when it is on but unsupported every command fails.

## Decision Log

- Decision: Name the variant `HarnessKind::Copilot`, identifier `copilot`, display name `Copilot CLI`, CLI `copilot`, home variable `COPILOT_HOME`, conventional home `~/.copilot`.
  Rationale: `copilot` is a fresh id in the store's `harness_kind` constraints; the display name distinguishes the CLI from Copilot's editor products.
  Date/Author: 2026-11-01 / Claude

- Decision: Authentication is the person's Copilot login in the profile home (`credential_file_name` = `config.json`; on macOS the login lives in the Keychain item `copilot-cli` and `keeps_login_in_home` is false, like Claude), with an optional `COPILOT_GITHUB_TOKEN` in the profile environment selecting `AuthScheme::ApiKey` instead. `COPILOT_GITHUB_TOKEN` in the environment Mjolnir started with is inherited like a Codex `env_key`. `GH_TOKEN` and `GITHUB_TOKEN` are never used; the worker strips them.
  Rationale: chosen by the user. The `gh auth token` fallback must never carry a session, because the session's `gh` serves a token Copilot cannot use.
  Date/Author: 2026-11-01 / Claude

- Decision: Install the pinned release tarball from GitHub, per platform with SHA-256, into the managed cache (`install_copilot`, `mj-core/assets/copilot/runtime.json`, entrypoint `bin/copilot`), the Kimi/Grok pattern. Containers install on demand the same way; the image gets no layer. `COPILOT_AUTO_UPDATE=false` is set on every launch and is a launch-owned variable.
  Rationale: chosen by the user; the binary needs no Node.js, the checksums are ours to verify, and auto-update would silently defeat the pin.
  Date/Author: 2026-11-01 / Claude

- Decision: Execution policy is realized with the `allow_all` config option on every open: `off` under configured approvals (every write, command and URL asks through Mjolnir), `on` when unconstrained. Copilot is not a guardian harness, so raw targets show the DANGER notice. `COPILOT_ALLOW_ALL` is launch-owned and refused in a profile's environment, because a launch-time allow-all baseline survives turning the option off.
  Rationale: chosen by the user; the bubblewrap sandbox is too demanding to require.
  Date/Author: 2026-11-01 / Claude

- Decision: Mjolnir's sub-agent and memory MCP servers reach Copilot through the staged home's `mcp-config.json` (`mcp_config_file`), written by the worker as Kimi's `mcp.json` is; `supports_delegation_tools` is true and `has_native_subagents` is true.
  Rationale: Copilot rejects stdio servers over ACP and accepts them from its config file.
  Date/Author: 2026-11-01 / Claude

- Decision: The profile instruction file is `copilot-instructions.md`, and `skills` is the synced skills directory.
  Rationale: both are Copilot's own personal locations under `COPILOT_HOME`, read with no extra environment.
  Date/Author: 2026-11-01 / Claude

- Decision: Plan mode maps onto the `mode` config option with Copilot's URL values; the surface gains a per-harness plan-mode id so the toggle, the active check and the config-pair check all use `…#plan` and `…#agent` for Copilot.
  Rationale: the existing helpers assume literal `plan`/`default` values.
  Date/Author: 2026-11-01 / Claude

- Decision: Token usage on the prompt response is recorded as per-turn (`UsageScope::Turn`), as for Claude.
  Rationale: ACP defines the field that way and the probe numbers sum the turn's requests.
  Date/Author: 2026-11-01 / Claude

- Decision: Checkpoints capture every regular file under `session-state/<id>/` except `inuse.*` and dot-files, and nothing under `.session-operation-locks`; restore rewrites nothing, because `session/load` takes its `cwd` from the request.
  Date/Author: 2026-11-01 / Claude

- Decision: Migration 103 admits `copilot` in both harness-kind constraints and raises the compatibility floor (breaking), as migration 92 did for DSH. Quota reads "not reported"; no utility model; import and SessionWiki say "not yet", like DSH.
  Date/Author: 2026-11-01 / Claude

## Outcomes & Retrospective

The harness is implemented and was exercised live in an isolated instance (`--instance copilot-live`) with a local bare target and the user's own GitHub account. Model discovery installed the pinned release archive from GitHub into the managed cache, verified against the digest in `runtime.json`, wrote the install receipt and listed Copilot's `model` option (only `auto` on this plan). A session created with `mj new` streamed `PONG` from the real Copilot CLI. The staged home carried the login (`config.json`), the personal instructions and Mjolnir's `mj-agents` server in `mcp-config.json`; the launch environment set `COPILOT_HOME` to the staged home and `COPILOT_AUTO_UPDATE=false`. Under configured approvals a prompt that wrote a file stopped at "Copilot CLI requests permission" showing the real shell command, `mj respond` with `allow_once` let it run, and the written file held the codeword from the staged `copilot-instructions.md`, proving the instructions were read. `mj suspend` produced a checkpoint whose native entries were exactly `session-state/<id>/events.jsonl`, `workspace.yaml` and `checkpoints/index.md` (no lock or `inuse.*` files); `mj resume` brought the same native session back and it recalled the first prompt's word. The session was destroyed and the daemon stopped.

Validation: targeted tests of every touched crate, then workspace clippy with `-D warnings` and the full suites of mj-core, mj-checkpoint, mj-client, mj-worker, mj-controller, mj-tui and the CLI crate, all passing.

Not exercised live: a container or EC2 target, the unconstrained `allow_all=on` policy, the Plan toggle through the TUI (its mapping is covered by `plan_exit_selects_each_harness_own_normal_mode`, and the probe accepted Copilot's URL mode ids over `session/set_mode` and `set_config_option`), a plan that offers several models, `mj login --profile copilot` through the ACP `terminal` method (the login was written with `copilot login --with-token` directly), and macOS Keychain detection (the `copilot-cli` service name comes from GitHub's documentation).

Remaining limitations: a re-login does not reach running sessions (Copilot's login carries no freshness); Copilot's premium-request quota is not reported; native import and SessionWiki indexing of Copilot sessions are not supported yet; the container image carries no Copilot layer, so a container session downloads the 129 MB archive on first use; a first model-discovery probe on a slow link can exceed the 300-second discovery budget and succeeds on the daemon's retry.

## Context and Orientation

`mj-core/src/config/harness.rs` defines the closed `HarnessKind` enum and every per-harness fact (identifier, display name, home variable, conventional home, credential file, instruction file, skills directories, execution enforcement, guardian support, native session directories, bridge arguments, MCP config file). `HarnessProfile` in the same file decides authentication (`auth_scheme`), readiness (`ensure_ready`) and environment inheritance (`inherit_provider_key`). Adding a variant makes every exhaustive `match` fail to compile; sites that use `==` or `matches!` must be found by searching for `HarnessKind::Dsh`.

`mj-core/src/harness_runtime.rs` holds the pinned versions, `pin()` and `fallback_launch()`. `mj-core/src/harness_install.rs` installs pinned runtimes into `~/.cache/mjolnir/harnesses/<id>/<install_id>` (`install_into`, `install_kimi`, `install_grok`, `extract_archive_file`). `mj-core/src/credentials.rs` reads credential files. `mj-core/src/acp/surface.rs` maps Plan mode onto each harness's controls. `mj-core/src/usage.rs` decides the usage scope.

`mj-controller/src/controller/worker_binary/harness.rs` stages a profile's allowlisted files into the per-session home (`profile_configuration_paths`). `mj-controller/src/database/schema.rs` migrates the store (`admit_harness_kind`). `mj-controller/src/setup.rs` detects installed agents and checks logins. `mj-controller/src/quota.rs`, `utility_llm.rs`, `import/native.rs`, `import.rs` and `sessionwiki/harness_adapters.rs` match on the harness. `mj-client/src/settings/schema.rs` lists profile kinds; `mj-tui/src/setup.rs` lists managed-setup harnesses.

`mj-worker/src/worker_runtime/subagents.rs` writes Mjolnir's MCP servers into staged homes (`configure_kimi_subagent_mcp`), called from `unix.rs`. `mj-worker/src/worker_runtime/harness_launch.rs` decides Node needs and ambient availability. `mj-checkpoint/src/checkpoint/native_scan.rs` and `restore.rs` capture and restore native files.

## Plan of Work

Core identity: add `Copilot` to `HarnessKind` and `ALL` (now nine). Facts: `COPILOT_HOME`, `.copilot`, `config.json`, `copilot`, `copilot-instructions.md`, `["skills"]`, no harness-owned skill paths, id `copilot`, display `Copilot CLI`, install advice naming npm, Homebrew and winget, `mcp_config_file` `mcp-config.json`, `native_session_dirs` `["session-state"]`, `bridge_args` `["--acp"]`, no extra entrypoint, no guardian, delegation tools yes, native sub-agents yes. `keeps_login_in_home` false on macOS. `launch_owned_environment`: `COPILOT_AUTO_UPDATE` and `COPILOT_ALLOW_ALL`. `configure_home_environment` sets `COPILOT_AUTO_UPDATE=false`. New `HarnessKind::optional_token_variable()` returns `COPILOT_GITHUB_TOKEN` for Copilot; `auth_scheme` returns `ApiKey` when the profile environment names it, `inherit_provider_key` inherits it, `authentication_marker` keeps the kind marker for that case, `ensure_ready` already refuses an unresolved key. Execution enforcement: `allow_all` `off`/`on`. `native_login_command` is `copilot login`; freshness and expiry are `None`. Add `credential_json(kind, bytes)` that strips Copilot's leading `//` lines, used by freshness, expiry, payload validation and login status; add `copilot_config_has_login(bytes)`.

Runtime: `COPILOT_VERSION = "1.0.105"`, pin `copilot-1.0.105` with entrypoint `bin/copilot`. `install_copilot` reads `assets/copilot/runtime.json` (`version`, `platforms.<os>-<arch>.{filename,sha256}`), downloads `https://github.com/github/copilot-cli/releases/download/v<version>/<filename>`, verifies, extracts `copilot` with `extract_archive_file`, sets mode 755. Add the asset to `mj-core/Cargo.toml` `include`. `fallback_launch` for Copilot execs `copilot --acp` from `PATH` or exits 127 with install advice. `harness_uses_node_runtime` stays false. Worker `ambient_native_harness_is_available` names `copilot`.

Surface: `execution_mode_id` returns Copilot's agent URL; new `plan_mode_id` returns its plan URL; `begin/finish_plan_mode_change`, `plan_mode_active`, `exact_config_has_pair` and `config_has_plan_pair` use it. Usage: `(Copilot, None) => Turn`. `error_reports_missing_native_session` is false.

Controller: staging allowlist `config.json`, `settings.json`, `mcp-config.json`. Migration 103. Quota `not_reported()`. Utility `None`. Import and SessionWiki not supported yet. Setup: a Copilot profile is authenticated when `config.json` holds `authTokens`, or on macOS when the Keychain has `copilot-cli`. TUI: add to `MANAGED_SETUP_HARNESSES` with description "GitHub Copilot subscription"; settings schema kind list; web viewer delegation list.

Worker: `configure_copilot_subagent_mcp` writes `mcp-config.json` with a `local` server for `hel worker subagent-mcp --socket <socket> --harness copilot --role <role>`; `unix.rs` calls it when a role exists.

Checkpoint: `is_copilot_session_artifact(relative, session_id)` selects `session-state/<session_id>/**` regular files whose name does not start with `inuse.` or `.`; restore leaves paths and bytes alone.

Image and docs: `containers/Containerfile.agent-dev` comment lists Copilot among on-demand harnesses; `AGENTS.md` Harness pins gets a Copilot bullet; docs tables (profiles, overview, targets, security, containers, custom images, quickstart, cli-reference, sessions, workspaces-bundles counts) and `mj-core/assets/skills/mj/references/configuration.md` gain Copilot; `tests/e2e/acp_real_harness.py` accepts `copilot`.

## Concrete Steps

Work from the repository root. Run every Cargo test outside the restricted sandbox. Focused checks while working:

    cargo test -p brokk-mj-core config
    cargo test -p brokk-mj-core surface
    cargo test -p brokk-mj-core copilot
    cargo test -p brokk-mj-checkpoint copilot
    cargo test -p brokk-mj-controller copilot
    cargo test -p brokk-mj-controller bridge_fallback_pins_match_the_agent_dev_containerfile
    cargo test -p brokk-mj-controller recent_revisions_upgrade_directly_and_preserve_user_data
    cargo test -p brokk-mj-worker copilot
    cargo test -p brokk-mj-tui settings

Final validation:

    cargo test -p <every touched crate>
    cargo clippy --all-targets -- -D warnings

Live check (inside a Mjolnir session the agent environment overrides `--instance`, so unset it):

    env -u MJ_CONFIG_DIR -u MJ_DATA_DIR -u MJ_SESSION_ID target/debug/mj --instance copilot-live daemon status

with an instance config holding `[profiles.copilot] kind = "copilot" home = "<isolated home with a plaintext login>"` and `[targets.localhost] kind = "bare"`.

## Validation and Acceptance

Acceptance is: a store at revision 102 upgrades to 103 keeping its sessions, and a fresh store admits `copilot`; a profile that sets `COPILOT_ALLOW_ALL` or `COPILOT_AUTO_UPDATE` fails to load; a Copilot `config.json` with comment lines and `authTokens` reads as signed in, and one without `authTokens` as signed out; a checkpoint keeps `session-state/<id>/events.jsonl` and `workspace.yaml` and drops `inuse.*` files; the plan toggle maps onto Copilot's URL mode values; and, live, a managed local bare session installs the verified binary, streams a reply, asks for permission before a command under configured approvals, shows Mjolnir's sub-agent tools, and resumes after suspend.

## Idempotence and Recovery

Managed harnesses use the existing content-addressed cache, staging directory, manifest receipt and lease; an interrupted install is removed on retry. The migration rebuilds the two tables in one transaction with foreign keys checked, and skips a table that already admits `'copilot'`. Live checks use an isolated instance only.

## Artifacts and Notes

Probe transcripts live in the session scratchpad; their essentials are recorded under Surprises & Discoveries.

## Interfaces and Dependencies

    pub enum HarnessKind { ..., Dsh, Copilot }
    pub const fn optional_token_variable(self) -> Option<&'static str>
    pub const COPILOT_VERSION: &str = "1.0.105";
    pub fn credential_json(kind: HarnessKind, bytes: &[u8]) -> Option<serde_json::Value>
    pub fn copilot_config_has_login(bytes: &[u8]) -> bool
    async fn install_copilot(staging: &Path, environment: &BTreeMap<String, String>) -> Result<()>
    pub(super) fn configure_copilot_subagent_mcp(root: &Path, session_id: &str, home: &Path, role: SubagentMcpRole) -> Result<()>
    pub(super) fn is_copilot_session_artifact(relative_path: &Path, session_id: &str) -> bool
