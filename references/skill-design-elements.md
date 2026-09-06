# Skill Design Elements

Complete reference for every element of a well-designed skill: definitions, intelligent defaults, writing guidance, and templates.

## Element Overview

| Element | Required | Complexity | Location |
|---------|----------|------------|----------|
| name | ✅ | All | frontmatter |
| description | ✅ | All | frontmatter |
| goal/overview | ✅ | All | body |
| prerequisites | ⭕ | Standard+ | body |
| workflow/steps | ✅ | All | body |
| code examples / scripts | ⭕ | Standard+ | body + scripts/ |
| output format | ⭕ | Standard+ | body |
| common errors | ⭕ | Complex | body |
| boundaries | ⭕ | Complex | body |
| fallback | ⭕ | Complex | body |
| completion checklist | ⭕ | Complex | body |
| worked example | ⭕ | Complex | body + examples/ |
| references | ⭕ | Any | references/ |
| assets | ⭕ | Any | assets/ |

---

## Frontmatter Elements

### name

**Definition**: Hyphen-case identifier for the skill. Must match the directory name.

**Rules**:
- Lowercase letters, digits, and hyphens only
- No leading/trailing hyphens, no consecutive hyphens
- Max 64 characters
- Descriptive but concise

**Examples**:
- ✅ `pdf-form-filler`
- ✅ `csv-log-cleaner`
- ✅ `brand-docx-generator`
- ❌ `PDF Form Filler` (spaces, uppercase)
- ❌ `my-skill` (not descriptive)
- ❌ `a-very-long-skill-name-that-exceeds-the-maximum-character-limit-of-sixty-four` (too long)

**Intelligent default**: Extract keywords from the goal. "Clean CSV logs and generate reports" → `csv-log-cleaner`.

---

### description

**Definition**: The primary triggering mechanism. The agent reads only name + description to decide whether to use the skill. All "when to use" information goes here, NOT in the body.

**Rules**:
- Include WHAT the skill does
- Include WHEN to use it (specific scenarios, phrases, file types, tasks)
- Be slightly pushy — agents under-trigger skills
- Max 1024 characters
- No angle brackets (`<` or `>`)

**Template**:
```
<What it does in one clause>. <Key capabilities>. Use whenever the user mentions 
<keyword 1>, <keyword 2>, <keyword 3>, or wants to <action> — even if they don't 
explicitly ask for a "skill".
```

**Bad vs Good**:

| Bad | Good |
|-----|------|
| "Process CSV files" | "Clean and validate CSV log files, generate anomaly reports with pass/fail statistics. Use whenever the user mentions CSV cleaning, log validation, data normalization, tabular data processing, or wants to detect anomalies in structured data — even if they don't say 'skill'." |
| "PDF tools" | "Extract text, merge, split, rotate, and fill forms in PDF documents. Use whenever the user works with PDF files for extraction, manipulation, form filling, or conversion — even if they don't explicitly request a 'PDF skill'." |
| "Write documents" | "Create and edit professional Word documents (.docx) with brand styling, tracked changes, comments, and template compliance. Use whenever the user mentions document creation, Word editing, report writing, proposal generation, or needs formatted .docx output — even if they don't say 'skill'." |

**Near-miss negatives** (for Complex skills): If there are adjacent domains where the skill should NOT trigger, implicitly distinguish by being specific about what it DOES. E.g., a "SEC filing analyzer" should mention "SEC filings, 10-K, 10-Q, financial reports" not just "financial documents" (which could trigger for invoices).

---

## Body Elements

### goal / overview

**Definition**: 1-2 sentences at the top of the body stating what the skill enables the agent to do.

**Template**:
```
# <Skill Title>

<One sentence: what this skill enables>.
<One sentence: when to use it / what problems it solves>.
```

**Example**:
```
# CSV Log Cleaner

Clean and validate raw CSV log files, producing standardized cleaned data plus an anomaly report.
Use this when working with tabular log data that requires field validation, format normalization, and error statistics.
```

**Writing guidance**:
- Imperative or declarative, not question form
- No "This skill will help you..." — state directly
- Include the core value proposition

---

### prerequisites

**Definition**: What must be true before the skill's workflow can execute. Environment, dependencies, input requirements, access permissions.

**Template**:
```
## Prerequisites

- <Environment requirement>
- <Dependency/library requirement>
- <Input file location/format>
- <Access/permission requirement>
```

