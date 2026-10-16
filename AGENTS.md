# Workspace Rules: Dart SDK Sandbox Layout

This workspace uses a specialized **Bare Repository + Sandbox Worktree** layout
to optimize disk space, handle `gclient` cleanly, and keep the root directory
pristine. All AI agents operating in this workspace MUST adhere to the following
rules.

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

## 1. Workspace Structure

The workspace root is `{workspace-root}` (typically `~/github/dart-sdk/`):

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

## 2. Rules for Agent Operations

1. **Rule 1: Never work in the root or bare directory**: You do not have a
   working tree at the root of `{workspace-root}` or `{bare-repo}`. Do not
   attempt to create files or run builds there.
2. **Rule 2: Always create an isolated Sandbox for your task**: Identify whether
   the task belongs to the **Core** (`core`) or **Bazel** (`bazel`) `{thread}`,
   and place your sandbox at `{workspace-root}/{thread}/agent-{task-name}/`.
3. **Rule 3: How to initialize your Sandbox (Automated Flow)**: To set up your
   workspace instantly with proper caching and remote wiring, you **MUST** use
   our automated helper script:
   ```bash
   .agents/scripts/mkagenttree {thread} <task-name> [base-ref]
   ```
   _(Automatically verifies `gcertstatus`, fetches the latest target remote ref
   `upstream-sdk/main` for Core or `origin/main` for Bazel, creates the sandbox
   worktree detached at the requested or default `base-ref`, sets up a decoupled
   `.gclient`, runs `gclient sync -D --no-history`, and empirically verifies all
   build tools before reporting success.)_
4. **Rule 4: Tooling Environment & PATH Invariant**: All SDK operations
   (`gclient`, `gn`, `ninja`, `python3 tools/build.py`,
   `tools/sdks/dart-sdk/bin/dart`) require `depot_tools` in your environment:
   ```bash
   export PATH="$HOME/github/depot_tools:$PATH"
   export DEPOT_TOOLS_UPDATE=0
   ```
   - If operating in an environment requiring corporate SSO authentication,
     ensure credentials (e.g. `gcert`) are active before running build or sync
     operations.
   - **NEVER manually copy or symlink `buildtools/` or
     `build/config/gclient_args.gni` between worktrees**; doing so creates
     broken GN configurations. If dependencies or build tools are missing,
     always run our automated self-healing command:
     ```bash
     .agents/scripts/mkagenttree --sync [path-to-sdk-or-sandbox]
     ```
5. **Rule 5: Clean up after yourself**: Once your task is complete, submitted,
   and approved by the user, reclaim disk space immediately using our cleanup
   script:
   ```bash
   .agents/scripts/rmagenttree {thread} <task-name>
   ```

## 3. Tips for Specific Workflows

- **Merging Upstream Core Dart into Bazel Thread**: Invoke
  [skills/dart-sdk-merge-upstream/SKILL.md](skills/dart-sdk-merge-upstream/SKILL.md)
  (`dart-sdk-merge-upstream`). Always merge `upstream-sdk/lkgr-dev` (or
  `dart-googlesource/lkgr-dev`), never raw `main`.
- **Wasm / `dart2wasm` & VM Build Targets**: For `runtime` vs. `create_sdk`
  targets, building `dart2wasm` (`~38s`), `tools/test.py` configurations, fast
  IR golden tests (`--src`), and the `15s` Flutter Web AOT snapshot swap fast
  path, read
  [references/wasm_and_vm_build_targets.md](references/wasm_and_vm_build_targets.md).
- **Dart VM C++ Best Practices**: For VM object model invariants, `Handle` vs.
  `ObjectPtr` safepoint discipline, memory allocation taxonomy, and Gerrit
  review standards, read
  [references/dart_vm_cpp_best_practices.md](references/dart_vm_cpp_best_practices.md).

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

## 5. Bazel Thread Tracking (`beads`) & Architectural Migration Guidelines

The **Bazel thread (`bazel`)** uses **beads** (`bd`) for local task tracking and
Dolt database synchronization:

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

- **Beads Workflow & Dolt/SSH Troubleshooting**: Refer to the `sdk-bazel-beads`
  skill and `../bazel/main/sdk/docs/bazel-migration/BEADS.md`
  (`{workspace-root}/bazel/main/sdk/docs/bazel-migration/BEADS.md`).
- **14 Bazel Architectural Migration Rules**: All agents on the `bazel` thread
  MUST adhere to our rules covering bottom-up hybrid migration, direct header
  deps, hermetic timestamps, `copy_file` rules, and determinism in
  `../bazel/main/sdk/docs/bazel-migration/GUIDELINES.md`
  (`{workspace-root}/bazel/main/sdk/docs/bazel-migration/GUIDELINES.md`).

## 6. Workspace Replication & Skill Discovery Protocol

- **Master Workspace Replication & `gclient` Architecture**: For instructions on
  duplicating this Bare-Repository + Sandbox Worktree environment on a new
  machine, or details on how `gclient` operates with `"managed": False` and
  shared disk caching, read [REPLICATION.md](./REPLICATION.md).
- **Check Workspace Skills Manifest (`{workspace-root}/.agents/skills.json`)**:
  Because `{workspace-root}/.agents/` is the workspace customization root,
  harnesses (Jetski, Gemini, Claude Code) ingest `skills.json` on startup to
  discover skills in repository subdirectories (`docs/bazel-migration/skills/`
  on `bazel`). Verify `{workspace-root}/.agents/skills.json` exists (and create
  it immediately on a new machine if missing):
  ```json
  {
    "entries": [
      { "path": ".agents/skills" },
      { "path": "bazel/main/sdk/docs/bazel-migration/skills" }
    ]
  }
  ```
