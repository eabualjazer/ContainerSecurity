# Frontend Context for Container Security Backend

This document provides context from the frontend platform repo (`../platform`) for coordinating backend changes.

## Frontend Repo

**Location:** `../platform` (Nx monorepo, React 19 + TypeScript)

**Relevant library:** `libs/cloud/react-ui-cloud-infra` - Container security UI components

## Current Feature Branch

**Branch:** `feature/AA-32278` (Cluster detail page improvements)

## Key Frontend Files

### Pages
- `libs/cloud/react-ui-cloud-infra/src/lib/pages/ClusterDetail.tsx` — Cluster detail page with stat cards (Nodes, Pods, Services, Namespaces) and tabbed navigation (Overview, Vulnerabilities, Malware, Security Events, Secrets, FIM)

### Components
- `libs/cloud/react-ui-cloud-infra/src/lib/components/clusters/ClusterOverviewTab.tsx` — Overview tab: Detected Issues badges, Cluster Information grid (Status, Connectivity, Provider, Region, Agent Version, Cluster/Compartment Name+ID), Node Details section, Timeline
- `libs/cloud/react-ui-cloud-infra/src/lib/components/clusters/ClusterNodeDetailsSection.tsx` — Node table with drawer showing node details + pods table with Kill Pod action
- `libs/cloud/react-ui-cloud-infra/src/lib/components/clusters/ClustersTable.tsx` — Clusters list table with Protect Cluster action

### API Hooks
- `libs/cloud/react-api-cloud-infra/src/lib/useGetClusterDetail/useGetClusterDetail.tsx` — Fetches cluster detail data (includes Nodes, Pods, counts, vulnerability stats)
- `libs/cloud/react-api-cloud-infra/src/lib/useGetClusters/` — Fetches clusters list
- `libs/cloud/react-api-cloud-infra/src/lib/useKillPod/` — Kill pod mutation
- `libs/cloud/react-api-cloud-infra/src/lib/useProtectCluster/` — Protect cluster mutation (calls POST `/container-security/protect`)

## Data Models (Frontend)

### ClusterPod
```typescript
{
  Id: string;
  Name: string;
  Namespace: string;
  Status: string;
  ProtectionStatus: string;
  CreatedDateTime?: string;
  LastEvaluatedDateTime?: string;
  IpAddress: string;
  Age: string;
}
```

### ClusterNode
```typescript
{
  Id: string;
  Name: string;
  Status: string;
  ProtectionStatus: string;
  OsVersion?: string;
  LaunchType?: string;
  KernelVersion?: string;
  CreatedDateTime?: string;
  LastEvaluatedDateTime?: string;
  IpAddresses?: string[];
  Pods?: ClusterPod[];
}
```

### GetClusterDetail Response
```typescript
{
  ClusterId: string;
  ClusterName: string;
  AccountId?: string;
  CloudConnectionId?: string | number;
  CompartmentId: string;
  CompartmentName?: string;
  Provider: string;
  ClusterConnectivity: string;
  ProtectionStatus: string;
  Region?: string;
  AgentVersion?: string;
  NodesCount?: number;
  PodsCount?: number;
  ServicesCount?: number;
  NamespacesCount?: number;
  Nodes?: ClusterNode[];
  VulnerabilityCriticalCount?: number;
  VulnerabilityHighCount?: number;
  VulnerabilityMediumCount?: number;
  VulnerabilityLowCount?: number;
  MalwareEventsCount?: number;
  SecretScanEventsCount?: number;
  FileIntegrityEventsCount?: number;
  SyscallEventsCount?: number;
  CreatedAt?: string;
  UpdatedAt?: string;
  LastScannedAt?: string;
}
```

## API Endpoints Consumed by Frontend

All paths below are relative to the `containerSecurityApiUrl()` base URL.

| Method | Path (relative) | Backend SAM Path | Frontend Hook | Purpose |
|--------|-----------------|------------------|---------------|---------|
| GET | `clusters` | `/clusters` | `useGetClusters` | List all clusters |
| GET | `clusters/{id}` | `/clusters/{clusterId}` | `useGetClusterDetail` | Cluster detail with nodes/pods |
| POST | `container-security/protect` | `/container-security/protect` | `useProtectCluster` | Register cluster for protection |
| POST | `clusters/{id}/actions/kill` | _(not yet in backend)_ | `useKillPod` | Kill a specific pod |
| GET | `clusters/{id}/vulnerabilities` | `/clusters/{clusterId}/vulnerabilities` | `useGetClusterVulnerabilities` | List vulnerabilities |
| GET | `clusters/{id}/events/malware` | `/clusters/{clusterId}/events/{eventType}` | `useGetClusterMalware` | List malware events |
| GET | `clusters/{id}/events/syscall` | `/clusters/{clusterId}/events/{eventType}` | `useGetClusterSecurityEvents` | List security events |
| GET | `clusters/{id}/events/secretScan` | `/clusters/{clusterId}/events/{eventType}` | `useGetClusterSecrets` | List secret scan events |
| GET | `clusters/{id}/events/fim` | `/clusters/{clusterId}/events/{eventType}` | `useGetClusterFim` | List FIM events |

## Recent Changes (feature/AA-32278)

- Added cloud connection ID to overview page
- Added node details section with pods table in cluster overview
- Compact cluster information layout (3-column CSS grid)
- Stat cards moved above tabs with inline layout
- Copy buttons for Cluster ID and Compartment ID
- Pods table converted from raw MUI to standard `@platform/react-ui` Table with pagination, sorting, and column filtering
- Kill pod: frontend now tracks "Deleting" → "Deleted" status transitions locally (see below)

## Kill Pod - Frontend Status Handling

The frontend manages pod deletion status transitions **client-side** using local React state. No backend changes are currently required.

**Current behavior:**
1. User confirms kill → pod status immediately changes to "Deleting" (warning chip)
2. Kill API succeeds → pod status changes to "Deleted" (error chip), Kill action disabled
3. Kill API fails → pod status reverts to original, Kill action re-enabled
4. Pod disappears from cluster detail API response after ~1 hour

**Kill API endpoint:** `POST /clusters/{clusterId}/actions/kill`

**Request body sent by frontend:**
```json
{
  "clusterId": "string",
  "cloudConnectionId": "string | number",
  "provider": "string",
  "namespace": "string",
  "podName": "string",
  "reason": "string"
}
```

**Expected response:** Any 2xx status (body is ignored by frontend). On error, frontend shows a generic failure snackbar.

**Post-success behavior:** Frontend invalidates the cluster detail query (`get-cluster-detail`) to re-fetch data. Local state overrides pod status to "Deleting"/"Deleted" until the pod disappears from the API response.

**Backend status:** The kill pod endpoint is **not yet implemented in the backend**. The frontend hook (`useKillPod`) exists but currently relies on the frontend platform's proxy routing. If the backend needs to handle this, a new handler would need to be added to `template.yml`.

**Future enhancement (optional):** If the backend adds a `DeletionStatus` field to the pod object in the cluster detail response (e.g., `"deleting"`, `"deleted"`, or `null`), the frontend can consume it directly instead of relying on local state. This would persist the status across page refreshes.
