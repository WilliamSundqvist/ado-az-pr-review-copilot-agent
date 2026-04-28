---
applyTo: '**'
description: Azure DevOps CLI runbook for PowerShell (Windows). Auto-applied to every chat in this workspace.
---

# Azure DevOps CLI Runbook  PowerShell

Recipes for everything the PR Review and PR Fix agents need to do, calibrated for PowerShell on Windows. Every recipe has been tested against the gotchas listed at the bottom  **do not deviate from the patterns shown**.

## Configuration

Replace these placeholders with your team's values (the scaffolding prompt does this for you):

- `__ORG_URL__`  e.g. `https://dev.azure.com/contoso/` or `https://contoso.visualstudio.com/`
- `__ADO_PROJECT__`  e.g. `MyProject`
- `__ADO_REPO__`  e.g. `my-repo`

## One-time setup

```powershell
az login
az extension add --name azure-devops
```

## Auth probe (always run first)

```powershell
az --version | Select-Object -First 1
az account show --query user.name -o tsv          # fails if not logged in
az extension list --query "['name=='azure-devops'].name" -o tsv   # empty if extension missing
```

If `az.cmd` is on PATH but `az` isn't, `az` still works because PowerShell resolves `.cmd` automatically. No special handling needed.

---

## List PRs

PRs awaiting your review:
```powershell
$me = az account show --query user.name -o tsv
az repos pr list --org "__ORG_URL__" --repository __ADO_REPO__ --project __ADO_PROJECT__ `
  --status active --reviewer $me --output table
```

PRs you opened:
```powershell
az repos pr list --org "__ORG_URL__" --repository __ADO_REPO__ --project __ADO_PROJECT__ `
  --status active --creator $me --output table
```

All active PRs:
```powershell
az repos pr list --org "__ORG_URL__" --repository __ADO_REPO__ --project __ADO_PROJECT__ `
  --status active --output table
```

---

## Get PR metadata

```powershell
$pr = <PR_ID>
$meta = az repos pr show --id $pr --org "__ORG_URL__" --output json | ConvertFrom-Json
$meta | Select-Object pullRequestId, title, status, isDraft, sourceRefName, targetRefName,
       @{n='source'; e={$_.lastMergeSourceCommit.commitId}},
       @{n='target'; e={$_.lastMergeTargetCommit.commitId}},
       @{n='author'; e={$_.createdBy.displayName}}
```

---

## Get the diff (the only correct way)

Always via local git, never via REST:

```powershell
$source = $meta.sourceRefName -replace '^refs/heads/', ''
$target = $meta.targetRefName -replace '^refs/heads/', ''
$srcSha = $meta.lastMergeSourceCommit.commitId
$tgtSha = $meta.lastMergeTargetCommit.commitId

