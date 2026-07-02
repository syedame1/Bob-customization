# Tax-doc-to-logic (Tax Circulars to Implementable Logic)

tax-doc-to-logic: Bob skill that converts indirect tax authority documents — rate notifications, circulars, exemption bulletins, and rulings — into structured decision tables and pseudocode that engineers can review and implement directly in tax calculation or determination engines.

Built for indirect tax domains: GST, VAT, Sales Tax, Customs Duty, and Excise across any jurisdiction.

---

## The problem it solves

Every time a tax authority publishes a new notification or circular, someone on the engineering team has to read it, interpret the rules, map conditions to logic, and translate that into code or configuration. That process is slow, inconsistent across team members, and nearly impossible to audit later — there's rarely a clean trail from "what the regulation says" to "what the code does."

This skill closes that gap. Give it a PDF or text document from a tax authority and it produces a structured decision table, pseudocode, and an audit summary — all with source traceability baked in — in minutes rather than hours.

---

## What it produces

For every document processed, the skill saves three files to a named output folder:

| File | Description |
|---|---|
| `decision_table.csv` | One row per atomic rule extracted, with conditions, outcome, effective dates, source clause reference, and a confidence flag |
| `pseudocode.txt` | A deterministic function taking a transaction object and an as-of date, returning a full tax determination — structured for direct translation into any language |
| `audit_summary.md` | Rule count by confidence level, cross-references that need follow-up, effective date conflicts, and anything flagged for tax counsel review |

All outputs go to `/mnt/user-data/outputs/<doc-slug>/` with fixed filenames so downstream tooling (test generators, config importers, CI checks) can rely on a stable path convention.

---

## Key design decisions

**Every rule traces back to its source clause.** Each row in the decision table and each branch in the pseudocode carries a reference to the exact section, entry, or paragraph it came from. Engineers and auditors can verify the implementation against the original document without reverse-engineering the logic.

**Ambiguity is flagged, never silently resolved.** When a rule's conditions are unclear, cross-reference an external document, or contain language that requires legal interpretation, the skill marks it `needs_review` in the decision table and inserts a `# TODO: confirm with tax counsel` comment in the pseudocode. It never guesses at a rate or threshold that isn't explicitly stated in the source.

**Rules are always date-bound.** The pseudocode function signature takes an `as_of_date` parameter so the same logic handles live transactions, historical recalculations for audits, and forward-dated rule previews for upcoming amendments — without needing separate code paths.

**Output is a draft, not a final implementation.** Every run explicitly marks its output as requiring engineer and tax counsel review before production use. The skill is a starting point that removes the manual extraction step, not a replacement for human sign-off.

---

## Skill structure

```
tax-doc-to-logic/
├── SKILL.md                          # Skill definition and workflow
├── assets/
│   └── decision_table_template.csv   # Column template for decision tables
└── references/
    └── pseudocode_patterns.md        # Pseudocode conventions and examples
```

---

## Supported document types

- Rate notifications (by HSN/SAC code, product category, or service type)
- Exemption notifications (with conditions and optional exemptions)
- Procedural circulars (reverse charge, place of supply, refund eligibility)
- Threshold and registration rules
- Revenue information bulletins and FAQ documents from tax authorities

---

## Tested with

| Document | Jurisdiction | Type | Rules extracted |
|---|---|---|---|
| GST Notification No. 14/2024 — Central Tax (Rate) *(synthetic)* | India | Rate + exemption + reverse charge | 10 rules, 11/11 test cases passing |
| Louisiana RIB 25-007 — State Sales Tax Rate | USA (Louisiana) | Rate change bulletin | Used as live skill test input |

---

## How to use

### In a chat with Bob

Paste a prompt like this:

```
Please run the tax-doc-to-logic skill on this document:
[document name and URL or upload the PDF]

Extract all atomic rules, build the decision table, generate pseudocode with
traceability comments, and produce an audit summary. Save all outputs to
/mnt/user-data/outputs/[doc-slug]/ as decision_table.csv, pseudocode.txt,
and audit_summary.md, and present the files for download when done.
```

### What to provide

- A direct PDF URL from the tax authority's official domain, or
- An uploaded PDF, or
- Pasted text of the notification/circular

### What to do with the output

1. Review `audit_summary.md` first — resolve all `needs_review` items with your tax counsel before touching the other files
2. Import `decision_table.csv` into your rules engine, config layer, or test case generator
3. Use `pseudocode.txt` as the implementation reference — translate into your target language, keeping the `rule_id` and source clause comments intact
4. Keep all three files in version control alongside the source document so the implementation is always traceable

---

## What this skill does not do

- It does not produce production-ready code. The pseudocode is a structured draft.
- It does not resolve cross-references to other notifications automatically. These are flagged in the audit summary for manual follow-up.
- It does not validate legal interpretation. Tax counsel review is always required before any output is used for compliance purposes.
- It does not replace a tax engine. It produces the rule logic that feeds into one.

---

## Contributing

When adding support for a new jurisdiction or document type:

- Add example documents to `tests/inputs/`
- Add expected decision table and pseudocode outputs to `tests/expected/`
- Update `references/pseudocode_patterns.md` if the jurisdiction introduces a new pattern (e.g. a new type of reverse charge mechanism or a cascading rate structure)
- Note any jurisdiction-specific column additions to the decision table schema in `assets/decision_table_template.csv` with a comment