**Example**:
```
## Prerequisites

- Python 3.10+ with pandas and pydantic installed
- Input CSV files in /workspace/input/ directory
- Files must be UTF-8 encoded (run fix_encoding.py first if not)
- Write access to /workspace/output/
```

**Intelligent defaults by domain**:
- Data processing: Python 3.10+, relevant library (pandas/numpy), input directory
- PDF: Python 3.10+, pdfplumber/PyPDF2, input PDF file
- DOCX: Python 3.10+, python-docx, template file
- API integration: API key in environment variable, network access
- Web/browser: Browser Use capability, target URL accessible

**When to skip**: If the skill is pure text instructions with no dependencies (e.g., "commit message format guide"), prerequisites can be omitted or reduced to "No external dependencies."

---

### workflow / steps

**Definition**: The ordered sequence of actions the agent should take. This is the core procedural knowledge that distinguishes a skill from a RAG document.

**Two patterns**:

#### Sequential Workflow (most common)
```
## Workflow

1. <Step 1: action + expected outcome>
2. <Step 2: action + expected outcome>
3. <Step 3: action + expected outcome>
```

#### Conditional Workflow (for branching logic)
```
## Workflow

1. Determine the case:
   - **<Case A>?** → Follow Section A below
   - **<Case B>?** → Follow Section B below

### Section A: <Case A name>
1. <step>
2. <step>

### Section B: <Case B name>
1. <step>
2. <step>
```

**Step writing rules**:
- Each step = one action + expected outcome
- Include "how" only when non-obvious (specific commands, API calls, scripts)
- Reference bundled scripts by name: `Run scripts/detect_format.py <input>`
- 3-7 steps is typical; more suggests splitting the skill
- Use imperative form: "Run", "Check", "Generate", not "You should run"

**Example**:
```
## Workflow

1. Detect the input format by running `scripts/detect_format.py <input_file>`. 
   Output: format type (csv/json/tsv). Do not trust file extension.

2. Validate fields against `references/schema.json`. 
   - Missing required fields → record in anomaly report, do not abort
   - Date fields must be ISO 8601 (Excel serial dates are a common failure — convert with scripts/excel_date.py)

3. Generate outputs to /workspace/output/:
   - `cleaned.csv`: records that passed validation
   - `report.json`: anomaly statistics + first 20 anomalous records
```

**Empirical guidance**: Skills with detailed, specific steps (including script names, file paths, and common pitfalls) gain significantly more than skills with vague high-level guidance. The detail should be what the agent cannot infer — not generic "process the data" but "run scripts/detect_format.py because file extensions are unreliable."

---

### code examples / scripts

**Definition**: Either inline code examples in the body or bundled executable scripts in `scripts/`.

**When to bundle a script**:
- The same code would be rewritten in every invocation
- Deterministic reliability is needed
- The code is >10 lines
- The code has non-obvious logic (encoding handling, edge cases)

**When to use inline examples**:
- Short snippets (<10 lines)
- Usage examples for bundled scripts
- Configuration templates

**Bundled script rules**:
- Place in `scripts/` directory
- Make executable (`chmod +x`)
- Include a docstring with purpose, usage, and expected I/O
- Test by running before finalizing
- Reference from SKILL.md with exact invocation command

**Example script reference in SKILL.md**:
```
## Step 2: Clean data

Run the cleaning pipeline:
```bash
python scripts/run_pipeline.py --input /workspace/input/data.csv --output /workspace/output/
```

This script handles encoding detection, field validation, and report generation in one pass.
```

---

### output format

**Definition**: Specification of what the skill's output should look like. Critical for verifiable skills.

**Template** (for strict format requirements):
```
## Output Format

ALWAYS produce these outputs:

1. `<filename>`: <description>
   - Format: <format>
   - Required fields: <field 1>, <field 2>, <field 3>

2. `<filename>`: <description>
   - Format: <format>
```

**Example**:
```
## Output Format

ALWAYS produce two files in /workspace/output/:

1. `cleaned.csv`: Valid records only, same columns as input, UTF-8 encoded.

2. `report.json`: Anomaly statistics with this exact structure:
```json
{
  "total_records": 0,
  "passed": 0,
  "failed": 0,
  "pass_rate": 0.0,
  "anomalies": [
    {"row": 0, "field": "", "reason": ""}
  ]
}
```
```

