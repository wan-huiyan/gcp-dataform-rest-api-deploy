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
author: Claude Code
version: 1.0.0
date: 2026-03-23
---

# GCP Dataform REST API Deploy

## Problem
Deploying .sqlx files to Dataform programmatically (from scripts, CI/CD, or
automation) without the Dataform CLI. The REST API has non-obvious endpoint
patterns and a specific workflow order.

## Context / Trigger Conditions
- Need to deploy Dataform SQL files from a script or automation pipeline
- Want to trigger specific Dataform targets after deployment
- Cloud Workflows needs to invoke Dataform compilations and invocations
- Error: 404 on Dataform API calls (URL formatting issue with colon actions)
- Error: "Unknown name" in commit payload (wrong field names)

## Solution

### Full Deployment Lifecycle

```bash
ACCESS_TOKEN=$(gcloud auth print-access-token)
BASE="https://dataform.googleapis.com/v1beta1"
PROJECT_ID="my-project"
REGION="europe-north1"  # Dataform region (often different from Cloud Run region)
REPO="my-repo"
WORKSPACE="my-workspace"
WS_PATH="projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/workspaces/$WORKSPACE"

# Step 1: Write file to workspace (content must be base64-encoded)
ENCODED=$(base64 -i path/to/file.sqlx)
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"path\": \"definitions/my_model.sqlx\", \"contents\": \"$ENCODED\"}" \
  "${BASE}/${WS_PATH}:writeFile"

# Step 2: Commit changes in the workspace
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "author": {"name": "Author", "emailAddress": "author@example.com"},
    "commitMessage": "feat: add new model",
    "paths": ["definitions/my_model.sqlx"]
  }' \
  "${BASE}/${WS_PATH}:commit"

# Step 3: Push workspace to default branch
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  "${BASE}/${WS_PATH}:push"

# Step 4: Compile from release config
COMPILE_RESULT=$(curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"releaseConfig\": \"projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/releaseConfigs/my_release\"}" \
  "${BASE}/projects/$PROJECT_ID/locations/$REGION/repositories/$REPO/compilationResults")
COMPILATION_NAME=$(echo $COMPILE_RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['name'])")

# Step 5: Invoke specific targets
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

# Step 6: Poll for completion
while true; do
  sleep 15
  STATE=$(curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
    "${BASE}/$INVOCATION_NAME" | python3 -c "import sys,json; print(json.load(sys.stdin).get('state','RUNNING'))")
  echo "State: $STATE"
  [[ "$STATE" == "SUCCEEDED" || "$STATE" == "FAILED" ]] && break
done
```

### Checking Failure Details

```bash
# Get detailed action-level status (shows SQL errors)
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

## Verification
- Empty `{}` response from writeFile, commit, push = success
- compilationResults response contains `name` field
- workflowInvocations response contains `name` field
- Poll returns `state: SUCCEEDED`
- Query target table in BQ to verify data

## Notes

### Critical Gotchas
1. **Colon-prefixed actions**: Endpoints use `:writeFile`, `:commit`, `:push` — the
   colon must be part of the URL path, not the workspace name. URL-encoding can break this.
2. **Base64 encoding**: `writeFile` requires `contents` to be base64-encoded, not raw text.
3. **Commit field names**: The commit endpoint uses `paths` (array of strings), NOT
   `fileOperations`. The error "Unknown name fileOperations" means wrong field.
4. **Dataform region**: Often different from Cloud Run region (e.g., `europe-north1` vs
   `us-central1`). Check your Dataform repo settings.
5. **Release config**: Compilation requires a release config reference. The full resource
   path format is `projects/{project}/locations/{region}/repositories/{repo}/releaseConfigs/{config}`.
6. **Success responses**: Successful writeFile/commit/push return empty `{}` — not an error.

### From Cloud Workflows
The same pattern works in YAML workflows using `http.post` with `auth: {type: OAuth2}`.
See the `dataform_wait_loop` subworkflow pattern for polling.