git fetch origin $source $target --quiet
git diff "$tgtSha...$srcSha"
```

The three-dot syntax shows changes on the source branch since it diverged from target  exactly what reviewers need. **Do not use `--query-parameters` against the `git/diffs` REST endpoint;** it's slower, has a smaller output limit, and offers nothing local git doesn't.

---

## Get linked work items

```powershell
$wiIds = az repos pr work-item list --id $pr --org "__ORG_URL__" --output tsv --query '[].id'
foreach ($wid in $wiIds) {
  az boards work-item show --id $wid --org "__ORG_URL__" --output json `
    --query '{id:id, title:fields."System.Title", type:fields."System.WorkItemType", state:fields."System.State"}'
}
```

---

## List existing comment threads (duplicate-prevention)

**Avoid `--query` here.** PowerShell's backtick is the escape character; JMESPath's backtick literals (e.g. `` ['isDeleted==`false`] ``) get mangled before reaching `az`. Filter in PowerShell instead:

```powershell
$tmpThreads = Join-Path $env:TEMP "pr-threads-$pr.json"

az devops invoke `
  --area git --resource pullRequestThreads `
  --route-parameters project=__ADO_PROJECT__ repositoryId=__ADO_REPO__ pullRequestId=$pr `
  --http-method GET `
  --org "__ORG_URL__" `
  --api-version "7.1" `
  --output json | Out-File -Encoding utf8 $tmpThreads

$threads = (Get-Content $tmpThreads -Raw | ConvertFrom-Json).value | Where-Object { -not $_.isDeleted }

$threads | Select-Object id, status,
  @{n='filePath'; e={$_.threadContext.filePath}},
  @{n='line';     e={$_.threadContext.rightFileStart.line}},
  @{n='author';   e={$_.comments[0].author.displayName}},
  @{n='preview';  e={ if ($_.comments[0].content) { $_.comments[0].content.Substring(0, [Math]::Min(120, $_.comments[0].content.Length)) } } }
```

> **Why temp file then read:** piping `az devops invoke` directly into another command is unreliable in PowerShell  the CLI's output buffering can deliver an empty stream to the next process.

---

## Post a comment  the only correct pattern

There is **one** pattern for posting comments. Multiple alternatives have been tried in the field and have failed in real-world runs. Use this pattern verbatim.

### The pattern

```powershell
$pr = <PR_ID>
$tmp = Join-Path $env:TEMP "pr-comment-$pr.json"

# Build the entire JSON body as a single-line string LITERAL (single-quoted in PowerShell).
# - Backticks in the comment body: pass through as-is. They are valid JSON.
# - Newlines in the comment body: use the two literal characters \ + n. ADO renders \n as a line break.
# - Double quotes in the comment body: escape as \" inside the string.
# - Do NOT use Unicode escapes like \u2014  PowerShell does not expand them in single-quoted strings.

$json = '{"comments":[{"parentCommentId":0,"content":"YOUR COMMENT TEXT HERE WITH \n FOR NEWLINES","commentType":1}],"status":1,"threadContext":{"filePath":"/src/auth/login.ts","rightFileStart":{"line":42,"offset":1},"rightFileEnd":{"line":42,"offset":1}}}'

# Write WITHOUT a UTF-8 BOM. Out-File -Encoding utf8 adds a BOM that az devops invoke rejects.
[System.IO.File]::WriteAllText($tmp, $json, [System.Text.UTF8Encoding]::new($false))

# POST. Capture the response.
$resp = az devops invoke `
  --area git --resource pullRequestThreads `
  --route-parameters project=__ADO_PROJECT__ repositoryId=__ADO_REPO__ pullRequestId=$pr `
  --http-method POST `
  --in-file $tmp `
  --org "__ORG_URL__" `
  --api-version "7.1" `
  --output json | ConvertFrom-Json

if ($LASTEXITCODE -eq 0 -and $resp.id) {
  "Posted thread $($resp.id)"
} else {
  "POST failed; do NOT retry without re-listing threads first"
}
```

### Top-level vs inline

- **Top-level (PR overview) comment:** drop the entire `threadContext` field from the JSON body.
  ```
  '{"comments":[{"parentCommentId":0,"content":"...","commentType":1}],"status":1}'
  ```
- **Inline (anchored to file/line):** include `threadContext`. `filePath` MUST start with `/`. For a single-line comment, `rightFileEnd.line` equals `rightFileStart.line`. For a multi-line range, set `rightFileEnd.line` to the last line.

### Why this pattern and not others

These have all been tried and have all failed in real-world runs:

| Alternative | What broke |
|---|---|
| `ConvertTo-Json` on a `@{}` hashtable | Works for simple bodies, but breaks with combinations of characters that the serializer mishandles. Real failures observed. |
| Double-quoted PowerShell string with `` \` `` escapes | Backtick is PowerShell's escape character; `` \` `` next to certain text breaks the parser entirely. The `$content` variable ends up empty and ADO returns *"A comment without any content cannot be added."* |
| `\u2014` Unicode escapes inside a string | PowerShell does **not** expand `\u` sequences. They become literal characters. |
| Here-strings (`@" ... "@`) | Hang the terminal when they contain backtick-fenced code blocks; the line-continuation parser waits for more input. |
| `Out-File -Encoding utf8` | Writes a UTF-8 BOM. `az devops invoke --in-file` rejects with *"Unexpected UTF-8 BOM"*. |

**The single-line JSON literal + `WriteAllText` pattern avoids all of these.** Use it. Don't substitute.

### A note on long bodies

For long comments (more than ~500 chars), the single-line literal is harder to read in the terminal but still works. Define it on its own line with no other code on the same line  don't try to break it across multiple lines with PowerShell continuation operators (backtick or pipe).

### Use a fresh terminal before posting

If recent commands have left the terminal with a large output buffer or trailing prompts, **start a new terminal session before posting**. After multiple failed attempts, terminal output buffer state in PowerShell becomes unreadable and you can't tell whether the POST succeeded. A clean prompt avoids this entirely.
---

## Vote / approve / reject (do not use during review)

The PR Review agent does not change PR state. Listed here only for completeness if the user explicitly asks:

```powershell
az repos pr set-vote --id $pr --org "__ORG_URL__" --vote approve|approve-with-suggestions|reject|wait-for-author|reset
```

---

## Idempotency cheat sheet

After a successful POST (`$LASTEXITCODE -eq 0` and `$resp.id` exists):
- Append to in-session `posted_findings`: `{ file, line, thread_id = $resp.id, content_hash }`.
- **Do not retry**, even if the user asks. Offer the thread URL instead:
  `__ORG_URL____ADO_PROJECT__/_git/__ADO_REPO__/pullrequest/$pr'_a=overview&discussionId=$($resp.id)`

After a failed POST:
- Re-run the threads listing. If the comment is now present, mark it posted (capture the `id`).
- If still absent, retry once.

---

## When things break

| Symptom | Cause | Fix |
|---|---|---|
| `could not convert string to float: '7.1.1'` | `--api-version 7.1-preview.1` parsed as float | Use `--api-version "7.1"` (quoted, no preview) |
| `Unexpected UTF-8 BOM` | `Out-File -Encoding utf8` writes BOM | Use `[System.IO.File]::WriteAllText` + `UTF8Encoding($false)` |
| `invalid jmespath_type value` with `\x0c` or other escapes | PowerShell ate JMESPath backticks | Drop `--query`; filter in PowerShell with `Where-Object` |
| `JSONDecodeError: Expecting value: line 1 column 1 (char 0)` | Piping `az devops invoke` output yields empty stream | Write to temp file first, then `Get-Content -Raw \| ConvertFrom-Json` |
| Terminal hangs waiting for input mid-command | Here-string with backtick code block confused PowerShell parser | Build JSON as a single-line string with `\n` escapes |
| `TF401019: ... does not exist or you do not have permissions` | Wrong `project=` / `repositoryId=` route param, or no access | Check the values; verify `az account show` is the right user |
| `InteractiveBrowserCredential authentication failed` | Token expired | `az login` |

---

## Useful query tips

For simple JSON shaping where backticks are NOT involved, `--query` works fine:

```powershell
az repos pr show --id $pr --org "__ORG_URL__" --query 'sourceRefName' -o tsv
```

For multi-value extraction, capture into an array:

```powershell
$vals = az repos pr show --id $pr --org "__ORG_URL__" --query '[sourceRefName, targetRefName]' -o tsv
$src = $vals[0]; $tgt = $vals[1]
```

But once a query needs `['field == \`literal\`]`  drop `--query` and filter in PowerShell.
