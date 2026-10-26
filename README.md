# agent-start

A Claude Code skill that takes a JIRA ticket from "here's the ID" to "here's the PR" in one command — but never runs unattended. Every stage stops for your explicit yes.

```
/agent-start PROJ-1234
```

## The flow

| # | What happens | Your gate |
|---|---|---|
| 1 | Parse the ticket ID (bare ID or JIRA URL) | — |
| 2 | Fetch the ticket, scan the workspace, draft an implementation plan | — |
| 3 | Show the plan | **yes/no** — run it past the review rules first? |
| 4 | *(if yes)* Check the **planned approach** against your engineering standards (if configured), revise, show what changed | — |
| 5 | Offer PO notes | **yes/no** — generate them? |
| 6 | *(if yes)* Draft PO notes from the approved plan | **yes/edit/no** — post to the ticket? |
| 7 | Offer to implement | **yes/no** — start building? |
| 7a | *(if yes)* `git checkout main && git pull && git checkout -b <type>/<TICKET-ID>/<slug>` | — |
| 7b | Implement, run tests | — |
| 8 | Offer to ship | **yes/no** — commit, push, open the PR? |
| 9 | Notify the configured reviewer that the PR is up | — (FYI only, see below) |

## Install

```
/plugin marketplace add anboli-foodhub/agent-start
/plugin install agent-start@anboli-agent-start
```

Or clone and install locally:

```bash
git clone git@github.com:anboli-foodhub/agent-start.git
```
```
/plugin marketplace add /path/to/agent-start
/plugin install agent-start@anboli-agent-start
```

## Setup

Open `skills/agent-start/SKILL.md` and configure two things for your own workspace before first use:

1. **Reviewer notification (Step 9)** — replace `<REVIEWER_EMAIL>` with who should be notified when a PR opens, or change the step to ask you each time.
2. **Engineering standards (Step 4)** — if you have standards/rules documents, point the skill at them; if not, leave it as-is — it already skips gracefully and says so.

The skill is marked user-only (`disable-model-invocation: true`) — it's always something you explicitly asked for, never something Claude triggers on its own, since it posts comments, commits, pushes, and opens PRs.

## Example run

```
/agent-start PROJ-1234
```

1. Fetches ticket `PROJ-1234` ("Add abandoned-checkout reminder email"), scans the workspace for relevant files, and drafts a plan: Summary, Scope, Files to create/modify, Step-by-step implementation, Risks, Complexity estimate.
2. Shows the plan and asks: *"Want me to check it against the review rules before we go further? (yes/no)"* — you say **yes**; it checks the plan against your configured standards, finds nothing to flag, and says so plainly.
3. Asks: *"Want me to generate PO notes for this ticket? (yes/no)"* — you say **yes**; it drafts a six-section plain-business summary and asks for approval to post. You say **yes**; it posts the comment.
4. Asks: *"Ready to implement? (yes/no)"* — you say **yes**; it creates branch `feature/PROJ-1234/abandoned-checkout-reminder`, implements, and runs tests.
5. Asks: *"Implementation's done — ready to commit, push, and open a PR? (yes/no)"* — you say **yes**; it commits, pushes, opens the PR, and reports the URL.
6. Sends a one-way notification (e.g. via Slack/Cliq/Teams) to your configured reviewer with the ticket, a one-line summary, and the PR link — informational only, never a wait-for-reply gate.

## Design decisions worth knowing

**Why it doesn't just call a separate `/analyse` or `/po-notes` skill.** Skills like those typically end in their own "yes → do it now" step. Chaining into them as-is would fire *their* approval loop instead of this one's checkpoints. So `agent-start` reproduces that planning/drafting logic and templates inline, and owns the full approval sequence itself. See `docs/adr/0001-inline-plan-and-po-notes-logic.md`.

**Why the review-rules check (step 4) doesn't use a PR-diff review tool.** At step 4 there's no code yet — just a plan — so this step reads standards files directly and reasons about whether the *planned* files/steps would violate them. See `docs/adr/0002-rules-check-reads-standards-not-a-diff.md`.

**Why the post-PR notification (step 9) is one-way.** Most chat-platform MCP tools can send messages but can't reliably read replies, so there's no way to actually wait for approval there without polling blind. The notification is explicitly FYI-only; the real gates are all in-chat (steps 3, 5, 7, 8). See `docs/adr/0003-notification-is-not-an-approval-gate.md`.

**Branch type inference.** The branch prefix (`bugfix` / `feature` / `hotfix` / `develop`) is picked from the ticket's JIRA issue type, per the default table in `SKILL.md` — adjust it to your own team's convention if different. Base branch is always `main`.

## Dependencies

- A JIRA-capable MCP server (a custom one, or Atlassian's official connector)
- A messaging MCP for step 9 (Slack, Zoho Cliq, Teams, etc.) — optional, the skill still works end-to-end without it, just skip step 9
- CLI: `git`, `gh`

## License

MIT
