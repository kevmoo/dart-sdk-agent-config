# Workspace Rules: Dart SDK Sandbox Layout

This workspace uses a specialized **Bare Repository + Sandbox Worktree** layout to optimize disk space, handle `gclient` cleanly, and keep the root directory pristine. 

All AI agents operating in this workspace MUST adhere to the following rules.

---

## 📖 Glossary & Variable Syntax

To ensure consistent operational syntax across all agents and human maintainers, the following terms and variable placeholders are formally defined:

* **`{workspace-root}`**: The root directory of the repository setup on the local machine (typically `~/github/dart-sdk/`, though customizable per machine).
* **`{thread}`**: The specific development stream within the repository:
  * `core`: Open-source Dart SDK thread (tracks `upstream-sdk/main`).
  * `bazel`: Internal Bazel migration and integration thread (tracks `origin/main`).
* **`{root-worktree}`**: The primary persistent main checkout for a thread, located at `{workspace-root}/{thread}/main/sdk`. Used as the baseline reference checkout.
* **`{sandbox-worktree}`** *(or task worktree)*: Short-lived, isolated worktrees created for specific agent tasks at `{workspace-root}/{thread}/agent-{task-name}/sdk`.
* **`{bare-repo}`**: The central git database at `{workspace-root}/.bare/` housing historical objects for all worktrees.

### Remote Naming Conventions
All worktrees and the central bare repository MUST configure remotes using the following standardized aliases:

| Remote Alias | Repository URL | Purpose / Scope |
| :--- | :--- | :--- |
| **`upstream-sdk`** | `https://github.com/dart-lang/sdk.git` | Canonical open-source Dart SDK upstream repository on GitHub. |
| **`dart-googlesource`** | `https://dart.googlesource.com/sdk.git` | Canonical GoogleSource internal mirror repository (used for `lkgr-dev` releases). |
| **`origin`** | `https://github.com/kevmoo/dart-sdk-bazel.git` | Primary remote fork for the `bazel` thread and Beads issue sync (`refs/dolt/data`). |
| **`kevmoo`** | `https://github.com/kevmoo/sdk.git` | Personal remote fork for open-source `core` Dart SDK contributions and PRs. |

---

## 1. Workspace Structure

The workspace root is `{workspace-root}` (typically `~/github/dart-sdk/`). 

*   **`{bare-repo}`** (`.bare/`): The central Git database (bare repository). **NEVER** attempt to run builds, edits, or standard git commands directly in this directory without the `--git-dir` flag.
*   **`{workspace-root}/.agents/`**: Our dedicated, standalone Git repository containing agent rules (`AGENTS.md`), automation scripts (`scripts/`), and specialized skills (`skills/`).
*   **`{workspace-root}/core/`**: Thread 1 - Core open-source Dart SDK work (`{root-worktree}` is `core/main/sdk`).
*   **`{workspace-root}/bazel/`**: Thread 2 - Heavy Bazel fork work (`{root-worktree}` is `bazel/main/sdk`).

---

## 2. Rules for Agent Operations

### Rule 1: Never work in the root or bare directory
You do not have a working tree at the root of `{workspace-root}` or `{bare-repo}`. Do not attempt to create files or run builds there.

### Rule 2: Always create an isolated Sandbox for your task
If you are assigned a task, you must create a dedicated sandbox. 
1.  Identify if the task belongs to **Core** (`core`) or **Bazel** (`bazel`) `{thread}`.
2.  Your sandbox directory should be named `{workspace-root}/{thread}/agent-{task-name}/`.

### Rule 3: How to initialize your Sandbox (Automated Flow)
To set up your workspace instantly with proper caching and remote wiring, you **MUST** use our automated helper script:
```bash
.agents/scripts/mkagenttree {thread} <task-name>
```
*(This automatically creates the directory, maps `upstream-sdk/main` for Core or `origin/main` for Bazel, sets up `.gclient`, and runs a fast `gclient sync`).*

**Manual Underlying Equivalence (For Reference Only):**
If you must run it manually:
1. `mkdir -p {workspace-root}/{thread}/agent-{task-name}`
2. Worktree add:
   * For Core: `git --git-dir={bare-repo} worktree add {workspace-root}/core/agent-{task-name}/sdk upstream-sdk/main`
   * For Bazel: `git --git-dir={bare-repo} worktree add {workspace-root}/bazel/agent-{task-name}/sdk origin/main`
3. Create `.gclient` and run `gclient sync`.

### Rule 4: Clean up after yourself
Once your task is complete, submitted, and approved by the user, you should reclaim disk space immediately using our cleanup script:
```bash
.agents/scripts/rmagenttree {thread} <task-name>
```

