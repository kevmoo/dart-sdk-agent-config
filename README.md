# Dart SDK Agent Configuration & Tooling

Operational playbook, automated sandbox tooling, and agent skills used to manage multi-thread development across the Dart SDK and specialized research forks using a shared **Bare Repository + Git Worktree** architecture.

---

## 🍴 Bespoke Dart SDK Forks

This configuration orchestrates local worktrees across several specialized forks of [dart-lang/sdk](https://github.com/dart-lang/sdk):

* **[kevmoo/sdk](https://github.com/kevmoo/sdk)** (`core`): Personal staging fork for day-to-day hacking, reproduction sandboxes, and contributions sent upstream to `dart-lang/sdk`.
* **[kevmoo/dart-sdk-bazel](https://github.com/kevmoo/dart-sdk-bazel)** (`bazel`): Long-running research fork exploring native Bazel compilation, hermetic toolchains, and rule migrations for the Dart SDK.
* **[kevmoo/dart-sdk-json-next](https://github.com/kevmoo/dart-sdk-json-next)** (`json-next`): Dedicated prototype fork optimizing core library JSON and UTF-8 serialization kernels for high-throughput workloads.

---

## 📚 Key Documentation

* **[AGENTS.md](./AGENTS.md)**: Master operational guidelines, remote naming conventions, sandbox rules, and safety constraints for human maintainers and coding agents.
* **[REPLICATION.md](./REPLICATION.md)**: Step-by-step setup instructions to replicate this bare-repository layout, cache configuration, and dependency synchronization on a new machine.
* **[skills/](./skills/)**: Reusable workflow skills for bootstrapping sandboxes, merging upstream LKGR releases, and executing pre-PR reviews.
