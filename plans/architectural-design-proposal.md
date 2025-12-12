# Architectural Design Proposal: Error Detection & GitHub Issue Creation System

## Executive Summary

This document proposes an architecture for an autonomous log monitoring and issue creation system that monitors Azure Log Diagnostics for failures, creates GitHub issues with detailed context, and assigns them to AI assistants (Claude or Copilot) for investigation.

---

## 1. Requirements Analysis

### 1.1 Core Requirements

**Functional Requirements:**
- Monitor Azure Log Diagnostics for failures (JSON format initially)
- Parse and analyze log entries to detect failures
- Create GitHub issues with failure context and details
- Implement duplicate detection to prevent redundant issues
- Assign created issues to Claude or Copilot
- Support extensibility for future log sources (multi-cloud)
- Support extensibility for future issue platforms (GitLab, Bitbucket)

**Configuration Requirements:**
- Configurable failure definition and evaluation criteria
- Configurable thresholds for issue generation (failure count, frequency)
- Configurable log analytics tables to monitor
- Configurable failure-to-repository mapping for proper issue routing

### 1.2 Key Constraints & Design Goals

- **Independence**: System must operate autonomously without manual intervention
- **Extensibility**: Architecture must support multiple log sources and issue platforms
- **Scalability**: Must handle high-volume log processing efficiently
- **Reliability**: Must not lose failure events or create duplicate issues
- **Configurability**: Support different failure patterns and thresholds per project

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Error Detection System                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   Log        │      │   Failure    │      │    Issue     │  │
│  │   Ingestion  │─────>│   Detection  │─────>│   Creation   │  │
│  │   Layer      │      │   Engine     │      │   Service    │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│         │                      │                      │          │
│         ▼                      ▼                      ▼          │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   Config     │      │  Duplicate   │      │  Assignment  │  │
│  │   Manager    │◄────│  Detection   │─────>│  Dispatcher  │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│         │                      │                      │          │
│         └──────────────────────┴──────────────────────┘          │
│                                │                                 │
│                         ┌──────▼──────┐                         │
│                         │   State     │                         │
│                         │   Store     │                         │
│                         └─────────────┘                         │
└─────────────────────────────────────────────────────────────────┘

