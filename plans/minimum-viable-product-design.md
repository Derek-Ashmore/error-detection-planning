# Minimum Viable Product Design - Log Analysis and GitHub Issue Creation

## MVP Objective
Prove the core concept: automatically detect failures in Azure logs and create GitHub issues, minimizing development effort while demonstrating value.

## MVP Scope

### IN SCOPE (MVP Features)
1. **Single Log Source**: Azure Log Analytics (JSON format only)
2. **Simple Failure Detection**: Pattern matching on single, predefined error patterns
3. **Basic GitHub Integration**: Create issues with error details
4. **Simple Duplicate Detection**: Hash-based matching on error message + stack trace
5. **Manual Configuration**: Single JSON config file
6. **Single Repository**: All detected errors create issues in one configured repository
7. **Low Volume**: Handle 1-2 applications with < 100 log entries per hour

### OUT OF SCOPE (Deferred to Full Product)
1. Multiple cloud providers (AWS, GCP)
2. Multiple log formats
3. Complex failure evaluation/thresholds
4. Multi-repository routing based on error context
5. AI-powered duplicate detection
6. Claude Code/Copilot assignment
7. Admin UI/dashboard
8. Azure Managed Identity (use service principal for MVP)
9. Multi-tenancy/team isolation
10. High-volume processing/streaming
11. Advanced configuration (thresholds, custom rules)

## MVP Architecture

### Components

#### 1. Log Fetcher (Azure Log Analytics Client)
- **Purpose**: Query Azure Log Analytics workspace periodically
- **Technology**: Node.js with `@azure/monitor-query` SDK
- **Scope**:
  - Single workspace
  - Single table (e.g., `AppTraces` or `AppExceptions`)
  - Simple time-windowed query (last 5 minutes)
- **Config**: Workspace ID, table name, query interval

#### 2. Failure Detector
- **Purpose**: Identify log entries matching failure patterns
- **Technology**: Simple regex/string matching in Node.js
- **Scope**:
  - Single predefined pattern (e.g., `"severity": "Error"` or `"level": "Error"`)
  - Extract: timestamp, message, stack trace (if present)
- **Config**: Error pattern definition

#### 3. Duplicate Checker
- **Purpose**: Prevent duplicate GitHub issues
- **Technology**: In-memory cache (Map) + SHA256 hash
- **Scope**:
  - Hash = SHA256(error_message + first_10_lines_of_stack)
  - Cache stores hashes in memory (lost on restart - acceptable for MVP)
  - No persistence layer
- **Config**: None needed

#### 4. GitHub Issue Creator
- **Purpose**: Create GitHub issues for new failures
- **Technology**: Octokit REST API
- **Scope**:
  - Single repository (owner/repo from config)
  - Simple issue template:
    - Title: First 80 chars of error message
    - Body: Timestamp, full message, stack trace
    - Labels: `["automated", "error-detection"]`
  - No assignee (deferred to full product)
- **Config**: GitHub token, repository owner/name

#### 5. Scheduler
- **Purpose**: Coordinate periodic execution
- **Technology**: Simple `setInterval` loop
- **Scope**:
  - Poll every 5 minutes (configurable)
  - Sequential execution (no concurrency)

### Deployment

#### Infrastructure
- **Service**: Azure Container Instances (simplest, no orchestration needed)
- **Image**: Single Docker container (Node.js app)
- **Auth**: Service Principal with environment variables
- **Config**: Mounted config file or environment variables
- **Logging**: stdout/stderr (captured by Azure Monitor)

#### Required Azure RBAC Permissions
- **Log Analytics Reader** on the target workspace
- No write permissions needed (read-only MVP)

## Configuration Schema

```json
{
  "azure": {
    "tenantId": "...",
    "clientId": "...",
    "clientSecret": "...",
    "workspaceId": "...",
    "tableName": "AppExceptions"
  },
  "detection": {
    "errorPattern": "\"severity\":\"Error\"",
    "pollIntervalMinutes": 5
  },
  "github": {
    "token": "ghp_...",
    "owner": "myorg",
    "repo": "myapp"
  }
}
```

## MVP Workflow

```
1. Timer triggers (every 5 minutes)
   ↓
2. Fetch logs from Azure Log Analytics (last 5 minutes)
   ↓
3. Filter entries matching error pattern
   ↓
4. For each error:
   a. Calculate hash (message + stack)
   b. Check if hash exists in cache
   c. If new:
      - Create GitHub issue
      - Store hash in cache
   d. If duplicate: skip
   ↓
5. Log summary (X errors, Y new issues created)
   ↓
6. Sleep until next interval
```

## Acceptance Test Strategy

### Automated Acceptance Tests

#### Test Environment
- **Setup**: Docker Compose with:
  - Mock Azure Log Analytics API (simple HTTP server)
  - Mock GitHub API (Octokit-compatible stub)
  - Application container

#### Test Scenarios

1. **Test: Detect and Create Issue for New Error**
   - Given: Mock Azure API returns 1 error log entry
   - When: Scheduler runs
   - Then:
     - GitHub API receives 1 issue creation request
     - Issue title contains error message
     - Issue body contains timestamp and stack trace

2. **Test: Skip Duplicate Errors**
   - Given: Mock Azure API returns same error twice (two poll cycles)
   - When: Scheduler runs twice
   - Then:
     - First run: 1 GitHub issue created
     - Second run: 0 GitHub issues created

3. **Test: Handle Multiple Distinct Errors**
   - Given: Mock Azure API returns 3 different errors
   - When: Scheduler runs
   - Then: 3 GitHub issues created with distinct titles

