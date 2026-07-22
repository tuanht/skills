## Tuan's writing style

This style is authoritative. When it conflicts with default PR/Jira formatting conventions (long-form summary prose, generic changelog phrasing, em-dash separators), this file wins. Keep only the structural scaffolding those templates require (section headers, checklists); all prose follows the rules below.

When drafting any written output for Tuan (Jira comments, PR descriptions, Slack messages, commit messages, emails, notes) — match this style exactly:

- Short, direct sentences. No preamble, no throat-clearing
- Active voice always ("we found", "the query returns", "this breaks")
- Numbered lists for findings or steps. No section headers unless the document is long (exception: commit message bodies — no numbered lists, use short prose sentences instead)
- Lowercase after colons in list items ("1. on production, ~40 records have...")
- Team voice: "we" not "I"
- Natural connectors: "then it will", "we must", "so it" — not formal alternatives
- Backticks for inline technical terms
- Parenthetical asides for brief context ("for more readability", "already handled")
- No period at the end of list items
- No semicolons, no em-dashes
- Approximate numbers with "~" ("~40 records")
- Contractions where natural ("it's", "we've") — don't force them
- "Either X is not a bug" style — direct verdict, no hedging
- When uncertain, state the open question directly ("unclear whether X causes Y — needs prod data")

Do NOT:
- Open with "The investigation shows that..." — just say "The investigation result:"
- Use bold headers inside short comments
- Add "In summary" or closing remarks
- Use passive voice ("it was found that", "it was noticed that")
- Open a paragraph with a framing label like "The gap:" / "The problem:" / "The issue:" — state the fact directly, or lead with "we" + what breaks (e.g. "But a single order can have multiple line items, so...")
