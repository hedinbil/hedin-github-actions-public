# Agent Renova - Repository Cleanup

This GitHub Action executes the Renova agent to clean up old workflow runs, artifacts, and stale branches from a repository. The agent helps maintain repository hygiene by removing outdated resources.

## Inputs

| Input                      | Description                                      | Required | Default             |
|----------------------------|--------------------------------------------------|----------|---------------------|
| `cleanup-type`             | Type of cleanup to perform                       | true     |                     |
|                            | Options: `all`, `workflow-runs`, `artifacts`, `branches` | | |
| `github-token`             | GitHub token for API operations                  | false    | `${{ github.token }}` |
| `workflow-runs-cutoff-days`| Number of days before workflow runs are considered old | false | `7` |
| `artifacts-cutoff-days`    | Number of days before artifacts are considered old | false | `7` |
| `branches-cutoff-days`     | Number of days before branches are considered stale | false | `30` |

## Usage

Below is an example of how to use this action in a GitHub Actions workflow.

```yaml
name: Agent Renova 🧹 - Repository Cleanup
run-name: Agent Renova 🧹 Repository Cleanup

on:
  workflow_dispatch:
    inputs:
      cleanup_type:
        description: 'Type of cleanup to perform'
        required: true
        default: 'all'
        type: choice
        options:
          - 'all'
          - 'workflow-runs'
          - 'artifacts'
          - 'branches'
  schedule:
    # Run every two hours
    - cron: '0 */2 * * *'

permissions:
  contents: write
  actions: write
  id-token: write
  pull-requests: read
  issues: read

jobs:
  cleanup:
    name: Repository Cleanup
    runs-on: ubuntu-latest
    steps:
      - name: Run Renova Agent
        uses: hedinbil/hedin-github-actions/agent-renova@main
        with:
          cleanup-type: ${{ github.event.inputs.cleanup_type || 'all' }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Steps Performed by the Action

1. **Determine Cleanup Tasks**: Analyzes the `cleanup-type` input to determine which cleanup operations should run.
2. **Cleanup Workflow Runs**: 
   - Lists all workflow runs in the repository
   - Deletes runs older than the specified cutoff (default: 7 days)
   - Skips runs that are currently in progress
3. **Cleanup Artifacts**: 
   - Lists all artifacts in the repository
   - Deletes artifacts older than the specified cutoff (default: 7 days)
4. **Cleanup Branches**: 
   - Fetches all branches in the repository
   - Identifies branches older than the specified cutoff (default: 30 days)
   - Checks for open pull requests associated with each stale branch
   - Deletes branches that have no open PRs
   - Preserves branches with active pull requests
5. **Summary**: Displays execution summary with cleanup statistics and timestamps.

## Cleanup Details

### Workflow Runs Cleanup
- Deletes workflow runs older than the cutoff period
- Automatically skips runs that are `in_progress`, `queued`, `waiting`, or `requested`
- Processes all workflow runs across the entire repository

### Artifacts Cleanup
- Deletes artifacts older than the cutoff period
- Processes all artifacts in the repository
- Uses pagination to handle repositories with many artifacts

### Branches Cleanup
- Only processes branches that are not `main` or `master`
- Checks the last commit date on each branch
- Verifies if branches have open pull requests before deletion
- Uses GraphQL to efficiently query for open PRs
- Preserves branches with active pull requests to prevent data loss

## Secrets Required

No additional secrets are required beyond the standard `GITHUB_TOKEN` which is automatically provided by GitHub Actions.

## How It Works

1. Triggered by `workflow_dispatch` (manual) or `schedule` (automated)
2. Determines which cleanup operations to perform based on `cleanup-type` input
3. For workflow runs and artifacts: Uses GitHub API via `actions/github-script@v7`
4. For branches: 
   - Checks out the repository
   - Installs GitHub CLI and jq if needed
   - Fetches all branches using pagination
   - Checks each branch's last commit date
   - Queries for open PRs using GraphQL
   - Deletes branches that are stale and have no open PRs
5. Displays a summary of all cleanup operations

## Notes

- The action is idempotent and safe to run multiple times
- Branches with open pull requests are never deleted, even if they are stale
- The action handles errors gracefully and continues processing even if individual deletions fail
- All cleanup operations use pagination to handle large repositories efficiently

