# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **coordination repository** for the Container Security feature (Jira: AA-32278). It holds cross-repo context documents but no application code directly. The actual implementation lives in two sibling repos:

| Repo | Path | Stack | Branch |
|------|------|-------|--------|
| Backend | `../container-security` | C# .NET 8.0, AWS SAM/Lambda, DynamoDB, SQS | `feature/AA-32278` |
| Frontend | `../platform` | Nx monorepo, React 19, TypeScript, MUI | `feature/AA-32278` |

## Architecture

The system manages Kubernetes cluster security through Trend Vision One integration:

- **Frontend** (`libs/cloud/react-ui-cloud-infra`) renders cluster list, detail pages with tabbed views (Overview, Vulnerabilities, Malware, Security Events, Secrets, FIM), and protection actions
- **Backend** exposes REST endpoints via API Gateway + Lambda handlers. Protection requests are async via SQS
- **Trend Vision One** is the upstream security API. The backend merges cloud provider cluster data with Trend security data

### Key Data Flow

1. `GET /clusters` merges cloud provider inventory with Trend security status per cluster
2. `POST /container-security/protect` queues an SQS message; installation is async. Poll cluster list for status transitions
3. `GET /clusters/{clusterId}` returns detail with nodes/pods (excluding `trendmicro-system` namespace)
4. Event endpoints (`/events/{malware|fim|syscall|secretScan}`) fetch from Trend API

### Critical Conventions

- `{clusterId}` in URLs must be **URL-encoded** (contains colons, slashes, dots depending on cloud provider)
- Pass `?trendClusterId=xxx` query param when available for faster Trend API lookups
- All endpoints require `Authorization` and `X-Account-Context` headers
- `CloudConnectionId` is always `int` in backend responses, but frontend types allow `string | number`
- Optional fields are **omitted** from JSON when null (not sent as `null`)
- Backend gracefully degrades: if Trend API is down, clusters still return with `ProtectionStatus: "Unprotected"`

### Protection Status Priority

`Failed` > `Protected`/`Needs Attention` > `Pending` > `Unprotected`

### Unimplemented

- Kill Pod (`POST /clusters/{clusterId}/actions/kill`) - frontend hook exists, backend handler does not

## Context Documents

- `BACKEND_CONTEXT.md` - Full API endpoint reference, request/response models, event type schemas
- `FRONTEND_CONTEXT.md` - Component locations, React hooks, frontend data models, kill pod state machine
