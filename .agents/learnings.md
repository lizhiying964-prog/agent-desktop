# Session learnings

## 2026-11-01 — Policies against real apps on Linux and Windows

- Linux: new harness leg `policies.owner_forbid_holds` (x11-openbox,
  headless and headed) on GNOME Calculator, verified through raw AT-SPI
  reads of the display; 2/2 passes, and a forbid on another id fails it.
  GTK4 under X11 reports screen extents (0, 0): build click points from
  `x11.client_origin` + `frame_extents` + window-relative extents, as the
  pointer legs do. The OrbStack VM needed gnome-calculator, openbox,
  xdotool, wmctrl, xclip, libglib2.0-bin, x11-apps, localsearch, nautilus,
  gnome-terminal, gnome-text-editor and gnome-control-center; Settings
  still never answers under openbox there, so warm-up was run without it
  through a throwaway wrapper. Suites retain their builds in
  `/run/user/<uid>` (a 1.2 GB tmpfs): clear them between runs.
- Windows: an ad-hoc interactive-session script (kept under
  `.compound-engineering/win-vm/run-policy-win.ps1`, not committed) on the
  WinForms fixture, read through separate `get` calls as the harness does
  (the managed UIA client in Windows PowerShell found no fixture elements
  on ARM64). 11/11 real, every refusal fails inverted. Traps: the fixture
  opens off a 1024x768 screen (move it with SetWindowPos first), status
  readouts are ValuePattern values not names, and a
  `C:\ProgramData\agent-desktop` lease folder created from SSH is owned by
  BUILTIN\Administrators, which makes interactive-session dispatch fail
  with `lease_directory_untrusted`; remove it first.

## 2026-11-01 — No more false "verified" text on macOS 27 SwiftUI fields

- `actions/swiftui_text.rs` runs before `set-value` and `type`: on macOS 27+
  (sysctl `kern.osproductversion`), an unfocused text field that reads
  `AXAutoInteractable` and `AXTraits` (SwiftUI-only; AppKit, Chromium lack
  them) is focused first when the policy allows focus, else refused
  unsent with `POLICY_DENIED` `swiftui_text_needs_focus`.
- `AXRespondsToInteraction` is listed on SwiftUI fields but fails to read
  (-25200), and `AXTraits` is a CFNumber: probe "reads successfully",
  not a boolean value.
- Verified through app-owned readouts: SwiftUI headless refused with the
  readout unchanged; `--headed set-value` reaches it; AppKit headless
  `set-value` still reaches the app (a throwaway AppKit app proved it);
  TextEditor unaffected. Disabling the guard makes the gate fail at
  set-value, so the E2E now treats a displayed-only write as a failure.
- Pre-existing flake, not from this change: `select combobox` in the
  interaction scenario returns ELEMENT_NOT_FOUND after the selection
  landed, 1 in 3 runs on the pre-guard build and 2 in 4 after.

## 2026-11-01 — macOS 27 SwiftUI text fields and the E2E gate

- Measured on macOS 27.0.1 with the fixture: a single-line SwiftUI
  `TextField` commits `AXValue`/`AXSelectedText` writes to the app's
  binding only while it has keyboard focus. Unfocused, `AXSelectedText`
  reports not settable (or is ignored even when settable), `AXValue`
  changes only the displayed value, `AXConfirm` commits nothing.
  `--headed focus` then `set-value`/`type`, or `--headed type`, reach it.
  Multi-line `TextEditor` fields are unaffected.
- Trap: `set-value` reports `delivered_verified` there, because its
  verification reads the AX value back, not the app's state. A value
  splice fallback for `type` would have been a false success too; it was
  written, measured and removed. Judge text entry by an app-owned readout.
- The gate (`safe-semantic.sh`) and `interaction.sh` treat those headless
  steps as a platform limit only when the outcome matches it exactly
  (`supported_action` refusal, or displayed-only write) and still fail
  otherwise; invert-checked. Gate: 3/3 passes; full suite then ran
  151 passed / 4 failed: the two text steps (now limits), an NT1
  notification permission for Script Editor (environment), and one
  `wait actionable` failure that passed 2/2 when the scenario ran alone.
- Fixture trick: `pkill -x AgentDeskFixture` between gate runs; a leftover
  instance or a frontmost app with no focused window makes the gate exit
  blocked, by design.

## 2026-11-01 — Owner folder from the account record

