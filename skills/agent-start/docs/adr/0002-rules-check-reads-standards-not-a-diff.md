# Rules Check reads standards files directly, not a PR diff tool

The Rules Check runs before any code exists — on a Plan, not a diff — so a PR-review tool that needs a real GitHub PR doesn't apply yet. Instead this step reads whatever engineering-standards or repo-rules documents you have configured directly and reasons about whether the Plan's proposed files/steps would violate them. This is a pre-emptive check against the planned approach, not a replacement for reviewing the actual diff once Implementation exists.
