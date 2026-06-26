---
name: dart-sdk-merge-upstream
description: >-
  Merge core upstream Dart SDK updates (upstream-sdk/lkgr-dev or dart-googlesource/lkgr-dev) into the Bazel thread fork safely.
---

# Merging Upstream Core Dart into Bazel Thread (`dart-sdk-merge-upstream`)

This skill defines the exact protocol for integrating continuous open-source Core Dart SDK developments into the long-running Bazel fork thread (`bazel`).

---

## When to Use This Skill
- Trigger when assigned a task to "merge upstream into bazel", "pull lkgr-dev updates", or "sync bazel thread with core SDK".

---

## 🔒 Mandatory Branch Selection Rule

> [!WARNING]
> **NEVER MERGE `main` DIRECTLY**: When merging upstream core Dart updates into the Bazel fork (`bazel` thread), you **MUST** merge `upstream-sdk/lkgr-dev` (or `dart-googlesource/lkgr-dev`), **NEVER** `upstream-sdk/main`. 
>
> `lkgr-dev` (Last Known Good Revision) represents the rolling revision that has successfully passed full continuous integration suites on builders. Merging raw `main` risks pulling unverified intermediate breakage.

---

## 🛠️ Step-by-Step Execution Workflow

### 1. Prerequisite Worktree Setup
Ensure you are operating inside a dedicated sandbox worktree for the Bazel thread:
```bash
# Verify you are in a Bazel sandbox worktree
cd {workspace-root}/bazel/agent-<task-name>/sdk
```

### 2. Fetch Latest Remote Revisions
Fetch the latest refs across all remotes:
```bash
git --git-dir={bare-repo} fetch --all
```

### 3. Execute the Merge Command
Inside your sandbox worktree (`sdk/`), execute the merge against `lkgr-dev`:
```bash
# Prefer upstream-sdk/lkgr-dev or dart-googlesource/lkgr-dev
git merge upstream-sdk/lkgr-dev
```

### 4. Conflict Resolution & Hermetic Validation
* If merge conflicts occur in C++ sources or Starlark files, resolve them carefully, preserving hermetic Bazel wrappers (`copy_file`, direct header declarations).
* Run local presubmit checks to verify build cleanliness:
  ```bash
  ./tools/bazel/presubmit.sh
  ```

---

## 🔒 Safety Safeguard
* **Explicit Approval Required**: Before pushing the merged branch or submitting a PR, request explicit confirmation from the human user.
