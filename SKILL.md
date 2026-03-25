---
name: gcp-dataform-rest-api-deploy
description: |
  Deploy .sqlx files to Google Cloud Dataform repositories via REST API without
  the Dataform CLI. Use when: (1) deploying Dataform SQL from CI/CD or scripts,
  (2) programmatically updating Dataform workspaces, (3) triggering Dataform
  invocations from Cloud Workflows or automation. Covers the full lifecycle:
  writeFile → commit → push → compile → invoke → poll for completion.
  Includes gotchas around base64 encoding, workspace path format, and the
  colon-prefixed action endpoints (:writeFile, :commit, :push).
  NOT for: Dataform CLI usage, BigQuery direct SQL, Dataform web UI operations,
  or non-GCP data pipeline tools (dbt, Airflow).
author: Claude Code
version: 1.1.0
date: 2026-03-25
triggers:
  positive:
    - "deploy sqlx files to dataform"
    - "dataform REST API"
    - "dataform API writeFile"
    - "dataform API commit push"
    - "programmatically deploy to dataform"
    - "dataform CI/CD deploy"
    - "cloud workflows dataform"
    - "dataform 404 colon endpoint"
    - "dataform unknown name fileOperations"
    - "dataform compilation invocation API"
    - "curl dataform API"
    - "automate dataform deployment"
    - "dataform workflowInvocations"
    - "dataform compilationResults API"
    - "dataform base64 writeFile"
  negative:
    - "dataform CLI install"
    - "dbt deploy"
    - "bigquery SQL query"
    - "dataform web UI"
    - "airflow DAG"
    - "dataform init project"
composability:
  consumes_from: ["gcloud-auth", "base64-encoding"]
  hands_off_to: ["bigquery-query-validation", "cloud-workflows-orchestration"]
  output_contract: |
    On success: workflow invocation name (resource path string) + SUCCEEDED state.
    On failure: FAILED state + action-level error details from :query endpoint.
  error_behavior: |
    - 404 → check colon-prefix on action endpoints (:writeFile, :commit, :push)
    - "Unknown name fileOperations" → use `paths` field instead
    - Empty {} response → success (not error) for writeFile/commit/push
    - FAILED state → query :query endpoint for per-action failure reasons
  idempotency: |
    writeFile is idempotent (overwrites). commit+push are NOT idempotent —
    re-committing unchanged files returns an error. Guard with a diff check.
---

# GCP Dataform REST API Deploy

## When to Use This Skill

| Scenario | Use this? | Why |
|---|---|---|
| Deploy .sqlx from CI/CD pipeline | Yes | REST API is the only option without CLI |
| Trigger Dataform from Cloud Workflows | Yes | HTTP calls with OAuth2 auth |
| Update workspace files programmatically | Yes | writeFile endpoint |
| Use Dataform CLI locally | No | CLI handles this natively |
| Write BigQuery SQL directly | No | Different API entirely |
| Manage Dataform via web console | No | No API needed |

## Deployment Lifecycle (6 Steps)

```bash
ACCESS_TOKEN=$(gcloud auth print-access-token)
BASE="https://dataform.googleapis.com/v1beta1"
PROJECT_ID="my-project"
REGION="europe-north1"  # Often different from Cloud Run region — check repo settings
REPO="my-repo"
WORKSPACE="my-workspace"
WS_PATH="projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/workspaces/$WORKSPACE"

# 1. Write file (content MUST be base64-encoded)
ENCODED=$(base64 -i path/to/file.sqlx)
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"path\": \"definitions/my_model.sqlx\", \"contents\": \"$ENCODED\"}" \
  "${BASE}/${WS_PATH}:writeFile"

# 2. Commit (use `paths`, NOT `fileOperations`)
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "author": {"name": "Author", "emailAddress": "author@example.com"},
    "commitMessage": "feat: add new model",
    "paths": ["definitions/my_model.sqlx"]
  }' \
  "${BASE}/${WS_PATH}:commit"

# 3. Push to default branch
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  "${BASE}/${WS_PATH}:push"

# 4. Compile from release config
COMPILE_RESULT=$(curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"releaseConfig\": \"projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/releaseConfigs/my_release\"}" \
  "${BASE}/projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/compilationResults")
COMPILATION_NAME=$(echo $COMPILE_RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['name'])")

# 5. Invoke specific targets
INVOKE_RESULT=$(curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"compilationResult\": \"$COMPILATION_NAME\",
    \"invocationConfig\": {
      \"includedTargets\": [
        {\"database\": \"my-project\", \"schema\": \"my_schema\", \"name\": \"my_model\"}
      ],
      \"transitiveDependenciesIncluded\": false
    }
  }" \
  "${BASE}/projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/workflowInvocations")
INVOCATION_NAME=$(echo $INVOKE_RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['name'])")

# 6. Poll for completion
while true; do
  sleep 15
  STATE=$(curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
    "${BASE}/$INVOCATION_NAME" | python3 -c "import sys,json; print(json.load(sys.stdin).get('state','RUNNING'))")
  echo "State: $STATE"
  [[ "$STATE" == "SUCCEEDED" || "$STATE" == "FAILED" ]] && break
done
```

