# Commands used to instruct agents

## Claude-Flow init

```
npx claude-flow@alpha init --force
```

## Command template

```
npx claude-flow@alpha swarm "" --claude
```

## Initial planning

> Initial design request

```
npx claude-flow@alpha swarm "This is a new application without existing functionality. Create an architectural design proposal based on specifics in file requirements/Requirements.md. Place the proposal in folder plans. Please show me your thinking step by step and document any assumptions you make. Please let me know if you need additional information." --claude
```
```
- This is a new application without existing functionality.
- Create an architectural design proposal based on specifics in file requirements/Requirements.md.
- Place the proposal in folder plans.
- Please show me your thinking step by step and document any assumptions you make.
- Please let me know if you need additional information.

```

Team response was:
```
  1. AI Assignment Mechanism: How should Claude/Copilot be assigned? Via GitHub API mentions, separate platform
  integration, or custom webhook?
  2. Azure Permissions: Will the system use Managed Identity or Service Principal? What RBAC roles needed?
  3. Issue Lifecycle: Should the system auto-close issues when failures stop, or only create/update?
  4. Multi-tenancy: Will this serve multiple teams? How should configuration be organized (per-team, per-project)?
  5. Log Volume: What's the expected log volume? (This affects whether polling vs streaming is appropriate)
```

Answers added to the requirements are:
Assume that either Claude or Copilot are defined to the repository. Let the installer configure which AI coding agent is available and who the assignee for the issue is. These coding agents will react to that issue aqssignment.
Azure permissions will be managed identity. Please specify any RBAC permissions the design needs.
The service will serve multiple teams. Assume a central administrator. there's no need for the complexity to support team-specific administrators.
I don't know log volume, but assume it to be high.

> Replan based on answers provided.

```
npx claude-flow@alpha swarm "Requirements have been updated to answer your questions. Please revise the architectural design based on the current version of file requirements/Requirements.md. Place the proposal in folder plans. Please show me your thinking step by step and document any assumptions you make. Please let me know if you need additional information." --claude
```

- Requirements have been updated to answer your questions. Please revise the architectural design based on the current version of file requirements/Requirements.md.
- Place the proposal in folder plans.
- Please show me your thinking step by step and document any assumptions you make.
- Please let me know if you need additional information.