External Systems:
┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐
│ Azure Log        │───>│  This System     │───>│   GitHub     │
│ Analytics        │    │                  │    │   API        │
└──────────────────┘    └──────────────────┘    └──────────────┘
```

### 2.2 Component Architecture

#### 2.2.1 Log Ingestion Layer

**Purpose**: Fetch and normalize logs from various sources

**Components**:
- **Log Source Adapters**: Pluggable adapters for different log sources
  - `AzureLogDiagnosticsAdapter` (initial implementation)
  - Future: `AWSCloudWatchAdapter`, `GCPLoggingAdapter`
- **Log Normalizer**: Converts different log formats to internal representation
- **Polling Scheduler**: Manages periodic log fetching with configurable intervals

**Design Pattern**: Adapter Pattern for log source abstraction

**Key Responsibilities**:
- Authenticate with log sources
- Fetch logs incrementally (track last processed timestamp)
- Transform logs to standard internal format
- Handle pagination and rate limiting
- Retry failed requests with exponential backoff

#### 2.2.2 Failure Detection Engine

**Purpose**: Analyze logs and identify failures based on configuration

**Components**:
- **Rule Engine**: Evaluates log entries against failure definitions
- **Pattern Matcher**: Supports regex, JSON path queries, and custom logic
- **Threshold Evaluator**: Tracks failure counts and frequencies
- **Context Extractor**: Extracts relevant context (stack traces, request IDs, etc.)

**Design Pattern**: Strategy Pattern for configurable failure detection rules

**Key Responsibilities**:
- Parse normalized log entries
- Apply failure detection rules (configurable per project)
- Extract failure metadata and context
- Apply threshold logic (e.g., 5 failures in 10 minutes)
- Enrich failures with additional context

#### 2.2.3 Duplicate Detection Service

**Purpose**: Prevent creating duplicate issues for the same failure

**Components**:
- **Fingerprint Generator**: Creates unique signatures for failures
- **Similarity Matcher**: Detects semantically similar failures
- **Deduplication Cache**: Stores recent failure fingerprints with TTL

**Design Pattern**: Template Method Pattern for fingerprint generation

**Key Responsibilities**:
- Generate failure fingerprints using multiple strategies:
  1. **Exact Match**: Hash of error message + stack trace
  2. **Structural Match**: Normalized stack trace (ignore variable values)
  3. **Semantic Match**: Error message similarity using embeddings (optional)
  4. **Temporal Window**: Group failures within time window
- Check if similar issue already exists in GitHub
- Maintain sliding window of processed failures
- Update existing issues if threshold changes

#### 2.2.4 Issue Creation Service

**Purpose**: Create well-formatted issues in appropriate repositories

**Components**:
- **Issue Platform Adapters**: Pluggable adapters for issue platforms
  - `GitHubIssueAdapter` (initial implementation)
  - Future: `GitLabAdapter`, `BitbucketAdapter`
- **Template Engine**: Renders issue content from templates
- **Repository Router**: Maps failures to correct repositories

**Design Pattern**: Adapter Pattern + Template Method

**Key Responsibilities**:
- Format issue title and body with failure details
- Include stack traces, logs, and context
- Add appropriate labels (auto-generated, failure-detection, severity)
- Link to log analytics queries
- Create issues in correct repository based on routing rules

#### 2.2.5 Assignment Dispatcher

**Purpose**: Assign created issues to AI assistants

**Components**:
- **Assignment Strategy**: Logic for choosing Claude vs Copilot
- **AI Platform Integrations**: APIs for Claude/Copilot assignment

**Design Pattern**: Strategy Pattern for assignment logic

**Key Responsibilities**:
- Determine which AI to assign (configurable per repo or failure type)
- Use GitHub API to assign issues
- Add assignment comments with investigation instructions
- Track assignment success/failure

#### 2.2.6 Configuration Manager

**Purpose**: Manage all system configuration

**Components**:
- **Config Loader**: Loads configuration from files/environment
- **Config Validator**: Validates configuration schema
- **Hot Reload**: Supports updating config without restart (optional)

**Configuration Schema** (see Section 5):
- Log sources and credentials
- Failure definitions and thresholds
- Repository routing rules
- Duplicate detection settings
- Assignment preferences

#### 2.2.7 State Store

**Purpose**: Persist system state for reliability and duplicate detection

**Components**:
- **Checkpoint Store**: Last processed log timestamps per source
- **Failure Cache**: Recent failure fingerprints with TTL
- **Issue Tracking**: Mapping of failures to created issues

**Storage Options** (see Section 3.1):
- Redis for in-memory caching
- Azure Table Storage or Cosmos DB for durable state

---

## 3. Technology Stack Recommendations

### 3.1 Programming Language: **Python**

**Rationale**:
- ✅ Excellent Azure SDK support (azure-monitor-query, azure-identity)
- ✅ Strong GitHub API libraries (PyGithub, ghapi)
- ✅ Rich ecosystem for log processing (pandas, jsonpath-ng)
- ✅ Great async support (asyncio) for concurrent log processing
- ✅ Easy configuration management (pydantic, python-dotenv)
- ✅ Simple deployment to Azure Functions or Container Apps
- ✅ Natural language processing libraries for semantic matching (optional)

**Alternative Considered**: TypeScript/Node.js
- Pros: Good Azure/GitHub SDK support, async-first
- Cons: Less mature for data processing and log analysis

### 3.2 Deployment Service: **Azure Container Apps**

**Rationale**:
- ✅ Supports long-running background processes (vs Functions' time limits)
- ✅ Built-in scaling (scale to zero when idle)
- ✅ Easy integration with Azure Log Analytics via managed identity
- ✅ Cost-effective for continuous monitoring workloads
- ✅ Supports environment variable configuration
- ✅ Built-in health checks and auto-restart

**Alternative Considered**: Azure Functions (Timer-triggered)
- Pros: Simpler deployment, serverless
- Cons: 10-minute execution limit, less suitable for continuous processing

**Alternative Considered**: Azure Kubernetes Service (AKS)
- Pros: Maximum flexibility, production-grade orchestration
- Cons: Overkill for single-service application, higher operational overhead

### 3.3 Core Dependencies

```python
# Azure Integration
azure-monitor-query>=1.2.0      # Query Log Analytics
azure-identity>=1.15.0          # Managed Identity auth
azure-keyvault-secrets>=4.7.0   # Secret management