## Debugging Failures

```bash
# Get per-action status with SQL error details
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "${BASE}/${INVOCATION_NAME}:query" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for action in data.get('workflowInvocationActions', []):
    target = action.get('target', {}).get('name', 'unknown')
    state = action.get('state', 'UNKNOWN')
    failure = action.get('failureReason', '')
    print(f'{target}: {state}')
    if failure:
        print(f'  Error: {failure}')
"
```

## Verification Checklist

| Step | Success Signal |
|---|---|
| writeFile | Empty `{}` response |
| commit | Empty `{}` response |
| push | Empty `{}` response |
| compilationResults | Response contains `name` field |
| workflowInvocations | Response contains `name` field |
| Poll | `state: SUCCEEDED` |
| End-to-end | Query target table in BigQuery |

## Gotcha Quick Reference

| Gotcha | Wrong | Right |
|---|---|---|
| Action endpoints | `workspace/writeFile` | `workspace:writeFile` (colon prefix) |
| File contents | Raw text in `contents` | Base64-encoded string |
| Commit field | `fileOperations: [...]` | `paths: ["file.sqlx"]` |
| Region | Using Cloud Run region | Dataform repo region (check settings) |
| Empty response | Treating `{}` as error | `{}` = success for write/commit/push |
| Release config path | Short name | Full resource path: `projects/{p}/locations/{r}/repositories/{repo}/releaseConfigs/{cfg}` |

## Prerequisites and Compatibility

Requires: `gcloud` CLI (for auth tokens), `bash`, `python3` (for JSON parsing), and `curl`.
Works with Dataform API v1beta1. Compatible with gcloud v400+.
Scoped to the `dataform.googleapis.com` REST surface only.
Expects `.sqlx` files as input and requires a valid GCP project ID, region, and Dataform repository name.

## Error Handling

If the writeFile call fails with 404, check the colon-prefix on action endpoints (e.g., `:writeFile` not `/writeFile`), since URL-encoding can break this.
If the commit fails with "Unknown name", use the `paths` field instead of `fileOperations`.
When the invocation fails with state `FAILED`, query the `:query` endpoint for per-action failure reasons.
On error from any step, verify the region matches your Dataform repo settings, because the Dataform region is often different from Cloud Run region.

## Handoff and Scope Boundaries

After deployment succeeds, then use BigQuery to validate the output tables.
For Dataform CLI operations, use the CLI directly instead of this REST workflow.
If you need to orchestrate multiple Dataform repos, for that use Cloud Workflows instead.

## Idempotency

`writeFile` is idempotent — safe to re-run with the same content.
`commit` + `push` are NOT idempotent — running again on unchanged files returns an error. Guard with a diff check before committing.

## Example 1: Deploy a Single Model

Deploy `definitions/staging/stg_orders.sqlx` to the `analytics` workspace:

1. Run `base64 -i definitions/staging/stg_orders.sqlx` to encode the file
2. Call `:writeFile` with path `definitions/staging/stg_orders.sqlx`
3. Call `:commit` with `paths: ["definitions/staging/stg_orders.sqlx"]`
4. Call `:push` to sync to the default branch
5. Create a compilation result referencing your release config
6. Invoke with `includedTargets` for `stg_orders`
7. Verify the invocation reaches `SUCCEEDED` state

## Cloud Workflows Integration

Use `http.post` with `auth: {type: OAuth2}` for the same lifecycle. Use a `dataform_wait_loop` subworkflow for polling with `sys.sleep`.
