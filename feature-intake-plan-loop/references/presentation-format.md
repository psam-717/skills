# Decision Presentation Format

When presenting multiple design decisions for a feature, use **compact tables**. One table per decision. No paragraphs for each option — the table is the spec.

## Preferred format

```
### Decision N: [Title]

**Context:** One-line trade-off summary (max 2 sentences).

| # | Option | Pro | Con |
|---|---|---|---|
| 1 | **Option A** — one-liner | Single benefit | Single drawback |
| 2 | **Option B** — one-liner | Single benefit | Single drawback |
| 3 | **Option C** — one-liner | Single benefit | Single drawback |

**My pick:** Option X because [one reason].
```

## Why

- The user can scan the entire decision in one glance
- Tables compress 3 paragraphs of text into 3 rows
- The "Pro" and "Con" columns force you to be concise — if you can't fit it in a few words, you don't understand the trade-off well enough
- The "My pick" line at the bottom gives the user a starting point to push against

## When to deviate

- If a single option needs detailed explanation (e.g., architectural pros/cons), add one sentence below the table: `Option 1 needs more detail: ...`
- If you have only 1–2 decisions, a slightly wordier format is fine — but still prefer a table for the options
