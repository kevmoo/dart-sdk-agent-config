---
name: dart-sdk-bootstrap
description: >-
  Bootstraps, verifies, and manages isolated task sandboxes and git worktrees
  for the Dart SDK bare-repository workspace. Use when starting a new Dart SDK
  task, syncing or self-healing an incomplete toolchain, cleaning up a task
  worktree, or verifying root repository health. Don't use for standard non-SDK
  GitHub repositories (use new-worktree).
---

# Dart SDK Bootstrap & Sandbox Management (`dart-sdk-bootstrap`)

This skill guides the interactive initialization, health checks, and teardown of
task worktrees. Refer to **[.agents/AGENTS.md](../../AGENTS.md)** for global
rules, terms (`{workspace-root}`, `{root-worktree}`), and remote conventions.

---

## 🚀 Step 1: Interactive Bootstrap & Health Check Protocol

Perform these steps sequentially when initializing a new task workspace:

1. **Pre-flight Tooling & Auth Checks**:
   - **depot_tools on PATH**: Ensure
     `export PATH="$HOME/github/depot_tools:$PATH"` and
     `export DEPOT_TOOLS_UPDATE=0`.
   - **Authentication / SSO & RBE Verification**: If operating in an environment
     with SSO tooling, verify credentials are valid (>30m remaining) before
     running git or gclient operations. When `DART_RBE=1` is enabled, verify RBE
     Application Default Credentials from inside an SDK checkout
     (`{workspace-root}/core/main/sdk` or `{root-worktree}`) via
     `siso auth-check -project flutter-rbe-prod -reapi_instance default` (if
     expired or if `~/.config/gcloud/application_default_credentials.json`
     contains `"quota_project_id"`, refresh via
     `gcloud auth application-default login --disable-quota-project`).
   - **Extract/Prompt Parameters (`ask_question`)**: Extract from user prompt or
     ask **Session Intent** (Create task worktree now vs. just explore).

2. **Inspect `{root-worktree}` Health**: Before creating a worktree or
   exploring, check `{workspace-root}/core/main/sdk`:
   - **Fetch Remotes**: `git --git-dir={bare-repo} fetch --all`
   - **Verify Cleanliness**: Run `git status` in `{root-worktree}`. If dirty,
     warn the user and use `ask_question` to resolve.
   - **Check Sync**: Compare `HEAD` to `upstream-sdk/main`. If out of date,
     prompt the user (`ask_question`) to sync/pull.

3. **Initialize Sandbox** (if creating worktree):
   - Run the automated setup script:
     ```bash
     .agents/scripts/mkagenttree core <task-name> [base-ref]
     ```
   - Change directory to `{sandbox-worktree}`:
     `cd {workspace-root}/core/agent-<task-name>/sdk`.
   - Ensure environment variables are active in your subagent shell:
     ```bash
     export PATH="$HOME/github/depot_tools:$PATH"
     export DEPOT_TOOLS_UPDATE=0
     ```

4. **Self-Healing & Re-Sync (If Toolchain is Incomplete)**:
   - If `buildtools/` (`gn`, `ninja`), `third_party/siso/siso`, CIPD SDK
     (`tools/sdks/dart-sdk`), `build/config/gclient_args.gni`, or
     `build/config/siso/.sisoenv` are missing, **never manually copy or symlink
     them across worktrees**. Run the self-healing command:
     ```bash
     .agents/scripts/mkagenttree --sync {workspace-root}/core/agent-<task-name>/sdk
     ```

---

## 🧹 Step 2: Sandbox Teardown

To reclaim disk space after task completion/approval, run:

```bash
.agents/scripts/rmagenttree core <task-name>
```
