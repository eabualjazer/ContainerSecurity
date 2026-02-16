# Backend Context for Container Security Frontend

This document provides context from the backend repo (`../container-security`) for coordinating frontend changes.

## Backend Repo

**Location:** `../container-security` (C# .NET 8.0, AWS SAM/Lambda)

**Current Feature Branch:** `feature/AA-32278`

## API Endpoints

All endpoints require `Authorization` and `X-Account-Context` headers. The `X-Account-Context` header provides the account ID used for scoping data.

### Endpoint Reference

| Method | Path | Handler | Trigger | Notes |
|--------|------|---------|---------|-------|
| **GET** | `/clusters` | `ListClustersHandler` | API Gateway | List all clusters for account |
| **GET** | `/clusters/{clusterId}` | `GetClusterDetailHandler` | API Gateway | Cluster detail with nodes/pods |
| **POST** | `/container-security/protect` | `ProtectClusterHandler` | API Gateway | Queue cluster for protection |
| **GET** | `/clusters/{clusterId}/vulnerabilities` | `GetClusterVulnerabilitiesHandler` | API Gateway | List vulnerabilities |
| **GET** | `/clusters/{clusterId}/events/{eventType}` | `GetClusterEventsHandler` | API Gateway | Unified events endpoint |
| _(internal)_ | SQS trigger | `InstallTrendAgentHandler` | SQS | Async cluster installation |
| _(internal)_ | SQS trigger | `DeleteClusterHandler` | SQS | Async cluster deletion |

### Path Parameter: `{clusterId}`

The `clusterId` is the cloud provider's native resource ID. It must be **URL-encoded** when passed in the path, since these IDs contain special characters:

- **OCI:** `ocid1.cluster.oc1.iad.aaaa...` (dots)
- **AWS:** `arn:aws:eks:us-east-1:123456789012:cluster/name` (colons, slashes)
- **Azure:** `/subscriptions/xxx/resourceGroups/...` (slashes)
- **GCP:** `projects/xxx/locations/...` (slashes)

The backend URL-decodes via `Uri.UnescapeDataString()`.

### Query Parameter: `trendClusterId`

Several endpoints accept an optional `?trendClusterId=` query parameter. When provided, the backend calls Trend Vision One's detail API directly using this internal ID instead of searching by name. This is faster and more reliable.

**Supported on:** `GET /clusters/{clusterId}`, `GET /clusters/{clusterId}/vulnerabilities`, `GET /clusters/{clusterId}/events/{eventType}`

### Event Types for `/clusters/{clusterId}/events/{eventType}`

The `{eventType}` path parameter must be one of:

| eventType | Description |
|-----------|-------------|
| `malware` | Malware detection events |
| `fim` | File integrity monitoring events |
| `syscall` | Runtime security / MITRE ATT&CK events |
| `secretScan` | Secret scan events |

## Request/Response Models

### Error Response (all endpoints)

All endpoints return errors in a consistent shape:

```json
{
  "success": false,
  "message": "Human-readable error description"
}
```

**HTTP Status Codes:**
| Code | Meaning |
|------|---------|
| 400 | Validation failure (missing header, invalid provider, missing clusterId, etc.) |
| 502 | Trend Vision One API error (upstream failure) |
| 503 | Failed to queue SQS message |
| 500 | Unhandled server error |

### POST `/container-security/protect`

**Request:**
```json
{
  "clusterId": "ocid1.cluster.oc1...",
  "provider": "OCI",
  "cloudConnectionId": 42,
  "region": "us-ashburn-1"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `clusterId` | string | yes | Cloud provider resource ID |
| `provider` | string | yes | Must be: `SELF`, `AWS`, `AZURE`, `GCP`, `OCI` (case-insensitive) |
| `cloudConnectionId` | int | yes | Must be > 0 |
| `region` | string | no | Cluster region |

**Response (200):**
```json
{
  "status": "queued",
  "clusterId": "ocid1.cluster.oc1...",
  "provider": "OCI",
  "message": "Installation request queued. Check cluster status for progress."
}
```

### GET `/clusters`

**Response (200):** Array of cluster objects.

```typescript
[
  {
    AccountId: string;
    ClusterId: string;          // cloud provider resource ID
    ClusterName: string;
    CompartmentId: string;
    CompartmentName: string;
    Provider: string;           // "oci"
    CloudConnectionId: number;  // int
    ClusterConnectivity: string; // OCI lifecycle state (e.g. "ACTIVE")
    ProtectionStatus: string;   // "Protected" | "Unprotected" | "Pending" | "Needs Attention" | "Failed"
    ResourceId: string;
    OciRegion: string;
    CreatedDateTime: string;    // empty string if unprotected
    UpdatedDateTime: string;    // empty string if unprotected
    TrendClusterName: string;
    TrendClusterId?: string;    // omitted if null
    TrendRuntimeSecurityEnabled?: boolean;
    TrendVulnerabilityScanEnabled?: boolean;
    TrendMalwareScanEnabled?: boolean;
    TrendSecretScanEnabled?: boolean;
    HasSecretScanEvents?: boolean;
    HasFileIntegrityEvents?: boolean;
    HasMalwareEvents?: boolean;
    HasSyscallEvents?: boolean;
    HasVulnerabilities?: boolean;
    VulnerabilityCriticalCount?: number;
    VulnerabilityHighCount?: number;
    VulnerabilityMediumCount?: number;
    VulnerabilityLowCount?: number;
    MalwareEventsCount?: number;
    FileIntegrityEventsCount?: number;
    SecretScanEventsCount?: number;
    SyscallEventsCount?: number;
  }
]
```

**Notes:**
- `CloudConnectionId` is always `int` (never string)
- Optional fields (`?`) are omitted from JSON when null (not sent as `null`)
- `ProtectionStatus` logic: `Failed` (RDS install failed) > `Protected`/`Needs Attention` (Trend match) > `Pending` (created < 24h or in_progress) > `Unprotected`

### GET `/clusters/{clusterId}`

**Query params:** `?trendClusterId=xxx` (optional, recommended for performance)

**Response (200):**

```typescript
{
  ClusterId: string;
  ClusterName: string;
  AccountId?: string;            // account ID from X-Account-Context header
  CompartmentId: string;
  CompartmentName: string;
  Provider: string;              // "OCI", "AWS", "Azure", "GCP", "unknown"
  ClusterConnectivity: string;   // "ACTIVE", "Unknown"
  ProtectionStatus: string;      // "Protected" | "Unprotected" | "Pending" | "Needs Attention"
  CloudConnectionId?: number;    // int, omitted if null
  Region?: string;
  Namespace?: string;            // defaults to "trendmicro-system"
  K8sVersion?: string;
  NodesCount?: number;
  PodsCount?: number;            // excludes trendmicro-system namespace pods
  ServicesCount?: number;
  NamespacesCount?: number;      // excludes trendmicro-system namespace
  CreatedAt?: string;            // ISO 8601
  UpdatedAt?: string;            // ISO 8601
  LastScannedAt?: string;        // ISO 8601
  AgentVersion?: string;
  VulnerabilityCriticalCount?: number;
  VulnerabilityHighCount?: number;
  VulnerabilityMediumCount?: number;
  VulnerabilityLowCount?: number;
  MalwareEventsCount?: number;
  FileIntegrityEventsCount?: number;
  SecretScanEventsCount?: number;
  SyscallEventsCount?: number;
  Nodes?: ClusterNode[];
}
```

**ClusterNode:**
```typescript
{
  Id: string;
  Name: string;
  Status: string;
  ProtectionStatus: string;
  LaunchType?: string;
  CreatedDateTime?: string;
  LastEvaluatedDateTime?: string;
  OsVersion?: string;
  KernelVersion?: string;
  IpAddresses?: string[];
  Pods?: ClusterPod[];         // excludes trendmicro-system namespace pods
}
```

**ClusterPod:**
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
  DeletionStatus?: string;      // "deleting" | "deleted" | omitted (null)
}
```

### GET `/clusters/{clusterId}/vulnerabilities`

**Query params:** `?trendClusterId=xxx` (optional)

**Response (200):** Array of vulnerability objects.

```typescript
[
  {
    id: string;                    // generated: "{cveId}_{hash}"
    cveId: string;                 // e.g. "CVE-2021-44228"
    severity: string;              // "Critical" | "High" | "Medium" | "Low" | "Unknown"
    packageName: string;
    packageVersion: string;
    fixedVersion?: string;
    imageName?: string;
    imageTag?: string;             // first 12 chars of digest
    namespace?: string;            // always null (not available from Trend)
    podName?: string;              // always null
    containerName?: string;        // always null
    detectedAt: string;            // ISO 8601
    description?: string;
    cvssScore?: number;            // 0.0 - 10.0
    link?: string;                 // NVD link
    clusterId?: string;            // Trend cluster ID
    clusterType?: string;
    imageId?: string;
    registry?: string;
    repository?: string;
    digest?: string;
    riskLevel?: string;
    firstDetectedDateTime?: string;
    lastDetectedDateTime?: string;
    cvssAttackVector?: string;
    packages?: VulnerabilityPackage[];
  }
]
```

### GET `/clusters/{clusterId}/events/{eventType}`

**Query params:** `?trendClusterId=xxx` (optional)

**Response (200):**
```typescript
{
  items: EventObject[];     // type depends on eventType
  count: number;
  eventType: string;        // echoes back the eventType
  nextLink?: string;        // pagination cursor (omitted if null)
}
```

**Event item shapes by type:**

#### `malware`
```typescript
{
  id: string; malwareName: string; severity: string;
  fileName: string; filePath: string; fileHash?: string;
  imageName?: string; imageTag?: string;
  namespace?: string; podName?: string; containerName?: string;
  detectedAt: string; status?: string; threatType?: string;
  description?: string; mitigation?: string; policyName?: string;
  scanType?: string; containerId?: string; imageDigest?: string;
}
```

#### `fim` (file integrity)
```typescript
{
  id: string; severity: string;
  fileName: string; filePath: string; fileHash?: string;
  eventAction?: string; fileOperation?: string;
  imageName?: string; imageTag?: string;
  namespace?: string; podName?: string; containerName?: string;
  detectedAt: string; status?: string; mitigation?: string;
  policyName?: string; ruleName?: string; ruleId?: string; ruleType?: string;
  containerId?: string; imageDigest?: string;
  processName?: string; processExecutable?: string;
  userName?: string; nodeHostname?: string;
  fileNameOld?: string; fileSize?: string; fileAttributes?: string;
  scanType?: string; clusterId?: string; clusterName?: string;
}
```

#### `syscall` (runtime security)
```typescript
{
  id: string; severity: string;
  ruleId: string; ruleName: string;
  eventCategory: string; eventAction: string;
  processName: string; processCommandLine: string;
  processExecutable: string; processId: string;
  parentProcessName: string; parentProcessCommandLine: string;
  userName: string;
  imageName: string; imageTag: string; imageDigest: string;
  namespace: string; podName: string;
  containerName: string; containerId: string;
  nodeHostname: string;
  detectedAt: string; status: string; mitigation: string;
  policyName: string; ruleType: string;
  tags: string[];                // MITRE ATT&CK tags
  rulesets: { id: string; name: string; }[];
}
```

#### `secretScan`
```typescript
{
  id: string; severity: string;
  secretRuleId: string; secretDescription: string; secretValue: string;
  fileName: string; filePath: string; fileType: string;
  startLine: string; endLine: string;
  startColumn: string; endColumn: string;
  scanType: string;
  imageName: string; imageTag: string; imageDigest: string;
  namespace: string; podName: string;
  containerName: string; containerId: string;
  nodeHostname: string;
  detectedAt: string; status: string; mitigation: string;
  policyName: string; ruleType: string;
}
```

## Important Backend Behaviors

### Protection Status Logic

The backend determines `ProtectionStatus` using this priority:

1. **"Failed"** - RDS installation record shows `failed` status
2. **"Protected"** - Trend API returns `ProtectionStatus: "HEALTHY"`
3. **"Needs Attention"** - Trend API returns a non-HEALTHY `ProtectionStatus`
4. **"Pending"** - Cluster created in Trend < 24 hours ago, or RDS shows `in_progress`
5. **"Unprotected"** - No Trend match found

### Pod/Namespace Filtering

The backend **excludes** pods and namespaces in `trendmicro-system` from:
- `PodsCount` in cluster detail
- `NamespacesCount` in cluster detail
- `Pods[]` array within each node

### Graceful Degradation

- If the Trend API is unreachable, the clusters list still returns with all clusters showing `ProtectionStatus: "Unprotected"`
- Individual summary/event-count failures per cluster are logged as warnings; other clusters still get their data

### Async Operations

`POST /container-security/protect` does **not** install immediately. It queues an SQS message and returns `{ status: "queued" }`. The actual installation happens asynchronously via `InstallTrendAgentHandler`. Poll `GET /clusters` to check when `ProtectionStatus` transitions from `"Pending"` to `"Protected"` or `"Failed"`.

### Kill Pod

The kill pod endpoint (`POST /clusters/{clusterId}/actions/kill`) is **not yet implemented in the backend**. The frontend hook (`useKillPod`) exists but currently relies on the frontend platform's proxy routing. If the backend needs to handle this, the handler would need to be added to `template.yml`.

## Supported Providers

| Provider Key | Cloud Platform |
|-------------|----------------|
| `SELF` | Self-managed Kubernetes |
| `AWS` | Amazon EKS |
| `AZURE` | Microsoft AKS |
| `GCP` | Google GKE |
| `OCI` | Oracle Cloud OKE |

Provider keys are **case-insensitive** in the protect endpoint.

## Recent Changes (feature/AA-32278)

- Added `GET /clusters/{clusterId}` detail endpoint with nodes, pods, and vulnerability/event counts
- Added `CloudConnectionId` to cluster list and detail responses
- Pod counts exclude `trendmicro-system` namespace (Trend internal pods)
- Added `trendClusterId` query parameter for direct Trend API lookups
- Added event counts (malware, FIM, secrets, syscall) to both list and detail responses
