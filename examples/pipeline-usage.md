# Pipeline Usage

How to use `aus-gov-style-rules` with a rule engine to check text against AGSM conventions.

The compiled ruleset in `schema/ruleset.json` follows a pattern inspired by the [Anthropic content moderation cookbook](https://docs.anthropic.com/en/docs/about-claude/models). Rules are typed so you can run them efficiently — regex rules need no model at all; deterministic LLM rules can use a fast/cheap model; assertion rules use a capable model for judgment.

---

## Rule types and how to run them

The ruleset has three `check_type` values:

| `check_type` | How to evaluate | Model needed |
|---|---|---|
| `regex` | Pattern match against the text | None |
| `llm-deterministic` | Send judge question + text to model | Yes — fast/cheap (e.g. Haiku) |
| `llm-assertion` | Send evaluation prompt + text to model | Yes — capable (e.g. Sonnet) |

**Recommended strategy:** Run regex rules first, free and instant. Then batch the LLM rules — group `llm-deterministic` rules into one call per chunk of text (they're yes/no questions), and evaluate `llm-assertion` rules individually or in small batches.

---

## 1. Loading the ruleset

```python
import json

with open("schema/ruleset.json") as f:
    ruleset = json.load(f)

rules = ruleset["rules"]
derived_fields = ruleset["derived_fields"]

# Split by type for staged evaluation
regex_rules = [r for r in rules if r["check_type"] == "regex"]
llm_det_rules = [r for r in rules if r["check_type"] == "llm-deterministic"]
llm_assert_rules = [r for r in rules if r["check_type"] == "llm-assertion"]

print(f"Regex: {len(regex_rules)}, LLM-det: {len(llm_det_rules)}, LLM-assert: {len(llm_assert_rules)}")
# Regex: 104, LLM-det: 92, LLM-assert: 74
```

---

## 2. Running regex rules

```python
import re

def run_regex_rules(text: str, rules: list) -> list[dict]:
    """Run all regex rules against text. Returns list of violations."""
    violations = []
    for rule in rules:
        pattern = rule["when"]["value"]
        if re.search(pattern, text, re.MULTILINE | re.IGNORECASE if "(?i)" in pattern else re.MULTILINE):
            violations.append({
                "rule_id": rule["id"],
                "title": rule["title"],
                "severity": rule["severity"],
                "action": rule["action"],
                "category": rule["category"],
                "check_type": "regex",
            })
    return violations

# Example
text = "The unemployment rate fell by 0.3 percent last quarter."
violations = run_regex_rules(text, regex_rules)
for v in violations:
    print(f"[{v['severity']}] {v['rule_id']}: {v['title']}")
# [error] percentage-per-cent-two-words: Write 'per cent' as two words in Australian English
```

---

## 3. Running LLM-deterministic rules

These rules use derived fields — each is a precise yes/no question answered by the model during extraction. All `llm-deterministic` rules that fire on `true` answers to the same derived field can be batched.

```python
import anthropic

client = anthropic.Anthropic()

def evaluate_derived_fields(text: str, derived_fields: dict, model="claude-haiku-4-5") -> dict:
    """
    Ask the model to evaluate all derived fields in one structured call.
    Returns dict of {field_name: bool | None}.
    """
    # Build a compact prompt asking all yes/no questions
    questions = "\n".join(
        f'{name}: {defn["judge"]}'
        for name, defn in derived_fields.items()
    )
    
    prompt = f"""Answer each yes/no question about the following text.
Respond with a JSON object where each key is the field name and the value is true (violation), false (no violation), or null (cannot determine).

Questions:
{questions}

Text:
{text}

Respond with JSON only."""

    response = client.messages.create(
        model=model,
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}]
    )
    
    return json.loads(response.content[0].text)


def run_llm_deterministic_rules(
    text: str,
    rules: list,
    derived_fields: dict,
    model="claude-haiku-4-5"
) -> list[dict]:
    """Evaluate all LLM-deterministic rules. Returns violations."""
    # Evaluate all derived fields in one pass
    field_values = evaluate_derived_fields(text, derived_fields, model)
    
    violations = []
    for rule in rules:
        field_name = rule["when"]["field"]
        value = field_values.get(field_name)
        
        # Rule fires when field value matches the 'when' condition
        expected = rule["when"]["value"]  # typically True
        if value == expected:
            violations.append({
                "rule_id": rule["id"],
                "title": rule["title"],
                "severity": rule["severity"],
                "action": rule["action"],
                "category": rule["category"],
                "check_type": "llm-deterministic",
            })
    
    return violations
```

> **Tip:** For large documents, chunk the text into ~500-word sections and evaluate each chunk. Aggregate violations across chunks.

---

## 4. Running LLM-assertion rules

Each `llm-assertion` rule has its own `evaluation_prompt` containing `{{text}}` — a placeholder for the text being checked.

```python
def run_single_assertion(
    text: str,
    rule: dict,
    model="claude-sonnet-4-6"
) -> dict | None:
    """Evaluate one llm-assertion rule. Returns violation dict or None."""
    prompt = rule["evaluation_prompt"].replace("{{text}}", text)
    
    response = client.messages.create(
        model=model,
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}]
    )
    
    result = json.loads(response.content[0].text)
    
    if result["verdict"] == "violation":
        return {
            "rule_id": rule["id"],
            "title": rule["title"],
            "severity": rule["severity"],
            "action": rule["action"],
            "category": rule["category"],
            "check_type": "llm-assertion",
            "explanation": result["explanation"],
        }
    return None


def run_llm_assertion_rules(
    text: str,
    rules: list,
    model="claude-sonnet-4-6",
    categories: list[str] | None = None
) -> list[dict]:
    """
    Evaluate llm-assertion rules. Optionally filter by category.
    Returns list of violations.
    """
    if categories:
        rules = [r for r in rules if r["category"] in categories]
    
    violations = []
    for rule in rules:
        result = run_single_assertion(text, rule, model)
        if result:
            violations.append(result)
    
    return violations
```

---

## 5. Full pipeline

```python
def check_agsm(
    text: str,
    ruleset_path="schema/ruleset.json",
    run_regex=True,
    run_llm_det=True,
    run_assertions=True,
    assertion_categories=None,  # None = all categories
    det_model="claude-haiku-4-5",
    assert_model="claude-sonnet-4-6",
) -> dict:
    """Run all enabled rule types and return a violation report."""
    with open(ruleset_path) as f:
        ruleset = json.load(f)
    
    all_rules = ruleset["rules"]
    derived_fields = ruleset["derived_fields"]
    
    violations = []
    
    if run_regex:
        regex_rules = [r for r in all_rules if r["check_type"] == "regex"]
        violations += run_regex_rules(text, regex_rules)
    
    if run_llm_det:
        det_rules = [r for r in all_rules if r["check_type"] == "llm-deterministic"]
        violations += run_llm_deterministic_rules(text, det_rules, derived_fields, det_model)
    
    if run_assertions:
        assert_rules = [r for r in all_rules if r["check_type"] == "llm-assertion"]
        violations += run_llm_assertion_rules(text, assert_rules, assert_model, assertion_categories)
    
    # Sort by severity
    severity_order = {"error": 0, "warning": 1, "suggestion": 2}
    violations.sort(key=lambda v: severity_order.get(v["severity"], 3))
    
    return {
        "text_length": len(text),
        "violations": violations,
        "summary": {
            "total": len(violations),
            "errors": sum(1 for v in violations if v["severity"] == "error"),
            "warnings": sum(1 for v in violations if v["severity"] == "warning"),
            "suggestions": sum(1 for v in violations if v["severity"] == "suggestion"),
        }
    }


# Example
report = check_agsm(
    "The department will endeavour to make a decision prior to 31st December, 2026.",
    run_llm_det=False,   # skip for this quick demo
    run_assertions=False,
)

print(json.dumps(report, indent=2))
```

Example output:
```json
{
  "text_length": 74,
  "violations": [
    {
      "rule_id": "date-no-ordinal-suffix-in-full-dates",
      "title": "Do not use ordinal suffixes in full date expressions",
      "severity": "error",
      "action": "block",
      "category": "numbers-dates",
      "check_type": "regex"
    },
    {
      "rule_id": "date-format-day-month-year",
      "title": "Write dates in day month year order with no commas",
      "severity": "error",
      "action": "block",
      "category": "numbers-dates",
      "check_type": "regex"
    }
  ],
  "summary": {
    "total": 2,
    "errors": 2,
    "warnings": 0,
    "suggestions": 0
  }
}
```

---

## 6. CI pipeline integration

Add a pre-commit check or CI step to run the ruleset against written content before publishing:

```yaml
# .github/workflows/agsm-check.yml
name: AGSM Style Check
on: [pull_request]

jobs:
  style-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run AGSM regex rules
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          python scripts/check_agsm.py \
            --files "content/**/*.md" \
            --run-regex \
            --fail-on-errors
```

---

## 7. Category-targeted usage

For a narrower check (e.g. inclusive language review only):

```python
report = check_agsm(
    text,
    run_regex=True,          # catches specific prohibited terms
    run_llm_det=False,       # skip for speed
    run_assertions=True,
    assertion_categories=["inclusive-language"],
    assert_model="claude-sonnet-4-6",
)
```

Useful category groups:

| Use case | Categories |
|---|---|
| Quick mechanical check | `grammar-punctuation`, `numbers-dates` |
| Web content review | `plain-english`, `structuring` |
| Inclusive language audit | `inclusive-language` |
| Policy document review | `capitalisation`, `referencing` |
| Full AGSM check | all categories (default) |

---

## 8. Working with the YAML source

For custom tooling that reads the YAML rule files directly:

```python
import yaml

with open("rules/numbers-dates.yaml") as f:
    rules = yaml.safe_load(f)

# Filter to deterministic rules only
deterministic = [r for r in rules["rules"] if r["type"] == "deterministic"]

# Get all pass/fail examples for testing
for rule in deterministic:
    print(f"Rule: {rule['id']}")
    for example in rule.get("test", {}).get("fail", []):
        print(f"  FAIL: {example}")
```

---

*Source: [bitsloppy/aus-gov-style-rules](https://github.com/bitsloppy/aus-gov-style-rules) · AGSM compiled 2026-09-22*