4. **Test: Ignore Non-Error Logs**
   - Given: Mock Azure API returns 5 info logs, 1 error log
   - When: Scheduler runs
   - Then: 1 GitHub issue created (only for error)

5. **Test: Handle Azure API Failure Gracefully**
   - Given: Mock Azure API returns 500 error
   - When: Scheduler runs
   - Then:
     - No crash
     - Error logged
     - Next poll cycle continues

#### Test Implementation
- **Framework**: Jest + Supertest
- **Mocking**: Nock (HTTP mocking) or custom Express servers
- **Assertions**: Issue creation payloads, call counts
- **Duration**: < 30 seconds total execution time

### Manual Validation Tests
1. Deploy to Azure Container Instances
2. Configure real Azure Log Analytics workspace (test app)
3. Configure real GitHub repository (test repo)
4. Trigger test error in application
5. Verify GitHub issue created within 5 minutes

## MVP Success Criteria

1. **Functional**:
   - Detects 1 error in Azure logs within 5 minutes
   - Creates GitHub issue with correct error details
   - Prevents duplicate issue for same error

2. **Technical**:
   - All automated acceptance tests pass
   - Deploys to Azure Container Instances successfully
   - Runs continuously for 24 hours without crashing

3. **Development Effort**:
   - < 3 days development time
   - < 500 lines of application code
   - < 100 lines of test code per acceptance test

## Technology Recommendations for MVP

### Programming Language: **Node.js (TypeScript)**
**Rationale**:
- Azure SDK well-supported (@azure/monitor-query)
- Octokit (GitHub API client) is native
- Fast development for simple I/O operations
- Easy containerization

### Deployment: **Azure Container Instances**
**Rationale**:
- No orchestration complexity (vs AKS)
- Pay-per-second billing
- Simple deployment (az container create)
- Adequate for low-volume MVP

### Alternative Considered (but rejected for MVP):
- **Azure Functions**: Requires timer trigger setup, more moving parts
- **Python**: Slower development (less mature Azure SDK experience)
- **Kubernetes**: Massive overkill for single container

## MVP Limitations & Future Enhancements

### Known MVP Limitations
1. **Lost state on restart**: In-memory cache means duplicates may occur after restarts
2. **Single repository**: All errors go to one repo, regardless of source application
3. **No AI/ML**: Simple pattern matching only
4. **No web UI**: Configuration requires file editing
5. **Low volume only**: Sequential processing, no streaming

### Path to Full Product
1. **Phase 2**: Add persistence (Azure Table Storage for duplicate cache)
2. **Phase 3**: Multi-repository routing (parse stack trace for file paths)
3. **Phase 4**: AI-powered duplicate detection (embedding similarity)
4. **Phase 5**: Claude Code/Copilot integration
5. **Phase 6**: Multi-cloud support (AWS CloudWatch, GCP Logging)
6. **Phase 7**: Admin dashboard (Azure Static Web Apps)

## Development Plan

### Sprint 1 (1 day): Core Components
- Azure Log Analytics client (fetch logs)
- Simple error pattern matcher
- GitHub issue creator
- In-memory duplicate cache

### Sprint 2 (1 day): Integration & Deployment
- Scheduler/main loop
- Docker containerization
- Azure Container Instances deployment
- Configuration loading

### Sprint 3 (1 day): Testing & Validation
- Automated acceptance tests
- Manual end-to-end test
- Documentation (README, deployment guide)

## Estimated Costs (MVP)

- **Azure Container Instances**: ~$10/month (1 vCPU, 1GB RAM, always on)
- **Azure Log Analytics**: Included (read-only, no ingestion cost)
- **GitHub**: Free (public/private repos both support API)
- **Development Time**: 3 days @ 1 developer

## Repository Structure (New Repo for Implementation)

```
error-detection-mvp/
├── src/
│   ├── azure-client.ts          # Log Analytics queries
│   ├── failure-detector.ts      # Pattern matching
│   ├── duplicate-checker.ts     # Hash-based cache
│   ├── github-client.ts         # Issue creation
│   └── main.ts                  # Scheduler/orchestrator
├── tests/
│   ├── acceptance/
│   │   ├── detect-new-error.test.ts
│   │   ├── skip-duplicates.test.ts
│   │   └── handle-multiple-errors.test.ts
│   └── mocks/
│       ├── azure-api-mock.ts
│       └── github-api-mock.ts
├── config/
│   ├── config.schema.json
│   └── config.example.json
├── Dockerfile
├── docker-compose.test.yml      # For acceptance tests
├── package.json
├── tsconfig.json
└── README.md
```

## Decision Log

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| Node.js over Python | Faster development, native GitHub/Azure SDKs | Less data science ecosystem (OK for MVP) |
| Container Instances over Functions | Simpler deployment, stateful loop | Slightly higher cost (~$10 vs ~$5/mo) |
| In-memory cache over DB | Zero dependencies, fast | Lost state on restart (acceptable for proof) |
| Single repo target over routing | Minimizes config complexity | All errors go to one place (OK for 1-2 apps) |
| Hash-based deduplication over AI | Deterministic, no model training | May miss semantic duplicates (OK for MVP) |
| Service Principal over Managed Identity | Simpler local development | Slightly less secure (OK for MVP) |

## Conclusion

This MVP design minimizes development effort (~3 days) while proving the core value proposition: **automated error detection and GitHub issue creation**. The scoped-down approach focuses on a single happy path with basic acceptance tests, deferring complexity (multi-cloud, AI deduplication, Claude Code integration) to post-MVP phases.

**Next Steps**:
1. Create new repository `error-detection-mvp`
2. Implement Sprint 1 components
3. Write acceptance tests
4. Deploy to Azure and validate with real logs
