# Revised Architectural Design: Automated Error Detection & GitHub Issue Creation System

## Executive Summary

This revised architecture addresses the clarified requirements for an autonomous, high-volume log monitoring system that detects failures in Azure Log Diagnostics, creates GitHub issues with context, and assigns them to AI assistants (Claude Code or Copilot). The design emphasizes streaming architecture for high-volume processing, multi-team support with centralized administration, and extensibility for future enhancements.

**Document Version**: 2.0
**Date**: 2025-12-08
**Status**: Revised Design Based on Updated Requirements

---

## 1. Requirements Analysis & Changes

### 1.1 Updated Requirements Summary

**Core Functional Requirements:**
- ✅ Monitor Azure Log Diagnostics for failures (JSON format initially)
- ✅ Create GitHub issues with failure context
- ✅ Implement duplicate detection to eliminate redundant issues
- ✅ Assign issues to Claude Code or Copilot (investigation is out of scope)
- ✅ Support multiple teams with central administrator
- ✅ Handle high log volumes
- ✅ Extensible for additional log sources and issue platforms

**Configuration Requirements:**
- ✅ Configurable failure definitions and evaluation criteria
- ✅ Configurable thresholds for issue generation
- ✅ Configurable Log Analytics tables to monitor
- ✅ Failure-to-repository mapping for proper routing
- ✅ Assignment specification (Claude Code vs Copilot)

**Azure Specifications:**
- ✅ Use Managed Identity for authentication
- ✅ Document required RBAC permissions

### 1.2 Key Changes from Previous Design

| Aspect | Previous Design | Revised Design | Rationale |
|--------|----------------|----------------|-----------|
| **Volume Assumption** | Moderate (< 10K/min) | High (unspecified, assume streaming) | Requirements explicitly state "high volume" |
| **Architecture Pattern** | Polling-based | Event-driven streaming | High volume requires real-time processing |
| **Deployment** | Azure Container Apps | Azure Container Apps + Event Hub | Streaming architecture for scalability |
| **Scope** | Issue creation + investigation prep | Issue creation + assignment ONLY | Investigation explicitly out of scope |
| **Multi-team** | Not specified | Multi-team with central admin | Requirements clarify multi-team support |
| **Assignment Detail** | Generic AI assignment | Claude Code/Copilot via GitHub | More specific assignment mechanism |

### 1.3 Design Goals

1. **High Throughput**: Handle high-volume log streams without data loss
2. **Real-time Processing**: Detect and report failures with minimal latency
3. **Reliability**: Ensure no failures are missed or duplicated
4. **Extensibility**: Support multiple log sources and issue platforms
5. **Maintainability**: Clear architecture for central administrator
6. **Cost Efficiency**: Optimize Azure resource usage

---

## 2. High-Level Architecture

