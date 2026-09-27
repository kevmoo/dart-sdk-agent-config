# Workspace Rules: Dart SDK Sandbox Layout

This workspace uses a specialized **Bare Repository + Sandbox Worktree** layout
to optimize disk space, handle `gclient` cleanly, and keep the root directory
pristine.

All AI agents operating in this workspace MUST adhere to the following rules.

---

## 📖 Glossary & Variable Syntax

To ensure consistent operational syntax across all agents and human maintainers,
the following terms and variable placeholders are formally defined:

- **`{workspace-root}`**: The root directory of the repository setup on the
  local machine (typically `~/github/dart-sdk/`, though customizable per
  machine).
- **`{thread}`**: The specific development stream within the repository:
  - `core`: Open-source Dart SDK thread (tracks `upstream-sdk/main`).
  - `bazel`: Internal Bazel migration and integration thread (tracks
    `origin/main`).
- **`{root-worktree}`**: The primary persistent main checkout for a thread,
  located at `{workspace-root}/{thread}/main/sdk`. Used as the baseline
  reference checkout. (The `core` root is typically a detached checkout at
  `upstream-sdk/main`, since worktrees are created detached at their tracked
  ref; the `bazel` root sits on the `main` branch tracking `origin/main`.)
- **`{sandbox-worktree}`** _(or task worktree)_: Short-lived, isolated worktrees
  created for specific agent tasks at
  `{workspace-root}/{thread}/agent-{task-name}/sdk`.
- **`{bare-repo}`**: The central git database at `{workspace-root}/.bare/`
  housing historical objects for all worktrees.

### Remote Naming Conventions

All worktrees and the central bare repository MUST configure remotes using the
following standardized aliases:

| Remote Alias            | Repository URL                                 | Purpose / Scope                                                                     |
| :---------------------- | :--------------------------------------------- | :---------------------------------------------------------------------------------- |
| **`upstream-sdk`**      | `https://github.com/dart-lang/sdk.git`         | Canonical open-source Dart SDK upstream repository on GitHub.                       |
| **`dart-googlesource`** | `https://dart.googlesource.com/sdk.git`        | Canonical GoogleSource internal mirror repository (used for `lkgr-dev` releases).   |
| **`origin`**            | `https://github.com/kevmoo/dart-sdk-bazel.git` | Primary remote fork for the `bazel` thread and Beads issue sync (`refs/dolt/data`). |
| **`kevmoo`**            | `https://github.com/kevmoo/sdk.git`            | Personal remote fork for open-source `core` Dart SDK contributions and PRs.         |

---

## 1. Workspace Structure

The workspace root is `{workspace-root}` (typically `~/github/dart-sdk/`).

- **`{bare-repo}`** (`.bare/`): The central Git database (bare repository).
  **NEVER** attempt to run builds, edits, or standard git commands directly in
  this directory without the `--git-dir` flag.
- **`{workspace-root}/.agents/`**: Our dedicated, standalone Git repository
  containing agent rules (`AGENTS.md`), automation scripts (`scripts/`), and
  specialized skills (`skills/`).
- **`{workspace-root}/core/`**: Thread 1 - Core open-source Dart SDK work
  (`{root-worktree}` is `core/main/sdk`).
- **`{workspace-root}/bazel/`**: Thread 2 - Heavy Bazel fork work
  (`{root-worktree}` is `bazel/main/sdk`).

---

## 2. Rules for Agent Operations

### Rule 1: Never work in the root or bare directory

You do not have a working tree at the root of `{workspace-root}` or
`{bare-repo}`. Do not attempt to create files or run builds there.

### Rule 2: Always create an isolated Sandbox for your task

If you are assigned a task, you must create a dedicated sandbox.

1.  Identify if the task belongs to **Core** (`core`) or **Bazel** (`bazel`)
    `{thread}`.
2.  Your sandbox directory should be named
    `{workspace-root}/{thread}/agent-{task-name}/`.

### Rule 3: How to initialize your Sandbox (Automated Flow)

To set up your workspace instantly with proper caching and remote wiring, you
**MUST** use our automated helper script:

```bash
.agents/scripts/mkagenttree {thread} <task-name> [base-ref]
```

_(This automatically verifies `gcertstatus`, fetches the latest target remote
ref (`upstream-sdk/main` for Core or `origin/main` for Bazel), creates the
sandbox worktree detached at the requested or default `base-ref`, sets up a
decoupled `.gclient`, runs `gclient sync -D --no-history`, and empirically
verifies all build tools before reporting success)._

