---
name: dart-sdk-bootstrap
description: >-
  Bootstrap, verify, and manage task sandboxes (git worktrees) for core or bazel threads. Use when starting a new task, cleaning up a worktree, or verifying root repository health. Don't use for generic git operations or standard code edits.
---

# Dart SDK Bootstrap & Sandbox Management (`dart-sdk-bootstrap`)

This skill guides the interactive initialization, health checks, and teardown of task worktrees. Refer to **[.agents/AGENTS.md](../../AGENTS.md)** for global rules, terms (`{workspace-root}`, `{thread}`, `{root-worktree}`), and remote conventions.

---

## 🚀 Step 1: Interactive Bootstrap & Health Check Protocol

Perform these steps sequentially when initializing a new task workspace:

1. **Pre-flight Tooling & Auth Checks**:
   - **depot_tools on PATH**: Ensure `export PATH="$HOME/github/depot_tools:$PATH"` and `export DEPOT_TOOLS_UPDATE=0`.
   - **Authentication / SSO Verification**: If operating in an environment with corporate SSO tooling (e.g. `gcertstatus`), verify credentials are valid (>30m remaining) before running git or gclient operations.
   - **Extract/Prompt Parameters (`ask_question`)**: Extract from user prompt or ask: **Work Thread** (`core` or `bazel`) and **Session Intent** (Create worktree now vs. just explore).

2. **Inspect `{root-worktree}` Health**:
   Before creating a worktree or exploring, check `{workspace-root}/{thread}/main/sdk`:
   - **Fetch Remotes**: `git --git-dir={bare-repo} fetch --all`
   - **Verify Cleanliness**: Run `git status` in `{root-worktree}`. If dirty, detached, or on a non-`main` branch, warn the user and use `ask_question` to resolve (stash/reset/checkout main).
   - **Check Sync**: Compare `HEAD` to the tracking remote (`upstream-sdk/main` for `core`, `origin/main` for `bazel`). If out of date, prompt the user (`ask_question`) to sync/pull.

3. **Initialize Sandbox** (if creating worktree):
   - Run the automated setup script:
     ```bash
     .agents/scripts/mkagenttree <core|bazel> <task-name> [base-ref]
     ```
   - Change directory to `{sandbox-worktree}`: `cd {workspace-root}/{thread}/agent-{task-name}/sdk`.
   - Ensure environment variables are active in your subagent shell:
     ```bash
     export PATH="$HOME/github/depot_tools:$PATH"
     export DEPOT_TOOLS_UPDATE=0
     ```

4. **Self-Healing & Re-Sync (If Toolchain is Incomplete)**:
   - If `buildtools/` (`gn`, `ninja`), CIPD SDK (`tools/sdks/dart-sdk`), or `build/config/gclient_args.gni` are missing, **never manually copy or symlink them across worktrees**. Run the self-healing command:
     ```bash
     .agents/scripts/mkagenttree --sync {workspace-root}/{thread}/agent-{task-name}/sdk
     ```

---

## 🧵 Step 2: Thread Operational Rules

### 🔵 Core Thread (`core`)
- **Tracking**: Use GitHub issues and standard PR workflows. Do **NOT** use `beads`.
- **Path**: Work inside `{workspace-root}/core/agent-{task-name}/sdk`.

### 🟢 Bazel Thread (`bazel`)
- **Tracking (`beads`)**: Adhere to `sdk-bazel-beads` skill. Canonical remote is `origin` (`refs/dolt/data`).
- **Lifecycle**: Keep beads in `IN_PROGRESS` until code lands on `main` (merge/push). Only then run `bd close`.
- **Update Dolt**: `bd dolt push` after updates.
- **Cleanup**: Follow `dart-sdk-cleanup-bead` skill.

---

## 🧹 Step 3: Sandbox Teardown

To reclaim disk space after task completion/approval, run:
```bash
.agents/scripts/rmagenttree <core|bazel> <task-name>
```