**When to use "ALWAYS"**: When the output is consumed by another system or verifier. When output is for human reading only, use softer language ("A sensible default format is...").

---

### common errors

**Definition**: Top 3-5 errors the agent is likely to encounter, with fixes. This is where domain expertise shines — these are the pitfalls the model doesn't know from pretraining.

**Template**:
```
## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| <error message> | <root cause> | <specific fix> |
| <error message> | <root cause> | <specific fix> |
| <error message> | <root cause> | <specific fix> |
```

**How to predict common errors**:
1. **Environment issues**: Wrong Python version, missing dependency, permission denied
2. **Input issues**: Wrong encoding (GBK vs UTF-8), malformed data, empty files, unexpected delimiters
3. **Process issues**: API rate limits, file locks, timeouts, edge cases in the workflow (e.g., all records fail validation)
4. **Output issues**: Wrong format, missing fields, path errors

**Example**:
```
## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `UnicodeDecodeError` | File is GBK encoded (common from Excel export on Windows) | Run `scripts/fix_encoding.py <file>` first |
| All dates flagged as anomalous | Excel serial date format (integers like 45231) | Convert with `scripts/excel_date.py` before validation |
| `report.json` missing | Output directory doesn't exist | The pipeline creates it automatically; if running scripts manually, `mkdir -p /workspace/output` |
| Delimiter detection fails | File uses `;` or tab instead of comma | Pass `--delimiter` explicitly: `python scripts/run_pipeline.py --delimiter ';'` |
```

---

### boundaries

**Definition**: Explicit statement of when NOT to use this skill. Prevents the agent from applying the skill inappropriately (a top cause of negative skill gains).

**Template**:
```
## When Not to Use This Skill

Do NOT use this skill for:
- <scenario 1 where it doesn't apply>
- <scenario 2 where it doesn't apply>

For those cases, <what to do instead>.
```

**Example**:
```
## When Not to Use This Skill

Do NOT use this skill for:
- Unstructured text logs (free-form application logs without tabular structure)
- Real-time streaming data (this skill is for batch file processing)
- Single-record manual fixes (use direct editing instead)

For unstructured logs, use general text processing. For streaming data, this batch pipeline is not appropriate.
```

**Why this matters**: 13 of 87 SkillsBench tasks showed negative gains. In every case, the skill prescribed a pipeline that was correct in principle but too heavy or wrong for the task. Explicit boundaries let the agent route around the skill when it doesn't fit.

---

### fallback

**Definition**: What the agent should do if the skill's approach fails or doesn't apply. A lightweight alternative path.

**Template**:
```
## Fallback

If <failure condition> (e.g., the pipeline errors out, input format is unsupported, or validation produces 0% pass rate):
1. Do NOT continue retrying the same approach.
2. <Lightweight alternative: report the specific error to the user / try a simpler method / ask for clarification>.
```

**Example**:
```
## Fallback

If the pipeline produces 0 valid records (all data flagged as anomalous), or if the input format is not CSV/JSON/TSV:
1. Do NOT retry with different parameters.
2. Report to the user: "The input file appears to be in an unsupported format or has structural issues. Detected format: <format>. Please verify the file is a valid CSV/JSON/TSV with the expected schema."
3. If the user confirms the format, ask for a sample of the first 5 lines to diagnose.
```

---

### completion checklist

**Definition**: A checklist the agent can use to verify the task is complete. Especially valuable for skills with multiple outputs or complex workflows.

**Template**:
```
## Completion Checklist

Before considering the task complete, verify:
- [ ] <check 1: output file exists>
- [ ] <check 2: output format correct>
- [ ] <check 3: key constraint met>
- [ ] <check 4: no temporary files left behind>
```

**Example**:
```
## Completion Checklist

Before reporting completion:
- [ ] `cleaned.csv` exists in /workspace/output/ and is non-empty
- [ ] `report.json` exists and has valid JSON with all required fields
- [ ] `passed + failed == total_records` in report.json
- [ ] `pass_rate` is between 0 and 1
- [ ] No temporary files remain in /workspace/input/
```

---

### worked example

**Definition**: A concrete input → output example showing what success looks like. More effective than verbose descriptions for communicating desired output quality and format.