- `AGENT_DESKTOP_HOME` (or `HOME`) relocation moved the owner's policies
  folder with the state root, so an agent escaped every owner rule by
  setting one variable. The gate now also reads
  `~/.agent-desktop/policies` under the account home from the system
  record (`getpwuid_r(geteuid())` on Unix, `SHGetKnownFolderPath(
  FOLDERID_Profile)` on Windows), as `Origin::Account`; deduped against the
  state-root folder by canonical path so one folder never opens two
  stores in one process.
- Test seam: `discover_from(account_folder, ...)`; never write the real
  account folder from tests. Consequence: tests running as a developer run
  under that developer's policies (AGENTS.md says to keep it empty).
- Cross-checks from the Mac for windows/linux targets now fail early in a
  dependency's C build script (`psm`, no cross compilers); rely on the VMs.

## 2026-11-01 — Policy layers, store growth, perf A/B

- Security hole found while writing skills: `--policy`/`--policy-dir`/
  `AGENT_DESKTOP_POLICY_DIR` *replaced* the owner's folder (first match
  wins), so `--policy-dir permit-all` switched owner rules off; and a
  broken caller source made reads fail open past the owner. Now every
  source is a layer (`discover` returns `Discovered { layers, failures }`),
  each with its own `PolicyGate`/store; all must allow. A failed layer
  refuses mutations and is skipped for reads without lifting others.
- The macOS command host applied its *own* startup environment to every
  caller; the client now resolves `AGENT_DESKTOP_POLICY_DIR` and forwards
  it in `HostRequest.policy_env_dir`. Lesson: anything read from env in
  the gate must be resolved client-side.
- Residual, documented: relocating `AGENT_DESKTOP_HOME` (or HOME) moves
  the owner folder; same trust tier as editing the files.
- Log growth: the engine checkpoints/prunes after 64 events within one
  open engine; one-shot CLI processes append 2, so on Windows/Linux the
  log was never pruned (11 MB at 4,000 commands, 64 -> 87 ms per command
  in 1,000). `OpenStore::drop` now checkpoints once 64 events gathered
  across processes (`checkpoint.offset`): 0.2 MB, flat 59 ms.
- Perf A/B vs merge-base without policies: within about 3% everywhere
  (snapshot, get, is, click, set-value); `type` fails on both under macOS
  27 (headless type has no semantic path).
- Stranger dogfood (binary + embedded skills + 3 docs pages, repo denied)
  found 15 issues; dispositions:
  - Fixed with an invert-verified test: macOS hit test used
    `kCGWindowListOptionAll`, which is NOT z-ordered (only on-screen-only
    is), so a click on Calculator over Claude judged as TextEdit — an owner
    forbid bypass; the old live test used the same list as its oracle and
    so could never fail (grid test now compares against the stacked list).
    Menu bar (layer 24, window-server owned, cross-uid) now judged as the
    frontmost app. All flags formed one layer, so an agent's `--policy`
    permit lifted a harness `--policy-dir` deny-by-default. `install`
    rebirths every policy with empty history; the gate now diffs canonical
    statements (`ParsedPolicySet::parse(...).expanded_source()`) into batch
    verbs, and stores key by layer location, not file list. Refusals were
    `retryable: true` with a retry recovery; now `details.retryable: false`.
    Suggestions blamed "the owner" for harness layers; now per-layer plus
    `details.layer`. Empty-policies refusal hints at an earlier step. `status`
    reports layers. Empty/relative `AGENT_DESKTOP_POLICY_DIR` is
    INVALID_ARGS. Lowercase AppId also matches.
  - Fixed in docs, verified by running the block: setup `mkdir -p` left the
    state root 755 (snapshots then fail INTERNAL); temporal example was
    inert next to allow-all; folder permission rule; id provenance; real
    envelope; input group list; `--help` env var; macOS skill claimed
    `--app` takes only display names (bundle ids work).
  - Accepted: other cross-uid windows under a point still fail closed
    (safe); Dogwood validation messages name `policy_1` not the `@id`
    (upstream engine); a batch whose entries are all refused is top-level
    `ok: true` (existing batch contract, per-entry results).
- Benchmark/E2E trap: a `screenshot --screen 0` on macOS 27 raises a
  system "bypass the private window picker" dialog; keep it out of
  unattended runs.

## 2026-11-01 — Policies against live apps with Accessibility granted

- `tests/e2e/scenarios/policies.sh` drives the fixture headless and headed
  under `--policy-dir` sets and checks every verdict through the fixture's
  readouts (click-status, text-echo, secure-echo). 24/24 pass on macOS
  27.0.1; with the rules pointed at nothing, all 14 refusal checks fail
  with visible effects (click fires, `x` typed, password set).
- `POLICY_DENIED` is not only Dogwood: core actionability refuses with the
  same code (e.g. headless `type` with no semantic path). Live assertions
  must also check `error.details.kind == "dogwood_policy"`.
