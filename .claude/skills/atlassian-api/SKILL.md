---
name: atlassian-api
description: >
  Use this skill to make authenticated calls to the Atlassian (Jira / Confluence)
  REST API. Trigger phrases include: "call the Jira API", "create a Jira ticket",
  "update a Jira issue", "link a story to an epic", "add to a sprint", "get Jira
  issue", "search Jira with JQL", "fetch sprint details", "post to Confluence".
  This skill is also invoked internally by the jira-story-refiner agent whenever
  it needs to read from or write to Jira.
version: 1.0.0
argument-hint: "[GET|POST|PUT] [/rest/api/3/<resource>] [json-body]"
allowed-tools: [Bash]
---

# Atlassian API Skill

## Overview

This skill executes authenticated Atlassian REST API calls using `curl`.
Credentials are read from environment variables — never hard-coded.

Supported APIs:
- **Jira REST API v3** — issues, epics, sprints, links, transitions
- **Jira Agile / Software API** — board and sprint management

---

## Environment Variables Required

| Variable | Description |
|---|---|
| `JIRA_BASE_URL` | Base URL of your Jira instance, e.g. `https://yourorg.atlassian.net` |
| `JIRA_USER_EMAIL` | Atlassian account email used for Basic Auth |
| `JIRA_API_TOKEN` | Atlassian API token (generate at id.atlassian.com) |
| `JIRA_PROJECT_KEY` | Default project key, e.g. `DE` or `PLAT` |

These must be present in the shell environment (e.g. sourced from `.env`).
If any are missing, report which ones are absent and stop.

---

## Instructions

### 1. Validate environment before every call

```bash
: "${JIRA_BASE_URL:?JIRA_BASE_URL is not set}"
: "${JIRA_USER_EMAIL:?JIRA_USER_EMAIL is not set}"
: "${JIRA_API_TOKEN:?JIRA_API_TOKEN is not set}"
```

### 2. Base curl pattern

All requests share the same auth header and content-type:

```bash
curl --silent --fail-with-body \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Content-Type: application/json" \
  --header "Accept: application/json" \
  "<REQUEST_URL>" \
  [--request POST --data '<JSON_BODY>']
```

Always pipe the response through `| python3 -m json.tool` (or `jq` if available)
for readable output. Capture the raw response first to check for error fields.

---

### 3. Create an issue (story, task, bug, etc.)

```bash
curl --silent --fail-with-body \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Content-Type: application/json" \
  --request POST \
  "${JIRA_BASE_URL}/rest/api/3/issue" \
  --data '{
    "fields": {
      "project":     { "key": "<PROJECT_KEY>" },
      "issuetype":   { "name": "Story" },
      "summary":     "<one-line summary>",
      "description": {
        "type":    "doc",
        "version": 1,
        "content": [
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<body text>" }]
          }
        ]
      },
      "priority":    { "name": "Medium" }
    }
  }'
```

The response contains `"key"` (e.g. `DE-42`) and `"self"` (the API URL).
Construct the browser URL as: `${JIRA_BASE_URL}/browse/<KEY>`

---

### 4. Link a story to an epic

Use the `parent` field when creating the issue (Jira Next-Gen / team-managed):

```bash
"parent": { "key": "<EPIC_KEY>" }
```

For classic (company-managed) projects, set the custom Epic Link field instead.
First discover the field ID:

```bash
curl --silent \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  "${JIRA_BASE_URL}/rest/api/3/field" \
  | python3 -c "
import sys, json
fields = json.load(sys.stdin)
for f in fields:
    if 'epic' in f.get('name','').lower():
        print(f['id'], f['name'])
"
```

Then include it in the create/update body:

```bash
"<epic_link_field_id>": "<EPIC_KEY>"
```

---

### 5. Add an issue to a sprint

First find the board and active/target sprint:

```bash
# List boards for the project
curl --silent \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  "${JIRA_BASE_URL}/rest/agile/1.0/board?projectKeyOrId=${JIRA_PROJECT_KEY}"

# List sprints on a board (replace <BOARD_ID>)
curl --silent \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  "${JIRA_BASE_URL}/rest/agile/1.0/board/<BOARD_ID>/sprint?state=active,future"
```

Then move the issue to the sprint:

```bash
curl --silent --fail-with-body \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Content-Type: application/json" \
  --request POST \
  "${JIRA_BASE_URL}/rest/agile/1.0/sprint/<SPRINT_ID>/issue" \
  --data '{ "issues": ["<ISSUE_KEY>"] }'
```

Alternatively, set `customfield_10020` (Sprint field) during issue creation
if you already know the sprint ID:

```bash
"customfield_10020": <SPRINT_ID>
```

---

### 6. Get an issue

```bash
curl --silent \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  "${JIRA_BASE_URL}/rest/api/3/issue/<ISSUE_KEY>"
```

---

### 7. Update an issue

```bash
curl --silent --fail-with-body \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  --header "Content-Type: application/json" \
  --request PUT \
  "${JIRA_BASE_URL}/rest/api/3/issue/<ISSUE_KEY>" \
  --data '{
    "fields": {
      "<field_name>": "<new_value>"
    }
  }'
```

---

### 8. Search with JQL

```bash
curl --silent \
  --user "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
  --get \
  --data-urlencode "jql=project=<PROJECT_KEY> AND sprint in openSprints() AND issuetype=Story" \
  --data-urlencode "fields=summary,status,assignee,priority" \
  "${JIRA_BASE_URL}/rest/api/3/search"
```

---

## Error Handling

- HTTP 401 → invalid credentials; check `JIRA_USER_EMAIL` and `JIRA_API_TOKEN`
- HTTP 403 → insufficient permissions on the project or issue
- HTTP 404 → issue key, board ID, or sprint ID does not exist
- HTTP 400 → malformed request body; print the full error response for diagnosis
- Always check the response body for an `"errors"` or `"errorMessages"` key
  even when the HTTP status is 2xx

---

## Best Practices

- Never log or echo `JIRA_API_TOKEN` in output
- Use `--fail-with-body` so `curl` exits non-zero on HTTP errors
- Prefer `python3 -m json.tool` for pretty-printing when `jq` is unavailable
- When creating issues, always return the browser URL (`/browse/<KEY>`) to the user
- Cache board/sprint IDs within a session to avoid redundant API calls