# GitHub Integration
PyGithub>=2.1.1                 # GitHub API client

# Data Processing
pydantic>=2.5.0                 # Configuration validation
jsonpath-ng>=1.6.0              # JSON log parsing
python-dotenv>=1.0.0            # Environment config

# Storage
redis>=5.0.0                    # Caching layer
azure-data-tables>=12.4.0       # Durable state (alternative: Cosmos DB)

# Utilities
tenacity>=8.2.3                 # Retry logic
structlog>=23.2.0               # Structured logging
schedule>=1.2.0                 # Job scheduling
```

### 3.4 Data Storage

**State Store**: Redis (Azure Cache for Redis)
- Failure fingerprint cache (TTL-based)
- Rate limiting counters
- Processing checkpoints (backup to durable storage)

**Durable Store**: Azure Table Storage
- Configuration persistence
- Checkpoint backups (last processed timestamps)
- Issue creation audit log
- Failure history (for analysis)

---

## 4. Data Flow & Sequence Diagrams

### 4.1 End-to-End Flow

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Timer   │────>│   Log    │────>│ Failure  │────>│Duplicate │
│ Trigger  │     │Ingestion │     │Detection │     │Detection │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                                          │
                                                          ▼
                                                    Is Duplicate?
                                                     ╱         ╲
                                                   Yes         No
                                                    │           │
                                                    ▼           ▼
                                              Skip/Update  ┌──────────┐
                                                           │  Issue   │
                                                           │ Creation │
                                                           └──────────┘
                                                                │
                                                                ▼
                                                           ┌──────────┐
                                                           │Assignment│
                                                           │Dispatch  │
                                                           └──────────┘
                                                                │
                                                                ▼
                                                           [GitHub
                                                            Issue
                                                            Created]
```

### 4.2 Detailed Processing Sequence

```
1. Log Fetching:
   System → Azure Log Analytics: Query logs since last checkpoint
   Azure → System: Return log entries (JSON)
   System → State Store: Update checkpoint timestamp

2. Failure Detection:
   For each log entry:
     System → Rule Engine: Evaluate log against failure rules
     Rule Engine → System: Return failure match + context
     System → Threshold Evaluator: Check if threshold met
     Threshold Evaluator → System: Return "create issue" decision

3. Duplicate Detection:
   System → Fingerprint Generator: Create failure signature
   System → State Store: Check if fingerprint exists
   State Store → System: Return existing issues (if any)
   System → GitHub API: Search for similar issues
   GitHub → System: Return search results
   System → Decision: Skip or create new issue

4. Issue Creation:
   System → Template Engine: Render issue content
   System → Repository Router: Determine target repository
   System → GitHub API: Create issue with labels
   GitHub → System: Return created issue URL
   System → State Store: Record failure → issue mapping

5. Assignment:
   System → Assignment Strategy: Determine AI assignment
   System → GitHub API: Assign issue to Claude/Copilot
   System → GitHub API: Add comment with instructions
```

---

## 5. Configuration Schema

### 5.1 Configuration File Structure (YAML)

