---
name: gcp-dataform-rest-api-deploy
description: |
  Deploy .sqlx files to Google Cloud Dataform repositories via REST API without
  the Dataform CLI. Use when: (1) deploying Dataform SQL from CI/CD or scripts,
  (2) programmatically updating Dataform workspaces, (3) triggering Dataform
  invocations from Cloud Workflows or automation, (4) adding NEW .sqlx files
  to a Dataform repo that also hosts other production release configs (v1.1.0:
  requires pre-merge workspace compile + post-merge cross-check gating to
  avoid breaking sibling release configs), (5) hitting "Only a commitish value
  of main is allowed" error when trying to compile a dev branch or non-main
  gitCommitish, (6) needing to discover exact target names with `_loader`
  suffix for workflowInvocations includedTargets, (7) renaming a .sqlx file
  whose `name:` (target name) is unchanged — both old and new produce the
  same target so they can't coexist (v1.2.0: atomic write+remove in one
  workspace commit + Gate-3 post-push compile against EVERY release config,
  with rollback recipe). Covers the full lifecycle:
  writeFile → commit → push → compile → invoke → poll for completion.
  Includes gotchas around base64 encoding, workspace path format, the
  colon-prefixed action endpoints (:writeFile, :commit, :push), the
  compile-main-only org policy workaround via workspace-level compile, the
  workspace field relative-path-only requirement, and target name discovery
  via compilationResults:query. (8) resolving a nightly-drift-check-filed
  GitHub issue (e.g. ADR-0016 style: workflow that opens/closes a
  `dataform-drift` issue) and post-sync re-verification surfaces a NEW
  `DF_AHEAD` file that was NOT in the original drift report (v1.4.0:
  pull-window race — your `:pull remoteBranch:main` reflects HEAD-at-pull,
  not the snapshot SHA cited in the issue, so Console-authored commits
  landing in the pull-window get absorbed; diagnostic via `:readFile` at
  the snapshot SHA distinguishes "pre-existing newer than snapshot"
  (file a follow-up, don't absorb) vs "drift-script bug" vs your sync
  introducing it).
author: Claude Code
version: 1.4.0
date: 2026-05-11
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
7. **`gcloud dataform ...` is NOT in the stock Cloud SDK**: `gcloud dataform repositories workspaces list` and every other `gcloud dataform` subcommand fails with `ERROR: (gcloud) Invalid choice: 'dataform'` — not even in `alpha` or `beta`. All Dataform operations MUST go through the REST API (`curl` + `$(gcloud auth print-access-token)`). Don't waste time trying to install a `dataform` component.
8. **Shell variable escape trap on colon actions**: `$CR:query` in a shell string is parsed as `${CR:query}` which is a bad substitution (or worse, silently drops the `:q`). ALWAYS use braces: `${CR}:query`. Symptom: curl returns 404 on a URL like `.../compilationResults/<uuid>uery` (the `:q` was eaten).
9. **Pull method is `:pull`, NOT `:pullGitCommits`**: A common mistake is to assume Dataform's pull method is `:pullGitCommits` (because the message field is `commitSha` etc.). It's not — that returns 404. The correct method is `:pull` with body `{remoteBranch, author}`.
10. **Dataform repo's git is SEPARATE from GitHub**: The `gitCommitish: main` in a release config refers to Dataform's INTERNAL main branch, not your GitHub main. Merging a PR on GitHub does NOT auto-update Dataform. You must propagate via workspace-based commit + push (see v1.1.0 section above). The two gits can have completely different SHAs. Verify which `resolvedGitCommitSha` is current via a fresh `POST /compilationResults` against the release config — if your PR content is not reflected, you still need to run the workspace-based push.

### From Cloud Workflows
The same pattern works in YAML workflows using `http.post` with `auth: {type: OAuth2}`.
See the `dataform_wait_loop` subworkflow pattern for polling.

---

## Advanced Defensive Patterns (v1.1.0 — added 2026-04-09)

When adding NEW `.sqlx` files to a Dataform repo that also hosts OTHER production
release configs (common in teams that manage multiple model versions like
`v9_per_term`, `v10_per_term`), a syntax error in the new file can break
compilation for every release config at the next daily invocation. The v1.0.0
happy-path deploy above doesn't protect against this. This section documents
the defensive gating required.

### Gotcha 7: Some repos enforce "Only a commitish value of main is allowed"

Some Dataform repos have an org-level policy that blocks `compilationResults`
against any gitCommitish other than `main`. Both of these attempts fail with
HTTP 400:

```bash
# Attempt A: create a dev release config pointing at a non-main branch
curl -X POST "${BASE}/${REPO_PATH}/releaseConfigs?releaseConfigId=v10_per_term_dev" \
  -d '{"gitCommitish":"dev/my-branch", "codeCompilationConfig":{}}'
# → Creates the release config successfully, BUT:

curl -X POST "${BASE}/${REPO_PATH}/compilationResults" \
  -d '{"releaseConfig":".../releaseConfigs/v10_per_term_dev"}'
# → HTTP 400: "Only a commitish value of 'main' is allowed."

# Attempt B: direct gitCommitish, bypassing the release config
curl -X POST "${BASE}/${REPO_PATH}/compilationResults" \
  -d '{"gitCommitish":"dev/my-branch", "codeCompilationConfig":{}}'
# → Same HTTP 400: "Only a commitish value of 'main' is allowed."
```

**The fallback: workspace-level compile as the pre-merge gate.** Workspace
compile compiles the workspace's in-memory file state, which does NOT involve
a git ref at all, so the main-only restriction doesn't apply:

```bash
curl -X POST "${BASE}/${REPO_PATH}/compilationResults" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"workspace": "projects/PROJECT/locations/REGION/repositories/REPO/workspaces/my-workspace"}'
```

### Gotcha 8: The `workspace` field needs the RELATIVE resource path, NOT the full URL

`compilationResults` returns HTTP 400 with "does not match projects/..." if you
pass the full HTTPS URL:

```bash
# WRONG — returns 400 INVALID_ARGUMENT
-d "{\"workspace\": \"${BASE}/${REPO_PATH}/workspaces/my-workspace\"}"
#                    ^^^^^^^^^^^^^^^^^^^^^^^ don't include the HTTPS URL

# RIGHT — use the relative resource path only
-d '{"workspace": "projects/PROJECT/locations/REGION/repositories/REPO/workspaces/my-workspace"}'
```

Same rule applies to `releaseConfig` fields in compile requests.

### The Three-Gate Defensive Deploy Pattern (for NEW FILE ADDITIONS)

When adding NEW `.sqlx` files (not in-place edits), use this defensive sequence
to protect sibling production release configs from compile errors:

```bash
#!/bin/bash
set -euo pipefail
export PATH="$HOME/google-cloud-sdk/bin:$PATH"
TOKEN=$(gcloud auth print-access-token)
BASE="https://dataform.googleapis.com/v1beta1"
REPO_PATH="projects/PROJECT/locations/REGION/repositories/REPO"

PROD_RELEASE="v9_per_term"      # EXISTING production release config
NEW_RELEASE="v10_per_term"      # NEW release config we're creating
WORKSPACE="dev-v10-new-files"
DEV_BRANCH="dev/v10-new-files"  # rollback snapshot (optional)

# --- Phase 1: Create workspace + pull main + write new files ---
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces?workspaceId=${WORKSPACE}" -d '{}'

curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces/${WORKSPACE}:pull" -d '{}'

for f in path/to/new/*.sqlx; do
  NAME=$(basename "$f")
  B64=$(base64 -i "$f" | tr -d '\n')
  curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "${BASE}/${REPO_PATH}/workspaces/${WORKSPACE}:writeFile" \
    -d "{\"path\":\"definitions/${NAME}\",\"contents\":\"${B64}\"}"
done

# --- GATE 1: Workspace compile (PRE-MERGE) ---
# Compiles every file in the workspace state (main files + new files)
# using codeCompilationConfig: {}. This is the only pre-merge gate on
# main-only-compile repos.
WS_COMPILE=$(curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/compilationResults" \
  -d "{\"workspace\": \"${REPO_PATH}/workspaces/${WORKSPACE}\"}")

ERRS=$(echo "$WS_COMPILE" | python3 -c "import json,sys;print(len(json.load(sys.stdin).get('compilationErrors',[])))")
if [[ "$ERRS" != "0" ]]; then
  echo "GATE 1 FAILED: $ERRS compile errors in workspace"
  echo "$WS_COMPILE" | python3 -c "import json,sys;[print(e.get('message','')[:200]) for e in json.load(sys.stdin).get('compilationErrors',[])]"
  exit 1
fi
echo "GATE 1 PASSED: workspace compile clean"

# --- Phase 2: Commit + push to dev branch (rollback snapshot) ---
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces/${WORKSPACE}:commit" \
  -d '{"author":{"name":"Author","emailAddress":"a@b.com"},"commitMessage":"feat: add new files"}'

curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces/${WORKSPACE}:push" \
  -d "{\"remoteBranch\":\"${DEV_BRANCH}\"}"

# --- Phase 3: Push workspace commits to main ---
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces/${WORKSPACE}:push" \
  -d '{"remoteBranch":"main"}'

# --- Phase 4: Create new release config pointing at main ---
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/releaseConfigs?releaseConfigId=${NEW_RELEASE}" \
  -d '{"gitCommitish":"main","codeCompilationConfig":{}}'

# --- GATE 2: Post-merge compile against new release config ---
NEW_COMPILE=$(curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/compilationResults" \
  -d "{\"releaseConfig\": \"${REPO_PATH}/releaseConfigs/${NEW_RELEASE}\"}")
ERRS=$(echo "$NEW_COMPILE" | python3 -c "import json,sys;print(len(json.load(sys.stdin).get('compilationErrors',[])))")
[[ "$ERRS" != "0" ]] && { echo "GATE 2 FAILED"; exit 1; }
echo "GATE 2 PASSED: ${NEW_RELEASE} compile clean"

# --- GATE 3: CRITICAL — cross-check the EXISTING production release config ---
# This catches the scenario where the new files broke compilation for OTHER
# release configs on the same repo. If this fails, main is broken for production.
PROD_COMPILE=$(curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/compilationResults" \
  -d "{\"releaseConfig\": \"${REPO_PATH}/releaseConfigs/${PROD_RELEASE}\"}")
ERRS=$(echo "$PROD_COMPILE" | python3 -c "import json,sys;print(len(json.load(sys.stdin).get('compilationErrors',[])))")
if [[ "$ERRS" != "0" ]]; then
  echo "GATE 3 FAILED: ${PROD_RELEASE} broke after adding new files!"
  echo "ROLLBACK: delete ${NEW_RELEASE}, use a workspace to remove the new files from main."
  exit 1
fi
echo "GATE 3 PASSED: ${PROD_RELEASE} still compiles clean — production safe"
```

**Why three gates instead of the happy-path two from v1.0.0?**

- **GATE 1** (workspace compile): catches syntax errors in the new files before
  they touch main
- **GATE 2** (new release config compile): confirms main is compilable via the
  new release config's lens
- **GATE 3** (production release config cross-check): **the one that actually
  matters** — confirms the NEW files didn't accidentally break the EXISTING
  production release configs. With empty `codeCompilationConfig: {}`, every
  release config compiles every file on main, so cross-version interference is
  possible via shared `includes/*.js`, shared lookup tables, or any other
  cross-reference

### Gotcha 9: `queryDirectoryContents` is GET, not POST

Most `workspaces/*:*` action endpoints are POST, but directory listing is GET
with a `path=` query parameter:

```bash
# WRONG — POST returns 404
curl -X POST "${BASE}/${WS_PATH}:queryDirectoryContents" -d '{"path":""}'

# RIGHT — GET with query string
curl "${BASE}/${WS_PATH}:queryDirectoryContents?path=definitions"
```

Useful to verify file structure before and after `writeFile` calls.

### Gotcha 10: Discover exact target names via `:query` on compilationResults

The `includedTargets` field in `workflowInvocations` requires EXACT target
names. The convention isn't always obvious:

- **`type: operations` sqlx files** (e.g., incremental loaders) usually have
  config `name: "my_table_loader"` — the target needs the `_loader` suffix
- **`type: table` / `view` sqlx files** usually have config `name: "my_table"`
  — no suffix
- If you drop the suffix incorrectly, you get:
  `"Requested target does not exist in compilation result: 'project.schema.my_table'"`

To discover the correct target names for a compilation result:

```bash
curl -sS "${BASE}/${REPO_PATH}/compilationResults/${COMPILE_ID}:query" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
for a in d.get('compilationResultActions', []):
    t = a.get('target', {})
    if t.get('schema') != 'dataform_assertions':  # skip generated assertions
        print(f\"{t.get('database')}.{t.get('schema')}.{t.get('name')}\")
"
```

Run this BEFORE building the `workflowInvocations` payload to verify names.
Copy the exact `name` field verbatim into the `includedTargets` array.

### Gotcha 11: `push` with `remoteBranch` creates NEW dev branches

The v1.0.0 skill shows `:push` with `-d '{}'` which pushes to the repo's
default branch. To push to a NEW remote branch (useful for rollback snapshots
or dev gates), add `remoteBranch`:

```bash
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${WS_PATH}:push" \
  -d '{"remoteBranch":"dev/my-feature-branch"}'
```

The branch is created on the Google-managed git if it doesn't exist. You can
push the same workspace's commits to multiple branches in sequence (e.g.,
first to `dev/v10` for the snapshot, then to `main` for the actual deploy).

