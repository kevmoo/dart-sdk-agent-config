# Master Workspace Replication Guide

This guide details how an AI agent or human maintainer can completely mirror this Bare-Repository + Sandbox Worktree environment on a brand new machine.

---

## 🚀 Machine Replication Setup Steps

### Step 1: Prepare Tooling (`depot_tools`)
```bash
mkdir -p ~/github
if [ ! -d "$HOME/github/depot_tools" ]; then
  git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git ~/github/depot_tools
fi
export PATH="$PATH:$HOME/github/depot_tools"
```

### Step 2: Build the Bare Repository Database
This central bare Git repository houses all historical objects and branches across multiple remotes, avoiding the need to duplicate 500MB+ Git databases for every individual task sandbox.
```bash
mkdir -p ~/github/dart-sdk/.bare
git init --bare ~/github/dart-sdk/.bare

# Add primary remotes
git --git-dir=$HOME/github/dart-sdk/.bare remote add upstream-sdk https://github.com/dart-lang/sdk.git
git --git-dir=$HOME/github/dart-sdk/.bare remote add dart-googlesource https://dart.googlesource.com/sdk.git
git --git-dir=$HOME/github/dart-sdk/.bare remote add origin https://github.com/kevmoo/dart-sdk-bazel.git
git --git-dir=$HOME/github/dart-sdk/.bare remote add kevmoo https://github.com/kevmoo/sdk.git

# Fetch all remote branches and objects (This may take several minutes)
git --git-dir=$HOME/github/dart-sdk/.bare fetch --all
```

### Step 3: Clone Our Solidified Agent Configurations
```bash
git clone git@github.com:kevmoo/dart-sdk-agent-config.git ~/github/dart-sdk/.agents
```

### Step 3.5: Configure Bazel Disk Caching & Optional Remote Cache
To prevent local disk space from growing unbounded while enabling fast rebuilds, configure local garbage-collected disk caching in `~/.bazelrc`:
```bash
# Append machine-global Bazel disk cache flags
cat >> ~/.bazelrc <<EOF
# A local disk cache must be set for the GC size/age limits below to apply.
build --disk_cache=~/.cache/bazel-disk-cache
build --experimental_disk_cache_gc_max_size=50G
build --experimental_disk_cache_gc_max_age=14d
build --experimental_repository_cache_hardlinks
EOF

# (Optional / Team Specific): If collaborating with the shared team GCS remote cache:
# cat >> ~/.bazelrc <<EOF
# build --config=remote-cache
# build --remote_cache_async
# EOF
# gcloud auth application-default login
```

### Step 4: Verify by Spinning Up Your First Sandbox
Use our automated helper script to instantly pre-warm and sanitize the root checkout (`gclient sync -D --force`), create a pristine task sandbox, set up hermetic dependencies, and run a lightning-fast `gclient sync` using a shared Git cache:
```bash
~/github/dart-sdk/.agents/scripts/mkagenttree core agent-first-setup
```

---

## ⚙️ Architectural Note: How `gclient` Operates Under the Hood

Our sandbox architecture introduces an exceptionally elegant dependency management mechanism to bridge `git worktree` and `gclient`:

1.  **Decoupled Code vs Dependencies:** When `mkagenttree` creates a new sandbox, it places a minimal `.gclient` solution file in the sandbox parent directory (e.g., `~/github/dart-sdk/core/agent-foo/.gclient`).
2.  **`"managed": False`:** This critical setting tells `depot_tools` **not** to manage or overwrite the primary `sdk/` Git checkout. Instead, it fully respects our pristine Git Worktree as the absolute source of truth.
3.  **Hermetic Tooling Sync:** When `gclient sync` runs, it simply reads the `DEPS` file located inside our active worktree (`sdk/DEPS`) and fetches only the required external third-party libraries, toolchains, and prebuilt binaries.
4.  **Shared Disk Caching:** By automatically enforcing `DEPOT_TOOLS_GIT_CACHE_DIR=~/github/dart-sdk/.git_cache`, all external Git dependency clones are cached globally on disk. Every new task sandbox links to this shared cache, reducing `gclient sync` times from several minutes to just a few seconds.
5.  **Root Checkout Sanitization:** Before cloning a new sandbox, `mkagenttree` automatically runs `gclient sync -D --force --no-history` inside the root checkout (`core/main/sdk` or `bazel/main/sdk`). This aggressively prunes removed dependencies and self-heals submodule pointers, ensuring new sandboxes never inherit broken or stale third-party states.
