---
name: databricks-api-calls
description: >
  Use this skill to make authenticated Databricks REST API calls, especially
  Workspace APIs. Trigger phrases include: "call Databricks API", "Databricks
  REST request", "list Databricks workspace objects", "create a Databricks
  workspace folder", "import notebook via API", "export notebook", "Databricks
  endpoint call", "Databricks API with curl".
version: 1.0.0
argument-hint: "[GET|POST|PUT|PATCH|DELETE] [/api/2.0/<resource>] [json-body]"
allowed-tools: [Bash]
---

# Databricks API Calls Skill

## Overview

This skill executes authenticated Databricks REST API requests using `curl`.
It is designed for Workspace APIs and general Databricks REST endpoints.

Authentication and host details must be loaded from the project's `.env` file
at runtime. Never hard-code or persist secret tokens in this skill.

Reference: https://docs.databricks.com/api/workspace/introduction

---

## Environment Variables

Load from `.env` each time before making a call.

Required:
- `DATABRICKS_WORKSPACE_URL` (example: `https://adb-xxxx.azuredatabricks.net`)

Token (at least one must exist in `.env`):
- `DATABRICKS_TOKEN`
- `DATABRICKS_PAT`
- `DATABRICKS_API_TOKEN`

Project convention supported:
- `DATABRICKS_API_TOKEN=${ANTHROPIC_AUTH_TOKEN}` in `.env`

Token precedence when multiple are present:
1. `DATABRICKS_TOKEN`
2. `DATABRICKS_PAT`
3. `DATABRICKS_API_TOKEN`
4. `ANTHROPIC_AUTH_TOKEN` (fallback)

If required variables are missing, stop and report which variable is absent.

---

## Instructions

### 1. Load `.env` before every API call

Run from repository root (where `.env` exists):

```bash
set -a
source ./.env
set +a
```

Then resolve token without printing it:

```bash
DBX_TOKEN="${DATABRICKS_TOKEN:-${DATABRICKS_PAT:-${DATABRICKS_API_TOKEN:-${ANTHROPIC_AUTH_TOKEN:-}}}}"
: "${DATABRICKS_WORKSPACE_URL:?DATABRICKS_WORKSPACE_URL is not set in .env}"
: "${DBX_TOKEN:?Set DATABRICKS_TOKEN, DATABRICKS_PAT, DATABRICKS_API_TOKEN, or ANTHROPIC_AUTH_TOKEN in .env}"
```

### 2. Base request template

```bash
curl --silent --show-error --fail-with-body \
  --request <METHOD> \
  --url "${DATABRICKS_WORKSPACE_URL}<ENDPOINT>" \
  --header "Authorization: Bearer ${DBX_TOKEN}" \
  --header "Content-Type: application/json" \
  --header "Accept: application/json" \
  [--data '<JSON_BODY>']
```

For readable output, prefer:

```bash
... | jq .
```

If `jq` is unavailable, use:

```bash
... | python3 -m json.tool
```

### 3. Workspace API examples

List objects in a workspace path:

```bash
curl --silent --show-error --fail-with-body \
  --request GET \
  --url "${DATABRICKS_WORKSPACE_URL}/api/2.0/workspace/list?path=/" \
  --header "Authorization: Bearer ${DBX_TOKEN}" \
  --header "Accept: application/json" \
| jq .
```

Create a workspace directory:

```bash
curl --silent --show-error --fail-with-body \
  --request POST \
  --url "${DATABRICKS_WORKSPACE_URL}/api/2.0/workspace/mkdirs" \
  --header "Authorization: Bearer ${DBX_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"path":"/Shared/my-folder"}' \
| jq .
```

Export a notebook:

```bash
curl --silent --show-error --fail-with-body \
  --request GET \
  --url "${DATABRICKS_WORKSPACE_URL}/api/2.0/workspace/export?path=/Shared/example&format=SOURCE" \
  --header "Authorization: Bearer ${DBX_TOKEN}" \
  --header "Accept: application/json" \
| jq .
```

Import a notebook:

```bash
CONTENT_B64="$(base64 -w0 ./local_notebook.py)"

curl --silent --show-error --fail-with-body \
  --request POST \
  --url "${DATABRICKS_WORKSPACE_URL}/api/2.0/workspace/import" \
  --header "Authorization: Bearer ${DBX_TOKEN}" \
  --header "Content-Type: application/json" \
  --data "{\"path\":\"/Shared/example\",\"format\":\"SOURCE\",\"language\":\"PYTHON\",\"overwrite\":true,\"content\":\"${CONTENT_B64}\"}" \
| jq .
```

### 4. Generic helper shell function (optional per session)

```bash
dbx_api() {
  local method="$1"
  local endpoint="$2"
  local body="${3:-}"

  set -a
  source ./.env
  set +a

  local token="${DATABRICKS_TOKEN:-${DATABRICKS_PAT:-${DATABRICKS_API_TOKEN:-${ANTHROPIC_AUTH_TOKEN:-}}}}"
  : "${DATABRICKS_WORKSPACE_URL:?DATABRICKS_WORKSPACE_URL is not set in .env}"
  : "${token:?Set DATABRICKS_TOKEN, DATABRICKS_PAT, DATABRICKS_API_TOKEN, or ANTHROPIC_AUTH_TOKEN in .env}"

  if [[ -n "$body" ]]; then
    curl --silent --show-error --fail-with-body \
      --request "$method" \
      --url "${DATABRICKS_WORKSPACE_URL}${endpoint}" \
      --header "Authorization: Bearer ${token}" \
      --header "Content-Type: application/json" \
      --header "Accept: application/json" \
      --data "$body"
  else
    curl --silent --show-error --fail-with-body \
      --request "$method" \
      --url "${DATABRICKS_WORKSPACE_URL}${endpoint}" \
      --header "Authorization: Bearer ${token}" \
      --header "Accept: application/json"
  fi
}
```

Example usage:

```bash
dbx_api GET "/api/2.0/workspace/list?path=/" | jq .
```

---

## Security Rules

- Never write tokens directly in commands, scripts, prompts, logs, or markdown
- Always load tokens from `.env` at call time
- Never print token values (`echo $DBX_TOKEN` is forbidden)
- If command output includes sensitive data, summarize without exposing secrets

---

## Error Handling

- `401 Unauthorized`: token missing, invalid, or expired
- `403 Forbidden`: token lacks required permissions
- `404 Not Found`: wrong endpoint or workspace path
- `400 Bad Request`: invalid JSON or unsupported request parameters

On errors, show HTTP response body (without exposing credentials) and propose
corrective action.

---

## Best Practices

- Keep endpoints explicit (e.g. `/api/2.0/workspace/list?path=/Shared`)
- Use `--fail-with-body` so failures return non-zero exit status
- Validate `.env` variables on every request, not once per conversation
- Prefer reusable helper functions for repeat calls in the same session
- For destructive APIs (`delete`, overwrite imports), confirm path and intent first