### Rule 4: Tooling Environment & PATH Invariant

All SDK operations (`gclient`, `gn`, `ninja`, `python3 tools/build.py`,
`tools/sdks/dart-sdk/bin/dart`) require `depot_tools` in your environment.
Always ensure:

```bash
export PATH="$HOME/github/depot_tools:$PATH"
export DEPOT_TOOLS_UPDATE=0
```

- If operating in an environment requiring corporate SSO authentication, ensure
  credentials (e.g. `gcert`) are active before running build or sync operations.
- **NEVER manually copy or symlink `buildtools/` or
  `build/config/gclient_args.gni` between worktrees**; doing so creates broken
  GN configurations. If dependencies or build tools are missing, always run our
  automated self-healing command:
  ```bash
  .agents/scripts/mkagenttree --sync [path-to-sdk-or-sandbox]
  ```

### Rule 5: Clean up after yourself

Once your task is complete, submitted, and approved by the user, you should
reclaim disk space immediately using our cleanup script:

```bash
.agents/scripts/rmagenttree {thread} <task-name>
```

---

## 3. Tips for Specific Workflows

### Merging Upstream Core Dart into Bazel Thread

When merging core Dart SDK updates into the Bazel fork (`bazel` thread), invoke
the dedicated skill: 👉 **`dart-sdk-merge-upstream`**. Always merge
`upstream-sdk/lkgr-dev` (never raw `main`).

### Wasm / dart2wasm & VM Build Targets

If your task involves WebAssembly, `dart2wasm`, or building local SDK binaries:

1.  **`runtime` vs. `create_sdk` (`out/ReleaseX64/dart` vs.
    `out/ReleaseX64/dart-sdk/bin/dart`):**
    - Building `runtime` produces `out/ReleaseX64/dart` (`~20s`), **NOT**
      `out/ReleaseX64/dart-sdk/bin/dart`.
    - `out/ReleaseX64/dart-sdk/bin/dart` (and `dart analyze` / `dart pub`
      subcommands) only exist if you build `create_sdk`. For fast `dart analyze`
      or `dart pub get` inside an SDK worktree without building `create_sdk`,
      use `tools/sdks/dart-sdk/bin/dart` or bare `dart` (`~/.local/bin/dart`).
2.  **Build Required Wasm Targets (`~38s`):** To run `dart2wasm` tests locally,
    build `dart2wasm` (the legacy `compile_dart2wasm` target was renamed to
    `dart2wasm`, which compiles the `dart2wasm` snapshot, platform dills, and
    `wasm-opt`):
    ```bash
    python3 tools/build.py -m release -a x64 dart2wasm
    ```
3.  **Running Tests:** Use `tools/test.py` with the `dart2wasm-linux-d8` or
    `dart2wasm-linux-chrome` configurations:
    ```bash
    python3 tools/test.py -n dart2wasm-linux-d8 language/exception/sync_throw_ref_test
    ```
    - _Fast IR Golden Tests (`~2s`)_: Run
      `pkg/dart2wasm/test/ir_test.dart --src` using
      `tools/sdks/dart-sdk/bin/dart` to compile directly from `pkg/dart2wasm`
      source without rebuilding snapshots.