### 2.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Error Detection System (Streaming)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐  │
│  │  Log Streaming   │      │   Stream         │      │   Failure        │  │
│  │  Ingestion       │─────>│   Processing     │─────>│   Detector       │  │
│  │  (Event Hub)     │      │   Pipeline       │      │   Engine         │  │
│  └──────────────────┘      └──────────────────┘      └──────────────────┘  │
│         ▲                           │                          │             │
│         │                           │                          ▼             │
│  ┌──────────────────┐               │                  ┌──────────────────┐ │
│  │  Diagnostic      │               │                  │   Duplicate      │ │
│  │  Settings        │               │                  │   Detection      │ │
│  │  Export          │               ▼                  │   Service        │ │
│  └──────────────────┘      ┌──────────────────┐       └──────────────────┘ │
│                            │   State Store    │               │             │
│                            │   (Redis/CosmosDB│◄──────────────┘             │
│                            └──────────────────┘                             │
│                                     │                                        │
│                                     ▼                                        │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐ │
│  │  Configuration   │◄────│   Issue          │◄────│   Repository     │ │
│  │  Manager         │      │   Creator        │      │   Router         │ │
│  └──────────────────┘      └──────────────────┘      └──────────────────┘ │
│                                     │                                        │
│                                     ▼                                        │
│                            ┌──────────────────┐                             │
│                            │   Assignment     │                             │
│                            │   Dispatcher     │                             │
│                            └──────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │   GitHub API     │
                            │   - Issue Create │
                            │   - Assign User  │
                            └──────────────────┘

External Azure Services:
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────┐
│  Azure Log          │───>│  Azure Event Hub    │───>│  This System    │
│  Analytics          │    │  (Stream Ingestion) │    │  (Container App)│
│  (Diagnostics)      │    └─────────────────────┘    └─────────────────┘
└─────────────────────┘
```

### 2.2 Data Flow Overview

```
1. Log Export:
   Azure Log Analytics → Diagnostic Settings → Event Hub (streaming)

2. Log Processing:
   Event Hub → Stream Processor → Failure Detection Engine

3. Duplicate Detection:
   Failure Detected → Fingerprint Generation → Cache/CosmosDB Lookup

4. Issue Creation:
   New Failure → Repository Router → GitHub Issue API

5. Assignment:
   Issue Created → Assignment Dispatcher → Assign to Claude Code/Copilot
```

---

## 3. Detailed Component Design

### 3.1 Log Streaming Ingestion Layer

**Purpose**: Receive and process log streams in real-time from Azure Log Analytics

#### Components:

1. **Azure Diagnostic Settings Export**
   - **What**: Native Azure feature to export logs to Event Hub
   - **Configuration**: Set up per Log Analytics workspace
   - **Tables to Export**: Configurable (AppServiceConsoleLogs, AppExceptions, etc.)
   - **Format**: JSON (native)

2. **Azure Event Hub Consumer**
   - **What**: Receives log events from Event Hub partitions
   - **Library**: `azure-eventhub` (Python)
   - **Pattern**: Consumer group for parallel processing
   - **Checkpointing**: Uses Azure Storage for offset tracking

3. **Stream Processor**
   - **What**: Processes events in batches for efficiency
   - **Batch Size**: Configurable (default: 100 events or 5 seconds)
   - **Error Handling**: Dead-letter queue for unparseable logs
   - **Backpressure**: Circuit breaker if downstream overwhelmed

#### Advantages over Polling:
- ✅ **Real-time**: Sub-second latency
- ✅ **Scalable**: Event Hub handles millions of events/sec
- ✅ **Reliable**: Built-in event persistence and replay
- ✅ **Cost-effective**: Pay for throughput units used
- ✅ **Native Integration**: First-class Azure support

#### RBAC Permissions Required:
```
Resource: Azure Event Hubs Namespace
Role: Azure Event Hubs Data Receiver
Scope: Event Hub instance

Resource: Azure Log Analytics Workspace
Role: Log Analytics Reader
Scope: Workspace (for validation/queries)

Resource: Azure Storage Account (checkpoint store)
Role: Storage Blob Data Contributor
Scope: Container for checkpoints
```

### 3.2 Failure Detection Engine

**Purpose**: Analyze log events and identify failures based on configurable rules

#### Components:

1. **Rule Engine**
   - **Pattern**: Strategy pattern for pluggable rules
   - **Rule Types**:
     - Field comparison (equals, regex, range)
     - JSON path queries for nested fields
     - Composite rules (AND/OR logic)
   - **Performance**: Compile rules to bytecode for fast evaluation

2. **Pattern Matcher**
   - **Regex Engine**: Precompiled patterns for performance
   - **JSON Path Queries**: Using `jsonpath-ng` library
   - **Custom Predicates**: Python functions for complex logic

3. **Threshold Evaluator**
   - **Window Types**:
     - **Sliding Window**: Last N minutes (e.g., 5 failures in 10 min)
     - **Tumbling Window**: Fixed time blocks (e.g., per hour)
     - **Session Window**: Burst detection
   - **Storage**: Redis with TTL for window state
   - **Reset Logic**: Configurable (reset after issue creation or continue counting)

4. **Context Extractor**
   - **What**: Extracts relevant fields from matched logs
   - **Extracts**:
     - Error messages and exception types
     - Stack traces (first 50 lines)
     - Request IDs, correlation IDs
     - User context (sanitized)
     - Timestamps and frequency
   - **Sanitization**: Removes PII, secrets, tokens

#### Example Rule Configuration:
```yaml
failure_definitions:
  - name: "http_5xx_spike"
    description: "HTTP 500-series errors exceeding threshold"
    condition:
      field: "$.ResultCode"  # JSON path
      operator: "regex"
      pattern: "^5\\d{2}$"
    context_fields:
      - "$.RequestUri"
      - "$.Message"
      - "$.StackTrace"
    threshold:
      count: 10
      window_minutes: 5
      window_type: "sliding"
    severity: "high"
    enabled: true
```

### 3.3 Duplicate Detection Service

**Purpose**: Prevent creating duplicate issues for the same root cause

#### Recommended Strategy: **Multi-Level Approach**

##### Level 1: Exact Fingerprint (Fast Path)
```python
def exact_fingerprint(failure):
    """O(1) lookup - catches identical failures"""
    data = f"{failure.type}:{failure.message}:{failure.stack_trace}"
    return hashlib.sha256(data.encode()).hexdigest()[:16]
```

**Cache**: Redis with 24-hour TTL
**Lookup Time**: < 1ms
**Accuracy**: 100% for identical failures

##### Level 2: Structural Fingerprint (Recommended)
```python
def structural_fingerprint(failure):
    """Normalized fingerprint - catches similar failures"""
    # Normalize stack trace
    normalized_trace = normalize_stack_trace(failure.stack_trace)
    normalized_msg = normalize_message(failure.message)

    # Use first 3 frames + exception type
    key_frames = extract_key_frames(normalized_trace, n=3)

    data = f"{failure.type}:{normalized_msg}:{key_frames}"
    return hashlib.sha256(data.encode()).hexdigest()[:16]

def normalize_stack_trace(stack):
    """Remove variable content"""
    normalized = re.sub(r'\b\d+\b', '<NUM>', stack)           # Numbers
    normalized = re.sub(r'0x[0-9a-fA-F]+', '<ADDR>', normalized)  # Memory addresses
    normalized = re.sub(r'\b[0-9a-f-]{36}\b', '<UUID>', normalized)  # UUIDs
    normalized = re.sub(r':\d+', ':<LINE>', normalized)       # Line numbers
    return normalized
```

**Cache**: Redis with 48-hour TTL
**Lookup Time**: < 5ms
**Accuracy**: ~90% for similar failures

##### Level 3: Temporal Grouping
- Group failures within 15-minute window
- Same repository + same failure type
- Update existing issue with occurrence count

##### Level 4: GitHub Issue Search (Fallback)
```python
def search_existing_issues(failure, repo):
    """Search GitHub for similar open issues"""
    query = f"repo:{repo} is:open label:auto-detected {failure.type}"
    issues = github_api.search_issues(query)

    # Check if any match structurally
    for issue in issues:
        if structural_match(failure, issue):
            return issue
    return None
```

**When**: If fingerprint cache misses
**Frequency**: Max once per failure type per minute (rate limited)

#### Duplicate Handling Logic:
```
if exact_fingerprint in cache:
    → Update existing issue comment with new occurrence
    → Increment counter
    → Skip issue creation

elif structural_fingerprint in cache:
    → Check if threshold changed (e.g., 10 → 50 occurrences)
    → Update issue if threshold crossed
    → Skip new issue creation

elif temporal_window matches:
    → Group into existing issue
    → Add comment with occurrence details

else:
    → Create new issue
    → Store fingerprints in cache
```

#### State Storage:
- **Redis Cache**: Fingerprints with TTL (fast lookups)
- **Cosmos DB**: Durable failure history (analysis, reporting)
- **Mapping**: Fingerprint → GitHub Issue URL + ID

### 3.4 Issue Creation Service

**Purpose**: Create well-formatted GitHub issues in appropriate repositories

#### Components:

1. **Repository Router**
   - **Configuration**: Maps failure types to repositories
   - **Fallback**: Default repository if no match
   - **Validation**: Checks repository exists and is accessible
   - **Multi-repo Support**: One failure type can map to multiple repos

2. **Issue Template Engine**
   - **Template Format**: Jinja2 templates
   - **Variables**: Failure context, metadata, query links
   - **Customization**: Per failure type or per repository

3. **GitHub Issue Adapter**
   - **Library**: PyGithub
   - **Operations**: Create issue, add labels, assign users
   - **Rate Limiting**: Respect GitHub API limits (5000/hour)
   - **Retry Logic**: Exponential backoff on failures

#### Issue Template Example:
```markdown
## 🚨 Automated Failure Detection

**Failure Type**: {{ failure.name }}
**Severity**: {{ failure.severity }}
**First Detected**: {{ failure.first_seen }}
**Occurrences**: {{ failure.count }} times in {{ failure.window }} minutes

### Error Details
{{ failure.message }}

### Stack Trace
```
{{ failure.stack_trace }}
```

### Context
- **Request ID**: {{ context.request_id }}
- **Operation**: {{ context.operation }}
- **Table**: {{ context.table_name }}

### Log Analytics Query
[View in Azure Portal]({{ azure_query_url }})

### Investigation Notes
This issue was automatically detected and assigned for investigation.

---
*Generated by Error Detection System*
*Fingerprint*: `{{ failure.fingerprint }}`
```

#### Labels Applied:
- `auto-detected` (all issues)
- `severity:high|medium|low` (based on configuration)
- `failure-type:{{ name }}` (e.g., `failure-type:http_5xx`)
- Custom labels from configuration

### 3.5 Assignment Dispatcher

**Purpose**: Assign created issues to Claude Code or Copilot for investigation

#### Assignment Mechanism:

**Option 1: GitHub Username Assignment (Recommended)**
```python
# Assign to GitHub user/bot account
github_api.assign_issue(
    repo="my-org/backend-api",
    issue_number=123,
    assignees=["claude-code-bot"]  # or "copilot-assistant"
)
```

**Option 2: GitHub Teams**
```python
# Assign to team (requires org permissions)
github_api.assign_issue_to_team(
    repo="my-org/backend-api",
    issue_number=123,
    team_slug="ai-investigators"
)
```

#### Assignment Strategy:

```yaml
assignment:
  default: "claude-code-bot"  # GitHub username

  # Override by failure type
  rules:
    - failure_type: "http_5xx_errors"
      assignee: "claude-code-bot"

    - failure_type: "frontend_errors"
      assignee: "copilot-assistant"

  # Rotation (optional)
  rotation:
    enabled: false
    agents: ["claude-code-bot", "copilot-assistant"]
    strategy: "round-robin"  # or "least-assigned"
```

#### Assignment Comment:
```python
def add_assignment_comment(issue, assignee):
    """Add investigation instructions"""
    comment = f"""
🤖 **Assigned to @{assignee} for investigation**

### Investigation Checklist
- [ ] Analyze error pattern and root cause
- [ ] Determine if this is a regression or new issue
- [ ] Assess severity and impact scope
- [ ] Note in comments: Investigation out of scope for this system

### Resources
- [Log Analytics Workspace](link)
- [Related Documentation](link)
- [Team Runbook](link)
"""
    github_api.create_comment(issue, comment)
```

**Note**: The system only creates and assigns issues. Investigation and PR creation are explicitly out of scope.

### 3.6 Configuration Manager

**Purpose**: Manage multi-team configuration with central administration

#### Configuration Storage Options:

**Option 1: Git-based Configuration (Recommended)**
```
config/
├── global.yaml              # Global settings
├── teams/
│   ├── team-a.yaml         # Team A specific config
│   ├── team-b.yaml         # Team B specific config
│   └── team-c.yaml
└── repositories/
    ├── repo-mapping.yaml   # Failure → Repo mapping
    └── templates/          # Issue templates per repo
```

**Advantages**:
- ✅ Version controlled
- ✅ Audit trail of changes
- ✅ GitOps workflow (PR-based changes)
- ✅ Easy rollback

**Option 2: Cosmos DB Configuration**
- Dynamic updates without restart
- UI for central admin
- More complex to implement

#### Configuration Schema:

```yaml
# global.yaml
system:
  admin_email: "admin@example.com"
  environment: "production"

azure:
  event_hub:
    namespace: "logs-streaming"
    hub_name: "diagnostics-export"
    consumer_group: "$Default"

  log_analytics:
    workspace_id: "${AZURE_WORKSPACE_ID}"

github:
  organization: "my-org"
  token: "${GITHUB_TOKEN}"
  rate_limit_buffer: 1000  # Reserve 1000 requests/hour

duplicate_detection:
  strategy: "structural"
  cache_ttl_hours: 48
  time_window_hours: 24

# teams/team-a.yaml
team:
  name: "team-a"
  display_name: "Backend Services Team"

failure_definitions:
  - name: "api_5xx_errors"
    condition:
      field: "$.ResultCode"
      operator: "regex"
      pattern: "^5\\d{2}$"
    threshold:
      count: 10
      window_minutes: 5
    context_fields:
      - "$.RequestUri"
      - "$.Message"

repository_routing:
  - failure: "api_5xx_errors"
    repository: "my-org/backend-api"
    labels: ["bug", "auto-detected", "severity:high"]
    assignee: "claude-code-bot"
```

#### Hot Reload:
```python
class ConfigManager:
    def __init__(self, config_path):
        self.config_path = config_path
        self.config = self.load_config()
        self.watch_for_changes()  # File watcher

    def watch_for_changes(self):
        """Reload config on file change"""
        # Use watchdog library for file monitoring
        # Validate config before applying
        # Gracefully update without restart
```

### 3.7 State Store

**Purpose**: Maintain system state for reliability and duplicate detection

#### Storage Architecture:

```
┌─────────────────────────────────────────────┐
│           State Store (Dual-tier)           │
├─────────────────────────────────────────────┤
│                                             │
│  ┌────────────────────────────────────┐    │
│  │   Redis Cache (Hot Data)           │    │
│  │   - Fingerprint cache (TTL)        │    │
│  │   - Threshold counters (TTL)       │    │
│  │   - Rate limit tracking            │    │
│  │   Retention: 24-48 hours            │    │
│  └────────────────────────────────────┘    │
│           │                                 │
│           │ (Write-through)                 │
│           ▼                                 │
│  ┌────────────────────────────────────┐    │
│  │   Cosmos DB (Durable Storage)      │    │
│  │   - Failure history (queryable)    │    │
│  │   - Issue creation audit log       │    │
│  │   - Configuration backups          │    │
│  │   Retention: 90 days default        │    │
│  └────────────────────────────────────┘    │
│                                             │
│  ┌────────────────────────────────────┐    │
│  │   Event Hub Checkpoint Store       │    │
│  │   (Azure Storage - Blob)           │    │
│  │   - Consumer offset tracking       │    │
│  │   - Partition assignment           │    │
│  └────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

#### Data Models:

**Redis Cache Keys:**
```
fingerprint:exact:{hash}        → {issue_url, created_at, count}
fingerprint:structural:{hash}   → {issue_url, created_at, count}
threshold:{failure_type}:{window} → {count, first_occurrence}
ratelimit:github:create         → {count, reset_at}
```

**Cosmos DB Collections:**
```javascript
// failures collection
{
  "id": "uuid",
  "fingerprint": "abc123",
  "failure_type": "http_5xx_errors",
  "message": "Internal Server Error",
  "stack_trace": "...",
  "severity": "high",
  "first_seen": "2025-12-08T10:00:00Z",
  "last_seen": "2025-12-08T10:15:00Z",
  "occurrence_count": 15,
  "github_issue_url": "https://github.com/org/repo/issues/123",
  "assigned_to": "claude-code-bot",
  "team": "team-a",
  "repository": "my-org/backend-api"
}

// audit_log collection
{
  "id": "uuid",
  "event_type": "issue_created",
  "timestamp": "2025-12-08T10:00:00Z",
  "failure_id": "uuid",
  "github_issue_number": 123,
  "repository": "my-org/backend-api",
  "assigned_to": "claude-code-bot",
  "details": {}
}
```

---

## 4. Technology Stack (Updated)

### 4.1 Programming Language: **Python 3.11+**

**Rationale** (unchanged from previous design, still optimal):
- ✅ Excellent Azure SDK support
- ✅ Strong async support for streaming
- ✅ Rich ecosystem for data processing
- ✅ Easy Event Hub integration

### 4.2 Deployment Architecture

**Primary**: **Azure Container Apps**
```yaml
Container App Configuration:
  - Replica count: 1-5 (auto-scale)
  - CPU: 0.5-2 vCPU
  - Memory: 1-4 GB
  - Scale rules:
    - Event Hub partition count
    - CPU utilization > 70%
    - Memory utilization > 80%
```

**Why Container Apps**:
- ✅ Supports long-running event processors
- ✅ Native Event Hub trigger scaling
- ✅ Built-in Managed Identity support
- ✅ Cost-effective (scale to zero when no events)

### 4.3 Core Dependencies

```python
# Azure Integration
azure-eventhub>=5.11.0              # Event Hub consumer
azure-eventhub-checkpointstoreblob>=1.1.4  # Checkpointing
azure-identity>=1.15.0              # Managed Identity auth
azure-keyvault-secrets>=4.7.0       # Secret management
azure-cosmos>=4.5.0                 # Durable storage

# GitHub Integration
PyGithub>=2.1.1                     # GitHub API client

# Data Processing
pydantic>=2.5.0                     # Configuration validation
pydantic-settings>=2.1.0            # Settings management
jsonpath-ng>=1.6.0                  # JSON log parsing

# Caching & State
redis>=5.0.0                        # Redis client
hiredis>=2.3.0                      # Redis protocol (performance)

# Utilities
tenacity>=8.2.3                     # Retry logic
structlog>=23.2.0                   # Structured logging
watchdog>=3.0.0                     # Config file watching

# Optional: Semantic matching
sentence-transformers>=2.2.0        # For semantic duplicate detection
```

### 4.4 Azure Services Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Services (Required)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────┐         ┌────────────────────┐          │
│  │  Log Analytics     │────────>│  Diagnostic        │          │
│  │  Workspace         │         │  Settings Export   │          │
│  └────────────────────┘         └────────────────────┘          │
│                                           │                      │
│                                           ▼                      │
│                                  ┌────────────────────┐          │
│                                  │  Event Hub         │          │
│                                  │  Namespace         │          │
│                                  │  - Standard tier   │          │
│                                  │  - 4 partitions    │          │
│                                  └────────────────────┘          │
│                                           │                      │
│                                           ▼                      │
│                                  ┌────────────────────┐          │
│                                  │  Container App     │          │
│                                  │  (This System)     │          │
│                                  │  - Managed Identity│          │
│                                  └────────────────────┘          │
│                                           │                      │
│                    ┌──────────────────────┴───────────────┐     │
│                    ▼                                       ▼     │
│          ┌────────────────────┐              ┌────────────────────┐
│          │  Redis Cache       │              │  Cosmos DB         │
│          │  (Azure Cache)     │              │  (NoSQL)           │
│          │  - Basic tier      │              │  - Serverless      │
│          └────────────────────┘              └────────────────────┘
│                                                                   │
│  ┌────────────────────┐         ┌────────────────────┐          │
│  │  Storage Account   │         │  Key Vault         │          │
│  │  (Checkpoints)     │         │  (Secrets)         │          │
│  └────────────────────┘         └────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

#### Cost Estimate (Monthly, USD):
- Event Hub (Standard, 4 partitions): ~$50
- Container Apps (1-2 replicas): ~$50-100
- Redis Cache (Basic C1, 1GB): ~$16
- Cosmos DB (Serverless, low usage): ~$25-50
- Storage Account (checkpoints): ~$1
- **Total**: ~$150-250/month

### 4.5 Required Azure RBAC Permissions

```yaml
# Managed Identity RBAC Assignments

# 1. Event Hub Access
Resource: Event Hubs Namespace
Role: "Azure Event Hubs Data Receiver"
Scope: Event Hub instance "diagnostics-export"
Actions:
  - Microsoft.EventHub/namespaces/eventhubs/consumergroups/read
  - Microsoft.EventHub/namespaces/eventhubs/read
  - Microsoft.EventHub/namespaces/eventhubs/messages/receive/action

# 2. Checkpoint Storage Access
Resource: Storage Account (for checkpoints)
Role: "Storage Blob Data Contributor"
Scope: Container "eventhub-checkpoints"
Actions:
  - Microsoft.Storage/storageAccounts/blobServices/containers/read
  - Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read
  - Microsoft.Storage/storageAccounts/blobServices/containers/blobs/write

# 3. Log Analytics (for validation/queries - optional)
Resource: Log Analytics Workspace
Role: "Log Analytics Reader"
Scope: Workspace
Actions:
  - Microsoft.OperationalInsights/workspaces/read
  - Microsoft.OperationalInsights/workspaces/query/read

# 4. Key Vault (for secrets)
Resource: Key Vault
Role: "Key Vault Secrets User"
Scope: Key Vault instance
Actions:
  - Microsoft.KeyVault/vaults/secrets/read

# 5. Cosmos DB Access
Resource: Cosmos DB Account
Role: "Cosmos DB Built-in Data Contributor"
Scope: Database "error-detection"
Actions:
  - Microsoft.DocumentDB/databaseAccounts/readMetadata
  - Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/*
```

---

## 5. Event-Driven Processing Flow

### 5.1 End-to-End Flow

```
┌──────────────────┐
│ 1. Log Generated │  Application logs error
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Log Ingestion │  Sent to Log Analytics
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Export Setup  │  Diagnostic Settings → Event Hub
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Event Stream  │  Event Hub receives log events
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Consumer      │  Container App processes events
│    Processing    │  - Batch: 100 events / 5 seconds
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. Rule          │  Evaluate each event against rules
│    Evaluation    │
└────────┬─────────┘
         │
         ├─── (no match) ─── → Discard
         │
         ├─── (match) ────────▼
         │              ┌──────────────────┐
         │              │ 7. Threshold     │
         │              │    Check         │
         │              └────────┬─────────┘
         │                       │
         │                       ├─── (below threshold) → Update counter, continue
         │                       │
         │                       ├─── (threshold met) ────▼
         │                                          ┌──────────────────┐
         │                                          │ 8. Fingerprint   │
         │                                          │    Generation    │
         │                                          └────────┬─────────┘
         │                                                   │
         │                                                   ▼
         │                                          ┌──────────────────┐
         │                                          │ 9. Duplicate     │
         │                                          │    Detection     │
         │                                          └────────┬─────────┘
         │                                                   │
         │                                                   ├─── (duplicate) → Update existing issue
         │                                                   │
         │                                                   ├─── (new) ────▼
         │                                                          ┌──────────────────┐
         │                                                          │ 10. Repository   │
         │                                                          │     Routing      │
         │                                                          └────────┬─────────┘
         │                                                                   │
         │                                                                   ▼
         │                                                          ┌──────────────────┐
         │                                                          │ 11. Issue        │
         │                                                          │     Creation     │
         │                                                          └────────┬─────────┘
         │                                                                   │
         │                                                                   ▼
         │                                                          ┌──────────────────┐
         │                                                          │ 12. Assignment   │
         │                                                          │     Dispatch     │
         │                                                          └────────┬─────────┘
         │                                                                   │
         │                                                                   ▼
         │                                                          ┌──────────────────┐
         │                                                          │ 13. Store State  │
         │                                                          │     & Audit      │
         │                                                          └────────┬─────────┘
         │                                                                   │
         └───────────────────────────────────────────────────────────────────┘
                                                                             │
                                                                             ▼
                                                                    ┌──────────────────┐
                                                                    │ 14. Checkpoint   │
                                                                    │     Progress     │
                                                                    └──────────────────┘
```

### 5.2 Event Processing Code Structure

```python
async def process_event_batch(events: List[EventData]):
    """Process a batch of events from Event Hub"""

    for event in events:
        try:
            # 1. Parse log entry
            log_entry = json.loads(event.body_as_str())

            # 2. Evaluate against all failure definitions
            for failure_def in config.failure_definitions:
                if rule_engine.matches(log_entry, failure_def):

                    # 3. Check threshold
                    if threshold_evaluator.should_create_issue(failure_def):

                        # 4. Extract context
                        failure_context = context_extractor.extract(
                            log_entry, failure_def
                        )

                        # 5. Generate fingerprint
                        fingerprint = duplicate_detector.fingerprint(
                            failure_context
                        )

                        # 6. Check for duplicates
                        existing_issue = await duplicate_detector.find_duplicate(
                            fingerprint, failure_context
                        )

                        if existing_issue:
                            # Update existing issue
                            await issue_updater.add_occurrence(
                                existing_issue, failure_context
                            )
                        else:
                            # 7. Route to repository
                            repo = repository_router.get_repository(failure_def)

                            # 8. Create issue
                            issue = await issue_creator.create_issue(
                                repo, failure_context, failure_def
                            )

                            # 9. Assign to AI assistant
                            await assignment_dispatcher.assign(
                                issue, failure_def.assignment_config
                            )

                            # 10. Store state
                            await state_store.record_failure(
                                fingerprint, issue, failure_context
                            )

                            # 11. Audit log
                            await audit_logger.log_issue_created(
                                issue, failure_context
                            )

        except Exception as e:
            # Log error and continue processing
            logger.error("Failed to process event", error=str(e), event=event)
            await dead_letter_queue.send(event)

    # 12. Checkpoint batch progress
    await checkpoint_store.update_checkpoint(events[-1].offset)
```

---

## 6. Duplicate Detection: Detailed Strategy

### 6.1 Recommended Multi-Level Approach

```
┌────────────────────────────────────────────────────┐
│         Duplicate Detection Pipeline               │
├────────────────────────────────────────────────────┤
│                                                     │
│  Level 1: Exact Match (Cache Hit)                 │
│  ├─ Hash: error_type + message + stack_trace       │
│  ├─ Lookup: Redis O(1)                             │
│  ├─ TTL: 24 hours                                  │
│  └─ Hit Rate: ~60%                                 │
│           │                                         │
│           ├─ (hit) ──> Update counter, skip        │
│           │                                         │
│           └─ (miss) ──▼                            │
│                                                     │
│  Level 2: Structural Match (Normalized)           │
│  ├─ Normalize: stack trace + message               │
│  ├─ Hash: error_type + normalized_content          │
│  ├─ Lookup: Redis O(1)                             │
│  ├─ TTL: 48 hours                                  │
│  └─ Hit Rate: ~30% of misses from Level 1         │
│           │                                         │
│           ├─ (hit) ──> Update counter, skip        │
│           │                                         │
│           └─ (miss) ──▼                            │
│                                                     │
│  Level 3: Temporal Window (Time-based)            │
│  ├─ Query: Same failure type in last 15 min       │
│  ├─ Lookup: Cosmos DB partition query              │
│  ├─ Match: Same repository + failure type          │
│  └─ Hit Rate: ~5% of remaining                     │
│           │                                         │
│           ├─ (hit) ──> Add to existing issue       │
│           │                                         │
│           └─ (miss) ──▼                            │
│                                                     │
│  Level 4: GitHub Issue Search (Fallback)          │
│  ├─ Search: repo + labels + status:open            │
│  ├─ API Call: GitHub Search API                   │
│  ├─ Rate Limited: Max 30 searches/min              │
│  └─ Hit Rate: ~1% of remaining                     │
│           │                                         │
│           ├─ (hit) ──> Add comment to issue        │
│           │                                         │
│           └─ (miss) ──▼                            │
│                                                     │
│  ✅ Create New Issue                               │
│  └─ Store all fingerprints in cache                │
│                                                     │
└────────────────────────────────────────────────────┘
```

### 6.2 Normalization Rules

```python
def normalize_stack_trace(stack_trace: str) -> str:
    """Normalize stack trace to catch similar errors"""

    # 1. Remove variable content
    normalized = re.sub(r'\b\d+\b', '<NUM>', stack_trace)
    normalized = re.sub(r'0x[0-9a-fA-F]+', '<ADDR>', normalized)
    normalized = re.sub(r'\b[0-9a-f-]{36}\b', '<UUID>', normalized)
    normalized = re.sub(r'\b[0-9a-fA-F]{8,}\b', '<HEX>', normalized)

    # 2. Normalize line/column numbers
    normalized = re.sub(r':\d+:\d+', ':<LINE>:<COL>', normalized)
    normalized = re.sub(r'line \d+', 'line <NUM>', normalized)

    # 3. Remove timestamps
    normalized = re.sub(
        r'\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}',
        '<TIMESTAMP>',
        normalized
    )

    # 4. Normalize file paths (keep relative structure)
    normalized = re.sub(
        r'[A-Za-z]:[\\\/].*?([\\\/][^\\\/]+[\\\/][^\\\/]+)$',
        r'<PATH>\1',
        normalized
    )

    # 5. Remove request IDs
    normalized = re.sub(
        r'request[_-]?id[=:]\s*[a-zA-Z0-9-]+',
        'request_id=<ID>',
        normalized,
        flags=re.IGNORECASE
    )

    # 6. Extract only first 5 frames (most relevant)
    frames = extract_stack_frames(normalized)
    key_frames = frames[:5]

    return "\n".join(key_frames)

def normalize_error_message(message: str) -> str:
    """Normalize error message"""

    # Remove specific values
    normalized = re.sub(r'\b\d+\b', '<NUM>', message)
    normalized = re.sub(r'["\'].*?["\']', '<STR>', normalized)
    normalized = re.sub(r'https?://\S+', '<URL>', normalized)

    # Lowercase for case-insensitive matching
    normalized = normalized.lower()

    return normalized
```

### 6.3 Performance Characteristics

| Level | Avg Latency | Hit Rate | Storage | Cost/lookup |
|-------|-------------|----------|---------|-------------|
| Level 1 (Exact) | < 1ms | 60% | Redis | $0.00001 |
| Level 2 (Structural) | < 5ms | 30% | Redis | $0.00001 |
| Level 3 (Temporal) | < 20ms | 5% | Cosmos DB | $0.0001 |
| Level 4 (GitHub) | 100-500ms | 4% | GitHub API | $0 (rate limited) |

**Overall Performance**:
- 90% of duplicates caught in < 5ms (Level 1-2)
- 95% caught in < 20ms (Level 1-3)
- 99% caught in < 500ms (all levels)
- New issues: ~1% of total failures

---

## 7. Configuration Management

### 7.1 Multi-Team Configuration Structure

```
config/
├── global.yaml                    # System-wide settings
├── teams/
│   ├── backend-team.yaml         # Backend team config
│   ├── frontend-team.yaml        # Frontend team config
│   ├── mobile-team.yaml          # Mobile team config
│   └── data-team.yaml            # Data team config
├── repositories/
│   ├── repo-mapping.yaml         # Failure → Repository mapping
│   └── templates/
│       ├── backend-api.md        # Issue template for backend-api
│       ├── frontend-web.md       # Issue template for frontend-web
│       └── default.md            # Default template
└── schemas/
    └── config-schema.json        # JSON schema for validation
```

### 7.2 Example Configuration Files

#### global.yaml
```yaml
# System-wide configuration
system:
  version: "2.0"
  admin_email: "platform-team@example.com"
  environment: "production"

azure:
  event_hub:
    namespace: "logs-streaming-prod"
    hub_name: "diagnostics-export"
    consumer_group: "error-detection-system"

  storage:
    checkpoint_account: "logstorageprod"
    checkpoint_container: "eventhub-checkpoints"

  redis:
    host: "error-detection-cache.redis.cache.windows.net"
    port: 6380
    ssl: true

  cosmos_db:
    account: "error-detection-db"
    database: "failures"
    container: "failure_history"

github:
  organization: "my-organization"
  api_url: "https://api.github.com"
  rate_limit:
    max_per_hour: 4000  # Reserve buffer for safety
    max_issue_creates_per_hour: 500

duplicate_detection:
  default_strategy: "structural"
  cache_ttl_hours: 48
  temporal_window_minutes: 15

  # Normalization settings
  normalization:
    max_stack_frames: 5
    remove_timestamps: true
    remove_line_numbers: true

rate_limits:
  max_issues_per_hour_global: 500
  max_issues_per_repository_per_hour: 50

logging:
  level: "INFO"
  format: "json"
  output: "stdout"
```

#### teams/backend-team.yaml
```yaml
team:
  name: "backend-team"
  display_name: "Backend Services Team"
  contact_email: "backend-team@example.com"

# Failure definitions specific to backend team
failure_definitions:
  - name: "api_5xx_errors"
    description: "HTTP 500-series errors in API endpoints"
    enabled: true

    # Condition to match
    condition:
      field: "$.ResultCode"
      operator: "regex"
      pattern: "^5\\d{2}$"
      additional_filters:
        - field: "$.Category"
          operator: "equals"
          value: "ApplicationLogs"

    # Context to extract
    context_fields:
      - "$.RequestUri"
      - "$.Message"
      - "$.StackTrace"
      - "$.OperationName"
      - "$.CorrelationId"

    # Threshold configuration
    threshold:
      count: 10
      window_minutes: 5
      window_type: "sliding"
      reset_after_issue: false

    # Severity and priority
    severity: "high"
    priority: 1

    # Repository routing
    repository: "my-organization/backend-api"
    labels:
      - "bug"
      - "auto-detected"
      - "severity:high"
      - "team:backend"

    # Assignment
    assignee: "claude-code-bot"

    # Issue template override (optional)
    template: "backend-api"  # Uses templates/backend-api.md

  - name: "database_connection_failures"
    description: "Database connection timeouts or failures"
    enabled: true

    condition:
      field: "$.Message"
      operator: "regex"
      pattern: "(connection.*timeout|failed to connect|database unavailable)"

    context_fields:
      - "$.Message"
      - "$.ExceptionType"
      - "$.StackTrace"
      - "$.DatabaseName"

    threshold:
      count: 5
      window_minutes: 3
      window_type: "sliding"

    severity: "critical"
    priority: 0
    repository: "my-organization/backend-api"
    labels:
      - "bug"
      - "auto-detected"
      - "severity:critical"
      - "database"
    assignee: "claude-code-bot"

  - name: "unhandled_exceptions"
    description: "Unhandled application exceptions"
    enabled: true

    condition:
      field: "$.SeverityLevel"
      operator: "equals"
      value: "Error"
      additional_filters:
        - field: "$.ExceptionType"
          operator: "not_equals"
          value: "System.OperationCanceledException"  # Ignore cancellations

    context_fields:
      - "$.ExceptionType"
      - "$.ExceptionMessage"
      - "$.StackTrace"
      - "$.OperationName"

    threshold:
      count: 1  # Create issue on first occurrence
      window_minutes: 1

    severity: "high"
    repository: "my-organization/backend-api"
    labels:
      - "bug"
      - "auto-detected"
      - "exception"
    assignee: "copilot-assistant"

# Notification preferences (optional)
notifications:
  enabled: true
  channels:
    - type: "slack"
      webhook_url: "${BACKEND_TEAM_SLACK_WEBHOOK}"
      on_severity: ["critical", "high"]
      throttle_minutes: 30
```

#### repositories/repo-mapping.yaml
```yaml
# Global repository routing rules
# These apply across all teams unless overridden in team config

repositories:
  - name: "my-organization/backend-api"
    default_assignee: "claude-code-bot"
    labels:
      - "auto-detected"
    teams:
      - "backend-team"

  - name: "my-organization/frontend-web"
    default_assignee: "copilot-assistant"
    labels:
      - "auto-detected"
      - "frontend"
    teams:
      - "frontend-team"

  - name: "my-organization/mobile-app"
    default_assignee: "claude-code-bot"
    labels:
      - "auto-detected"
      - "mobile"
    teams:
      - "mobile-team"

# Fallback configuration
default:
  repository: "my-organization/platform-issues"
  assignee: "claude-code-bot"
  labels:
    - "auto-detected"
    - "needs-triage"
```

### 7.3 Configuration Validation

```python
from pydantic import BaseModel, Field, validator
from typing import List, Optional, Literal

class FailureCondition(BaseModel):
    field: str = Field(..., description="JSON path to field")
    operator: Literal["equals", "regex", "range", "contains", "not_equals"]
    pattern: Optional[str] = None
    value: Optional[str] = None
    additional_filters: Optional[List['FailureCondition']] = None

class ThresholdConfig(BaseModel):
    count: int = Field(..., ge=1, description="Number of failures")
    window_minutes: int = Field(..., ge=1, le=60)
    window_type: Literal["sliding", "tumbling"] = "sliding"
    reset_after_issue: bool = False

class FailureDefinition(BaseModel):
    name: str = Field(..., regex=r'^[a-z_]+$')
    description: str
    enabled: bool = True
    condition: FailureCondition
    context_fields: List[str]
    threshold: ThresholdConfig
    severity: Literal["low", "medium", "high", "critical"]
    priority: int = Field(..., ge=0, le=10)
    repository: str = Field(..., regex=r'^[\w-]+/[\w-]+$')
    labels: List[str]
    assignee: str
    template: Optional[str] = None

    @validator('name')
    def name_must_be_unique(cls, v):
        # Validation logic to ensure uniqueness across all configs
        return v

class TeamConfig(BaseModel):
    team: dict
    failure_definitions: List[FailureDefinition]
    notifications: Optional[dict] = None

    class Config:
        extra = "forbid"  # Reject unknown fields

# Usage
def load_and_validate_config(config_path: str) -> TeamConfig:
    with open(config_path) as f:
        data = yaml.safe_load(f)

    try:
        config = TeamConfig(**data)
        return config
    except ValidationError as e:
        logger.error(f"Invalid configuration: {e}")
        raise
```

---

## 8. Assumptions & Trade-offs

### 8.1 Key Assumptions

1. **Log Volume**: "High volume" assumes 10K-100K events/min
   - **Validation Needed**: Measure actual volume in production
   - **Impact**: May need to scale Event Hub throughput units

2. **Claude Code/Copilot Assignment**: Assumes assignable via GitHub username
   - **Validation Needed**: Test assignment mechanism
   - **Alternative**: Use GitHub teams or project boards

3. **Network Reliability**: Assumes stable Azure → GitHub connectivity
   - **Mitigation**: Retry logic with exponential backoff
   - **Fallback**: Dead-letter queue for failed operations

4. **Issue Creation Scope**: Only creates and assigns issues, no investigation
   - **Clarification**: Claude Code/Copilot perform investigation independently
   - **Out of Scope**: PR creation, code fixes, automated remediation

5. **Multi-team Model**: Central admin manages all configurations
   - **Assumption**: Teams don't need self-service config updates
   - **Alternative**: Add web UI for team-specific config (future phase)

6. **Authentication**: Managed Identity for all Azure services
   - **RBAC**: Requires specific role assignments (documented in Section 4.5)
   - **GitHub**: PAT token stored in Key Vault

7. **Duplicate Detection**: Structural matching sufficient for 90% accuracy
   - **Assumption**: Normalized fingerprints catch most duplicates
   - **Future**: Add semantic matching for edge cases

8. **Failure Window**: 15-minute temporal window for grouping
   - **Assumption**: Failures within 15 min are likely related
   - **Configurable**: Can be adjusted per failure type

### 8.2 Critical Trade-offs

#### Trade-off 1: Event Hub vs Polling

| Aspect | Event Hub (Chosen) | Polling (Previous) |
|--------|-------------------|-------------------|
| **Latency** | < 1 second | 30-60 seconds |
| **Complexity** | Higher (streaming, checkpoints) | Lower (simple queries) |
| **Cost** | ~$50/month (Standard tier) | $0 (included in Log Analytics) |
| **Scalability** | Millions of events/sec | Limited by query performance |
| **Reliability** | Built-in durability, replay | Dependent on checkpoint storage |

**Decision**: Event Hub for high-volume requirement

#### Trade-off 2: Stateful vs Stateless

| Aspect | Stateful (Chosen) | Stateless |
|--------|------------------|-----------|
| **Duplicate Detection** | Possible with cache | Not possible reliably |
| **Complexity** | Higher (state management) | Lower (no state) |
| **Reliability** | Requires state recovery | Simple retry logic |
| **Scalability** | Moderate (cache size) | High (no state to sync) |

**Decision**: Stateful required for duplicate detection

#### Trade-off 3: Configuration Storage

| Option | Git-based (Recommended) | Database (Cosmos DB) |
|--------|------------------------|---------------------|
| **Auditability** | ✅ Full Git history | ⚠️ Requires custom audit log |
| **Validation** | ✅ Pre-commit hooks, CI | ⚠️ Runtime validation only |
| **Rollback** | ✅ Git revert | ⚠️ Manual restore |
| **Hot Reload** | ⚠️ Requires file watching | ✅ Immediate updates |
| **Complexity** | Lower (standard Git workflow) | Higher (database management) |

**Decision**: Git-based for Phase 1, add database UI in Phase 2

#### Trade-off 4: Issue Update Strategy

**Options for Duplicate Failures**:

1. **Update Existing Issue** (Chosen)
   - Add comment with new occurrence details
   - Update occurrence count in issue body
   - Keep issue open until manually closed

2. **Create New Issue with Reference**
   - Create new issue, link to original
   - Allows separate tracking per occurrence
   - Risk: Too many issues

**Decision**: Update existing issue to reduce noise

---

## 9. Additional Recommendations

### 9.1 Programming Language Selection

**Recommendation**: **Python 3.11+**

**Rationale**:
- ✅ Best-in-class Azure SDK support (azure-eventhub, azure-identity, azure-cosmos)
- ✅ Mature async/await for concurrent event processing
- ✅ Rich data processing libraries (pandas, jsonpath-ng)
- ✅ Strong typing support (pydantic, mypy) for configuration validation
- ✅ Easy deployment to Azure Container Apps (Docker)
- ✅ Excellent GitHub API library (PyGithub)

**Alternatives Considered**:
- **Go**: Fast, efficient, good for high-throughput
  - Cons: Less mature Azure SDK, fewer data processing libraries
- **TypeScript/Node.js**: Good async support, decent Azure SDK
  - Cons: Less mature for data processing and log analysis
- **C#**: First-class Azure support
  - Cons: Heavier runtime, less flexible for log parsing

### 9.2 Azure Deployment Service Selection

**Recommendation**: **Azure Container Apps**

**Rationale**:
- ✅ Supports long-running event processors (no time limit)
- ✅ Native Event Hub trigger scaling (scales with partition count)
- ✅ Built-in Managed Identity support
- ✅ Auto-scaling based on event volume
- ✅ Cost-effective (scale to zero when no events)
- ✅ Easy CI/CD integration

**Deployment Configuration**:
```yaml
# Azure Container App configuration
properties:
  configuration:
    activeRevisionsMode: Single
    ingress:
      external: false  # No public endpoint needed

    secrets:
      - name: github-token
        keyVaultUrl: "https://keyvault.vault.azure.net/secrets/github-token"

    registries:
      - server: myregistry.azurecr.io
        identity: system

  template:
    containers:
      - name: error-detection-system
        image: myregistry.azurecr.io/error-detection:latest
        resources:
          cpu: 1.0
          memory: 2Gi
        env:
          - name: AZURE_CLIENT_ID
            value: "managed-identity-client-id"
          - name: GITHUB_TOKEN
            secretRef: github-token

    scale:
      minReplicas: 1
      maxReplicas: 5
      rules:
        - name: event-hub-scaling
          custom:
            type: azure-eventhub
            metadata:
              eventHubName: diagnostics-export
              consumerGroup: error-detection-system
              unprocessedEventThreshold: '100'
```

**Alternatives Considered**:
- **Azure Functions**: Simpler but has 10-minute execution limit
- **AKS (Kubernetes)**: Overkill for single-service application
- **Azure Batch**: Not designed for streaming workloads
- **VM**: Higher operational overhead

### 9.3 Additional Configuration Items for Developers

Based on common needs, recommend adding:

#### 9.3.1 Severity Classification & Priority
```yaml
severity_rules:
  - name: "Critical - Service Down"
    condition:
      error_rate_per_minute: "> 100"
      or:
        - contains: "OutOfMemoryError"
        - contains: "Service Unavailable"
    severity: "critical"
    priority: 0
    sla_response_minutes: 15

  - name: "High - Frequent Errors"
    condition:
      error_count: "> 50"
      time_window_minutes: 10
    severity: "high"
    priority: 1
    sla_response_minutes: 60
```

#### 9.3.2 Business Hours & On-Call Configuration
```yaml
assignment:
  business_hours:
    enabled: true
    timezone: "America/New_York"
    schedule:
      weekdays:
        start: "09:00"
        end: "17:00"
      weekends: false

  after_hours:
    behavior: "create_without_assignment"  # or "queue_for_next_business_day"
    escalation:
      if_critical: "assign_to_oncall"
      oncall_github_team: "backend-oncall"
```

#### 9.3.3 Notification Channels
```yaml
notifications:
  - channel: "slack"
    webhook_url: "${SLACK_WEBHOOK}"
    conditions:
      - severity: ["critical", "high"]
      - failure_type: ["api_5xx_errors", "database_connection_failures"]
    throttle_minutes: 30
    message_template: |
      🚨 *Failure Detected: {{ failure.name }}*
      Severity: {{ failure.severity }}
      Occurrences: {{ failure.count }}
      Issue: {{ github_issue_url }}

  - channel: "email"
    recipients: ["oncall@example.com"]
    conditions:
      - severity: ["critical"]
    throttle_minutes: 15

  - channel: "pagerduty"
    integration_key: "${PAGERDUTY_KEY}"
    conditions:
      - severity: ["critical"]
      - outside_business_hours: true
```

#### 9.3.4 Custom Context Extractors
```yaml
context_extractors:
  - name: "user_id_extractor"
    field: "$.Message"
    pattern: "user_id[=:]\\s*([0-9]+)"
    output_field: "user_id"

  - name: "tenant_extractor"
    field: "$.Properties.tenant"
    output_field: "tenant_id"

  - name: "api_version_extractor"
    field: "$.RequestUri"
    pattern: "/api/v(\\d+)/"
    output_field: "api_version"
```

#### 9.3.5 Rate Limiting & Throttling
```yaml
rate_limits:
  global:
    max_issues_per_hour: 500
    max_issues_per_day: 2000

  per_repository:
    max_issues_per_hour: 50
    max_issues_per_day: 200

  per_failure_type:
    max_issues_per_hour: 20

  github_api:
    max_requests_per_hour: 4000  # Leave buffer for manual use
    max_search_queries_per_minute: 30
```

#### 9.3.6 Auto-Close Configuration
```yaml
issue_lifecycle:
  auto_close:
    enabled: true
    conditions:
      - no_occurrences_for_hours: 48
      - failure_type: ["http_5xx_errors"]
    close_comment: |
      Automatically closing this issue as no new occurrences detected in 48 hours.
      Will reopen if failure recurs.

  auto_reopen:
    enabled: true
    reopen_comment: |
      Failure has recurred. Reopening for investigation.
```

### 9.4 Repository Organization

**Recommendation**: Separate implementation repository from planning

#### Planning Repository (Current)
```
error-detection-planning/
├── requirements/
│   └── Requirements.md
├── plans/
│   ├── architectural-design-proposal.md (original)
│   └── revised-architectural-design.md (this document)
├── research/
│   ├── duplicate-detection-research.md
│   ├── azure-event-hub-evaluation.md
│   └── github-api-investigation.md
└── decisions/
    ├── 001-use-event-hub-not-polling.md
    ├── 002-python-language-choice.md
    └── 003-structural-fingerprinting.md
```

#### Implementation Repository (New)
```
error-detection-system/
├── src/
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── event_hub_consumer.py
│   │   └── stream_processor.py
│   ├── detection/
│   │   ├── __init__.py
│   │   ├── rule_engine.py
│   │   ├── pattern_matcher.py
│   │   └── threshold_evaluator.py
│   ├── deduplication/
│   │   ├── __init__.py
│   │   ├── fingerprint.py
│   │   ├── duplicate_detector.py
│   │   └── strategies.py
│   ├── issues/
│   │   ├── __init__.py
│   │   ├── github_adapter.py
│   │   ├── issue_creator.py
│   │   └── repository_router.py
│   ├── assignment/
│   │   ├── __init__.py
│   │   └── dispatcher.py
│   ├── config/
│   │   ├── __init__.py
│   │   ├── loader.py
│   │   ├── validator.py
│   │   └── models.py
│   ├── state/
│   │   ├── __init__.py
│   │   ├── redis_cache.py
│   │   ├── cosmos_store.py
│   │   └── checkpoint_manager.py
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── logging.py
│   │   └── retry.py
│   └── main.py
├── config/
│   ├── global.yaml
│   ├── teams/
│   ├── repositories/
│   └── schemas/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── deployment/
│   ├── Dockerfile
│   ├── container-app.yaml
│   └── terraform/
├── docs/
│   ├── setup.md
│   ├── configuration.md
│   ├── deployment.md
│   └── operations.md
├── requirements.txt
├── pyproject.toml
└── README.md
```

---

## 10. Implementation Roadmap

### Phase 1: MVP (Weeks 1-4)

**Goals**:
- Core event-driven processing
- Basic failure detection
- GitHub issue creation
- Structural duplicate detection

**Deliverables**:
- ✅ Event Hub consumer with checkpoint management
- ✅ Rule engine with threshold evaluation
- ✅ Structural fingerprinting
- ✅ GitHub issue creation
- ✅ Basic assignment (Claude Code or Copilot)
- ✅ Redis cache for fingerprints
- ✅ Configuration via YAML files
- ✅ Docker container

**Success Criteria**:
- Process 1000 events/minute without data loss
- < 100ms processing latency per event
- 90% duplicate detection accuracy
- Zero failed issue creations (with retry)

### Phase 2: Multi-Team Support (Weeks 5-6)

**Goals**:
- Support multiple teams
- Enhanced configuration
- Cosmos DB for durable storage
- Monitoring and metrics

**Deliverables**:
- ✅ Multi-team configuration structure
- ✅ Cosmos DB integration for audit log
- ✅ Prometheus metrics export
- ✅ Health check endpoints
- ✅ Configuration validation
- ✅ Automated tests (unit + integration)

**Success Criteria**:
- Support 5+ teams with separate configs
- < 1% configuration errors
- 99.9% uptime
- Full audit trail of all operations

### Phase 3: Enhanced Features (Weeks 7-8)

**Goals**:
- Notification system
- Advanced duplicate detection
- Rate limiting
- Issue lifecycle management

**Deliverables**:
- ✅ Slack/email notifications
- ✅ Temporal window grouping
- ✅ GitHub issue search fallback
- ✅ Rate limiting per repository
- ✅ Auto-close inactive issues
- ✅ Business hours configuration

**Success Criteria**:
- < 5 second notification latency
- 95% duplicate detection accuracy
- Zero GitHub API rate limit violations
- Automated issue lifecycle

### Phase 4: Production Readiness (Weeks 9-10)

**Goals**:
- Production deployment
- Monitoring and alerting
- Documentation
- Security hardening

**Deliverables**:
- ✅ Terraform infrastructure code
- ✅ CI/CD pipelines (GitHub Actions)
- ✅ Comprehensive documentation
- ✅ Security review and audit
- ✅ Load testing (10K events/min)
- ✅ Operational runbooks
- ✅ On-call procedures

**Success Criteria**:
- Zero-downtime deployment
- < 0.1% error rate
- All security recommendations implemented
- Complete operational documentation

---

## 11. Success Metrics & Monitoring

### 11.1 Functional Metrics

```yaml
metrics:
  detection:
    - name: "failure_detection_rate"
      description: "% of actual failures detected"
      target: "> 95%"
      measurement: "Compare with manual review"

    - name: "false_positive_rate"
      description: "% of non-failures flagged"
      target: "< 5%"
      measurement: "Developer feedback"

    - name: "duplicate_detection_accuracy"
      description: "% of duplicates correctly identified"
      target: "> 90%"
      measurement: "Manual validation of sample"

  performance:
    - name: "event_processing_latency_p99"
      description: "99th percentile processing time"
      target: "< 500ms"
      measurement: "Prometheus histogram"

    - name: "issue_creation_time"
      description: "Time from failure to issue created"
      target: "< 10 seconds"
      measurement: "End-to-end tracing"

    - name: "assignment_success_rate"
      description: "% of issues successfully assigned"
      target: "100%"
      measurement: "GitHub API responses"

  reliability:
    - name: "system_uptime"
      description: "% availability"
      target: "99.9%"
      measurement: "Health check monitoring"

    - name: "event_loss_rate"
      description: "% of events not processed"
      target: "0%"
      measurement: "Checkpoint vs received count"

    - name: "error_rate"
      description: "% of failed operations"
      target: "< 0.1%"
      measurement: "Exception logging"
```

### 11.2 Prometheus Metrics

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Event processing
events_received_total = Counter(
    'events_received_total',
    'Total events received from Event Hub',
    ['partition_id']
)

events_processed_total = Counter(
    'events_processed_total',
    'Total events successfully processed',
    ['partition_id']
)

event_processing_duration = Histogram(
    'event_processing_duration_seconds',
    'Time spent processing events',
    ['operation']
)

# Failure detection
failures_detected_total = Counter(
    'failures_detected_total',
    'Total failures detected',
    ['failure_type', 'severity']
)

duplicates_detected_total = Counter(
    'duplicates_detected_total',
    'Total duplicate failures detected',
    ['level']  # exact, structural, temporal, github
)

# Issue creation
issues_created_total = Counter(
    'issues_created_total',
    'Total GitHub issues created',
    ['repository', 'failure_type']
)

issues_updated_total = Counter(
    'issues_updated_total',
    'Total GitHub issues updated',
    ['repository']
)

github_api_calls_total = Counter(
    'github_api_calls_total',
    'Total GitHub API calls',
    ['endpoint', 'status_code']
)

github_rate_limit_remaining = Gauge(
    'github_rate_limit_remaining',
    'Remaining GitHub API rate limit'
)

# State management
cache_hits_total = Counter(
    'cache_hits_total',
    'Total cache hits',
    ['cache_type']  # exact, structural
)

cache_misses_total = Counter(
    'cache_misses_total',
    'Total cache misses',
    ['cache_type']
)

# System health
checkpoint_lag_seconds = Gauge(
    'checkpoint_lag_seconds',
    'Lag between current time and last checkpoint',
    ['partition_id']
)
```

### 11.3 Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: error_detection_system
    interval: 30s
    rules:
      - alert: HighEventProcessingLatency
        expr: histogram_quantile(0.99, event_processing_duration_seconds_bucket) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High event processing latency"
          description: "P99 latency is {{ $value }}s (> 1s threshold)"

      - alert: GitHubRateLimitLow
        expr: github_rate_limit_remaining < 500
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "GitHub rate limit running low"
          description: "Only {{ $value }} requests remaining"

      - alert: EventProcessingStalled
        expr: rate(events_processed_total[5m]) == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Event processing has stalled"
          description: "No events processed in 5 minutes"

      - alert: HighErrorRate
        expr: rate(events_processing_errors_total[5m]) > 0.01
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate in event processing"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: CheckpointLagging
        expr: checkpoint_lag_seconds > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Checkpoint lagging behind"
          description: "Checkpoint is {{ $value }}s behind"
```

---

## 12. Security Considerations

### 12.1 Authentication & Authorization

```yaml
security:
  azure:
    authentication:
      method: "managed_identity"
      identity_type: "system_assigned"

    rbac_assignments:
      - resource: "Event Hub Namespace"
        role: "Azure Event Hubs Data Receiver"
        scope: "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{ns}/eventhubs/{hub}"

      - resource: "Storage Account"
        role: "Storage Blob Data Contributor"
        scope: "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{sa}/blobServices/default/containers/checkpoints"

      - resource: "Key Vault"
        role: "Key Vault Secrets User"
        scope: "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{kv}"

      - resource: "Cosmos DB"
        role: "Cosmos DB Built-in Data Contributor"
        scope: "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DocumentDB/databaseAccounts/{db}"

  github:
    authentication:
      method: "personal_access_token"
      storage: "azure_key_vault"
      rotation_days: 90

    permissions_required:
      - "repo" (for private repos)
      - "public_repo" (for public repos)
      - "write:discussion" (for issue creation)

    rate_limiting:
      max_per_hour: 5000
      buffer: 1000  # Reserve for safety

  secrets_management:
    provider: "azure_key_vault"
    secrets:
      - name: "github-token"
        rotation: "manual"
        last_rotated: "2025-12-01"
      - name: "slack-webhook"
        rotation: "manual"
```

### 12.2 Data Protection

```yaml
data_protection:
  log_sanitization:
    enabled: true
    rules:
      - type: "pii"
        patterns:
          - regex: '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
            replacement: '<EMAIL>'
          - regex: '\b\d{3}-\d{2}-\d{4}\b'
            replacement: '<SSN>'
          - regex: '\b\d{16}\b'
            replacement: '<CARD>'

      - type: "credentials"
        patterns:
          - regex: '(?i)(password|pwd|secret|token)["\s:=]+[^\s"]+'
            replacement: '<REDACTED>'
          - regex: 'Bearer [A-Za-z0-9\-._~+/]+'
            replacement: 'Bearer <TOKEN>'

      - type: "internal_ips"
        patterns:
          - regex: '\b10\.\d{1,3}\.\d{1,3}\.\d{1,3}\b'
            replacement: '<INTERNAL_IP>'
          - regex: '\b172\.(1[6-9]|2[0-9]|3[0-1])\.\d{1,3}\.\d{1,3}\b'
            replacement: '<INTERNAL_IP>'

  encryption:
    at_rest:
      - service: "Redis Cache"
        enabled: true
        method: "Azure managed keys"
      - service: "Cosmos DB"
        enabled: true
        method: "Azure managed keys"
      - service: "Storage Account"
        enabled: true
        method: "Azure managed keys"

    in_transit:
      - protocol: "TLS 1.2+"
        enforcement: "mandatory"
```

### 12.3 Network Security

```yaml
network_security:
  container_app:
    ingress:
      external: false  # No public endpoint
      allow_insecure: false

    vnet_integration:
      enabled: true  # Future enhancement
      subnet: "container-apps-subnet"

  private_endpoints:
    - service: "Redis Cache"
      enabled: true  # Future enhancement
    - service: "Cosmos DB"
      enabled: true  # Future enhancement
    - service: "Storage Account"
      enabled: true  # Future enhancement

  firewall_rules:
    - service: "Key Vault"
      allow_azure_services: true
      allowed_ips: []  # Use Managed Identity only
```

---

## 13. Open Questions & Next Steps

### 13.1 Questions for Stakeholders

1. **Volume Validation**: What is the actual expected log volume?
   - Current average events/minute?
   - Peak events/minute?
   - Expected growth rate?

2. **Claude Code/Copilot Assignment**:
   - What are the GitHub usernames for Claude Code and Copilot bots?
   - Are they already set up with repository access?
   - What permissions do they need?

3. **Repository Structure**:
   - How many repositories need monitoring?
   - How many teams will use this system?
   - What is the team-to-repository mapping?

4. **Issue Lifecycle**:
   - Should the system auto-close resolved issues?
   - Should it reopen issues if failures recur?
   - What is the expected issue resolution SLA?

5. **Budget & Resources**:
   - What is the monthly Azure budget?
   - Are there existing Event Hub/Redis/Cosmos DB resources?
   - Who will manage infrastructure (Terraform)?

6. **Multi-team Configuration**:
   - Who are the team admins/contacts?
   - What are team-specific SLAs?
   - Are there compliance requirements per team?

7. **Deployment Timeline**:
   - When is the target production date?
   - Is there a pilot phase with one team first?
   - What is the rollout strategy?

### 13.2 Validation Tasks

- [ ] **Test Event Hub throughput**: Validate that Standard tier handles expected volume
- [ ] **Test GitHub assignment**: Confirm Claude Code/Copilot can be assigned via API
- [ ] **Measure Log Analytics export lag**: Determine latency from log generation to Event Hub
- [ ] **Test duplicate detection accuracy**: Use sample logs to validate fingerprinting
- [ ] **Validate RBAC permissions**: Ensure Managed Identity has correct roles
- [ ] **Estimate costs**: Calculate actual Azure costs based on volume

### 13.3 Recommended Next Steps

1. **Week 1-2: Foundation**
   - Create implementation repository
   - Set up Azure resources (Event Hub, Redis, Cosmos DB, Key Vault)
   - Configure Managed Identity and RBAC
   - Implement Event Hub consumer with checkpointing
   - Set up CI/CD pipeline

2. **Week 3-4: Core Features**
   - Implement rule engine and threshold evaluation
   - Build structural fingerprinting
   - Integrate GitHub API (issue creation, assignment)
   - Add Redis caching
   - Create basic configuration loader

3. **Week 5-6: Multi-Team Support**
   - Implement multi-team configuration structure
   - Add Cosmos DB integration for audit log
   - Build configuration validation
   - Add Prometheus metrics
   - Write unit and integration tests

4. **Week 7-8: Enhanced Features**
   - Add temporal window grouping
   - Implement GitHub issue search fallback
   - Build notification system (Slack/email)
   - Add rate limiting
   - Implement issue lifecycle management

5. **Week 9-10: Production Readiness**
   - Complete Terraform infrastructure code
   - Set up monitoring and alerting
   - Conduct security review
   - Load testing (10K events/min)
   - Write operational documentation
   - Deploy to production

---

## 14. Conclusion

This revised architecture addresses the clarified requirements with a focus on:

1. **High-Volume Processing**: Event-driven streaming architecture with Azure Event Hub
2. **Multi-Team Support**: Centralized configuration with team-specific failure definitions
3. **Reliability**: Stateful processing with duplicate detection and retry logic
4. **Simplicity**: Out-of-scope: investigation and PR creation (only issue creation + assignment)
5. **Extensibility**: Pluggable components for future log sources and issue platforms

### Key Design Decisions:

| Decision | Rationale |
|----------|-----------|
| **Event Hub over Polling** | High-volume requirement demands streaming architecture |
| **Python** | Best Azure/GitHub SDK support, mature data processing |
| **Azure Container Apps** | Supports long-running processes, auto-scaling, cost-effective |
| **Structural Fingerprinting** | Balances accuracy (90%) with performance (< 5ms) |
| **Git-based Configuration** | Auditability, rollback, standard GitOps workflow |
| **Managed Identity** | Secure, no credential management in code |

### Success Factors:

- ✅ Start with MVP (Event Hub + basic detection + GitHub)
- ✅ Validate duplicate detection early with real logs
- ✅ Pilot with one team before rolling out to all teams
- ✅ Monitor GitHub rate limits and adjust throttling
- ✅ Iterate on configuration based on developer feedback
- ✅ Plan for extensibility (multi-cloud, GitLab, Bitbucket)

### Next Actions:

1. **Validate assumptions**: Confirm log volume, GitHub assignment mechanism
2. **Set up Azure infrastructure**: Event Hub, Redis, Cosmos DB, Container Apps
3. **Create implementation repository**: Follow recommended structure (Section 9.4)
4. **Begin Phase 1 development**: Event Hub consumer + basic detection + GitHub integration
5. **Pilot with one team**: Backend team as initial user, gather feedback

---

## Appendix A: Glossary

- **Event Hub**: Azure's high-throughput event streaming service
- **Checkpoint**: Last successfully processed event offset in a partition
- **Fingerprint**: Unique identifier for a failure used in duplicate detection
- **Structural Matching**: Duplicate detection using normalized error signatures
- **Temporal Window**: Time-based grouping of related failures
- **Managed Identity**: Azure AD identity for secure service-to-service authentication
- **Container Apps**: Azure's serverless container platform with auto-scaling
- **Throughput Units**: Event Hub capacity units (1 TU = 1 MB/s ingress)
- **Partition**: Event Hub data shard for parallel processing

## Appendix B: References

- **Azure Event Hubs**: https://learn.microsoft.com/en-us/azure/event-hubs/
- **Azure Container Apps**: https://learn.microsoft.com/en-us/azure/container-apps/
- **Azure Monitor Diagnostics**: https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings
- **Azure Managed Identity**: https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/
- **GitHub REST API**: https://docs.github.com/en/rest
- **Python Azure SDK**: https://learn.microsoft.com/en-us/python/api/overview/azure/
- **Pydantic**: https://docs.pydantic.dev/

---

**Document Version**: 2.0
**Last Updated**: 2025-12-08
**Author**: Architecture Team
**Status**: Revised Design for Review
