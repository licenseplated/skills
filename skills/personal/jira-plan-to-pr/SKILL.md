---
name: jira-plan-to-pr
description: Take a Jira ticket plus its attached implementation plan, branch off `dev` in every repo the plan touches, deploy to a named testbed, validate per the plan, then open PRs into `dev` and post a summary comment back to the ticket. Use when the user supplies a Jira URL and a target testbed and wants the plan executed end-to-end, or invokes /jira-plan-to-pr.
---

# jira-plan-to-pr

Drive a planned Yardi fix from ticket to open PRs.

Inputs the user must supply (or the skill must extract from the request):

- `JIRA_URL` — e.g. `https://yardi.atlassian.net/browse/CLDPLT-123`
- `TESTBED` — testbed key, e.g. `mmz`, `tobyg`. Must already exist in `platform-testbed/terraform/main/testbeds.auto.tfvars` and be `active: true`.

The plan document is expected as an attachment on the Jira ticket (typically produced by `/jira-issue-plan`). If no plan attachment is present, stop and tell the user.

## Auth

Source the Yardi env file before any Atlassian or `gh` call (the Yardi `GITHUB_TOKEN` shadows the general one for this shell):

```bash
set -a; . ~/.yardi.env; set +a
```

If the file is stale, refresh from Vault per `/home/mmz/projects/CLAUDE.md`.

## Workflow

Use a TaskList to track the phases below — each one ends with state the user can see (a branch, a deploy, a PR, a comment) and is worth marking complete as you go.

### 1. Read the ticket and the plan

Extract the issue key (`KEY=CLDPLT-123`). Fetch the ticket and its attachments:

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" \
  "$ATLASSIAN_ADDR/rest/api/3/issue/$KEY?fields=summary,description,status,attachment,issuelinks" | jq .
```

Find the most recent markdown attachment (typically `jira-plan-$KEY.md`), download it, and read it end-to-end:

```bash
ATTACH_URL=$(curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" \
  "$ATLASSIAN_ADDR/rest/api/3/issue/$KEY?fields=attachment" \
  | jq -r '.fields.attachment | sort_by(.created) | reverse | map(select(.filename | endswith(".md")))[0].content')
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" -L "$ATTACH_URL" -o /tmp/plan-$KEY.md
```

From the plan, extract:

- **Repos touched** — every repo that needs a code change (one branch + one PR per repo).
- **Validation steps** — what proves the fix works on the testbed. Trust these; don't invent your own.
- **Out-of-scope notes** — don't expand the diff past the plan.

If the plan is ambiguous about which repos or what validation looks like, stop and ask. Don't guess the diff.

### 2. Branch and implement

For each repo in scope (`cd` into it first):

```bash
git fetch origin
git checkout dev
git pull --ff-only
git checkout -b "bugfix/$KEY-<short-slug>"
```

Implement the plan's changes for that repo. Commit with the issue key in the message so Jira's dev panel auto-links:

```bash
git commit -m "$KEY: <short summary from the plan>"
git push -u origin HEAD
```

Pushing the branch triggers CI to build a docker image tagged with the sanitized branch name (slashes → `-`, e.g. `bugfix-CLDPLT-123-foo`). That's the tag you'll point the testbed at in step 3.

### 3. Deploy to the testbed

`platform-testbed` is the source of truth for how testbeds are reconfigured. **Read `~/projects/yardidev/platform-testbed/CLAUDE.md` before choosing a path** — its sections on `Testbed Redeploy`, `code-server` (`switch-tags` helper), and `Per-testbed divergence from golden` describe the available mechanisms. The right choice depends on whether the testbed is `protected` and whether you want the new tags committed to git or applied as a runtime override.

Two common shapes:

**(a) Override path (no tfvars commit yet — preferred for test-before-merge):**
SSH into the target testbed's infra L1 and use the `switch-tags` helper to point services at the new branch tags. The override survives reconciles while `protected: true`. This lets you validate before any platform-testbed PR exists.

**(b) Commit path (tfvars edit + `Testbed Redeploy`):**
Open a PR to `platform-testbed` editing `terraform/main/testbeds.auto.tfvars` for the target testbed's `yvm_svc_tag` / `yks_svc_tag` / `yvm_ui_tag`, merge it, then dispatch:

```bash
gh workflow run testbed-redeploy.yml \
  --repo yardidev/platform-testbed -f testbed=$TESTBED
```

Pick (a) unless the plan or the user calls for the commit path. Either way, **wait for the deploy to finish and the new image tags to be live on the testbed before validating** — `gh run watch <id> --repo yardidev/platform-testbed` for path (b); for path (a), check `docker compose ps` / `docker images` on the infra L1.

### 4. Validate

Run the validation steps from the plan exactly as written. If they pass, proceed. If any fail, stop, report which step failed and the observed output, and wait for the user — don't open PRs against a fix that hasn't been confirmed.

### 5. Open PRs into `dev`

For each repo that received a branch in step 2, open a PR with the issue key as the first token of the title (Jira's dev panel auto-links on issue key in the PR title and branch name):

```bash
gh pr create \
  --base dev \
  --title "$KEY: <short summary>" \
  --body "$(cat <<EOF
## Summary
<1-3 bullets from the plan>

## Validation
Deployed to \`$TESTBED\` testbed and ran the validation steps from the plan attached to [$KEY]($ATLASSIAN_ADDR/browse/$KEY). All passed.

## Test plan
- [ ] CI green
- [ ] Reviewer sanity-check on \`$TESTBED\` if needed
EOF
)"
```

Capture each PR URL — you'll need them for the Jira comment.

If the deploy used path (b), the platform-testbed tfvars PR is already merged; mention it in the comment but don't double-open.

### 6. Update the Jira ticket

Post a single summary comment on the ticket with the PR links and the deploy outcome. Keep it terse — the dev panel already shows the PRs, so the comment's job is to record what happened, not duplicate metadata.

```bash
COMMENT_BODY=$(jq -Rs '{
  body: {
    type: "doc", version: 1,
    content: [{
      type: "paragraph",
      content: [{ type: "text", text: . }]
    }]
  }
}' <<EOF
Implemented per the attached plan. Deployed to \`$TESTBED\` testbed; validation steps from the plan all passed.

PRs opened into \`dev\`:
- <repo-1>: <pr-url-1>
- <repo-2>: <pr-url-2>
EOF
)

curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST -d "$COMMENT_BODY" \
  "$ATLASSIAN_ADDR/rest/api/3/issue/$KEY/comment" | jq '{id, created}'
```

Confirm a non-empty `id` came back. Then report to the user: the comment URL, each PR URL, and any caveats from validation.

## Stop conditions

Stop and ask the user before continuing if any of these happen:

- No plan attachment on the ticket.
- The plan is ambiguous about repos in scope or validation steps.
- A repo's `dev` branch can't fast-forward (local divergence — investigate, don't force).
- A validation step fails on the testbed.
- The target testbed isn't `active: true` in `testbeds.auto.tfvars`, or doesn't exist.

## Don't

- Don't open PRs against `main` — base is always `dev` for this workflow.
- Don't expand scope past what the plan calls for, even if you spot adjacent issues. Note them for a follow-up ticket instead.
- Don't skip validation because "the change looks obvious."
- Don't comment on the Jira ticket per-step — one summary comment at the end is the agreed shape.
