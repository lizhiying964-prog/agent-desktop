# agent-desktop-ffi

A C-ABI library over the [`PlatformAdapter`](../../crates/core/src/adapter/mod.rs) trait. It builds a shared library for C, C++, Python (ctypes), Swift, Go (cgo) and Node (ffi-napi). Calls run in process, so a program does not start the CLI for each command.

The full guide is at https://agent-browser.dev/desktop/ffi. It covers linking, the API, ownership and threading. The agent-facing version is the [`agent-desktop-ffi` skill](../../skills/agent-desktop-ffi/SKILL.md).

## Build

```bash
cargo build --profile release-ffi -p agent-desktop-ffi
```

The output is in `target/release-ffi/`: `libagent_desktop_ffi.dylib` on macOS, `libagent_desktop_ffi.so` on Linux, and `agent_desktop_ffi.dll` on Windows.

Use `--profile release-ffi`, not `--release`. The `release` profile sets `panic = "abort"`, which turns off the `catch_unwind` traps at the `extern "C"` boundary without any error. The `release-ffi` profile keeps `panic = "unwind"`.

A source build also needs the platform helper beside the library: `agent-desktop-macos-helper` on macOS, `agent-desktop-linux-clipboard-owner` on Linux.

For CI and tests that need no accessibility permission, build the stub adapter. It returns `PLATFORM_NOT_SUPPORTED` for every adapter call.

```bash
cargo build --profile release-ffi -p agent-desktop-ffi --features stub-adapter
```

## Link and load

The header is `crates/ffi/include/agent_desktop.h`. cbindgen generates it, so do not edit it. Regenerate it with `scripts/update-ffi-header.sh`.

After loading the library, call `ad_init(AD_ABI_VERSION_MAJOR)`. It returns `AD_RESULT_ERR_INVALID_ARGS` when the header and the library disagree on the ABI major version. The header marks the call as recommended, but skipping it leaves a mismatch undetected.

On macOS the install name is `@rpath/libagent_desktop_ffi.dylib`. A linking program needs `-rpath` pointing at the library directory. `crates/ffi/build.rs` sets only that install name.

ABI 6 and `AD_EXACT_REF_ENTRY_VERSION` 2 add `bounds_space` to `AdRefGeometry`, inside `AdExactRefEntry`. It is an `AD_BOUNDS_SPACE_*` constant that names the frame of `bounds`. Bounds are screen-relative on macOS and Windows and window-relative on Linux.

Command-backed JSON entrypoints such as `ad_snapshot` and `ad_wait` return the CLI JSON envelope. `ad_execute_by_ref` and `ad_wait` accept only snapshot-qualified refs (`@<snapshot_id>:eN`).

## Threads and errors

Any host thread may call the library. A native handle (`AdNativeHandle`) is thread-affine. Resolve, use and free it on the same thread, with the adapter that created it.

Mutations, such as ref actions, clipboard writes and app or window changes, run one at a time across threads and processes for the same OS user, under an interaction lock. Reads can overlap them.

A failing call sets a thread-local last error. Read it with `ad_last_error_code()`, `ad_last_error_message()` and `ad_last_error_suggestion()`. A pointer from these accessors stays valid until the next failing call on the same thread. Successful calls do not change it.

For the ownership rules and the full error table, see https://agent-browser.dev/desktop/ffi/ownership-and-threading.

## Source layout

Each command-backed JSON entrypoint has one canonical module under `crates/ffi/src/commands/`.
