# Dart SDK Agent Configuration & Automation

This repository ("Solidified Agent Config") is a standalone Git repository residing at `~/github/dart-sdk/.agents/`. It is completely decoupled from the large Dart SDK checkouts, allowing you to independently version control your AI agent rules, scripts, and specialized skills.

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

## Setup & Onboarding for AI Agents

Whenever an AI agent is working within the `~/github/dart-sdk/` directory, it automatically discovers and loads the rules defined in `AGENTS.md`.

To perform work, agents should use the helper scripts located in `scripts/`:
*   `~/.agents/scripts/mkagenttree <core|bazel> <task-name>`
*   `~/.agents/scripts/rmagenttree <core|bazel> <task-name>`