Even if the repo enforces "only main commitish allowed" for compilation, you
CAN still push to non-main branches — the restriction is on compilation, not
on git operations. A dev branch is useful as a rollback target even if you
can't compile it directly.

---

## v1.2.0 — Atomic Target-Collision Rewrite (added 2026-04-27)

### Gotcha 12: Renaming a `.sqlx` file that produces the same target name will collide if old + new exist on main simultaneously

**Symptom:** You want to swap `v9_term_enrollment_predictions_enriched.sqlx` for
`v10_term_enrollment_predictions_enriched.sqlx`. Both files have
`name: "term_enrollment_predictions_enriched"` in their `config{}` block
(intentional — the target table is the same, the file rename is just a DAG-
metadata refresh). If both files exist in `definitions/` simultaneously,
**every release config compile fails** with a duplicate-target error.

**Naive sequence that breaks production:**
1. `:writeFile new.sqlx` (the v10 version)
2. `:commit` + `:push` — **at this point both v9 and v10 are on main; ALL release configs fail to compile until you also remove v9**
3. `:removeFile old.sqlx`
4. `:commit` + `:push` — fixed, but you've left main broken between steps 2 and 4. If the daily scoring window fires in between, the workflow fails.

### The Atomic Rewrite Pattern

Stage write + remove in the SAME workspace commit, then run Gate-3 against
EVERY release config (not just the one you're targeting) before push:

```bash
TOKEN=$(gcloud auth print-access-token)
BASE="https://dataform.googleapis.com/v1beta1"
REPO_PATH="projects/PROJECT/locations/REGION/repositories/REPO"
WS_PATH="${REPO_PATH}/workspaces/atomic-rewrite-$(date +%s)"

# Step 1: Create workspace + pull main
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces?workspaceId=$(basename $WS_PATH)" -d '{}'
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${WS_PATH}:pull" -d '{"author":{"name":"deploy","emailAddress":"deploy@example.com"},"remoteBranch":"main"}'

# Step 2: Write new + remove old
B64=$(base64 -i path/to/v10_new.sqlx | tr -d '\n')
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${WS_PATH}:writeFile" \
  -d "{\"path\":\"definitions/v10_new.sqlx\",\"contents\":\"${B64}\"}"
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${WS_PATH}:removeFile" \
  -d '{"path":"definitions/v9_old.sqlx"}'

# Step 3: Sanity check workspace state
curl -sS -H "Authorization: Bearer $TOKEN" \
  "${BASE}/${WS_PATH}:queryDirectoryContents?path=definitions" \
  | python3 -c "
import json, sys
d=json.load(sys.stdin)
files=sorted([e['file'] for e in d.get('directoryEntries',[]) if 'file' in e])
print('v9 absent:', not any('v9_old' in f for f in files))
print('v10 present:', any('v10_new' in f for f in files))
print('total:', len(files))"

# Step 4: GATE 1 — workspace compile (must be 0 errors)
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/compilationResults" \
  -d "{\"workspace\":\"${WS_PATH}\"}" \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
e=len(d.get('compilationErrors',[]))
print(f'GATE 1: {e} errors')
exit(1 if e else 0)"

# Step 5: Atomic commit + push (BOTH paths in one :commit)
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${WS_PATH}:commit" \
  -d '{
    "author":{"name":"deploy","emailAddress":"deploy@example.com"},
    "commitMessage":"atomic: replace v9_old with v10_new (same target name)",
    "paths":["definitions/v10_new.sqlx","definitions/v9_old.sqlx"]
  }'
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${WS_PATH}:push" -d '{"remoteBranch":"main"}'

# Step 6: GATE 3 — compile against EVERY release config in the repo
# CRITICAL: this is broader than the v1.1.0 three-gate pattern. Every release
# config compiles every .sqlx in definitions/, so any release config could
# surface cross-version interference. The 3-of-9 release configs in this
# project's repo with non-empty `codeCompilationConfig{tablePrefix}` are the
# ones MOST likely to surface issues, but check ALL.
RELEASE_CONFIGS=$(curl -sS -H "Authorization: Bearer $TOKEN" \
  "${BASE}/${REPO_PATH}/releaseConfigs" \
  | python3 -c "import json,sys; print(' '.join(rc['name'].split('/')[-1] for rc in json.load(sys.stdin).get('releaseConfigs',[])))")

for RC in $RELEASE_CONFIGS; do
  ERRS=$(curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "${BASE}/${REPO_PATH}/compilationResults" \
    -d "{\"releaseConfig\":\"${REPO_PATH}/releaseConfigs/${RC}\"}" \
    | python3 -c "import json,sys; print(len(json.load(sys.stdin).get('compilationErrors',[])))")
  echo "RC=$RC errors=$ERRS"
  if [[ "$ERRS" != "0" ]]; then
    echo "GATE 3 FAILED: $RC. ROLLBACK NOW — re-push a workspace that restores v9_old."
    exit 1
  fi
done
echo "GATE 3 PASSED: all $(echo $RELEASE_CONFIGS | wc -w) release configs compile clean"
```

### Why TWO gates instead of one

- **Gate 1 (workspace compile, pre-push)** catches syntax errors in the new
  file. Workspace compile uses in-memory file state, no git ref involved, so
  even on repos with "Only main commitish allowed" policy (Gotcha 7) this
  works.
- **Gate 3 (post-push compile against EVERY release config)** is the one that
  matters for production safety. Workspace compile uses default
  `codeCompilationConfig{}`. Each release config may apply its own
  `tablePrefix` or other config, which can surface schema collisions
  workspace compile missed. Cross-checking all release configs catches
  cross-version interference (e.g., the new file's `dependencies:` reference
  a target that DOES exist for v10_per_term but NOT for v8_per_term's
  prefixed schema).

### Rollback if Gate 3 fails

You're already past push, so main is in a broken state. Roll back by
re-pushing a workspace that restores the old file:

```bash
ROLLBACK_WS_PATH="${REPO_PATH}/workspaces/rollback-$(date +%s)"
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/workspaces?workspaceId=$(basename $ROLLBACK_WS_PATH)" -d '{}'
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${ROLLBACK_WS_PATH}:pull" -d '{"remoteBranch":"main"}'

# Re-write the OLD file (you should have its content saved before the rewrite)
B64_OLD=$(base64 -i /tmp/v9_old_backup.sqlx | tr -d '\n')
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${ROLLBACK_WS_PATH}:writeFile" \
  -d "{\"path\":\"definitions/v9_old.sqlx\",\"contents\":\"${B64_OLD}\"}"
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${ROLLBACK_WS_PATH}:removeFile" \
  -d '{"path":"definitions/v10_new.sqlx"}'
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${ROLLBACK_WS_PATH}:commit" \
  -d '{
    "author":{"name":"rollback","emailAddress":"rollback@example.com"},
    "commitMessage":"rollback: restore v9_old after Gate 3 failure",
    "paths":["definitions/v9_old.sqlx","definitions/v10_new.sqlx"]
  }'
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${ROLLBACK_WS_PATH}:push" -d '{"remoteBranch":"main"}'
```

**Pre-flight backup:** Before starting the atomic rewrite, save the old
file's content locally:

```bash
SHA=$(curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "${BASE}/${REPO_PATH}/compilationResults" \
  -d "{\"releaseConfig\":\"${REPO_PATH}/releaseConfigs/v10_per_term\"}" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['resolvedGitCommitSha'])")
curl -sS -H "Authorization: Bearer $TOKEN" \
  "${BASE}/${REPO_PATH}:readFile?commitSha=${SHA}&path=definitions/v9_old.sqlx" \
  | python3 -c "import json,sys,base64; print(base64.b64decode(json.load(sys.stdin)['contents']).decode())" \
  > /tmp/v9_old_backup.sqlx
```

### Verified examples
- 2026-04-27 (S107b on <DATA-PROJECT> / <PROPENSITY-DATASET>):
  v9_term_enrollment_predictions_enriched.sqlx → v10_term_enrollment_predictions_enriched.sqlx,
  9 release configs (3 with non-empty `tablePrefix`), Gate 1 + Gate 3 clean,
  daily workflow SUCCEEDED in 7 min, BQ verify 11,079 rows / 0 dups.

---

## v1.3.0 — Gotcha 13: `dependencies:` arrays use the target's `name:` field, not the source filename (added 2026-05-10)

### Gotcha 13: A `dependencies:` array entry that names the source filename instead of the target's `name:` field fails Gate 1 with "Action depends on X which does not exist"

**Symptom:** A new `.sqlx` file declares `dependencies: ["v10_my_table"]` (matching
the source basename `v10_my_table.sqlx`). Workspace compile (Gate 1) returns
HTTP 200 but `compilationErrors[].message`:

```
Missing dependency detected: Action "<DATA-PROJECT>.schema.my_new_table" depends on
"{"name":"v10_my_table","includeDependentAssertions":false}" which does not exist
```

**Root cause:** Dataform resolves `dependencies` against each target sqlx's
`config.name` field — NOT the source filename. A common drift pattern:

```sqlx
// File: definitions/v10_my_table.sqlx     ← source basename has v10_ prefix
config {
  name: "my_table",                          // ← target name has NO v10_ prefix
  schema: "ml_features",
  type: "table"
}
```

A consumer that declares `dependencies: ["v10_my_table"]` (the filename)
will never resolve. The fix is `dependencies: ["my_table"]` (the `name:` value).

### Diagnostic recipe

When Gate 1 surfaces "Action depends on X which does not exist":

1. **Find the target sqlx** that produces the X resource:
   ```bash
   grep -ln "name:.*\"X\"" definitions/
   ```
2. **Note the source filename and the `name:` value separately:**
   ```bash
   awk '/^config/,/^}/' definitions/v10_*.sqlx | grep -E "^  (name|schema):"
   ```
3. **Pin the dependency reference** in the new file to match the `name:` value, NOT the filename.

### Why this is a project-specific drift

Some projects keep filename and target name aligned (`v10_foo.sqlx` →
`name: "v10_foo"`). Others rename targets across versions while keeping a
neutral table name (`v10_foo.sqlx` → `name: "foo"`) so consumers don't have
to update their references each version cut. Both conventions are valid;
the gotcha bites when adding NEW files into the second convention.

### Pre-flight check

Before declaring `dependencies` for a new sqlx, run this lookup once:

```bash
# For each dependency you want to declare, find its target name
for DEP_FILENAME in v10_term_enrollment_predictions_enriched; do
  DEP_NAME=$(awk '/^config/,/^}/' "definitions/${DEP_FILENAME}.sqlx" | \
    grep -E '^  name:' | sed -E 's/.*name: *"([^"]+)".*/\1/')
  echo "$DEP_FILENAME → $DEP_NAME"
done
```

Use `$DEP_NAME` (not `$DEP_FILENAME`) in the `dependencies` array.

### Verified example
- 2026-05-10 (S166 on <DATA-PROJECT> / <PROPENSITY-DATASET>):
  new `v10_term_enrollment_realised_outcomes.sqlx` declared
  `dependencies: ["v10_term_enrollment_predictions_enriched"]` (filename);
  Gate 1 failed with the above error. Fix: change to
  `dependencies: ["term_enrollment_predictions_enriched"]` (the target's
  `name:` value, no `v10_` prefix). Required a 1-line hotfix PR (#679 in
  the project) since the original sqlx had already merged.

## v1.4.0 — Gotcha 14: Drift-check-driven sync `:pull` reflects HEAD-at-pull, not the drift-check snapshot SHA (added 2026-05-11)

### Gotcha 14: When resolving a nightly-drift-check-filed issue, expect post-sync verification to surface NEW drifts you didn't introduce

**Symptom:** A scheduled drift-check workflow (per ADR-0016 style: opens/closes a
`dataform-drift` GitHub issue on state change) files an issue listing N specific
drifted files at a specific snapshot SHA (e.g. `Dataform main SHA: 1651ece...`).
You run the standard GH→DF sync per this skill — `:pull remoteBranch:main` →
`:writeFile` the N files → Gate 1 → commit → `:push` → Gate 3 — and all gates
pass clean. Then you re-run the drift script and a DIFFERENT file appears as
`DF_AHEAD` (present in Dataform main, missing from GitHub) that was NOT in the
original issue.

**Root cause:** Your `:pull remoteBranch:main` returns whatever is HEAD-of-
Dataform-main at the moment of the pull — NOT the snapshot SHA cited in the
issue. Any Console-authored commits that landed between the nightly drift-check
snapshot and your pull-time get silently absorbed into your workspace state.
When you `:push` back to main, those Console-authored files are still there
(unchanged), but they ARE genuinely DF-ahead of GitHub.

It looks like your sync introduced them. It didn't. They were pre-existing in
the pull-window.

### Diagnostic: did my sync introduce this drift, or was it already there?

Read the surprise file from Dataform main at the SNAPSHOT SHA (from the original
drift issue body), NOT at current HEAD:

```bash
TOKEN=$(gcloud auth print-access-token)
BASE="https://dataform.googleapis.com/v1beta1"
REPO_PATH="projects/PROJECT/locations/REGION/repositories/REPO"
SNAPSHOT_SHA="<SHA from the drift issue body>"     # e.g. 1651ece78358931a...
SURPRISE_FILE="<DF_AHEAD filename from post-sync verification>"

curl -sS -H "Authorization: Bearer $TOKEN" \
  "${BASE}/${REPO_PATH}:readFile?commitSha=${SNAPSHOT_SHA}&path=definitions/${SURPRISE_FILE}"
```

Interpretation:
- **`404 NOT_FOUND`** → file landed in the pull-window. NOT your fault. Pre-
  existing Console work newer than the snapshot. File a follow-up issue.
- **File exists at snapshot but differs from current** → drift script bug
  (allow-list mis-config, byte-comparison edge case, etc.). Investigate the
  drift script.
- **File exists at snapshot AND matches current** → impossible by definition,
  but if you see this, your re-run is reading wrong data — re-verify with
  the actual SHAs you pushed.

### Resolution path

Resist the temptation to absorb the surprise drift into the current sync:

1. **Close the original drift issue** with a comment confirming the originally-
   reported N items are now `MATCH` and explicitly noting the post-sync-
   surfaced item as out-of-scope.
2. **File a follow-up issue** for the surprise file with: (a) the pull-time
   Dataform main SHA, (b) the `:readFile` content recipe so the next operator
   can fetch the file, (c) recommended action (import to GitHub per ADR-0016,
   or allow-list per `dataform_drift_github_only_allowlist.txt` if baker-owned).
3. **Do NOT add it to the current workspace's commit.** Even if it would be
   "one more file in the same push," conflating two issues complicates the
   audit trail and makes the next nightly drift report harder to interpret.

### Why this matters

A drift-check workflow that opens/closes a single labelled issue (e.g. ADR-0016
style) will reopen tomorrow when the nightly run sees the residual DF_AHEAD.
Filing the follow-up explicitly today gives the next operator the context they
need; relying on the workflow alone produces a fresh issue with NO context
about the prior sync that surfaced it.

### Verified example
- 2026-05-11 (issue #694 on <PROPENSITY-REPO>): drift-check filed
  3 GH→DF drifts at snapshot `1651ece...`. Standard 3-gate sync resolved all
  3 clean (Dataform main advanced to `e1cc96f...`). Post-sync drift script
  surfaced `DF_AHEAD student_identity_index.sqlx`. `:readFile` at
  `commitSha=1651ece...` returned `404 NOT_FOUND` → confirmed Console-authored
  in the pull-window for issue #715 (sitewide student lookup). Closed #694
  with clean resolution note + filed follow-up #722 for the GitHub import.

## References
- [Dataform REST API reference — compilationResults.create](https://cloud.google.com/dataform/reference/rest/v1beta1/projects.locations.repositories.compilationResults/create)
- [Dataform workspaces API reference](https://cloud.google.com/dataform/reference/rest/v1beta1/projects.locations.repositories.workspaces)
- Related skill: `gcp-cloudrun-job-args-override` — different gcloud Cloud Run gotcha
- Related skill: `gcp-cloudrun-job-polling-url` — polling Cloud Run jobs from workflows