**Template**:
```
## Example

**Input**:
<sample input data or file content>

**Expected output**:
<sample output showing the desired format and content>

**Key things to notice**:
- <point 1 about the output>
- <point 2 about the output>
```

**Example**:
```
## Example

**Input** (`sample.csv`):
```
name,date,amount
Alice,2024-01-15,100.50
Bob,invalid-date,200.00
Charlie,2024-01-16,not-a-number
```

**Output**:

`cleaned.csv`:
```
name,date,amount
Alice,2024-01-15,100.50
```

`report.json`:
```json
{
  "total_records": 3,
  "passed": 1,
  "failed": 2,
  "pass_rate": 0.333,
  "anomalies": [
    {"row": 2, "field": "date", "reason": "invalid date format: 'invalid-date'"},
    {"row": 3, "field": "amount", "reason": "not a number: 'not-a-number'"}
  ]
}
```

**Key things to notice**: Only valid records go to cleaned.csv; ALL records are counted in the report; anomalies include row number, field, and specific reason.
```

---

## Bundled Resources

### scripts/

Executable code for deterministic/repetitive tasks. See "code examples / scripts" above.

### references/

Documentation loaded into context as needed. Use when:
- Information is too detailed for SKILL.md (>100 lines)
- Information is only needed in specific scenarios
- Multiple variants exist (e.g., AWS/GCP/Azure deployment patterns)

**Rules**:
- Include a table of contents if >100 lines
- Reference from SKILL.md with clear "when to read this" guidance
- Keep one level deep (no references/ inside references/)
- Don't duplicate information that's in SKILL.md

**Example reference pattern**:
```
## Advanced Topics

- **API reference**: See `references/api_reference.md` for complete endpoint documentation. Read when you need to make API calls beyond the basic workflow.
- **Schema details**: See `references/schema.md` for field definitions and validation rules. Read when validating input data.
```

### assets/

Files used in output, NOT loaded into context. Use when:
- The skill produces documents using templates
- Brand assets (logos, fonts) are needed
- Boilerplate code is copied into output

**Rules**:
- Don't put documentation here (that's references/)
- Don't put executable scripts here (that's scripts/)
- These files are copied/used, not read for instruction

---

## SKILL.md Structure Templates

### Template A: Workflow-Based (best for sequential processes)

```markdown
---
name: <skill-name>
description: <pushy description>
---

# <Title>

<goal overview>

## Prerequisites
- <dep 1>
- <dep 2>

## Workflow
1. <step 1>
2. <step 2>
3. <step 3>

## Output Format
<specification>

## Common Errors & Fixes
| Error | Cause | Fix |
|-------|-------|-----|

## When Not to Use
<boundaries>

## Fallback
<fallback path>

## Completion Checklist
- [ ] <check>
```

### Template B: Task-Based (best for tool collections)

```markdown
---
name: <skill-name>
description: <pushy description>
---

# <Title>

<goal overview>

## Quick Start
<most common operation, one example>

## Task Category 1: <name>
### Sub-task 1
<steps>
### Sub-task 2
<steps>

## Task Category 2: <name>
...

## Common Errors & Fixes
...
```

### Template C: Reference/Guidelines (best for standards)

```markdown
---
name: <skill-name>
description: <pushy description>
---

# <Title>

<goal overview>

## Core Principles
1. <principle>
2. <principle>

## Specifications
### <spec area 1>
<details>
### <spec area 2>
<details>

## Examples
<worked examples>
```

---

## Intelligent Defaults Quick Reference

When filling slots with defaults (Quick Mode or "just draft it"):

| Slot | Default Strategy |
|------|-----------------|
| name | Extract keywords from goal, hyphen-case |
| description | Use pushy template + goal keywords + common trigger phrases |
| goal | 1-2 sentences paraphrasing user's description |
| prerequisites | Python 3.10+ + most likely library + input/output dirs |
| steps | 3-5 steps inferred from goal: detect/input → process → validate → output |
| needs_scripts | Yes if goal mentions "process", "convert", "generate", "analyze"; No if pure guidelines |
| output_format | Specify output files + basic structure |
| common_errors | Top 3: encoding issue, missing dependency, format mismatch |
| boundaries | "Not for unstructured text or real-time streaming" (generic, adjust per domain) |
| fallback | "If pipeline fails, report error and ask for clarification" |
| checklist | Output exists + non-empty + format valid + no temp files |
