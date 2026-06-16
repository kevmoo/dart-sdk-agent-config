# Dart SDK Agent Configuration & Automation

This repository ("Solidified Agent Config") is a standalone Git repository containing your AI agent rules, scripts, and specialized skills. It typically resides within your personal Dart SDK development workspace (most commonly under `~/github/dart-sdk/.agents/`), ensuring it is completely decoupled from your massive Dart SDK source checkouts.

## Structure

```text
.agents/
├── AGENTS.md           # Master behavioral rules & architecture guidelines (Natively loaded by AI agents)
├── README.md           # This onboarding document
├── scripts/
│   ├── mkagenttree     # Automated script to spin up a fully configured task sandbox
│   └── rmagenttree     # Automated script to tear down a task sandbox
└── skills/             # Directory for future project-scoped reusable AI skills
```

## Setup & Onboarding for Active AI Agents

Whenever an AI agent is working within the `~/github/dart-sdk/` directory, it automatically discovers and loads the rules defined in `AGENTS.md`.

To perform work, agents should use the helper scripts located in `scripts/`:
*   `~/.agents/scripts/mkagenttree <core|bazel> <task-name>`
*   `~/.agents/scripts/rmagenttree <core|bazel> <task-name>`

---

## 🌐 Master Workspace Replication Guide

If you are an AI agent (or a human maintainer) tasked with completely mirroring this highly efficient Bare-Repository + Sandbox Worktree setup on a brand new machine, execute the following multi-step workflow exactly:

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
git --git-dir=$HOME/github/dart-sdk/.bare remote add origin https://github.com/kevmoo/dart-sdk-bazel.git

# Fetch all remote branches and objects (This may take several minutes)
git --git-dir=$HOME/github/dart-sdk/.bare fetch --all
```

### Step 3: Clone Our Solidified Agent Configurations
```bash
git clone git@github.com:kevmoo/dart-sdk-agent-config.git ~/github/dart-sdk/.agents
```

### Step 4: Verify by Spinning Up Your First Sandbox
Use our automated helper script to instantly create a pristine task sandbox, set up hermetic dependencies, and run a lightning-fast `gclient sync` using a shared Git cache:
```bash
~/github/dart-sdk/.agents/scripts/mkagenttree core agent-first-setup
```

---

## 🧠 Architectural Note: How `gclient` Operates Under the Hood

Our sandbox architecture introduces an exceptionally elegant dependency management mechanism to bridge `git worktree` and `gclient`:

1.  **Decoupled Code vs Dependencies:** When `mkagenttree` creates a new sandbox, it places a minimal `.gclient` solution file in the sandbox parent directory (e.g., `~/github/dart-sdk/core/agent-foo/.gclient`).
2.  **`"managed": False`:** This critical setting tells `depot_tools` **not** to manage or overwrite the primary `sdk/` Git checkout. Instead, it fully respects our pristine Git Worktree as the absolute source of truth.
3.  **Hermetic Tooling Sync:** When `gclient sync` runs, it simply reads the `DEPS` file located inside our active worktree (`sdk/DEPS`) and fetches only the required external third-party libraries, toolchains, and prebuilt binaries.
4.  **Shared Disk Caching:** By automatically enforcing `DEPOT_TOOLS_GIT_CACHE_DIR=~/github/dart-sdk/.git_cache`, all external Git dependency clones are cached globally on disk. Every new task sandbox links to this shared cache, reducing `gclient sync` times from several minutes to just a few seconds.
