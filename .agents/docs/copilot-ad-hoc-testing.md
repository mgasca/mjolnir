# Copilot CLI harness: ad hoc testing runbook

Agent-facing runbook for exercising the Copilot CLI harness (`kind = "copilot"`,
added in commit `2616c1968`, plan `.agents/plans/add-copilot-cli-harness.md`) by
hand. Every scenario runs in a named instance so the person's own Mjolnir, its
daemon and its sessions are never touched. Each scenario names what to run,
what to look for, and where the evidence is.

## Prerequisites

- A build of this checkout: `cargo build -p brokk-mjolnir -p brokk-mj-worker`
  (debug is fine). Below, `MJ="target/debug/mj --instance copilot-test"`.
- A GitHub account with Copilot. The Free and Pro plans expose one model
  (`gpt-6-luna` through Auto); Pro+, Business and Enterprise expose several.
- A way to sign the test profile in. Either a token to paste (see "Getting
  a token" below) or a browser on this machine for Copilot's own login flow.
- A shell that is not pointed at the person's own instance. A shell opened
  inside a Mjolnir session (a coding agent's terminal, for example) inherits
  `MJ_CONFIG_DIR`, `MJ_DATA_DIR`, `MJ_SESSION_ID` and `MJ_SESSION_BASH_ENV`,
  and the first two override `--instance`, so every `mj` command below would
  act on the live instance instead of the test one. Check and clear them at
  the start of the shell, once:

      env | grep '^MJ_'                                   # anything printed here must go
      unset MJ_CONFIG_DIR MJ_DATA_DIR MJ_SESSION_ID MJ_SESSION_BASH_ENV
      env | grep '^MJ_' || echo "no MJ_ variables; safe to use --instance"

  `unset` affects only the current shell, so repeat it in every new terminal
  or tmux window used for the test, or add it to the top of a test script.
  Alternatively run each command with them removed:
  `env -u MJ_CONFIG_DIR -u MJ_DATA_DIR -u MJ_SESSION_ID -u MJ_SESSION_BASH_ENV target/debug/mj --instance copilot-test ...`.
  A plain terminal outside any session has none of these set and needs
  nothing.

## Instance and profile home

Create `~/.config/mjolnir/instances/copilot-test/config.toml`:

    version = 19

    [targets.localhost]
    kind = "bare"

    [profiles.copilot]
    kind = "copilot"
    home = "/path/to/copilot-test-home"

Prepare the profile home. Copilot CLI keeps its login in the OS keychain
when there is one, and otherwise (Linux without a secret service, WSL) asks
to store it as plaintext in `COPILOT_HOME/config.json`. Mjolnir stages that
file into every session, so for this runbook the plaintext file is what you
want on Linux. `copilot` in the commands below is any Copilot CLI 1.0.105:
the managed one Mjolnir installs at
`~/.cache/mjolnir/harnesses/copilot/copilot-1.0.105/bin/copilot` after the
first discovery (scenario A), or `npx -y @github/copilot@1.0.105`, or a
global `npm install -g @github/copilot@1.0.105`.

Getting a token. Copilot accepts three kinds, and refuses classic `ghp_`
personal access tokens:

1. The GitHub CLI's own OAuth token, the quickest if `gh` is already signed
   in on this machine (`gh auth status` says so; otherwise `gh auth login`
   first). Print it and copy it to the clipboard in one step:

       gh auth token | clip.exe        # WSL
       gh auth token | pbcopy          # macOS
       gh auth token | xclip -sel c    # Linux with X11

   or just run `gh auth token` and copy the `gho_...` line it prints. It is
   a credential for your account; do not paste it into a chat, a log or a
   commit.
2. A fine-grained personal access token from
   https://github.com/settings/personal-access-tokens/new: resource owner
   your own account, no repository access needed, and under "Account
   permissions" grant "Copilot Requests". Copy it once at creation; GitHub
   does not show it again.
3. A token from Copilot's own login (option B below), which needs no copying.

Option A, paste a token:

    mkdir -m 700 /path/to/copilot-test-home
    COPILOT_HOME=/path/to/copilot-test-home copilot login --with-token

   Copilot prints "Paste your authentication token:"; paste it and press
   Enter (nothing is echoed). It then asks "System keychain unavailable.
   Store token in plaintext config file? (y/N)"; answer `y`. Expect "Signed in
   successfully as <login>" and a `config.json` in the home containing
   `authTokens`. Run it in a real terminal: the token prompt reads standard
   input, but the y/N question needs a terminal, so piping the token in
   (`gh auth token | copilot login --with-token`) logs in and then reports
   "the token was not saved".

Option B, Copilot's browser or device-code login, no token handling:

    mkdir -m 700 /path/to/copilot-test-home
    COPILOT_HOME=/path/to/copilot-test-home copilot login

   On a desktop it opens the browser; over SSH it prints a device code to
   enter at https://github.com/login/device. Answer `y` to the same plaintext
   question. This is also what `mj login --profile copilot` runs for you once
   the profile exists (scenario G), so you can skip preparing the login here
   and sign in through Mjolnir instead.

Either way, check the result without printing the token:

    grep -c authTokens /path/to/copilot-test-home/config.json    # expect 1

Write the canary instruction scenario D checks. This overwrites any existing
`copilot-instructions.md` in the home, so use it only on a test home. Copy and
paste the whole block; it is plain `printf`, so it survives the indentation of
this document:

    export COPILOT_TEST_HOME=/path/to/copilot-test-home
    printf '%s\n' '# Test instructions' 'When asked for the staged codeword, answer exactly STAGED-HOME-7731 and nothing else.' > "$COPILOT_TEST_HOME/copilot-instructions.md"
    grep -c STAGED-HOME-7731 "$COPILOT_TEST_HOME/copilot-instructions.md"    # expect 1

Do this before creating the session: staging copies the home when a session
starts, so a session that is already running keeps the instructions it began
with.

Where things live afterwards: instance data under
`~/.local/share/mjolnir/instances/copilot-test/`; per session,
`workers/<session>/launch.json` (environment the harness got), `profile/` (the
staged home: `config.json`, `copilot-instructions.md`, `mcp-config.json`,
`session-state/<uuid>/`), `relay-journal/active.jsonl` (every ACP update) and
`worker.log`; daemon logs under `logs/`; Copilot's own debug logs under the
staged home's `logs/`. Checkpoints are `sessions/<session>-*.hel.zip`.

## A. Discovery and managed install (bare localhost)

    $MJ models --profile copilot --json

Expect the daemon to start, download `copilot-linux-x64.tar.gz` (129 MB) from
GitHub, verify it against `mj-core/assets/copilot/runtime.json`, and leave
`~/.cache/mjolnir/harnesses/copilot/copilot-1.0.105/{bin/copilot,mj-harness.json,.lease}`.
The answer lists the `model` option: `auto` alone on Free/Pro, the plan's
models otherwise. On a link slower than about 450 KB/s the first call fails
with "timed out after 300 s while discover profile configuration"; the daemon
retries the install with backoff, and a second call a few minutes later
answers from the cache. A wrong digest in `runtime.json` must fail the install
loudly; do not expect a fallback.

## B. First session and streamed reply

    $MJ workspaces create test --json
    SID=$($MJ new --workspace test --profile copilot --target localhost \
          --project-directory /path/to/a/git/checkout --json \
          "Reply with exactly the word PONG and nothing else." | python3 -c 'import sys,json;print(json.load(sys.stdin)["session_id"])')
    $MJ wait --session $SID --timeout 240 --json
    $MJ transcript --session $SID

Expect `PONG` as an agent entry. In `workers/$SID/launch.json` the environment
holds `COPILOT_HOME=<staged home>` and `COPILOT_AUTO_UPDATE=false`, and no
`GH_TOKEN` or `GITHUB_TOKEN`. `relay-journal/active.jsonl` ends the turn with
`"stopReason":"end_turn"` and a `usage` object; `$MJ usage --session $SID`
records it as per-turn usage. A `usage_update` notification with
`used`/`size` also appears in the journal; it is Copilot's context-window
gauge and is not an error.

## C. Permission requests under configured approvals

Bare localhost preserves configured approvals, which selects Copilot's
`allow_all` option `off` on every session open.

    $MJ prompt --session $SID --return-on-input --timeout 240 --json \
        "Using the shell, create a file named proof.txt containing the word hello."
    $MJ elicitations --session $SID --json
    $MJ respond --session $SID --elicitation tool-permission-1 \
        '{"action":"accept","content":{"choice":"allow_once"}}'
    $MJ wait --session $SID --timeout 240 --json
    $MJ fetch --session $SID --path proof.txt --stdout

Expect `outcome: input_required`, one elicitation titled "Copilot CLI requests
permission" whose message shows the real shell command (Copilot's own title is
a summary; the command comes from the request's `rawInput`), choices
`allow_once`, `allow_always`, `reject_once`, and the file afterwards. Answer
`reject_once` in a second attempt and expect no file and a turn that ends
without it. Reads outside the workspace ("Access paths outside trusted
directories") and URL fetches ask the same way.

## D. Personal instructions and skills

Ask "What is the staged codeword?" and expect exactly `STAGED-HOME-7731`, the
canary written into the profile home's `copilot-instructions.md` in the
profile-home step, which staging copied into the session home. If it answers
that it does not know, the file was written after the session started or into
a different home than the profile's `home`; `diff` the profile copy against
`workers/<session>/profile/copilot-instructions.md`.
Put a skill at `<profile home>/skills/<name>/SKILL.md` and expect Copilot to
list it on `/skill list` inside the TUI or use it when asked. Copilot also
reads the workspace's own `AGENTS.md`; that is Copilot's behavior, not
Mjolnir's staging.

## E. Unconstrained policy on a container

Add a container target to the instance config (Podman or Docker, see the
Containers docs) and create a session on it. Containers run unconstrained,
which selects `allow_all` `on`: the same file-writing prompt as in C must
complete with no elicitation. The container has no Copilot layer, so the first
session on a fresh image downloads the archive into the container user's
cache; the provisioning stage reads "Installing Copilot CLI". A profile that
sets `COPILOT_ALLOW_ALL` or `COPILOT_AUTO_UPDATE` in its `environment` must
fail to load with a message naming the variable and who owns it.

## F. Token authentication instead of a login

Make a second profile whose home has no `config.json` login and set the token
in its environment, either directly or from `secrets.toml`:

    [profiles.copilot-token]
    kind = "copilot"
    home = "/path/to/empty-copilot-home"
    environment = { COPILOT_GITHUB_TOKEN = { from_secret = "COPILOT_GITHUB_TOKEN" } }

`$MJ doctor` and the Settings login status must show the profile as needing
no login ("COPILOT_GITHUB_TOKEN API key"); a session must answer. Remove the
secret and expect `mj new` to be refused before launch with a message that
says where to set the variable. Also confirm that exporting only `GH_TOKEN`
before starting `mj` does not authenticate a Copilot profile: the worker
strips it, and a profile with neither login nor `COPILOT_GITHUB_TOKEN` fails
at `session/new` with Copilot's "Authentication required".

## G. `mj login` through the ACP terminal method

With a profile home that has no login:

    $MJ login --profile copilot

Copilot advertises a `copilot-login` method carrying a terminal command, so
Mjolnir runs the managed `copilot login` with `COPILOT_HOME` set to the profile
home: a browser flow on a desktop, a device code over SSH. On Linux without a
keychain answer `y` to plaintext storage so the login lands in `config.json`.
Afterwards `$MJ doctor` reports the profile signed in. On macOS the login goes
to the Keychain item `copilot-cli` and the profile reads signed in through
`security find-generic-password -s copilot-cli`; this path was not exercised
during development and is the first thing to verify on a Mac.

## H. Plan mode

In the TUI, toggle Plan on a Copilot session and prompt. Copilot's mode ids
are URLs (`https://agentclientprotocol.com/protocol/session-modes#plan` and
`#agent`); the journal shows `session/set_config_option` on `mode` with those
values, and `current_mode_update` notifications echo them. From the CLI,
`$MJ set-config --session $SID` lists the `mode` and `allow_all` options and
`$MJ set-config --session $SID --key mode --value <plan URL>` switches it.

## I. Sub-agents

Create the session with `--subagents mjolnir` (needs `[subagents.straightforward]`
configured in the instance). The staged home's `mcp-config.json` must hold an
`mj-agents` server of type `local` running `hel worker subagent-mcp --harness
copilot --role parent`. Ask Copilot to list its tools or to spawn a sub-agent
for a trivial task, and watch the child appear in `$MJ sessions`. Calls to the
`mj-agents` tools under configured approvals currently ask like any other
tool; if that proves too noisy, extend `OwnedMcpApprovals` in
`mj-worker/src/acp/permissions.rs` with Copilot's permission title shape, as
DSH has.

## J. Suspend, resume, move

    $MJ suspend --session $SID --acknowledge-unpublished-work --json
    python3 -c 'import zipfile,sys;print("\n".join(n for n in zipfile.ZipFile(sys.argv[1]).namelist() if "native" in n))' \
        ~/.local/share/mjolnir/instances/copilot-test/sessions/$SID-*.hel.zip
    $MJ resume --session $SID --json
    $MJ prompt --session $SID --timeout 240 --json "What word did I ask you to reply with first? Answer with the word only."

Expect the checkpoint's native entries to be `session-state/<uuid>/events.jsonl`,
`workspace.yaml` and `checkpoints/index.md`, never `inuse.*` or `.workspace-fork.lock`,
and the resumed session to answer `PONG`. Resume refuses while the suspend is
still in progress; poll `$MJ sessions --session $SID --json` for `suspended`
first. A move to another target (`$MJ move`) restores the same native state;
`session/load` takes the new directory from Mjolnir, so no path rewrite is
involved.

## K. Model selection on a richer plan

On a plan with several models, `$MJ models --profile copilot --json` lists
them, `$MJ new --model <id>` starts a session on one, `/model` in the TUI
switches, and `relay-journal` shows `config_option_update` with the new
`model` value. On Free/Pro only `auto` exists and a requested model is
silently ignored by Copilot; that is the plan, not the harness.

## L. Setup detection and doctor

With `~/.copilot` or the `copilot` command present and no profile configured,
`$MJ setup` offers a Copilot CLI profile; it reads signed in only when
`config.json` holds `authTokens` (or the Keychain holds `copilot-cli`), never
merely because `config.json` exists, which it does from Copilot's first launch.
`$MJ doctor` names Copilot CLI among the agents it looks for.

## Cleanup

    $MJ destroy --session $SID --json
    $MJ daemon stop

Then remove `~/.config/mjolnir/instances/copilot-test/` and
`~/.local/share/mjolnir/instances/copilot-test/` if the instance is no longer
wanted. The managed install under `~/.cache/mjolnir/harnesses/copilot/` is
shared with every instance on the machine; leave it.
