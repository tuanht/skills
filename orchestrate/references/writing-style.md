## Tuan's writing style

Style authoritative. Conflicts default PR/Jira conventions (long-form summary prose, generic changelog phrasing, em-dash separators) — this file win. Keep only structural scaffolding those templates need (section headers, checklists); all prose follow rules below.

Drafting any written output for Tuan (Jira comments, PR descriptions, Slack messages, commit messages, emails, notes) — match this style exact:

- Short, direct sentences. No preamble, no throat-clearing
- Active voice always ("we found", "the query returns", "this breaks")
- Numbered lists for findings or steps. No section headers unless doc long (exception: commit message bodies — no numbered lists, short prose sentences instead)
- Lowercase after colons in list items ("1. on production, ~40 records have...")
- Team voice: "we" not "I"
- Natural connectors: "then it will", "we must", "so it" — not formal alternatives
- Backticks for inline technical terms
- Parenthetical asides for brief context ("for more readability", "already handled")
- No period at end of list items
- No semicolons, no em-dashes
- Approximate numbers with "~" ("~40 records")
- Contractions where natural ("it's", "we've") — don't force
- "Either X is not a bug" style — direct verdict, no hedging
- Uncertain? State open question direct ("unclear whether X causes Y — needs prod data")

Do NOT:
- Open with "The investigation shows that..." — just say "The investigation result:"
- Bold headers inside short comments
- Add "In summary" or closing remarks
- Passive voice ("it was found that", "it was noticed that")
- Open paragraph with framing label like "The gap:" / "The problem:" / "The issue:" — state fact direct, or lead with "we" + what breaks (e.g. "But a single order can have multiple line items, so...")
- List changed/added/test files in PR body (no "Scope"/"Files changed" section) — reviewer see file list in GitHub UI already, restating noise