- The full harness cannot run on macOS 27.0.1 today: its first gate
  (`safe-semantic.sh`, headless AXSelectedText `type`) fails on `main` too,
  and its frontmost-identity checks flake. Run a single scenario through a
  scratch copy of `run.sh` without that gate.
- Accessibility is granted to the app that spawns the shell (Claude here),
  not the copied binary; `agent-desktop permissions --guide` opens the pane.

## 2026-11-01 — Policies judge the app itself, not the name typed

- The gate used the `--app` text verbatim as the resource, so `--app
  1password` or `--app com.1password.1password` slipped past a forbid on
  `"1Password"`, and window-scoped commands without `--app`
  (`minimize --window-id`, plain `snapshot`) were judged as `system`.
  Display names are also localized (System Settings is "Impostazioni di
  Sistema" on an Italian Mac), so name rules silently miss.
- Fix: every target is canonicalized before the verdict
  (`src/command_policy/dogwood_target.rs` `Resolver`): typed names go
  through `AppInfo::matches_identifier` over the running apps, then the
  platform's `list_apps_scoped` (aliases), then the new
  `find_installed_app`; refs and windows are named by the owning pid via
  the new cheap `list_app_identities`. The resource entity
  `Desktop::App::"<name>"` gets parent `Desktop::AppId::"<bundle id>"`
  (`EventBuilder::parents_for`), plus `context.input.app_id`.
- Cedar scope `resource in [..]` is invalid (only actions take a list in
  the scope); write `when { resource in [A, B] }`. The design doc had it
  wrong.
- macOS `list_apps` costs ~70 ms (ps + stabilization); the workspace
  snapshot alone (`workspace_apps::list_apps_inventory_until`) is the
  cheap identity source. macOS installed lookup mirrors launch:
  `URLForApplicationWithBundleIdentifier` for bundle-id-looking input,
  else `fullPathForApplication`, then `NSBundle` display name.
- Windows app names are image names (`notepad.exe`) with no bundle id;
  canonicalize the resolved path to get the on-disk letter case. Don't
  write temporal rules on `input.app_id`: empty on Windows, so every app
  compares equal.
- `CommandFacts` had 10 fields (rule: at most 7); the target part is now
  `TargetFacts`, reused as the binary's resolver output.
- Cost under policies (release, median): `--app` on a running or an
  installed app +7-8 ms, unknown names +82 ms (the platform alias lookup
  runs `ps` on macOS, so it is the last fallback), system targets
  unchanged. Ordering the fallbacks running -> installed -> alias search
  took the installed case from +86 ms to +7 ms.
- Benchmark pitfalls: an A/B policy must be valid for both schemas (a
  policy naming `Desktop::AppId` fails to install on the old binary, which
  then fails open for reads and reopens the store every command, ~40 ms);
  and `cp` over an existing signed binary on macOS gets it SIGKILLed
  (exit 137): `rm` it first.

## 2026-11-01 — Global input judged by the app it lands in

- Input naming no ref (`press`, `key-*`, `mouse-*`, `hover --xy`, `drag
  --*-xy`) used to be judged as `system`, bypassing app-scoped forbids.
  The gate now plans targets from arguments only (`dogwood_plan.rs`, the
  exhaustive command match), discovers policies, then resolves
  (`dogwood_target.rs`): focused window for keys, `--window-id` owner,
  window on top at the point for pointers, both ends of a drag.
  Undeterminable receiver -> POLICY_DENIED `dogwood_policy_target_unknown`.
- New core capability `window_at_point`. macOS: CoreGraphics list is
  front to back across all layers; skip alpha-0 windows and windows of
  our own executable (the cursor overlay ignores mouse events). Windows:
  `WindowFromPoint` -> `GA_ROOT` matched to `list_windows` ids (`w-<hwnd>`),
  else owner via `list_apps`. Linux: only answer when all windows under the
  point belong to one app (no stacking order on Wayland).
- Test pitfalls hit: two `Scenario`s alive in one test self-deadlock on the
  process-wide `HomeGuard` mutex (diagnose with `sample <pid>`); the UTM VM
  screen is 1024x768 and Windows fixtures can open off-screen, where
  `WindowFromPoint` is unspecified — move fixtures on screen first.
- Cross-checks from the Mac (`cargo check --target
  aarch64-pc-windows-msvc` / `aarch64-unknown-linux-gnu`) catch compile
  errors but not clippy; Windows clippy found a `clone_on_copy` only there.

## 2026-11-01 — Concurrent sessions sharing the policy store

- redb takes a single-process lock and the macOS command host held the
  gate (and the store) for its 15-minute life, so a second session's host
  failed closed with `dogwood_policy_unavailable`. Reproduced in-process
  (two gates on one store) and end to end (two `session start` ids).
- Reopening per decision is correct but slow: release-build profile is
  redb open ~4 ms, engine recovery ~12 ms, reloading installed policies
  ~35 ms, redb close ~16 ms. Measure before choosing; the design doc's
  "microseconds" guess was off by four orders of magnitude.
- The fix keeps the store open behind core's `FileLock` and hands it
  over on request: waiters refresh `store.wanted`, the holder's watcher
  thread drops the store between decisions. Single session ~16 ms median
  (same as before), interleaved sessions ~108 ms per command.
- Release binary with the policy gate is 7.0 MB (CI cap 15 MB).
- Benchmark harness pattern: build old/new binaries by `git stash push --
  <files>`, copy each `target/release/agent-desktop` aside, then time N
  sequential CLI runs under a fresh AGENT_DESKTOP_HOME.

## 2026-11-01 — Cross-OS verification (Linux and Windows VMs)

- Running the branch on other OSes caught five real defects macOS alone
  hid: the policy crate was in no CI package list; the MSRV lane still
  pinned 1.89; tests pin ci.yml text — src/cli/contract_tests.rs pins
  the Windows clippy command, and two Windows-crate tests find the step
  named exactly "Core and Windows unit tests" with its exact cargo
  command (add new crates as separate steps, never rename that one);
  a Unix-only fn body left an unused variable on Windows; clippy 1.95's
  manual_take fired on Windows-only code.
- Run the repo's own gates, not just fmt/clippy/tests:
  scripts/check-rust-file-size.sh (400-line cap, no non-doc comments),
  check-architecture.sh, check-no-phase-references.sh. The pre-commit
  hook runs them; enable it per clone with
  `git config core.hooksPath .githooks`.
- macOS `list-apps` timing out ("ps: timed out draining stdout") under
  host load also happens on the pre-branch build, so it is environmental;
  policy process tests now use permission-free, fast `list-displays`.
- Linux VM: OrbStack (`orb create ubuntu`), repo mounted at the same
  path; run lanes under `dbus-run-session -- xvfb-run -a`.
- Windows 11 ARM64 VM on UTM, scripted via AppleScript: removable drives
  attach as CD-ROM (Windows can't read FAT there); extra QEMU devices
  need `bootindex=` or ArmVirtQemu firmware never connects them; QEMU
  arguments can carry `file urls` for sandbox access. Boot WinPE from a
  FAT32 NVMe image (no "press any key"), install from the ISO as a CD.
  UTM forwards guest input only to a frontmost window.
- Shipping a tree to Windows: `COPYFILE_DISABLE=1 tar --no-mac-metadata`,
  or Windows tar extracts AppleDouble `._*.rs` files that break the
  command-surface contract tests. UI Automation and capture tests need an
  interactive session: run them via `schtasks /ru <user> /it`, not SSH.
  In the interactive session 9 adapter tests and the FFI live round-trip
  (`ad_execute_by_ref` -> ErrInternal) still fail identically on the
  pre-branch commit (shell surfaces, interaction-lease SIDs, VM
  clipboard): environment, not regressions. When polling a done-marker
  after `schtasks /run`, also check no cargo process is alive: a stale
  marker from the previous run satisfies the first poll.

## 2026-10-31 — Flattened policy layout

- Policy rules moved from `~/.agent-desktop/policy/policies/` to flat
  `~/.agent-desktop/policies/`, engine state to a sibling `policy-state/`.
  The nested shape failed its own hardening story: `chflags uchg` on the
  shared `policy/` root would freeze the engine's event log, which must
  stay writable. The lockable unit is exactly the rules folder, so the
  layout now separates owner-edited from engine-owned at the top level,
  matching the `snapshots/` naming precedent.
- `PolicyGate::open_at` now takes the engine state dir directly (no
  implicit `/state` join); `policy_root()` became `engine_state_dir()`.

## 2026-10-31 — Real-case scenario tests for the policy gate

- Scenario testing caught two real defects: (1) reloading an edited policy
  set opened a second engine on the same redb store while the cached gate
  still held its single-process lock — drop the cached gate before
  reopening; (2) the documented secure-field rule could never match: a
  password field's canonical role is plain `textfield` (AXSecureTextField
  normalizes to it), so secure-ness must be read from the ref's `secure`
  state token — now a dedicated `secure: Bool` fact.
- Test placement matters for the GATE process-static: in-process scenario
  tests pass policies via `CommandContext::with_policy_paths` (never the
  default folder), so concurrently running tests with empty policy paths
  can't discover them; `HomeGuard` serializes the rest.
- Assertions against the real binary must not assume OS grants: a command
  that passes the gate can still fail with PERM_DENIED/AX errors on an
  unprivileged machine — assert "not POLICY_DENIED", not "ok".
- Keep process tests side-effect-free: list-apps/list-windows/version and
  denied mutations only; an allowed clipboard-set/clear in a test clobbers
  the developer's real clipboard even under an isolated AGENT_DESKTOP_HOME.

## 2026-10-31 — Dogwood gate implementation (branch dogwood-policies)

- The real enforcement choke point is `dispatch::dispatch`, not the main.rs
  preflights: batch items on non-macOS (and in tests) call dispatch directly,
  bypassing `execute_with_adapter`. Policy paths therefore travel on
  `CommandContext` (new `PolicyPaths` in core), which batch item contexts and
  host-forwarded commands inherit for free.
- dogwood crates (1.1.x) require newer Rust than the repo pinned: redb wants
  1.90 and dogwood-language uses if-let guards, stable only since 1.95
  (Apr 2026). Toolchain bumped 1.89 → 1.95; that surfaced clippy's new
  `result_large_err` (allowed workspace-wide, AdapterError-by-value is the
  repo contract) and two small macOS-adapter lints, one a clippy false
  positive on CFArray's inherent `into_iter`.
- Dogwood engine facts: default event schema pins temporal matching to
  `callerPrincipal`; `context.<group>` reads come from `request_context`,
  temporal patterns from `field` — the same value must go to both. Response
  kind "response" vs "error" distinguishes outcomes without an output field.
- Permission preflight still runs before the gate, so an unprivileged binary
  reports PERM_DENIED before POLICY_DENIED — policy denial is verifiable
  e2e with permission-free commands (list-apps --app X).
- Smoke-testing note: `clipboard-clear` under an isolated AGENT_DESKTOP_HOME
  still clears the real system clipboard; avoid mutating commands in smokes.

## 2026-10-31 — Dogwood policy design

- Repo cloned from vercel-labs/agent-desktop into this folder's root (was empty).
- Decision: propose embedding AWS's Dogwood Local Engine (`dogwood-local-engine`
  + `dogwood-language`, Rust, redb-backed) as a user-owned guardrail layer;
  design captured in `docs/dogwood-policies-design.md`.
