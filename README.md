# Azure DevOps PR Review Agent (CLI Edition)

Review Azure DevOps pull requests with GitHub Copilot from inside VS Code, driven entirely by the **Azure CLI** — no MCP server, no PAT, no extra services. Drop one prompt file in your repo, run it, and you get a fully tailored PR Review agent in minutes.

> **Origin:** This is a CLI rewrite of [segunak/azure-devops-github-copilot-pr-review](https://github.com/segunak/azure-devops-github-copilot-pr-review). The structure (scaffolding prompt → generated files → review agent + instructions + slash command) is preserved; the MCP server is replaced by CLI recipes baked into an instructions file. Credit to that project for the original design.

## Scope

This agent **reviews PRs and posts inline comments**. That's it. It does not edit code, does not implement fixes, does not push commits. The reviewer (you) makes any changes the comments suggest. This is a deliberate scope choice — keeping the agent narrow makes it more reliable and the prompts shorter.

## Why CLI

- **Lower context cost.** No MCP tool schemas to load. The model already knows `az`.
- **Composability.** `az repos pr show ... | jq ... | other-tool` is trivial.
- **Debuggability.** Failures show the exact command, not "tool call failed".
- **No extra runtimes.** No Node.js, no npx, no MCP server lifecycle.
- **Source-of-truth alignment.** `az` and the REST API are authoritative; MCP wrappers occasionally lag.

The one thing MCP did better — posting structured inline comments — is solved here by `az devops invoke` against the threads API. The instructions file documents the exact command shape, calibrated against real-world failures.

## Two scaffolds, one for each shell

The Windows version is calibrated against PowerShell quirks (UTF-8 BOM, JMESPath backtick mangling, `--api-version` float-parsing, here-string interrupts, `ConvertTo-Json` edge cases). The Unix version is plain bash. They are otherwise functionally identical.

| Your machine | Use this prompt |
|---|---|
| Windows (PowerShell) | [`create-pr-review-agent-windows.prompt.md`](./create-pr-review-agent-windows.prompt.md) |
| macOS / Linux (bash, zsh) | [`create-pr-review-agent-unix.prompt.md`](./create-pr-review-agent-unix.prompt.md) |

## Prerequisites

- **VS Code 1.106+** with [GitHub Copilot](https://docs.github.com/en/copilot). [Custom agents were introduced in 1.106.](https://code.visualstudio.com/updates/v1_106)
- **Git**.
- **[Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)** with the `azure-devops` extension:
  ```
  az login
  az extension add --name azure-devops
  ```
- **`jq`** (Unix only) — `brew install jq` on macOS, your package manager on Linux.

That's it. No Node.js, no npx, no MCP server, no PAT.

## Quick Start

The scaffolding prompt is **self-contained** — every file it generates is inlined inside it.

1. Pick the prompt for your shell (table above) and place it at `.github/prompts/create-pr-review-agent.prompt.md` in your repo. The path must be exact.
2. Open VS Code → Copilot Chat → type `/createPrReviewAgent`.
3. Confirm the auto-detected ADO org/project/repo and default branch (or correct them).
4. Review the generated `.github/instructions/code-review.instructions.md` (the AI's best guess at your team's standards — edit as needed).
5. Commit and push.

This generates (or merges into):

| File | Purpose |
| --- | --- |
| `.github/agents/pr-review.agent.md` | The PR Review custom agent |
| `.github/instructions/code-review.instructions.md` | Review standards, tailored to your codebase |
| `.github/instructions/azure-devops-cli.instructions.md` | CLI runbook with all recipes the agent calls |
| `.github/prompts/pr-review.prompt.md` | `/prReview` slash command |
| `.github/copilot-instructions.md` | Repo context (created or merged) |

No `scripts/` folder. No `.vscode/mcp.json`. No fix agent. The CLI runbook is everything.

## Using the Agent

**Option A:** In Copilot Chat, click the agent dropdown and select **pr-review**, then paste a PR number or URL.

**Option B:** Type `/prReview` in chat. It asks for the PR number (or lists your open PRs if you don't give one) and an optional focus area.

The agent then:

1. Verifies `az` auth and the `azure-devops` extension.
2. Inventories repo markdown and reads team conventions (filtering by name patterns rather than reading everything).
3. Fetches PR metadata, the diff (via local `git diff target...source`), linked work items, and existing comment threads.
4. Reads surrounding code at the PR's source commit (via `git show <sha>:path`) for context.
5. Forms findings, grouped by severity (blocker / major / minor / nit).
6. Presents findings numbered and asks **via interactive carousel** (radio + multi-select buttons): *"Post these as PR comments? (all / pick which / walk through / skip)"*. Pick mode shows a checkbox list of every finding. No typing required.
7. Maintains an in-session list of posted findings to prevent duplicate comments.

## Manual Setup

Prefer to wire it up by hand?

1. Copy the matching `example-windows/` or `example-unix/` tree's `.github/` folder into your repo root.
2. In `pr-review.agent.md`, find the **Team Configuration** comment block and replace `__ADO_ORG__`, `__ADO_PROJECT__`, `__ADO_REPO__`, `__DEFAULT_BRANCH__`, `__ORG_URL__` with your team's values.
3. In `azure-devops-cli.instructions.md`, do the same find-and-replace.
4. Edit `code-review.instructions.md` to match your team's standards.
5. Commit and push.

## Customisation

- **Add review standards.** Drop more `*.instructions.md` files into `.github/instructions/`. They auto-apply.
- **Change review categories.** Edit Phase 5 in `pr-review.agent.md`.
- **Pin a model.** Add `model: ['Claude Opus 4.7']` (or similar) to the agent's YAML frontmatter.
- **Add a new CLI recipe.** Edit `azure-devops-cli.instructions.md`.

## Limitations

- **Local only.** Runs in each engineer's VS Code. No webhook/pipeline trigger. (Pair with a marketplace extension like [`ado-copilot-code-review`](https://github.com/little-fort/ado-copilot-code-review) if you want automatic-on-PR-open.)
- **VS Code only.** Custom agents are a VS Code feature. The setup ports to Claude Code or Cursor with frontmatter changes; not 1-to-1.
- **Review only.** No fix agent. The reviewer implements suggested changes themselves.
- **Not a replacement for human review.** It assists; the engineer makes the call.

## Design notes

### Interactive carousels via `vscode/askQuestions`

The agent uses VS Code's built-in `vscode/askQuestions` tool to render decision points as proper UI carousels (radio buttons, checkboxes, text input) rather than plain text questions. Five decision points get carousels:

1. **Review focus** at the start (Standard / Security / Correctness / Light / Custom + optional free-text)
2. **Posting decision** after findings are presented (Post all / Pick which / Walk through / Skip)
3. **Per-finding approval** in walk-through mode (Post / Skip / Edit)
4. **Failure handler** when a POST errors (Retry / Skip / Show error / Stop)
5. **End of session** (Add another finding / Export / Done)

Each carousel is a single tool call regardless of how many questions it bundles, so polish costs nothing in model requests. If the tool isn't available (older VS Code, etc.), the agent falls back to plain-text questions.

### Idempotency

The agent maintains an in-session list of `{file, line, thread_id, content_hash}` for every comment posted. Before each post it (1) re-fetches the PR threads and (2) checks against this list. After a successful post (the API returned a thread `id`), the finding is marked **done** — the agent will refuse to post it again, even if the user asks. This was added after a real-world run produced six comments for three findings.

### `git diff` is the diff method

`git diff <target-commit>...<source-commit>` (three dots) is fast, complete, and gives the agent text it can read directly. Faster and more complete than the ADO REST diff endpoint; no fallback in either direction.

### `git show <sha>:path` for surrounding code

The review agent never modifies files, so it doesn't need a worktree or a checkout. It reads source files at the PR's source commit via `git show`, which is one bash command per file with no setup or cleanup. Earlier iterations considered adding a worktree for review-time browsing, but the cost (network fetch, filesystem create, cleanup, platform quirks) outweighs the per-file convenience for typical PR sizes.

### `--api-version "7.1"`, not `7.1-preview.1`

PowerShell's argument parser strips `preview.` and tries to interpret `7.1.1` as a float, producing `could not convert string to float: '7.1.1'`. The threads API works fine on `7.1`. The Unix runbook uses the same value for consistency.

### Single-line JSON literal + `WriteAllText` (Windows)

The Windows runbook prescribes one and only one pattern for posting comments:

1. Build the JSON body as a single-quoted PowerShell string literal containing the raw JSON.
2. Write it via `[System.IO.File]::WriteAllText` with `UTF8Encoding($false)` to avoid the BOM.
3. POST via `az devops invoke --in-file`.

Alternatives — `ConvertTo-Json` on a hashtable, double-quoted strings with backtick escapes, here-strings, `Out-File -Encoding utf8` — have all failed in real-world runs in different ways. The runbook documents what each one breaks. Don't substitute.

### Why no fix agent

An earlier version included a companion "PR Fix" agent that implemented suggested fixes on a git worktree. It was removed because the scope creep wasn't worth it: review and fix are different cognitive tasks, the fix flow needs a much different prompt structure, and most reviewers prefer to implement fixes themselves anyway. Keeping the agent narrow makes it more reliable.

### Why no scripts

An earlier iteration shipped a `scripts/` folder of bash wrappers. A real-world run on Windows revealed the bash scripts were dead weight: most Windows developers don't have WSL or Git Bash configured, so the agent had to abandon them and improvise raw `az` commands anyway. The scripts also created a layer of indirection that was harder to debug than just calling `az` directly. The current design — recipes in an instructions file, agent calls `az` directly — works on every platform that has `az` installed.

## Repository structure

```
create-pr-review-agent-windows.prompt.md   # PowerShell scaffold
create-pr-review-agent-unix.prompt.md      # bash scaffold
example-windows/                           # PowerShell reference set
  .github/
    agents/pr-review.agent.md
    instructions/{code-review,azure-devops-cli}.instructions.md
    prompts/pr-review.prompt.md
    copilot-instructions.md
example-unix/                              # bash reference set (same structure)
README.md
```

## License

MIT (matching the upstream project).