### Rule 5: Mandatory Backlog Board Synchronization
Whenever an agent creates, transitions, updates, or closes an issue in the `beads` database (`bd`), the agent **MUST**:
1. Execute `tools/sdks/dart-sdk/bin/dart docs/bazel-migration/gen_board_from_beads.dart`.
2. Stage and commit the resulting `docs/bazel-migration/BACKLOG.md` and `BACKLOG_HISTORY.md` updates to `main`.
3. Request explicit user authorization to execute `bd dolt push` alongside `git push`.

*(Note: Beads issue tracking and backlog synchronization apply **EXCLUSIVELY to the Bazel thread (`bazel/`)**. Do not use Beads for Core (`core/`) SDK tasks).*

---

## 3. Tips for Specific Workflows

### Merging Upstream Core Dart into Bazel Thread
When merging core Dart SDK updates into the Bazel fork (`bazel` thread), invoke the dedicated skill: 👉 **`dart-sdk-merge-upstream`**. Always merge `upstream-sdk/lkgr-dev` (never raw `main`).

### Wasm / dart2wasm Development
If your task involves WebAssembly or `dart2wasm`:
1.  **Enable Emscripten in `.gclient`:** Set `"download_emscripten": True` in `custom_vars` in your `.gclient` file *before* running `gclient sync`. (If using `mkagenttree`, edit the generated `.gclient` file and run `gclient sync` again).
2.  **Build Required Targets:** To run tests, you must build both the compiler and the optimizer. Run:
    ```bash
    python3 tools/build.py -m release -a x64 dart2wasm wasm-opt
    ```
3.  **Running Tests:** Use `tools/test.py` with the `dart2wasm` compiler:
    ```bash
    python3 tools/test.py -c dart2wasm language/exception/sync_throw_ref_test
    ```

---

## 4. CRITICAL: State-Changing and Remote-Write Safeguards

> [!CAUTION]
> ### 🚨 ABSOLUTE RULE FOR ALL AGENTS: NO INFERRED REMOTE WRITES OR FORCE PUSHES
> 
> 1. **IMMEDIATE EXPLICIT AUTHORIZATION:** Every single `git push` (especially `git push --force` or `git push origin ... --force`) and every single `bd dolt push` (or any remote database write) **MUST** be explicitly authorized by the human user **immediately before** it is executed.
> 2. **NO INFERRED PERMISSION:** Permission **CANNOT** be inferred from earlier statements in the conversation (such as *"let's do that"* or *"please"*). Even if a multi-step plan containing a push was previously approved, the agent **MUST pause and ask for a final, explicit confirmation** right before running the actual push command.
> 3. **HISTORY MODIFICATIONS:** The same rule applies to `git commit --amend`, `git rebase`, or any other history-modifying command. These are destructive operations and **MUST** be explicitly approved immediately before execution.
> 4. **PENALTY:** Any violation of this rule is considered a critical breach of safety protocols and will result in immediate termination of the agent's session.

---

## 5. Task Tracking & Backlog Sync (`beads` - Bazel Thread Only)

The **Bazel thread (`bazel`)** uses **beads** (`bd`) for local task tracking and Dolt database synchronization.

> [!IMPORTANT]
> **BAZEL THREAD EXCLUSIVE**: Beads issue tracking applies **EXCLUSIVELY to Bazel work on the `bazel` thread**. Do NOT use Beads for Core (`core`) SDK development (which uses standard GitHub issues).

For the complete guide on setup, daily workflows, board generation (`gen_board_from_beads.dart`), multi-user `--repo` routing guardrails, and Dolt/SSH push troubleshooting, refer to: 👉 **[docs/bazel-migration/BEADS.md](../../bazel/main/sdk/docs/bazel-migration/BEADS.md)**.

---

## 6. Architectural Migration Guidelines (Bazel Thread Only)

All agents operating on the Bazel thread (`bazel`) MUST strictly adhere to our 14 Bazel architectural migration rules (covering bottom-up hybrid migration, direct header deps, hermetic timestamps, `copy_file` rules, and determinism).

For the full architectural rulebook, refer to: 👉 **[docs/bazel-migration/GUIDELINES.md](../../bazel/main/sdk/docs/bazel-migration/GUIDELINES.md)**.

---

## 7. Master Workspace Replication & Architecture Guide

For instructions on duplicating this Bare-Repository + Sandbox Worktree environment on a new machine, or for details on how `gclient` operates under the hood with `"managed": False` and shared disk caching, refer to: 👉 **[REPLICATION.md](./REPLICATION.md)**.



