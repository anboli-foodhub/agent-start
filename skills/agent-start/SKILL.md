---
name: agent-start
description: This skill should be used when the user wants to run a full JIRA ticket lifecycle in one command — plan, optional review-rules check, optional PO notes, implementation, and PR — each stage gated by explicit approval rather than running unattended. Invoked as /agent-start <TICKET-ID>.
disable-model-invocation: true
---

# Agent Start Skill

Runs a full ticket lifecycle in one command, but stays gated: it stops for your approval at each major checkpoint rather than running end-to-end unattended. It owns its own approval sequence — it does **not** shell out to an `/analyse` or `/po-notes` skill as sub-invocations (that would trigger their own independent yes/no loops and implement/post immediately on "yes", which would collide with the checkpoints below). Instead it reuses their logic and templates inline, in this order.

## Setup (do this once before first use)

Open this file and configure two things for your own workspace before relying on the gates below:

1. **Reviewer notification (Step 9).** Replace `<REVIEWER_EMAIL>` with the email of whoever should be notified when a PR opens, or change Step 9 to ask you for a recipient each time instead of hardcoding one.
2. **Engineering standards (Step 4).** This step optionally checks a plan against your own standards docs if you have them (e.g. a repo-specific rules file, or a separate "engineering standards" skill). If you don't have anything like that yet, leave Step 4 as a no-op — it already degrades gracefully.

## How to invoke

```
/agent-start PROJ-1234
```

## Step 1 — Parse the ticket ID

The argument may be a bare ID (`[A-Z]+-[0-9]+`) or a URL containing `/browse/` or `/issues/` — extract the ID either way. If missing, ask for it and stop.

## Step 2 — Build the implementation plan

1. Fetch the ticket: try a custom JIRA MCP server first if one is configured (e.g. `mcp__jira-analyser__fetch_jira_ticket { "ticketId": "<TICKET-ID>" }`). If it fails or isn't configured, fall back to Atlassian's official MCP connector (`mcp__claude_ai_Atlassian_Rovo_2__getJiraIssue`), using your own `cloudId` (call `getAccessibleAtlassianResources` once if you don't know it), and note the fallback.
2. Extract title, description, acceptance criteria, ticket type, labels, priority, linked issues, comments.
3. Run `git log main..HEAD --oneline --no-merges` and `git diff main..HEAD --stat` to see current branch state, then scan the workspace for files relevant to the ticket's key topic (via your JIRA MCP's workspace-scan tool if you have one, or a plain `grep`/`rg` across your repos otherwise). Read the most relevant ones.
4. Draft the plan using this template: **Summary, Scope, Files to create/modify, Step-by-step implementation, Risks & considerations, Complexity estimate.**

Keep the ticket's issue type (Bug / Story / Task / etc.) — it decides the branch prefix in Step 7.

## Step 3 — Share the plan, then offer the review-rules check

Display the full plan. Then ask exactly:
> "Here's the implementation approach. Want me to check it against the review rules before we go further? (**yes** / **no**)"

- **no** → skip to Step 5.
- **yes** → continue to Step 4.

## Step 4 — Check the plan against review rules

There is no diff yet, so this is a read of your standards against the *planned* approach, not a PR review:

1. If you have engineering-standards or review-rules documents (a dedicated skill, a `pr-review-rules.md`-style file, a wiki page — whatever you use), read them now. **If you don't have anything like that configured, say so plainly and skip to Step 5** — don't invent standards to check against.
2. Walk the plan's "Files to create/modify" and "Step-by-step implementation" sections against whatever you loaded. Flag anything the *planned* approach would violate (e.g. a planned write without a transaction where one's needed, a planned query against the wrong data source, missing audit/PII handling, an undeclared cross-repo dependency).
3. Revise the plan to close any real gaps found. If nothing applies, say so plainly — don't invent findings.

Then show the **updated plan summary** (just the sections that changed, plus a one-line note on what the rules check caught or confirmed clean).

## Step 5 — Offer PO notes

Ask exactly:
> "Want me to generate PO notes for this ticket? (**yes** / **no**)"

- **no** → skip to Step 7.
- **yes** → continue to Step 6.

## Step 6 — Draft and post PO notes

