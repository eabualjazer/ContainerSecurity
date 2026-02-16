# Agents System — Overview

## Purpose

This repo contains the Claude Code multi-agent system that manages bidirectional
synchronization between two repos:

- **`platform`** — React frontend (multiple apps)
- **`container-security`** — AWS Lambda backend

When either repo changes, the relevant agents are notified so they can handle
their part of the update. All changes are submitted as **pull requests** for
human review before merging.

---

## Repo Structure

~/projects/
├── platform/                   # React frontend repo
├── container-security/         # AWS Lambda backend repo
└── agents/                     # This repo — Claude Code agent definitions
    ├── OVERVIEW.md
    ├── WORKFLOWS.md
    ├── orchestrator.md
    ├── frontend-agent.md
    ├── backend-agent.md
    ├── testing-agent.md
    └── documentation-agent.md

---

## Agent Roles

| Agent               | Repo                | Responsibility                              |
|---------------------|---------------------|---------------------------------------------|
| Orchestrator        | N/A                 | Routes events, coordinates all agents       |
| Frontend Agent      | platform            | React components, API hooks, UI updates     |
| Backend Agent       | container-security  | Lambda functions, API responses, schemas    |
| Testing Agent       | Both                | Runs tests, validates changes               |
| Documentation Agent | Both                | Keeps docs in sync with code changes        |

---

## Communication Model

All agent coordination goes through the Orchestrator.
No agent directly calls another.

User or trigger
      │
      ▼
 Orchestrator
   /   |   \
  FE   BE  ...
Agent Agent
   \   /
  Testing
   Agent
      │
 Documentation
    Agent

### Event Types

| Event                        | Source          | Triggers                              |
|------------------------------|-----------------|---------------------------------------|
| frontend.api_hook.changed    | Frontend Agent  | Backend Agent, Testing Agent          |
| frontend.component.added     | Frontend Agent  | Documentation Agent                   |
| frontend.ui.changed          | Frontend Agent  | Testing Agent, Documentation Agent    |
| backend.api.changed          | Backend Agent   | Frontend Agent, Testing Agent         |
| backend.schema.changed       | Backend Agent   | Frontend Agent, Testing Agent, Docs   |
| backend.lambda.added         | Backend Agent   | Documentation Agent                   |
| validation.passed            | Testing Agent   | Orchestrator (proceed to PR)          |
| validation.failed            | Testing Agent   | Orchestrator (halt, notify)           |
| docs.updated                 | Docs Agent      | Orchestrator (include in PR)          |

---

## Output: Pull Requests

Every completed workflow results in one or more PRs:
- Changes to platform → PR opened in platform repo
- Changes to container-security → PR opened in container-security repo
- Both PRs are linked and reference the originating change
- No code is merged automatically — all PRs require human review