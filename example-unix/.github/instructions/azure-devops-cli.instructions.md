---
applyTo: '**'
description: Azure DevOps CLI runbook for bash (macOS / Linux). Auto-applied to every chat in this workspace.
---

# Azure DevOps CLI Runbook  bash

Recipes for everything the PR Review and PR Fix agents need to do, calibrated for bash on macOS or Linux. This file complements the agents  they reference these patterns rather than re-deriving CLI syntax.

## Configuration

Replace these placeholders with your team's values (the scaffolding prompt does this for you):

- `__ORG_URL__`  e.g. `https://dev.azure.com/contoso/` or `https://contoso.visualstudio.com/`
- `__ADO_PROJECT__`  e.g. `MyProject`
- `__ADO_REPO__`  e.g. `my-repo`

## Prerequisites

- `az` CLI with the `azure-devops` extension
- `jq`  used for JSON shaping. Install with `brew install jq` (macOS) or your package manager (Linux).
- `git`

## One-time setup

```bash
az login
az extension add --name azure-devops
```

## Auth probe (always run first)

```bash
az --version | head -1
az account show --query user.name -o tsv          # fails if not logged in
az extension list --query "['name=='azure-devops'].name" -o tsv   # empty if extension missing
command -v jq >/dev/null || echo "jq is not installed; install before proceeding"
```

---

## List PRs

PRs awaiting your review:
```bash
me=$(az account show --query user.name -o tsv)
az repos pr list --org "__ORG_URL__" --repository __ADO_REPO__ --project __ADO_PROJECT__ \
  --status active --reviewer "$me" --output table
```

PRs you opened:
```bash
az repos pr list --org "__ORG_URL__" --repository __ADO_REPO__ --project __ADO_PROJECT__ \
  --status active --creator "$me" --output table
```

All active PRs:
```bash
az repos pr list --org "__ORG_URL__" --repository __ADO_REPO__ --project __ADO_PROJECT__ \
  --status active --output table
```

---

## Get PR metadata

```bash
PR=<PR_ID>
az repos pr show --id $PR --org "__ORG_URL__" --output json | jq '{
  id: .pullRequestId,
  title, status, isDraft,
  source: .sourceRefName, target: .targetRefName,
  sourceSha: .lastMergeSourceCommit.commitId,
  targetSha: .lastMergeTargetCommit.commitId,
  author: .createdBy.displayName
}'
```

---

## Get the diff (the only correct way)

Always via local git, never via REST:

```bash
read SOURCE TARGET SRC_SHA TGT_SHA < <(
  az repos pr show --id $PR --org "__ORG_URL__" --output tsv \
    --query '[sourceRefName, targetRefName, lastMergeSourceCommit.commitId, lastMergeTargetCommit.commitId]'
)
SOURCE=${SOURCE#refs/heads/}
TARGET=${TARGET#refs/heads/}

git fetch origin "$SOURCE" "$TARGET" --quiet
git diff "$TGT_SHA...$SRC_SHA"
```

The three-dot syntax shows changes on the source branch since it diverged from target  exactly what reviewers need. **Do not use the `git/diffs` REST endpoint;** it's slower, has a smaller output limit, and offers nothing local git doesn't.

---

## Get linked work items

```bash
ids=$(az repos pr work-item list --id $PR --org "__ORG_URL__" --output tsv --query '[].id')
for id in $ids; do
  az boards work-item show --id "$id" --org "__ORG_URL__" --output json \
    --query '{id:id, title:fields."System.Title", type:fields."System.WorkItemType", state:fields."System.State"}'
done
```

---

## List existing comment threads (duplicate-prevention)

```bash
az devops invoke \
  --area git --resource pullRequestThreads \
  --route-parameters project=__ADO_PROJECT__ repositoryId=__ADO_REPO__ pullRequestId=$PR \
  --http-method GET \
  --org "__ORG_URL__" \
  --api-version "7.1" \
  --output json \
  | jq '[ .value[] | select(.isDeleted == false) | {
      id, status,
      filePath: .threadContext.filePath,
      line:     .threadContext.rightFileStart.line,
      author:   .comments[0].author.displayName,
      preview:  (.comments[0].content // "" | .[0:120])
    } ]'
```