Follow the `po-notes` skill's template and rules exactly if you have it installed (resolve `template.md` from `.claude/skills/po-notes/template.md` then `~/.claude/skills/po-notes/template.md`; no file/function/table/repo names ever; 1–3 sentences per section). The one difference: since no code exists yet, base "What's changing" on the **approved plan from Step 2/4**, not a git diff.

Show the draft, then ask exactly:
> "Does this look good? Reply **yes** to post it to {TICKET-ID}, **edit** to modify, or **no** to cancel."

- **yes** → post via your JIRA MCP's comment tool, falling back to Atlassian's official connector if that errors. Confirm success.
- **edit** → revise, show again, re-ask.
- **no** → "Notes discarded. Nothing was posted to JIRA." and move on.

## Step 7 — Approval to implement

Ask exactly:
> "Ready to implement? Reply **yes** to start or **no** to cancel."

On **no**: "Implementation skipped. The plan above is ready whenever you want to proceed." and stop.

On **yes**, continue:

### 7a — Create the dedicated branch

```bash
git checkout main
git pull origin main
git checkout -b <type>/<TICKET-ID>/<short-kebab-description>
```

Pick `<type>` from the ticket's issue type. A sensible default mapping — adjust to your own team's branching convention if different:

| Ticket type | Branch prefix |
|---|---|
| Bug | `bugfix` |
| Hotfix / production incident | `hotfix` |
| Story / Task / Feature | `feature` |
| Epic-level integration branch | `develop` |

`<short-kebab-description>` is a concise kebab-case slug of the ticket title. Base branch is always `main`, confirmed up to date via the `git pull` above.

### 7b — Implement

1. Use a todo list to break the plan's numbered steps into tasks.
2. Before writing or editing any client-side file (anything rendered in a browser: views/templates, `.html`, `.css`/`.scss`, `.js`/`.jsx`/`.ts`/`.tsx`/`.vue` under a frontend/public/assets path), consult whatever frontend-conventions skill or guide you have available first, if any.
3. Implement fully, following existing conventions in the touched repo(s).
4. Run relevant tests/build checks.
5. Mark tasks done as you go; summarise what was built.

## Step 8 — Approval to commit, push, and open the PR

Ask exactly:
> "Implementation's done — summary above. Ready to commit, push, and open a PR? (**yes** / **no**)"

On **no**: stop, leave the working tree as-is.

On **yes**:
1. `git add` the relevant files (never a blanket `-A`; review `git status` first for anything unexpected).
2. Commit following your own team's commit-message convention (e.g. `TICKET-ID - Short message`, if that's what you use).
3. `git push origin <branch>` (or with `-u` on first push of a new branch, per your own preference).
4. `gh pr create` with a concise title and a body covering what changed and a test plan, base `main`.
5. Report the PR URL.

## Step 9 — Notify for approval

Once the PR exists, send a one-way notification to whoever should review it (e.g. via Zoho Cliq, Slack, or your team's usual channel):

```
mcp__zoho-cliq__send_message_to_user { "email": "<REVIEWER_EMAIL>", "message": "<summary + PR link>" }
```

Message content: ticket ID + title, one-line summary of the change, and the PR URL from Step 8 — enough for the recipient to review and approve from the PR itself.

This is a **notify-only** step, not a gate: if there is no tool available to read replies in your messaging platform, do not pause or wait for a response here. The real approval loop already happened in this chat (Steps 3–8); the notification is visibility for the reviewer, nothing more. Report to the user that the notification was sent (or if sending errors, say so plainly — don't retry silently or fall back to a different channel without asking).

## Ground rules

- Never skip a checkpoint (Steps 3, 5, 7, 8) — each requires an explicit answer before moving on.
- Never post to JIRA or push/open a PR without the matching explicit **yes**.
- Don't re-litigate the plan after Step 4 unless the user raises something in response to a checkpoint question — these are approval gates, not open discussion loops (if the user wants to discuss, follow their lead and re-show the affected section before continuing).
- If the review-rules check (Step 4) finds nothing to flag, or has nothing configured to check against, say so plainly rather than manufacturing findings.
- Keep `<type>` selection and the branch table above in sync with your own team's branching convention if that ever changes.
- Step 9's notification is informational only — never treat silence or a delay in the messaging platform as approval or disapproval of anything; the PR is already open regardless.
