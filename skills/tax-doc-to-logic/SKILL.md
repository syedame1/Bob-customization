---
name: tax-doc-to-logic
description: Converts indirect tax authority guidance (VAT/GST/Sales Tax PDFs, circulars, notifications, rate schedules) into structured decision tables and pseudocode that engineers can implement directly in tax calculation/determination engines. Use this skill whenever the user uploads or references a tax authority document, circular, notification, ruling, or rate schedule and wants it turned into implementable logic, decision tables, rule specs, or pseudocode for a tax application. Also trigger for requests like "turn this regulation into code logic", "extract the rules from this tax notice", "build a decision table from this VAT guidance", or "what would the implementation logic look like for this GST circular". Applies to any jurisdiction's indirect tax (VAT, GST, Sales Tax, Customs duty, Excise).
---

# Tax Documentation-to-Logic

Converts indirect tax authority source documents (PDFs, notifications, circulars, rate schedules, rulings) into engineer-ready artifacts: a structured decision table (CSV/JSON) and corresponding pseudocode, plus an audit trail mapping each rule back to its source clause.

## When to use this skill

Trigger on requests to:
- Extract rules/logic from a tax authority PDF, circular, or notification
- Build a "decision table" or "rule matrix" from regulatory text
- Draft pseudocode for a tax determination/calculation feature based on a regulation
- Summarize "what code needs to change" given a new tax notice

## Output location and file-saving (required for every run)

This skill ALWAYS saves its output as files on disk — never leave the decision table, pseudocode, or audit summary as chat-only text. Chat text is a readable preview of the files, not a substitute for them.

### Output folder

Create the output folder **first, before generating any content**, at:

```
/mnt/user-data/outputs/<doc-slug>/
```

`<doc-slug>` is a short slug derived from the source document's name/number (e.g. notification "14/2024" → `notification-14-2024`). If no clear identifier exists, use `tax-doc-<yyyymmdd>` with today's date. If the user gives a project/feature name, use that instead.

### Required files (fixed names, every run)

Save exactly these files inside that folder — do not rename, merge, or skip any of them, even for a short document:

| File | Format | Content |
|---|---|---|
| `decision_table.csv` | CSV | One row per atomic rule, columns per Step 3 |
| `pseudocode.txt` | Plain text | Pseudocode generated per Step 4, in the same structural style as `references/pseudocode_patterns.md` |
| `audit_summary.md` | Markdown | Coverage stats, flags, gaps, per Step 5 |
| `source_document.<ext>` | Same as original | Only if the source document should be retained for traceability |

If the user explicitly asks for a different format (e.g. JSON instead of CSV, or a single combined file), follow that instead — but confirm the substitution back to them in one line so it's not silently different from this default.

### Procedure

1. Create the output folder first.
2. Write `decision_table.csv`, then `pseudocode.txt`, then `audit_summary.md` — in that order, so each later file can reference rule_ids already defined in the earlier ones.
3. After all files are written, call `present_files` once on the full set so the user gets one complete batch. Do not present files individually as they're created, and never write the only copy outside `/mnt/user-data/outputs/`.
4. In the chat response itself, still show: a 2-3 sentence summary, the decision table as a markdown table, and the pseudocode block — as a readable preview — always followed by the actual file links. Never substitute the chat preview for the files.

### Checklist (confirm before ending the turn)

- [ ] Output folder created at `/mnt/user-data/outputs/<doc-slug>/`
- [ ] All required files written, non-empty, and internally consistent (every `rule_id` in the pseudocode exists in the decision table and vice versa)
- [ ] Files shared via a single `present_files` call
- [ ] Chat response includes the readable preview, not just a "see attached" pointer

## Workflow

### Step 1: Ingest the source document

Read the document (PDF, text, or pasted content). If a PDF, use the pdf-reading skill to extract text content first — read the whole document, don't sample.

Identify the document type:
- **Rate notification**: lists rates by HSN/SAC code, product category, or service type
- **Exemption notification**: lists exempted goods/services, often with conditions
- **Procedural circular**: describes a process (e.g., reverse charge mechanism, place of supply rules, refund eligibility)
- **Threshold/registration rules**: numeric thresholds triggering obligations

### Step 2: Extract atomic rules

For each distinct rule in the document, extract:
- **Condition(s)**: the input attributes that determine applicability (product category, transaction type, value thresholds, jurisdiction, buyer/seller status, date ranges)
- **Outcome**: the tax treatment (rate %, exemption, reverse charge applicable, place of supply jurisdiction, etc.)
- **Effective date**: when the rule starts/ends applying (critical — tax rules are always date-bound)
- **Source reference**: section/clause/paragraph number in the source document, quoted minimally (under 15 words) or paraphrased

Watch for:
- **Conditional exceptions** ("...except where the supply is made to a registered person under composition scheme")
- **Cross-references** to other notifications (flag these — they may need separate extraction)
- **Ambiguous language** — flag for human review rather than guessing; tax logic errors have compliance consequences

### Step 3: Build the decision table

Use `assets/decision_table_template.csv` as the structure. Each row = one atomic rule. Columns should be adapted to the document but typically include:

`rule_id, condition_field_1, condition_field_2, ..., outcome_field, effective_from, effective_to, source_clause, confidence`

The `confidence` column flags rules extracted with ambiguity (`high` / `needs_review`).

Write the completed table directly to `<output-folder>/decision_table.csv` (see Output location above). Also render it as a markdown table inline in the chat response for quick review — but the CSV file is the deliverable, not optional.

### Step 4: Generate pseudocode

From the decision table, generate pseudocode following the pattern in `references/pseudocode_patterns.md`. Key conventions:
- Express as a function taking a transaction object and an "as-of date", returning a tax determination object
- Order conditions from most specific to most general (specific exemptions before general rate rules)
- Include the rule_id and source_clause as inline comments for traceability
- Flag any `needs_review` rules with a `# TODO: confirm with tax counsel` comment — never silently resolve ambiguity

Write the pseudocode directly to `<output-folder>/pseudocode.txt` (see Output location above).

### Step 5: Produce the audit summary

Write a summary directly to `<output-folder>/audit_summary.md` (see Output location above) listing:
- Total rules extracted, broken down by confidence level
- Any cross-referenced notifications that weren't available and may need follow-up
- Any rules where the document's effective date conflicts with or supersedes prior rules (if prior context is available)

## Critical guardrails

- **Never invent rates, thresholds, or conditions not present in the source.** If a value is unclear or missing, mark as `needs_review` with a note rather than guessing.
- **Always preserve source traceability** — every rule must map back to a clause/section reference.
- **Treat this output as a draft for engineer + tax-counsel review**, not a final implementation. State this explicitly in the summary.
- Respect copyright: paraphrase regulatory text; quotes under 15 words only, one per source section.