# Wasm / `dart2wasm` & VM Build Targets

Reference playbook for WebAssembly (`dart2wasm`), local Dart VM/SDK build
targets, test suites, and end-to-end Flutter Web AOT snapshot swapping.

## 1. `runtime` vs. `create_sdk` (`out/ReleaseX64/dart` vs. `out/ReleaseX64/dart-sdk/bin/dart`)

- Building `runtime` produces `out/ReleaseX64/dart` (`~20s`), **NOT**
  `out/ReleaseX64/dart-sdk/bin/dart`.
- `out/ReleaseX64/dart-sdk/bin/dart` (and `dart analyze` / `dart pub`
  subcommands) only exist if you build `create_sdk`. For fast `dart analyze` or
  `dart pub get` inside an SDK worktree without building `create_sdk`, use
  `tools/sdks/dart-sdk/bin/dart` or bare `dart` (`~/.local/bin/dart`).

## 2. Build Required Wasm Targets (`~38s`)

To run `dart2wasm` tests locally, build `dart2wasm` (the legacy
`compile_dart2wasm` target was renamed to `dart2wasm`, which compiles the
`dart2wasm` snapshot, platform dills, and `wasm-opt`):

```bash
python3 tools/build.py -m release -a x64 dart2wasm
```

## 3. Running Tests

Use `tools/test.py` with the `dart2wasm-linux-d8` or `dart2wasm-linux-chrome`
configurations:

```bash
python3 tools/test.py -n dart2wasm-linux-d8 language/exception/sync_throw_ref_test
```

- _Fast IR Golden Tests (`~2s`)_: Run `pkg/dart2wasm/test/ir_test.dart --src`
  using `tools/sdks/dart-sdk/bin/dart` to compile directly from `pkg/dart2wasm`
  source without rebuilding snapshots.

## 4. Testing Local `pkg/dart2wasm` Changes End-to-End in Flutter (`15s` AOT Snapshot Fast Path)

- **NEVER copy `out/ReleaseX64/dart2wasm.snapshot` or
  `out/ReleaseX64/dartaotruntime` into `~/github/flutter/bin/cache/dart-sdk/`**:
  - `ninja -C out/ReleaseX64 dart2wasm` embeds
    `-Dsdk_hash=<dart-sdk-worktree-hash>` into the snapshot, whereas
    `flutter build web --wasm` loads
    `~/github/flutter/bin/cache/flutter_web_sdk/kernel/dart2wasm_platform.dill`
    (compiled with Flutter's pinned `dart_revision`). CFE's `verifySdkHash` will
    throw `InvalidKernelSdkVersionError` in `_runCfePhase`, and overwriting
    `dartaotruntime` invalidates `frontend_server_aot.dart.snapshot`.
- **Fast Path (`~15s`, No Engine or Web SDK Rebuild Needed)**: As long as
  `Tag.BinaryFormatVersion` (`pkg/kernel/lib/binary/tag.dart`) matches Flutter's
  pinned Dart revision, compile `pkg/dart2wasm/bin/dart2wasm.dart` directly
  using **Flutter's cached `dart` binary** without `-Dsdk_hash` (which defaults
  `sdk_hash` to the `'0000000000'` wildcard and matches
  `~/github/flutter/bin/cache/dart-sdk/bin/dartaotruntime` ABI):
  ```bash
  ~/github/flutter/bin/cache/dart-sdk/bin/dart compile aot-snapshot \
    /path/to/dart-sdk/pkg/dart2wasm/bin/dart2wasm.dart \
    -o /tmp/dart2wasm_custom.snapshot
  ```
- Temporarily swap `/tmp/dart2wasm_custom.snapshot` into
  `~/github/flutter/bin/cache/dart-sdk/bin/snapshots/dart2wasm_product.snapshot`
  using an `EXIT` `trap` to restore the original backup automatically after
  `flutter build web --wasm` or `flutter_web_perf` completes.
