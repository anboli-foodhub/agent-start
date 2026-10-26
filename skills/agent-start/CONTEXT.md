# Agent Start

The domain of `/agent-start`: taking a single JIRA ticket through planning, review, PO communication, implementation, and shipping — as one gated run.

## Language

**Ticket**:
The JIRA issue `/agent-start` is run against — its ID, title, description, acceptance criteria, and issue type are the single source of truth for scope.
_Avoid_: issue, story, JIRA ticket.

**Plan**:
The structured implementation plan drafted from a Ticket and a workspace scan (Summary, Scope, Files, Steps, Risks, Complexity). A Plan may be revised by a Rules Check, but a revision only changes *how* the Ticket gets built, never *what* it asked for.
_Avoid_: implementation plan, proposal, spec, design.

**Rules Check**:
The optional pass that reads engineering standards and repo rules (if any are configured) against a drafted Plan, before any code exists, and revises the Plan accordingly. Distinct from a PR review, which happens later against real code.
_Avoid_: review, PR review, lint, compliance check.

**PO Notes**:
The plain-business, non-technical summary drafted from the Plan and optionally posted as a Ticket comment. Always describes the *planned* change — it can exist before a single line of code is written.
_Avoid_: release notes, changelog, summary.

**Approval Gate**:
A single yes/no (or yes/edit/no) checkpoint that must receive an explicit answer before the run proceeds past it. A "no" ends the run; it does not undo anything already done at earlier gates.
_Avoid_: checkpoint, confirmation, prompt.

**Implementation**:
The actual code change, written only after the Implement Approval Gate returns yes. A Plan is not considered real work until Implementation exists.
_Avoid_: code, changes, diff, build.

**Notification**:
The one-way message sent after a PR opens, to whatever platform your team uses. It carries no approval semantics — the run never waits on or reads a reply to it.
_Avoid_: approval request, platform approval, alert.