- Key codebase facts discovered:
  - Single enforcement choke point: `execute_with_adapter` in `src/main.rs`
    (permission preflight in `src/command_policy/mod.rs` → `dispatch::dispatch`);
    batch items also pass through it.
  - On macOS most commands forward to a persistent per-namespace command host
    (`src/command_host/`, 15-min idle exit) — natural home for a single-writer
    temporal engine; host already serializes requests.
  - State root is `~/.agent-desktop` or `$AGENT_DESKTOP_HOME`
    (`crates/core/src/storage/state_root.rs`), with user-ownership validation —
    same pattern proposed for policy-file loading.
  - `.claude` in this repo is a symlink to `.agents`.
- Dogwood facts: Cedar superset (deny-by-default, forbid > permit), `when
  temporal` clauses over an ordered request/response event log; engine issues
  verdicts only, harness enforces; reference interpreter not hardened for
  production enforcement; rolling limits can be exceeded by concurrent in-flight
  requests (mitigated by host serialization).
- Owner policies are enforced only in the binary's `dispatch()`; `crates/ffi` does
  not depend on `agent-desktop-policy`, so the C library is never gated. Every
  trust-boundary doc (README, docs/policies.md, the guide, the FFI skill and page)
  must say so, or an owner assumes a library-loading program is bound.
- Statics are never dropped, so anything that must run on close (the policy
  store checkpoint) needs an explicit release before a one-shot process exits;
  `main()` calls `command_policy::dogwood::release_stores()`. The macOS host
  calls `crate::execute` per request, so the release belongs in `main()`, not
  in `execute`.
- OrbStack's mount can hide a macOS-side edit from cargo's mtime check: a
  "Finished in 0.06s" after an edit means a stale binary. `touch` the file
  inside the VM before an invert-verify run there.
- When the SSH agent cannot sign (`communication with agent failed`), push
  once over HTTPS with `git -c credential.helper= -c
  'credential.helper=!gh auth git-credential' push https://github.com/<fork>.git
  <branch>`, which changes no config.
- The live `window_at_point` tests meet WindowServer windows (another user);
  the hit test refuses those by design, and `kill(pid, 0)` returning `EPERM`
  is the independent oracle for "another user's process".
