# GCP Dataform REST API Deploy

A Claude Code skill for deploying `.sqlx` files to Google Cloud Dataform repositories via REST API — without the Dataform CLI.

## Quick Start

```
You: Deploy this new .sqlx model to Dataform and run it
Claude: [writes file to workspace → commits → pushes → compiles → invokes → polls until SUCCEEDED]
```

Or use the documented bash patterns directly in CI/CD pipelines, Cloud Workflows, or deployment scripts.

## Installation

**Claude Code:**
```bash
# Git clone
git clone https://github.com/wan-huiyan/gcp-dataform-rest-api-deploy.git ~/.claude/skills/gcp-dataform-rest-api-deploy
```

**Cursor (2.4+):**
```bash
# Per-project rule (most reliable)
mkdir -p .cursor/rules
# Copy SKILL.md content into .cursor/rules/gcp-dataform-rest-api-deploy.mdc with alwaysApply: true

# Manual global install
git clone https://github.com/wan-huiyan/gcp-dataform-rest-api-deploy.git ~/.cursor/skills/gcp-dataform-rest-api-deploy
```

## What You Get

- **Complete 6-step lifecycle**: writeFile → commit → push → compile → invoke → poll
- **Failure diagnostics**: Action-level error extraction showing exact SQL errors
- **6 documented gotchas** that cause silent failures or cryptic errors
- **Cloud Workflows integration** notes for YAML-based orchestration

## The Problem

The Dataform REST API (`v1beta1`) has non-obvious patterns that cause frustrating failures:

| What Goes Wrong | Symptom | Root Cause |
|----------------|---------|------------|
| 404 on API calls | `Not Found` on valid workspace | Colon-prefixed actions (`:writeFile`) get URL-encoded or appended wrong |
| Commit fails | `Unknown name "fileOperations"` | Wrong field name — API uses `paths`, not `fileOperations` |
| File write seems to fail | Empty `{}` response | That's actually success — empty response = OK |
| Invocation runs wrong SQL | Old version executes | Forgot to push workspace before compiling |
| Region mismatch | Various auth/not-found errors | Dataform region often differs from Cloud Run region |
| Date parsing errors | `No matching signature for TIMESTAMP_SECONDS` | Column types in BQ don't match what you'd expect from field names |

## How It Works

| Step | API Endpoint | What It Does |
|------|-------------|--------------|
| 1. Write | `{workspace}:writeFile` | Upload base64-encoded `.sqlx` to workspace |
| 2. Commit | `{workspace}:commit` | Commit with author, message, and file paths |
| 3. Push | `{workspace}:push` | Push workspace changes to default branch |
| 4. Compile | `{repo}/compilationResults` | Compile from release config |
| 5. Invoke | `{repo}/workflowInvocations` | Run specific targets |
| 6. Poll | `GET {invocation}` | Poll `state` until `SUCCEEDED` or `FAILED` |

## Key Gotchas

### 1. Colon-Prefixed Action Endpoints
```
# WRONG — colon gets eaten or URL-encoded
${BASE}/${WS_PATH}writeFile

# RIGHT — colon is part of the path
${BASE}/${WS_PATH}:writeFile
```

### 2. Base64 Encoding Required
`writeFile` expects `contents` as base64, not raw text:
```bash
ENCODED=$(base64 -i path/to/file.sqlx)
```

### 3. Commit Uses `paths`, Not `fileOperations`
```json
// WRONG
{"fileOperations": {"definitions/model.sqlx": {"writeFile": {}}}}

// RIGHT
{"paths": ["definitions/model.sqlx"], "author": {...}, "commitMessage": "..."}
```

### 4. Empty `{}` = Success
`writeFile`, `commit`, and `push` all return empty `{}` on success. Don't treat it as an error.

### 5. Dataform Region != Cloud Run Region
Dataform repos are often in `europe-north1` while Cloud Run services are in `us-central1`. Always check your Dataform repo settings.

### 6. Release Config Full Path Required
```
projects/{project}/locations/{region}/repositories/{repo}/releaseConfigs/{config}
```

## Checking Failure Details

When an invocation fails, get action-level errors:
```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "${BASE}/${INVOCATION_NAME}:query" | python3 -c "
import sys, json
for action in json.load(sys.stdin).get('workflowInvocationActions', []):
    print(f\"{action['target']['name']}: {action['state']}\")
    if action.get('failureReason'):
        print(f\"  Error: {action['failureReason']}\")
"
```

## Limitations

- Covers `v1beta1` API only — Google may change endpoints when GA
- Polling is simple sleep-based; no exponential backoff
- Does not handle Dataform workspace creation (assumes workspace exists)
- Does not cover Dataform assertion verification post-invocation
- Bash examples only — no Python SDK wrapper (yet)

## Dependencies

| Dependency | Required | Purpose |
|-----------|----------|---------|
| `gcloud` CLI | Yes | Authentication (`gcloud auth print-access-token`) |
| `curl` | Yes | HTTP requests to Dataform API |
| `base64` | Yes | Encoding file contents for writeFile |
| `python3` | Yes | JSON parsing of API responses |
| GCP project with Dataform | Yes | Target repository and workspace |

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-03-23 | Initial release — full lifecycle, 6 gotchas, failure diagnostics |

## License

MIT

---

*Built with [Claude Code](https://claude.ai/claude-code)*