```yaml
# config.yaml
log_sources:
  - name: "azure-production-logs"
    type: "azure_log_analytics"
    workspace_id: "${AZURE_WORKSPACE_ID}"
    credentials:
      type: "managed_identity"  # or "service_principal"
    tables:
      - "AppServiceConsoleLogs"
      - "AppExceptions"
      - "AppDependencies"
    poll_interval_seconds: 60
    lookback_window_minutes: 5

failure_definitions:
  - name: "http_5xx_errors"
    description: "HTTP 500-series errors"
    condition:
      field: "ResultCode"
      operator: "regex"
      pattern: "^5\\d{2}$"
    context_fields:
      - "RequestUri"
      - "Message"
      - "StackTrace"
    threshold:
      count: 5
      window_minutes: 10
      reset_window: true

  - name: "unhandled_exceptions"
    description: "Application exceptions"
    condition:
      field: "SeverityLevel"
      operator: "equals"
      value: "Error"
    context_fields:
      - "ExceptionType"
      - "ExceptionMessage"
      - "StackTrace"
      - "OperationName"
    threshold:
      count: 1  # Create issue on first occurrence
      window_minutes: 1

repository_routing:
  - failure_name: "http_5xx_errors"
    repository: "my-org/backend-api"
    labels:
      - "bug"
      - "auto-detected"
      - "severity:high"

  - failure_name: "unhandled_exceptions"
    repository: "my-org/backend-api"  # Default
    labels:
      - "bug"
      - "auto-detected"
      - "needs-investigation"

duplicate_detection:
  strategy: "structural"  # Options: exact, structural, semantic
  time_window_hours: 24
  similarity_threshold: 0.85  # For semantic matching
  cache_ttl_hours: 48

assignment:
  default_assignee: "claude"  # Options: claude, copilot, none
  assignment_rotation: false  # Round-robin between AIs
  assignment_comment_template: |
    🤖 Auto-assigned for investigation.

    Please analyze this failure and:
    1. Identify the root cause
    2. Assess severity and impact
    3. Draft a fix if straightforward

    Failure context:
    - Detection time: {{ timestamp }}
    - Occurrence count: {{ count }}
    - Log query: {{ log_query_url }}

github:
  token: "${GITHUB_TOKEN}"
  issue_template: |
    ## Failure Detected

    **Failure Type**: {{ failure_name }}
    **Detection Time**: {{ timestamp }}
    **Occurrences**: {{ count }} in {{ window }} minutes

    ### Error Details
    ```
    {{ error_message }}
    ```

    ### Stack Trace
    ```
    {{ stack_trace }}
    ```

    ### Context
    {{ context }}

    ### Log Analytics Query
    [View in Azure]( {{ log_query_url }} )

    ---
    *Auto-generated by Error Detection System*
```

### 5.2 Environment Variables

```bash
# Required
AZURE_WORKSPACE_ID=xxx
GITHUB_TOKEN=ghp_xxx

# Optional (for service principal auth)
AZURE_TENANT_ID=xxx
AZURE_CLIENT_ID=xxx
AZURE_CLIENT_SECRET=xxx

# Redis connection
REDIS_HOST=xxx.redis.cache.windows.net
REDIS_PASSWORD=xxx

# Deployment
ENVIRONMENT=production
LOG_LEVEL=INFO
```

---

## 6. Duplicate Detection Strategies

### 6.1 Strategy 1: Exact Match (Default)

**Approach**: Hash error message + stack trace + failure type

**Pros**:
- Fast (O(1) lookup)
- No false positives
- Simple implementation

**Cons**:
- Misses similar errors with different variable values
- May create duplicates for same root cause

**Implementation**:
```python
def fingerprint(failure):
    data = f"{failure.type}:{failure.message}:{failure.stack_trace}"
    return hashlib.sha256(data.encode()).hexdigest()
```

### 6.2 Strategy 2: Structural Match (Recommended)

**Approach**: Normalize stack traces and error messages

**Normalization Rules**:
- Replace variable values with placeholders
- Remove request IDs, timestamps, IP addresses
- Normalize file paths
- Extract exception type and first N frames of stack trace

**Pros**:
- Catches same error with different runtime values
- More accurate than exact match
- Still fast (O(1) after normalization)

**Cons**:
- More complex implementation
- May have false positives if normalization too aggressive

**Implementation**:
```python
def normalize_stack_trace(stack):
    # Remove numbers, IDs, timestamps
    normalized = re.sub(r'\b\d+\b', '<NUM>', stack)
    normalized = re.sub(r'0x[0-9a-fA-F]+', '<ADDR>', normalized)
    normalized = re.sub(r'\b[0-9a-f-]{36}\b', '<UUID>', normalized)
    return normalized

def fingerprint(failure):
    normalized_msg = normalize_stack_trace(failure.message)
    normalized_trace = normalize_stack_trace(failure.stack_trace[:500])
    data = f"{failure.type}:{normalized_msg}:{normalized_trace}"
    return hashlib.sha256(data.encode()).hexdigest()
```