> Use `--api-version "7.1"` (quoted, no `-preview.1` suffix)  see "When things break" below.

---

## Post a comment

### Top-level (overview) comment

```bash
TMP=$(mktemp -t pr-comment.XXXXXX.json)
trap 'rm -f "$TMP"' EXIT

jq -n --arg content "Comment text here" '{
  comments: [{ parentCommentId: 0, content: $content, commentType: 1 }],
  status: 1
}' > "$TMP"

resp=$(az devops invoke \
  --area git --resource pullRequestThreads \
  --route-parameters project=__ADO_PROJECT__ repositoryId=__ADO_REPO__ pullRequestId=$PR \
  --http-method POST \
  --in-file "$TMP" \
  --org "__ORG_URL__" \
  --api-version "7.1" \
  --output json)

if [ $' -eq 0 ] && echo "$resp" | jq -e '.id' >/dev/null; then
  thread_id=$(echo "$resp" | jq -r '.id')
  echo "Posted thread $thread_id"
else
  echo "POST failed; do NOT retry without re-listing threads"
fi
```

### Inline comment (anchored to a file/line)

```bash
FILE="/src/auth/login.ts"   # MUST start with /
LINE=42

jq -n \
  --arg content "Inline comment text" \
  --arg file    "$FILE" \
  --argjson line $LINE \
  '{
    comments: [{ parentCommentId: 0, content: $content, commentType: 1 }],
    status: 1,
    threadContext: {
      filePath: $file,
      rightFileStart: { line: $line, offset: 1 },
      rightFileEnd:   { line: $line, offset: 1 }
    }
  }' > "$TMP"

az devops invoke \
  --area git --resource pullRequestThreads \
  --route-parameters project=__ADO_PROJECT__ repositoryId=__ADO_REPO__ pullRequestId=$PR \
  --http-method POST \
  --in-file "$TMP" \
  --org "__ORG_URL__" \
  --api-version "7.1" \
  --output json
```

For a multi-line range, set `rightFileEnd.line` to the last line.

### Multi-line comment bodies

`jq -n --arg content "..."` handles newlines and quotes safely  pass a heredoc:

```bash
content=$(cat <<'EOF'
Title here

Detail line 1.
Detail line 2.

Suggestion:
- Use X instead of Y.
EOF
)
jq -n --arg content "$content" '{
  comments: [{ parentCommentId: 0, content: $content, commentType: 1 }],
  status: 1
}' > "$TMP"
```

---

## Vote / approve / reject (do not use during review)

The PR Review agent does not change PR state. Listed here only for completeness if the user explicitly asks:

```bash
az repos pr set-vote --id $PR --org "__ORG_URL__" \
  --vote approve|approve-with-suggestions|reject|wait-for-author|reset
```

---

## Idempotency cheat sheet

After a successful POST (`$' -eq 0` and the response has an `id`):
- Append to in-session `posted_findings`: `{ file, line, thread_id, content_hash }`.
- **Do not retry**, even if the user asks. Offer the thread URL instead:
  `__ORG_URL____ADO_PROJECT__/_git/__ADO_REPO__/pullrequest/$PR'_a=overview&discussionId=<thread_id>`

After a failed POST:
- Re-run the threads listing. If the comment is now present, mark it posted.
- If still absent, retry once.

---

## When things break

| Symptom | Cause | Fix |
|---|---|---|
| `could not convert string to float: '7.1.1'` | `--api-version 7.1-preview.1` parsed as float | Use `--api-version "7.1"` (quoted, no preview) |
| `The requested resource does not support http method 'POST'` | api-version too old | Use `7.1` or newer |
| `TF401019: ... does not exist or you do not have permissions` | Wrong route params or no access | Check `project=` / `repositoryId=`; verify `az account show` |
| `InteractiveBrowserCredential authentication failed` | Token expired | `az login` |
| `'az' is not recognized` (rare on macOS/Linux) | PATH issue | Reinstall Azure CLI; check `command -v az` |