4.  **Testing Local `pkg/dart2wasm` Changes End-to-End in Flutter (`15s` AOT
    Snapshot Fast Path):**
    - **NEVER copy `out/ReleaseX64/dart2wasm.snapshot` or
      `out/ReleaseX64/dartaotruntime` into
      `~/github/flutter/bin/cache/dart-sdk/`**:
      - `ninja -C out/ReleaseX64 dart2wasm` embeds
        `-Dsdk_hash=<dart-sdk-worktree-hash>` into the snapshot, whereas
        `flutter build web --wasm` loads
        `~/github/flutter/bin/cache/flutter_web_sdk/kernel/dart2wasm_platform.dill`
        (compiled with Flutter's pinned `dart_revision`). CFE's `verifySdkHash`
        will throw `InvalidKernelSdkVersionError` in `_runCfePhase`, and
        overwriting `dartaotruntime` invalidates
        `frontend_server_aot.dart.snapshot`.
    - **Fast Path (`~15s`, No Engine or Web SDK Rebuild Needed)**: As long as
      `Tag.BinaryFormatVersion` (`pkg/kernel/lib/binary/tag.dart`) matches
      Flutter's pinned Dart revision, compile `pkg/dart2wasm/bin/dart2wasm.dart`
      directly using **Flutter's cached `dart` binary** without `-Dsdk_hash`
      (which defaults `sdk_hash` to the `'0000000000'` wildcard and matches
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

---

## 4. CRITICAL: State-Changing and Remote-Write Safeguards

> [!CAUTION]
>
> ### 🚨 ABSOLUTE RULE FOR ALL AGENTS: NO INFERRED REMOTE WRITES OR FORCE PUSHES
>
> 1. **IMMEDIATE EXPLICIT AUTHORIZATION:** Every single `git push` (especially
>    `git push --force` or `git push origin ... --force`) and every single
>    `bd dolt push` (or any remote database write) **MUST** be explicitly
>    authorized by the human user **immediately before** it is executed.
> 2. **NO INFERRED PERMISSION:** Permission **CANNOT** be inferred from earlier
>    statements in the conversation (such as _"let's do that"_ or _"please"_).
>    Even if a multi-step plan containing a push was previously approved, the
>    agent **MUST pause and ask for a final, explicit confirmation** right
>    before running the actual push command.
> 3. **HISTORY MODIFICATIONS:** The same rule applies to `git commit --amend`,
>    `git rebase`, or any other history-modifying command. These are destructive
>    operations and **MUST** be explicitly approved immediately before
>    execution.
> 4. **PENALTY:** Any violation of this rule is considered a critical breach of
>    safety protocols and will result in immediate termination of the agent's
>    session.

---

## 5. Task Tracking & Backlog Sync (`beads` - Bazel Thread Only)

The **Bazel thread (`bazel`)** uses **beads** (`bd`) for local task tracking and
Dolt database synchronization.

> [!IMPORTANT]
>
> **BAZEL THREAD EXCLUSIVE & MANDATORY SKILL**: Beads issue tracking applies
> **EXCLUSIVELY to Bazel work on the `bazel` thread**. Whenever starting,
> updating, or completing tasks on the `bazel` thread, all agents **MUST load
> and adhere to the `sdk-bazel-beads` skill**.
>
> **BEAD CLOSURE RULE**: Keep tasks in `IN_PROGRESS` status throughout active
> development, local testing, and PR review. **ONLY run `bd close` after code
> has landed on `main`** (immediately after a direct push to `main`, or after a
> PR merges into `main`).

For the complete guide on setup, daily workflows, multi-user `--repo` routing
guardrails, and Dolt/SSH push troubleshooting, refer to the `sdk-bazel-beads`
skill and 👉
**[docs/bazel-migration/BEADS.md](../bazel/main/sdk/docs/bazel-migration/BEADS.md)**.

---

## 6. Architectural Migration Guidelines (Bazel Thread Only)

All agents operating on the Bazel thread (`bazel`) MUST strictly adhere to our
14 Bazel architectural migration rules (covering bottom-up hybrid migration,
direct header deps, hermetic timestamps, `copy_file` rules, and determinism).

For the full architectural rulebook, refer to: 👉
**[docs/bazel-migration/GUIDELINES.md](../bazel/main/sdk/docs/bazel-migration/GUIDELINES.md)**.

---

## 7. Master Workspace Replication & Architecture Guide

For instructions on duplicating this Bare-Repository + Sandbox Worktree
environment on a new machine, or for details on how `gclient` operates under the
hood with `"managed": False` and shared disk caching, refer to: 👉
**[REPLICATION.md](./REPLICATION.md)**.

---

## 8. Environment Setup & Skill Verification Protocol

To ensure AI agent harnesses (such as Jetski, Gemini, Claude Code, etc.)
correctly discover custom skills defined within repository subdirectories (such
as `docs/bazel-migration/skills/` on the `bazel` thread), all agents operating
in `{workspace-root}` MUST follow this verification protocol:

1. **Check Workspace Skills Manifest**: Verify that
   `{workspace-root}/.agents/skills.json` exists and registers the repository
   skills directory:
   ```json
   {
     "entries": [
       { "path": ".agents/skills" },
       { "path": "bazel/main/sdk/docs/bazel-migration/skills" }
     ]
   }
   ```
2. **Harness Discovery**: Because `{workspace-root}/.agents/` is the workspace
   customization root, standard harnesses ingest `skills.json` by default upon
   starting work in `{workspace-root}`. If operating on a new or unconfigured
   machine where `{workspace-root}/.agents/skills.json` is missing, create it
   immediately.
