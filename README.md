# aus-gov-style-rules

**The Australian Government Style Manual (AGSM), compiled as portable, AI-native rules.**

Not an app. Just the rules — in structured formats you can drop into a system prompt, feed to a rule engine, or reference from any tool stack.

[![Licence: MIT (tooling)](https://img.shields.io/badge/Licence-MIT-blue.svg)](./LICENCE)
[![Licence: CC-BY 4.0 (rules content)](https://img.shields.io/badge/Licence-CC--BY%204.0-lightgrey.svg)](./LICENCE-CONTENT)
[![AGSM source date](https://img.shields.io/badge/AGSM%20compiled-2026--09--22-green.svg)](./AGSM-VERSION.md)

---

## What's in here

```
aus-gov-style-rules/
├── prompts/
│   ├── full.md              # Full AGSM system prompt drop-in
│   ├── quick.md             # Top ~20 mechanical rules, minimal tokens
│   └── by-category/         # Per-category prompts (grammar, numbers, etc.)
├── rules/
│   ├── grammar-punctuation.yaml
│   ├── numbers-dates.yaml
│   ├── capitalisation.yaml
│   ├── inclusive-language.yaml
│   ├── plain-english.yaml
│   ├── referencing.yaml
│   └── structuring.yaml
├── schema/
│   ├── extraction-schema.json   # JSON Schema for typed rule fields
│   └── ruleset.json             # Compiled rules (cookbook rule engine format)
└── examples/
    ├── before-after.md          # Violations and corrections
    └── pipeline-usage.md        # How to use with the Anthropic cookbook pattern
```

### Three output formats, one repo

| Format | Location | Use it for |
|---|---|---|
| System prompt drop-in | `prompts/` | Paste into Claude, ChatGPT, Copilot, or any LLM |
| Structured YAML | `rules/` | Rule engines, linters, custom tooling |
| JSON schema | `schema/` | The Anthropic cookbook rule engine pattern |

---

## Two kinds of rules

**Deterministic** — binary, consistent, no judgment needed:
- Date format: `21 September 2026` not `21/09/2026`
- Numbers: spell out one to nine; use numerals for 10 and above
- Percentage: `per cent` in body text, not `%`
- Headings: sentence case, not title case
- Lists: Oxford comma required for three or more items
- Acronyms: define on first use — `Australian Taxation Office (ATO)`
- Em dashes: spaced `—` not unspaced `—`

**LLM-assertion** — model-evaluated, requires judgment:
- Active vs passive voice
- Plain English (reading level, sentence length, nominalisations)
- Bureaucratic language and unnecessary hedging
- Inclusive language

---

## Current status

> **Phase 0 complete** — repo scaffolded.
> **Phase 1 in progress** — reading AGSM sitemap, mapping categories, beginning rule extraction.

See the [phase plan](./_project-status.md) for what's coming.

---

## How to use it (once Phase 4 is done)

### Quickest: system prompt drop-in

Copy `prompts/quick.md` into your system prompt. That's it.

### Rule engine: Anthropic cookbook pattern

The `schema/ruleset.json` file is designed for use with the
[Anthropic content moderation cookbook](https://platform.claude.com/cookbook/capabilities-content-moderation-guide)
rule engine pattern. See `examples/pipeline-usage.md` for integration details.

### Structured YAML

The `rules/` directory contains one YAML file per AGSM category. Import them into your own tooling.

---

## How rules are compiled

1. Claude reads each AGSM category page from `stylemanual.gov.au`
2. Rules are **re-expressed in original language** — the model extracts meaning and writes rules in its own words; it does not copy AGSM prose
3. Each rule cites its AGSM source URL and carries an `agsm_source_date` field
4. Deterministic rules are validated with pass/fail test cases
5. LLM-assertion rules are validated against a repair loop (cookbook pattern)

The AGSM is a living document — rules here reflect the site as of the date in [AGSM-VERSION.md](./AGSM-VERSION.md).

---

## Related: Octavius

[Octavius](https://github.com/Thomas-Amann-IPAustralia/Octavius) (IP Australia) is a full Streamlit + spaCy application for AGSM linting. It has a well-structured `library_of_rules/` directory and deterministic regex/NLP rules.

**This repo is a different thing, not a competitor:**

| | Octavius | aus-gov-style-rules |
|---|---|---|
| Type | App (Streamlit + spaCy) | Portable data (no app) |
| Rules | Deterministic (regex/NLP) | Deterministic + LLM-assertion |
| Output format | Running linter | System prompts, YAML, JSON |
| Use case | Run it locally | Drop into any tool stack |
| LLM-aware | No | Yes — designed for LLM enforcement |

If you want a linter to run on your machine, use Octavius. If you want rules you can paste into Claude or drop into a custom pipeline, use this.

---

## Copyright and licensing

The AGSM is © Commonwealth of Australia 2020, published under standard Crown copyright with no Creative Commons licence.

**Rules in this repository are not verbatim reproductions of AGSM prose.** They are re-expressed in original language compiled by AI, citing source URLs. Rules themselves (principles, facts) are not copyrightable — only specific expression is.

Each rule cites its AGSM source URL (e.g. `source: https://www.stylemanual.gov.au/grammar-punctuation-and-conventions/numbers`).

This repository is dual-licensed:
- **MIT** — tooling, scripts, schema definitions
- **CC-BY 4.0** — compiled rules content (`rules/`, `prompts/`, `schema/ruleset.json`)

Attribution: [bitsloppy/aus-gov-style-rules](https://github.com/bitsloppy/aus-gov-style-rules)

See [AGSM-VERSION.md](./AGSM-VERSION.md) for full copyright notes.

---

## Contributing

Phase 1 (rule extraction) uses Claude with a consistent schema. If you spot an error or want to add a rule category, open an issue or PR — just keep to the re-expression approach, no verbatim AGSM prose.
