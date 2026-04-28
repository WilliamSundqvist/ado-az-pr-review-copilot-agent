# Azure DevOps PR Review Agent

Review Azure DevOps pull requests with GitHub Copilot inside VS Code, using the **Azure CLI**. Drop in one prompt file, run `/createPrReviewAgent`, and get a repo-tailored PR review agent in minutes.

The generated review flow is designed to **save tokens** by keeping the review automatic up front and only asking for user input at the comment-posting and deeper-investigation stages.

This project started as a CLI adaptation of the MCP-based Azure DevOps PR review work from [segunak/azure-devops-github-copilot-pr-review](https://github.com/segunak/azure-devops-github-copilot-pr-review).

## What It Does

This project generates a PR review agent that:
- Reviews Azure DevOps pull requests
- Reads PR diffs and surrounding code
- Applies your repo's review standards
- Lets you choose which findings to post as inline comments

It does **not** edit code, push commits, or change PR state.

## Choose Your Prompt

| Your machine | Use this prompt |
|---|---|
| Windows (PowerShell) | [`create-pr-review-agent-windows.prompt.md`](./create-pr-review-agent-windows.prompt.md) |
| macOS / Linux (bash, zsh) | [`create-pr-review-agent-unix.prompt.md`](./create-pr-review-agent-unix.prompt.md) |

## Prerequisites

- **VS Code 1.106+** with [GitHub Copilot](https://docs.github.com/en/copilot)
- **Git**
- **[Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)** with the `azure-devops` extension
- **`jq`** on macOS/Linux

Setup:

```bash
az login
az extension add --name azure-devops
```

## Quick Start

1. Copy the prompt for your shell to `.github/prompts/create-pr-review-agent.prompt.md`.
2. Open VS Code and run `/createPrReviewAgent` in Copilot Chat.
3. Confirm the detected Azure DevOps org, project, repo, and default branch.
4. Review the generated `.github/instructions/code-review.instructions.md`.
5. Commit and push the generated files.

This generates:

| File | Purpose |
| --- | --- |
| `.github/agents/pr-review.agent.md` | The PR review agent |
| `.github/instructions/code-review.instructions.md` | Review standards tailored to your repo |
| `.github/instructions/azure-devops-cli.instructions.md` | Azure DevOps CLI recipes the agent uses |
| `.github/prompts/pr-review.prompt.md` | `/prReview` slash command |
| `.github/copilot-instructions.md` | Repo context |

## Using the Agent

Use either flow:

- Select the **pr-review** agent in Copilot Chat and paste a PR number or URL
- Run `/prReview` and provide a PR number or URL

The agent will:

1. Verify Azure CLI access
2. Read relevant repo instructions
3. Fetch PR metadata and diff
4. Review changed code and gather context
5. Group findings into:
   - ready to post
   - needs more evidence
   - likely noise
6. Use Copilot's popup UI to let you:
   - post all ready findings
   - pick which findings to post
   - investigate uncertain findings first

## Manual Setup

If you prefer to wire it up manually:

1. Copy `example-windows/.github/` or `example-unix/.github/` into your repo.
2. Replace `__ADO_ORG__`, `__ADO_PROJECT__`, `__ADO_REPO__`, `__DEFAULT_BRANCH__`, and `__ORG_URL__`.
3. Edit `code-review.instructions.md` to match your team's standards.
4. Commit and push.

## Customization

- Add more review standards in `.github/instructions/`
- Edit review categories in `.github/agents/pr-review.agent.md`
- Pin a model in the agent frontmatter if you want

## Limitations

- Runs locally in VS Code
- Designed for Azure DevOps PR review only
- Not a replacement for human review

## Repository Structure

```text
create-pr-review-agent-windows.prompt.md
create-pr-review-agent-unix.prompt.md
example-windows/
example-unix/
README.md
```

## License

MIT
