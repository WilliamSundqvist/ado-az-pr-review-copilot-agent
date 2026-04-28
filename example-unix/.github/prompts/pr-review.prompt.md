---
description: Review an Azure DevOps pull request
mode: agent
agent: pr-review
---

Review the Azure DevOps pull request the user specifies.

If the user did not include a PR number or URL, list their open PRs first using the patterns in `.github/instructions/azure-devops-cli.instructions.md` (under "List PRs"), and ask which one to review.

Accept either a bare number (e.g. `4567`) or a full URL (extract the trailing PR ID).

Then proceed through the phases defined in your agent instructions:

1. Verify setup
2. Discover repo conventions
3. Gather PR context
4. Read surrounding code
5. Form findings
6. Present grouped findings
7. Use `vscode/askQuestions` for posting decisions or deeper investigation
8. Post approved findings (with the strict idempotency rules in Phase 6 â€” never post the same finding twice)
9. Final summary

If `vscode/askQuestions` is unavailable, fall back to plain text questions for each carousel.
