# Repository Context for GitHub Copilot

This repository uses Azure DevOps for source control and pull requests.

For PR review and fix tasks, this repo provides custom agents:
- **pr-review**  review an Azure DevOps PR and post inline comments. Invoke via the agents dropdown or `/prReview`.
- **pr-fix**  implement review findings on a worktree of the PR branch. Reachable as a handoff from pr-review.

Both agents drive the **Azure CLI (`az`)** directly. There is no MCP server and no wrapper scripts. All recipes  auth, listing PRs, fetching diffs, posting comments  live in `.github/instructions/azure-devops-cli.instructions.md` and are auto-applied to every chat in this workspace.

Team coding standards live in `.github/instructions/`. The PR Review agent is required to consult them when forming findings.
