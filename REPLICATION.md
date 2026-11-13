# Master Workspace Replication Guide

This guide details how an AI agent or human maintainer can completely mirror
this Bare-Repository + Sandbox Worktree environment on a brand new machine.

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

This central bare Git repository houses all historical objects and branches
across multiple remotes, avoiding the need to duplicate 500MB+ Git databases for
every individual task sandbox.

```bash
mkdir -p ~/github/dart-sdk/.bare
git init --bare ~/github/dart-sdk/.bare

# Add primary remotes
git --git-dir=$HOME/github/dart-sdk/.bare remote add upstream-sdk https://github.com/dart-lang/sdk.git
git --git-dir=$HOME/github/dart-sdk/.bare remote add dart-googlesource https://dart.googlesource.com/sdk.git
git --git-dir=$HOME/github/dart-sdk/.bare remote add kevmoo https://github.com/kevmoo/sdk.git
git --git-dir=$HOME/github/dart-sdk/.bare remote add json-next https://github.com/kevmoo/dart-sdk-json-next.git

# Fetch all remote branches and objects (This may take several minutes)
git --git-dir=$HOME/github/dart-sdk/.bare fetch --all
```

### Step 3: Clone Our Solidified Agent Configurations

```bash
git clone git@github.com:kevmoo/dart-sdk-agent-config.git ~/github/dart-sdk/.agents
```

### Step 4: Verify by Spinning Up Your First Sandbox

Use our automated helper script to fetch the latest upstream ref, create a
pristine task sandbox, set up hermetic dependencies, and run a lightning-fast
`gclient sync` using a shared Git cache:

```bash
~/github/dart-sdk/.agents/scripts/mkagenttree core agent-first-setup
```

---

## ⚙️ Architectural Note: How `gclient` Operates Under the Hood

Our sandbox architecture introduces an exceptionally elegant dependency
management mechanism to bridge `git worktree` and `gclient`:

1.  **Decoupled Code vs Dependencies:** When `mkagenttree` creates a new
    sandbox, it places a minimal `.gclient` solution file in the sandbox parent
    directory (e.g., `~/github/dart-sdk/core/agent-foo/.gclient`).
2.  **`"managed": False`:** This critical setting tells `depot_tools` **not** to
    manage or overwrite the primary `sdk/` Git checkout. Instead, it fully
    respects our pristine Git Worktree as the absolute source of truth.
3.  **Hermetic Tooling Sync:** When `gclient sync` runs, it simply reads the
    `DEPS` file located inside our active worktree (`sdk/DEPS`) and fetches only
    the required external third-party libraries, toolchains, and prebuilt
    binaries.
4.  **Shared Disk Caching:** By configuring `GIT_CACHE_PATH=~/.cache/git_cache`,
    `depot_tools` maintains bare mirror clones of external Git dependencies
    centrally on disk. Every new task sandbox syncs from this shared local
    cache, reducing `gclient sync` times from several minutes over the network
    to just a few seconds.
5.  **Deterministic Base Ref & Dependency Pruning:** Before syncing a new
    sandbox, `mkagenttree` fetches the target remote ref in `.bare`, checks out
    the requested or default `base-ref` (`upstream-sdk/main` for `core`) in
    detached `HEAD` mode, runs `gclient sync -D --no-history` inside the new
    sandbox to prune removed dependencies and hydrate CIPD toolchains, and
    restores `remote.origin.pushurl = no_push` if an `origin` remote is present
    (which `depot_tools` git-cache mirrors otherwise overwrite to the `.gclient`
    solution URL).
