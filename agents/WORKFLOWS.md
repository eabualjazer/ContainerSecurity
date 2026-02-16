# Agent Workflows

## Workflow 1: Frontend Change → Backend Update

**Trigger:** Developer asks for a frontend change that requires a new or
modified API call.

1. Orchestrator receives task
2. Orchestrator spawns **Frontend Agent**
3. Frontend Agent implements React/hook changes
4. Frontend Agent emits `frontend.api_hook.changed`
5. Orchestrator spawns **Backend Agent** with API contract details
6. Backend Agent updates Lambda to support new API shape
7. Backend Agent emits `backend.api.changed`
8. Orchestrator spawns **Testing Agent** across both repos
9. Testing Agent emits `validation.passed` (or halts on failure)
10. Orchestrator spawns **Documentation Agent**
11. Documentation Agent emits `docs.updated`
12. Orchestrator opens PRs in both `platform` and `container-security`

---

## Workflow 2: Backend Change → Frontend Update

**Trigger:** Developer changes a Lambda API response or adds a new endpoint.

1. Orchestrator receives task
2. Orchestrator spawns **Backend Agent**
3. Backend Agent updates Lambda function
4. Backend Agent emits `backend.api.changed` with new schema
5. Orchestrator spawns **Frontend Agent** with new contract
6. Frontend Agent updates API hooks and UI components
7. Frontend Agent emits `frontend.ui.changed`
8. Orchestrator spawns **Testing Agent** across both repos
9. Testing Agent emits `validation.passed` (or halts on failure)
10. Orchestrator spawns **Documentation Agent**
11. Documentation Agent emits `docs.updated`
12. Orchestrator opens PRs in both repos

---

## Workflow 3: Isolated Frontend Change (No Backend Impact)

**Trigger:** UI-only change, no API involvement.

1. Orchestrator receives task
2. Orchestrator spawns **Frontend Agent**
3. Frontend Agent implements changes, no API hook events emitted
4. Orchestrator spawns **Testing Agent** (platform only)
5. Testing Agent emits `validation.passed`
6. Orchestrator spawns **Documentation Agent** if component added
7. Orchestrator opens PR in `platform` only

---

## Workflow 4: Isolated Backend Change (No Frontend Impact)

**Trigger:** Internal Lambda refactor, no API contract change.

1. Orchestrator receives task
2. Orchestrator spawns **Backend Agent**
3. Backend Agent implements changes, no API contract events emitted
4. Orchestrator spawns **Testing Agent** (container-security only)
5. Testing Agent emits `validation.passed`
6. Orchestrator spawns **Documentation Agent** if Lambda added
7. Orchestrator opens PR in `container-security` only

---

## Failure Handling

If any agent emits a failure event:
- Orchestrator halts the entire workflow
- No PRs are opened
- Orchestrator reports the failure details to the user
- User decides how to proceed