### 6.3 Strategy 3: Semantic Match (Advanced)

**Approach**: Use embeddings to find semantically similar errors

**Components**:
- Embed error messages using sentence transformers
- Compute cosine similarity
- Threshold-based matching (e.g., > 0.85 similarity)

**Pros**:
- Catches rephrased or similar errors
- Best duplicate detection accuracy

**Cons**:
- Slower (requires model inference)
- Requires ML model deployment
- Higher resource usage

**Implementation**:
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

def fingerprint(failure):
    # Still use structural for primary key
    return structural_fingerprint(failure)

def find_similar(failure, recent_failures, threshold=0.85):
    embedding = model.encode(failure.message)
    for recent in recent_failures:
        recent_embedding = model.encode(recent.message)
        similarity = cosine_similarity(embedding, recent_embedding)
        if similarity > threshold:
            return recent
    return None
```

### 6.4 Recommendation

**Phase 1**: Implement **Structural Match** (Strategy 2)
- Balances accuracy and performance
- Handles 90%+ of duplicate scenarios
- Simple to deploy and maintain

**Phase 2 (Optional)**: Add **Semantic Match** (Strategy 3)
- Use as secondary check for edge cases
- Configurable per failure type
- Improves accuracy for complex errors

---

## 7. Repository Organization

### 7.1 Recommended Structure

```
error-detection-system/
├── src/
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── base.py                    # Abstract log source
│   │   ├── azure_adapter.py           # Azure Log Analytics
│   │   └── normalizer.py              # Log normalization
│   ├── detection/
│   │   ├── __init__.py
│   │   ├── rule_engine.py             # Failure detection
│   │   ├── pattern_matcher.py         # Pattern matching
│   │   ├── threshold.py               # Threshold evaluation
│   │   └── context.py                 # Context extraction
│   ├── deduplication/
│   │   ├── __init__.py
│   │   ├── fingerprint.py             # Fingerprint generation
│   │   ├── strategies.py              # Match strategies
│   │   └── cache.py                   # Dedup cache
│   ├── issues/
│   │   ├── __init__.py
│   │   ├── base.py                    # Abstract issue platform
│   │   ├── github_adapter.py          # GitHub implementation
│   │   ├── templates.py               # Issue templates
│   │   └── router.py                  # Repository routing
│   ├── assignment/
│   │   ├── __init__.py
│   │   ├── dispatcher.py              # Assignment logic
│   │   └── strategies.py              # Assignment strategies
│   ├── config/
│   │   ├── __init__.py
│   │   ├── loader.py                  # Config loading
│   │   ├── validator.py               # Schema validation
│   │   └── models.py                  # Pydantic models
│   ├── state/
│   │   ├── __init__.py
│   │   ├── checkpoints.py             # Checkpoint store
│   │   ├── cache.py                   # Redis cache
│   │   └── audit.py                   # Audit logging
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── logging.py                 # Structured logging
│   │   └── retry.py                   # Retry helpers
│   └── main.py                        # Application entry point
├── tests/
│   ├── unit/
│   │   ├── test_ingestion.py
│   │   ├── test_detection.py
│   │   ├── test_deduplication.py
│   │   ├── test_issues.py
│   │   └── test_assignment.py
│   ├── integration/
│   │   ├── test_azure_integration.py
│   │   ├── test_github_integration.py
│   │   └── test_end_to_end.py
│   └── fixtures/
│       ├── sample_logs.json
│       └── test_config.yaml
├── config/
│   ├── config.yaml                    # Default config
│   ├── config.production.yaml         # Production overrides
│   └── config.schema.json             # JSON schema
├── deployment/
│   ├── Dockerfile
│   ├── docker-compose.yaml            # Local development
│   ├── azure-container-app.yaml       # Azure deployment
│   └── terraform/                     # Infrastructure as code
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── docs/
│   ├── architecture.md                # This document
│   ├── configuration.md               # Config guide
│   ├── deployment.md                  # Deployment guide
│   └── development.md                 # Development guide
├── scripts/
│   ├── setup.sh                       # Local setup
│   ├── deploy.sh                      # Deployment script
│   └── test_connection.py             # Connection testing
├── .github/
│   └── workflows/
│       ├── ci.yaml                    # CI pipeline
│       └── deploy.yaml                # CD pipeline
├── requirements.txt                   # Python dependencies
├── requirements-dev.txt               # Dev dependencies
├── pyproject.toml                     # Project metadata
├── pytest.ini                         # Test configuration
├── .env.example                       # Environment template
├── .gitignore
└── README.md
```

### 7.2 Planning Repository (Current)

Keep this repository (`error-detection-planning/`) for:
- Architecture documentation
- Requirements gathering
- Design decisions and ADRs (Architecture Decision Records)
- Prototypes and experiments

Create separate implementation repository:
- `error-detection-system/` (or similar name)

---

## 8. Assumptions & Trade-offs

### 8.1 Assumptions

1. **Log Volume**: Assumes moderate log volume (< 10K events/minute)
   - *Impact*: Higher volumes may require streaming architecture (Azure Event Hub)

2. **GitHub Rate Limits**: Assumes rate limits won't be exceeded
   - *Mitigation*: Implement rate limiting and request throttling

3. **Network Reliability**: Assumes stable connectivity to Azure and GitHub
   - *Mitigation*: Implement retry logic and circuit breakers

4. **Configuration Updates**: Assumes configuration changes are infrequent
   - *Impact*: Frequent changes may require hot-reload capability

5. **Issue Assignment**: Assumes Claude/Copilot can be assigned via GitHub API
   - *Verification Needed*: Confirm assignment mechanism with AI platforms

6. **Authentication**: Assumes Azure Managed Identity for production
   - *Alternative*: Service Principal with Key Vault for secrets

7. **Single Region**: Initial deployment in single Azure region
   - *Future*: Multi-region for high availability

8. **Failure Window**: Assumes failures within 5-minute window are related
   - *Configurable*: Can be adjusted per failure type

### 8.2 Trade-offs

#### 8.2.1 Polling vs Streaming

**Decision**: Use polling (query logs periodically)

**Pros**:
- Simpler implementation
- Easier debugging
- Lower operational complexity
- Sufficient for moderate volumes

**Cons**:
- Higher latency (up to poll interval)
- Less real-time
- May miss events if polling fails

**Alternative**: Streaming (Azure Event Hub + Stream Analytics)
- Pros: Real-time, handles high volume
- Cons: More complex, higher cost

**Recommendation**: Start with polling, migrate to streaming if needed

#### 8.2.2 Stateful vs Stateless

**Decision**: Stateful (with durable checkpoint storage)

**Pros**:
- Prevents reprocessing logs
- Enables duplicate detection
- Tracks issue creation history

**Cons**:
- Requires state management
- More complex failure recovery

**Alternative**: Stateless with idempotent operations
- Pros: Simpler, easier to scale
- Cons: Risk of duplicate issues, reprocessing logs

**Recommendation**: Stateful is necessary for duplicate detection

#### 8.2.3 Duplicate Detection Accuracy vs Performance

**Decision**: Structural matching (normalized fingerprints)

**Pros**:
- Good accuracy (90%+)
- Fast (O(1) lookup)
- Simple to implement

**Cons**:
- May miss semantically similar errors
- Requires careful normalization rules

**Alternative**: Semantic matching with embeddings
- Pros: Highest accuracy
- Cons: Slower, requires ML model

**Recommendation**: Structural for Phase 1, add semantic as opt-in

#### 8.2.4 Deployment: Container Apps vs Functions

**Decision**: Azure Container Apps

**Pros**:
- No execution time limit
- Better for long-running processes
- Easier scaling configuration
- More control over runtime

**Cons**:
- Slightly more operational overhead
- May not scale to zero as quickly

**Alternative**: Azure Functions (Timer-triggered)
- Pros: True serverless, simpler deployment
- Cons: 10-minute timeout, less suitable for continuous processing

**Recommendation**: Container Apps for flexibility and reliability

---

## 9. Additional Configuration Recommendations

Based on common developer needs, consider adding:

### 9.1 Severity Classification

```yaml
severity_rules:
  - condition: "ResultCode >= 500"
    severity: "high"
    priority: 1

  - condition: "ExceptionType == 'OutOfMemoryError'"
    severity: "critical"
    priority: 0

  - condition: "ResultCode >= 400 AND ResultCode < 500"
    severity: "medium"
    priority: 2
