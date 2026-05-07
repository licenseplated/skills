---
name: jira-issue-plan
description: Read a Jira ticket, draft an implementation plan, then interview the user relentlessly to refine it before uploading the final plan as an attachment on the ticket. Use when the user provides a Jira URL or issue key and wants a planning session, or invokes /jira-issue-plan.
---

Read the Jira issue, draft an implementation plan, then interview me relentlessly until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, propose your recommended answer with the reasoning. Then write the plan to a markdown file and upload it as an attachment on the ticket.

Ask questions one at a time, waiting for feedback before continuing. If a question can be answered by exploring the codebase, explore the codebase instead.

## Auth

Yardi tickets live on Atlassian Cloud. Source the env file before any API call:

```bash
set -a; . ~/.yardi.env; set +a
```

That sets `ATLASSIAN_ADDR`, `ATLASSIAN_EMAIL`, `ATLASSIAN_TOKEN`. If the env file is stale, refresh it from Vault — see `/home/mmz/projects/CLAUDE.md`.

## Workflow

### 1. Read the ticket

Extract the issue key from the URL (e.g., `CLDPLT-123` from `https://yardi.atlassian.net/browse/CLDPLT-123`):

```bash
KEY=CLDPLT-123
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" \
  "$ATLASSIAN_ADDR/rest/api/3/issue/$KEY?fields=summary,description,status,issuetype,labels,parent,issuelinks" | jq .
```

The description comes back as ADF JSON — read it as a tree, don't expect plain text. Pull linked tickets too if they look load-bearing.

### 2. Sketch the draft plan

Identify the modules touched and the interfaces involved. Explore the relevant code areas before asking anything — many "questions" answer themselves once you read the code. List the open questions that genuinely need the user.

### 3. Interview

For each open question:

- State the question, your recommended answer, and the reasoning
- Wait for feedback
- Update the plan inline as decisions land

Walk the dependency tree: settle foundational decisions before branching ones. If the user uses fuzzy or overloaded terms, push for a precise canonical name and capture it in the glossary. Stress-test boundary conditions with concrete scenarios.

### 4. Write the plan

Save to `/tmp/jira-plan-<KEY>.md`. Suggested sections:

- **Context** — your reading of what the ticket asks for
- **Modules touched** — code areas and interfaces
- **Decisions** — what was settled in the interview, with the reasoning
- **Open questions** — anything still unresolved (ideally none)
- **Out of scope**
- **Glossary** — sharpened terms

### 5. Attach to the ticket

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@/tmp/jira-plan-$KEY.md" \
  "$ATLASSIAN_ADDR/rest/api/3/issue/$KEY/attachments" | jq '.[] | {id, filename, size}'
```

Confirm the response includes a non-empty entry with the attachment ID, then tell the user the attachment is live.
