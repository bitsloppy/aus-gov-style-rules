# aus-gov-style-rules

**The Australian Government Style Manual (AGSM), compiled as portable, AI-native rules.**

Not an app. Just the rules — in formats you can paste into a system prompt, feed to a rule engine, or reference from any tool stack.

[![Licence: MIT (tooling)](https://img.shields.io/badge/Licence-MIT-blue.svg)](./LICENCE)
[![Licence: CC-BY 4.0 (rules content)](https://img.shields.io/badge/Licence-CC--BY%204.0-lightgrey.svg)](./LICENCE-CONTENT)
[![AGSM source date](https://img.shields.io/badge/AGSM%20compiled-2026--09--22-green.svg)](./AGSM-VERSION.md)
[![270 rules](https://img.shields.io/badge/rules-270-informational.svg)](./schema/ruleset.json)

---

## Quickstart

**Paste into any LLM** — copy [`prompts/quick.md`](./prompts/quick.md) into your system prompt. Done.

**Use the rule engine** — load [`schema/ruleset.json`](./schema/ruleset.json) and run it against text. See [`examples/pipeline-usage.md`](./examples/pipeline-usage.md).

---

## What's in here

```
aus-gov-style-rules/
│
├── prompts/
│   ├── quick.md              ← top ~20 mechanical rules, minimal tokens — paste anywhere
│   ├── full.md               ← complete all-category AGSM system prompt, ~15KB
│   └── by-category/
│       ├── grammar-punctuation.md
│       ├── numbers-dates.md
│       ├── capitalisation.md
│       ├── plain-english.md
│       ├── inclusive-language.md
│       ├── referencing.md
│       └── structuring.md
│
├── schema/
│   ├── ruleset.json          ← 270 machine-runnable rules (cookbook rule engine format)
│   └── extraction-schema.json
│
├── rules/                    ← 251 source rules in YAML, one file per category
│   ├── grammar-punctuation.yaml
│   ├── numbers-dates.yaml
│   ├── capitalisation.yaml
│   ├── plain-english.yaml
│   ├── inclusive-language.yaml
│   ├── referencing.yaml
│   └── structuring.yaml
│
└── examples/
    ├── before-after.md       ← 40+ worked violation + correction examples
    └── pipeline-usage.md     ← how to run the rule engine (Python code)
```

---

## Three ways to use it

### 1. System prompt drop-in (zero setup)

Copy [`prompts/quick.md`](./prompts/quick.md) into the system prompt of any LLM. It covers the top ~20 mechanical AGSM rules — dates, time, numbers, percentages, currency, capitalisation, plain English, inclusive language — in the most compact form.

For a complete drop-in (all 7 categories), use [`prompts/full.md`](./prompts/full.md).

For a targeted prompt (one topic), use a file from [`prompts/by-category/`](./prompts/by-category/).

```
Workflow:
  1. Open prompts/quick.md
  2. Copy the contents
  3. Paste into your LLM system prompt
  4. Done — the LLM now enforces AGSM rules
```

### 2. Rule engine (pipeline integration)

[`schema/ruleset.json`](./schema/ruleset.json) contains 270 compiled rules in a structured format suitable for automated checking.

**Rule types:**

| Type | Count | How it runs |
|---|---|---|
| `regex` | 104 | Pattern match — no model needed, instant |
| `llm-deterministic` | 92 | Yes/no judge question — fast model (e.g. Haiku) |
| `llm-assertion` | 74 | Full evaluation prompt — capable model (e.g. Sonnet) |
| **Total** | **270** | |

See [`examples/pipeline-usage.md`](./examples/pipeline-usage.md) for complete Python code.

Quick example:

```python
import json, re

with open("schema/ruleset.json") as f:
    ruleset = json.load(f)

# Run regex rules instantly — no model
text = "The unemployment rate fell by 0.3 percent last quarter."
violations = []
for rule in ruleset["rules"]:
    if rule["check_type"] == "regex":
        if re.search(rule["when"]["value"], text, re.MULTILINE):
            violations.append(rule["id"])

print(violations)
# ['percentage-per-cent-two-words']
```

### 3. YAML source (build your own tooling)

The [`rules/`](./rules/) directory contains 251 source rules in human-readable YAML, one file per AGSM category. Each rule has:

- `id` — stable kebab-case identifier
- `type` — `deterministic` or `llm-assertion`
- `title`, `description` — in plain English, not AGSM prose
- `test.pass` / `test.fail` — example text
- `severity` — `error`, `warning`, or `suggestion`
- `source` — AGSM page URL
- `agsm_source_date` — when the rule was compiled

```yaml
- id: percentage-per-cent-two-words
  type: deterministic
  category: numbers-dates
  title: Write 'per cent' as two words in Australian English
  description: >
    The spelled-out form in Australian English is always two words: per
    cent. The single-word form "percent" is not standard Australian spelling.
  test:
    pass:
      - "Fifty-five per cent of the council's revenue came from rates."
    fail:
      - "Fifty-five percent of the council's revenue came from rates."
  severity: error
  source: https://www.stylemanual.gov.au/grammar-punctuation-and-conventions/numbers-and-measurements/percentages
  agsm_source_date: "2026-09-22"
```

---

## Rule coverage

| Category | YAML rules | Ruleset entries |
|---|---|---|
| Grammar + punctuation | 48 | 61 |
| Numbers + dates | 43 | 41 |
| Capitalisation | 30 | 29 |
| Plain English | 30 | 30 |
| Inclusive language | 38 | 39 |
| Referencing | 28 | 28 |
| Structuring | 34 | 42 |
| **Total** | **251** | **270** |

> Ruleset entries exceed YAML source rules because the `llm-deterministic` and `llm-assertion` splits produce separate compiled entries from the same source.

---

## Worked examples

[`examples/before-after.md`](./examples/before-after.md) has 40+ worked before/after examples across all categories. A sample:

**Date format**
```
❌ The consultation period ends December 31, 2026.
✅ The consultation period ends 31 December 2026.
   Rule: date-format-day-month-year
```

**Nominalisation**
```
❌ Please make an application before the closing date.
✅ Please apply before the closing date.
   Rule: avoid-nominalisation
```

**Disability language**
```
❌ Disabled people can access support through this service.
✅ People with disability can access support through this service.
   Rule: disability-person-first-default
```

---

## Related: Octavius

[Octavius](https://github.com/Thomas-Amann-IPAustralia/Octavius) (IP Australia) is a full Streamlit + spaCy application for AGSM linting with deterministic regex/NLP rules.

**This repo is a different thing, not a competitor:**

| | Octavius | aus-gov-style-rules |
|---|---|---|
| Type | App (Streamlit + spaCy) | Portable data (no app) |
| Rules | Deterministic (regex/NLP) | Deterministic + LLM-assertion |
| Output | Running linter | System prompts, YAML, JSON |
| Use case | Run it locally | Drop into any tool stack |
| LLM-aware | No | Yes — evaluation prompts included |

If you want a linter to run locally, use Octavius. If you want rules to paste into Claude or drop into a pipeline, use this.

---

## How rules are compiled

1. Claude reads each AGSM category page from `stylemanual.gov.au`
2. Rules are **re-expressed in original language** — the model extracts meaning and writes rules in its own words; AGSM prose is never copied verbatim
3. Each rule cites its AGSM source URL and carries an `agsm_source_date`
4. Deterministic rules are compiled into the rule engine format with regex patterns and LLM judge questions
5. LLM-assertion rules carry evaluation prompts — self-contained instructions for a capable model to judge violations

The AGSM is a living document — rules here reflect the site as of the date in [AGSM-VERSION.md](./AGSM-VERSION.md). To update, re-extract the relevant category.

---

## Copyright and licensing

The AGSM is © Commonwealth of Australia 2020, published under standard Crown copyright with no Creative Commons licence.

**Rules in this repository are not verbatim reproductions of AGSM prose.** They are re-expressed in original language compiled by AI, citing source URLs. Rules themselves (principles, facts) are not copyrightable — only specific expression is.

This repository is dual-licensed:
- **MIT** — tooling, scripts, schema definitions
- **CC-BY 4.0** — compiled rules content (`rules/`, `prompts/`, `schema/ruleset.json`)

Attribution: [bitsloppy/aus-gov-style-rules](https://github.com/bitsloppy/aus-gov-style-rules)

See [AGSM-VERSION.md](./AGSM-VERSION.md) for full copyright notes.

---

## Contributing

Rules extracted using Claude with a consistent schema. To contribute:

- Keep to the re-expression approach — no verbatim AGSM prose
- Follow the YAML schema in `schema/extraction-schema.json`
- Include `source` URL and `agsm_source_date` on every rule
- Add pass/fail examples that unambiguously test the rule

Open an issue if you spot an error or want to add a rule category.