```

### 9.2 Notification Preferences

```yaml
notifications:
  - channel: "slack"
    webhook_url: "${SLACK_WEBHOOK}"
    on_severity: ["critical", "high"]
    throttle_minutes: 30

  - channel: "email"
    recipients: ["oncall@example.com"]
    on_severity: ["critical"]
```

### 9.3 Business Hours Configuration

```yaml
issue_creation:
  business_hours_only: false
  business_hours:
    timezone: "America/New_York"
    days: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
    start: "09:00"
    end: "17:00"
  after_hours_behavior: "create_without_assignment"
```

### 9.4 Rate Limiting

```yaml
rate_limits:
  max_issues_per_hour: 50
  max_issues_per_repository_per_hour: 10
  max_duplicate_updates_per_hour: 100
```

### 9.5 Custom Context Extractors

```yaml
context_extractors:
  - name: "user_id_extractor"
    pattern: "user_id=([0-9]+)"
    field: "user_id"

  - name: "request_id_extractor"
    pattern: "request_id=([a-f0-9-]+)"
    field: "request_id"
```

### 9.6 Integration Webhooks

```yaml
webhooks:
  on_issue_created:
    url: "https://example.com/webhook"
    headers:
      Authorization: "Bearer ${WEBHOOK_TOKEN}"
    payload_template: |
      {
        "event": "issue_created",
        "issue_url": "{{ issue_url }}",
        "failure_type": "{{ failure_type }}",
        "severity": "{{ severity }}"
      }
