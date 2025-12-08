## Purpose
This process will act independently. It does the following:
- Monitors logs for failures. Initially monitor Azure Log Diagnostics, but I will add other sources later, possibly using other clouds.
- Initial log format supported is JSON for Log Diagnostics. Others will be added later.
- Creates a GitHub issue describing the failures detected and providing any information available regarding defect context. GitLab and Bitbucket might be added later.
- Evaluates failures detected to eliminate duplicate issue generation
- Assigns the issue to Claude or Copilot for investigation and possibly draft PR to fix. Note that creating the issue with the assignment is the objective. Investigating the failure is out of scope.

## Configuration options
Installers of this process will provide a runtime configuration.  Configurable items will include:
- Definition of a failure to evaluate and processs
- Threshold for issue generation (e.g. number of failures, number off failures )
- Which log analytics tables to monitor
- Register failure to repository so that the issue can be created in the repository containing relevant source code.

## Recommendations sought
I would like your recommendations on the following items:
- Different methods of detecting which failures are duplicate.
- Programming language selection
- Deployment service selection for Azure
- Additional configuration items that support developers may find useful.
- Repository organization for implementation of this process. This repository is for planning only.