```

---

## 10. Implementation Phases

### Phase 1: MVP (Weeks 1-3)

**Goals**:
- Core failure detection for Azure Log Analytics
- GitHub issue creation
- Basic duplicate detection (structural matching)
- Single repository support

**Deliverables**:
- Log ingestion for Azure (JSON format)
- Rule-based failure detection
- Structural duplicate detection
- GitHub issue creation and assignment
- Configuration via YAML
- Docker container deployment

### Phase 2: Enhanced Features (Weeks 4-6)

**Goals**:
- Multi-repository support
- Enhanced duplicate detection
- Notification system
- Better monitoring

**Deliverables**:
- Repository routing rules
- Configurable thresholds
- Slack/email notifications
- Prometheus metrics
- Health check endpoints
- Automated tests

### Phase 3: Extensibility (Weeks 7-8)

**Goals**:
- Support additional log sources
- Support additional issue platforms
- Advanced configuration

**Deliverables**:
- Plugin architecture for log sources
- Plugin architecture for issue platforms
- Semantic duplicate detection (optional)
- Rate limiting
- Business hours support

### Phase 4: Production Hardening (Weeks 9-10)

**Goals**:
- High availability
- Monitoring and alerting
- Documentation

**Deliverables**:
- Terraform infrastructure code
- CI/CD pipelines
- Comprehensive documentation
- Load testing
- Security review
- Operational runbooks

---

## 11. Success Metrics

### 11.1 Functional Metrics

- **Detection Accuracy**: % of actual failures detected
- **False Positive Rate**: % of non-failures flagged
- **Duplicate Rate**: % of issues correctly identified as duplicates
- **Issue Creation Time**: Time from failure to issue creation
- **Assignment Success Rate**: % of issues successfully assigned

### 11.2 Operational Metrics

- **System Uptime**: % availability
- **Processing Latency**: Time to process log batch
- **Error Rate**: % of failed operations
- **Resource Usage**: CPU, memory, storage consumption

### 11.3 Business Metrics

- **MTTR (Mean Time to Resolution)**: Average time to resolve auto-detected issues
- **Issue Volume Reduction**: Reduction in duplicate issue creation
- **Developer Satisfaction**: Feedback on issue quality and relevance

---

## 12. Security Considerations

### 12.1 Authentication & Authorization

- Use Azure Managed Identity for Azure resources (no credentials in code)
- Store GitHub token in Azure Key Vault
- Rotate secrets regularly
- Use least-privilege access (read-only for logs, issue creation only for GitHub)

### 12.2 Data Handling

- Never log sensitive data (passwords, tokens, PII)
- Sanitize error messages before creating issues
- Encrypt state data at rest (Azure Storage encryption)
- Use TLS for all external communications

### 12.3 Deployment Security

- Container image scanning (Azure Container Registry)
- Network isolation (VNet integration)
- Private endpoints for Azure services
- Regular dependency updates (Dependabot)

---

## 13. Open Questions & Next Steps

### 13.1 Questions for Stakeholders

1. **AI Assignment**: How exactly should Claude/Copilot be assigned to issues? Via GitHub API, separate platform, or custom integration?

2. **Log Access**: What Azure RBAC roles does the system need? Will it use Managed Identity or Service Principal?

3. **Repository Permissions**: Should the system auto-create repositories if they don't exist, or only work with pre-configured repos?

4. **Issue Lifecycle**: Should the system close issues automatically if failures stop, or only create/update?

5. **Multi-tenancy**: Will this serve multiple teams/projects? If so, how should configuration be organized?

6. **Compliance**: Any specific compliance requirements (SOC 2, HIPAA, etc.)?

### 13.2 Recommended Next Steps

1. **Validate Assumptions**:
   - Confirm AI assignment mechanism
   - Test Azure Log Analytics query performance
   - Verify GitHub rate limits with expected load

2. **Proof of Concept**:
   - Build minimal version with single failure type
   - Test end-to-end flow (Azure → System → GitHub)
   - Validate duplicate detection accuracy

3. **Architecture Review**:
   - Review with security team
   - Review with Azure/GitHub experts
   - Get feedback from potential users (developers)

4. **Create Implementation Repository**:
   - Set up `error-detection-system` repository
   - Initialize project structure
   - Set up CI/CD pipeline
   - Create development environment

5. **Detailed Design**:
   - Create detailed API specifications
   - Define configuration schema formally
   - Design database schema for state store
   - Create sequence diagrams for edge cases

---

## 14. Conclusion

This architecture proposes a modular, extensible system for automated error detection and issue creation. The design prioritizes:

- **Reliability**: Stateful processing with duplicate detection
- **Extensibility**: Pluggable adapters for logs and issue platforms
- **Maintainability**: Clear separation of concerns and well-defined interfaces
- **Scalability**: Efficient caching and async processing
- **Developer Experience**: Configurable, transparent, well-documented

The recommended technology stack (Python + Azure Container Apps) balances simplicity, performance, and Azure integration. The phased implementation approach allows for iterative validation and refinement.

**Key Success Factors**:
- Start simple with MVP (Phase 1)
- Validate duplicate detection strategies early
- Get developer feedback on issue quality
- Monitor and iterate on configuration
- Plan for extensibility from the start

---

## Appendix A: Glossary

- **Failure**: An error or anomalous condition detected in logs
- **Fingerprint**: A unique identifier for a failure used in duplicate detection
- **Threshold**: Minimum count/frequency of failures before creating an issue
- **Duplicate Detection**: Process of identifying similar/identical failures
- **Checkpoint**: Last processed timestamp for a log source
- **Repository Routing**: Mapping failures to appropriate code repositories
- **Assignment**: Process of assigning created issues to AI assistants

## Appendix B: References

- Azure Monitor Query API: https://learn.microsoft.com/en-us/python/api/azure-monitor-query
- GitHub REST API: https://docs.github.com/en/rest
- Azure Container Apps: https://learn.microsoft.com/en-us/azure/container-apps/
- Pydantic Configuration: https://docs.pydantic.dev/

---

*Document Version: 1.0*
*Last Updated: 2025-12-08*
*Author: Architecture Team*
