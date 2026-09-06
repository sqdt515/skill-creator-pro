---
name: skill-creator-pro
description: Create, design, and optimize high-quality agent Skills across any framework (Claude, OpenAI, Gemini, domestic platforms like Coze/Dify/百炼/元器). Use when users want to create a skill from scratch, improve an existing skill, design skill architecture, turn a workflow into a reusable agent capability, or align skill requirements through guided interviews. Supports both quick-draft and guided-alignment modes. Make sure to use this skill whenever the user mentions "create a skill", "build a skill", "skill for X", "design a skill", "improve my skill", "optimize skill", "turn this into a skill", or wants to extend agent capabilities with specialized knowledge or workflows.
---

# Skill Creator Pro

A cross-framework skill creation tool with dual modes: quick-draft for rapid prototyping and guided-alignment for production-grade skills.

## Core Workflow Map

```
User wants to create/improve a skill
        │
        ▼
┌─────────────────────────┐
│  Mode Selection         │
│  Quick / Guided?        │
│  (also auto-classify    │
│   complexity)           │
└──────────┬──────────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌─────────┐  ┌──────────────┐
│ Quick   │  │ Guided       │
│ Mode    │  │ Mode         │
│ 1 ask → │  │ Grouped      │
│ draft   │  │ alignment    │
│         │  │ (propose+    │
│         │  │  confirm)    │
└────┬────┘  └──────┬───────┘
     │               │
     └───────┬───────┘
             ▼
    Generate SKILL.md + resources
             │
             ▼
    Validate (quick_validate.py)
             │
             ▼
    Test & Iterate (cross-framework)
```

## Step 1: Mode Selection

Always start by offering both modes. If the user already described their skill in detail, auto-recommend a mode based on complexity classification (see below).

```
User: "I want to make a skill"

Agent: "Two modes:

  🚀 Quick Mode: Tell me what the skill does in one sentence. I draft immediately, then we iterate through testing. Best for simple tools, prototyping, or experts who know what they want.

  🎯 Guided Mode: I align requirements in 3 groups (Positioning → Process → Robustness). For each group I propose specifics and you confirm/correct. Best for production-grade, complex, or domain-specific skills.

Which mode? Or describe your skill and I'll recommend."
```

### Complexity Classification (Internal)

After the user describes what the skill does, classify internally (do not ask the user to classify):

| Dimension | Check |
|-----------|-------|
| Output verifiable? | File transform / data extraction / code generation → yes; writing style / art → no |
| External dependencies? | API / SDK / specific libraries → yes |
| Multi-step process? | >3 steps → yes |
| Specialized domain? | Security / finance / science / healthcare → yes |
| Edge cases? | Multiple input formats / error paths → yes |

- 0-1 "yes" → **Simple** → recommend Quick Mode
- 2-3 "yes" → **Standard** → user's choice
- 4-5 "yes" → **Complex** → recommend Guided Mode

The user can override the recommendation at any time.

## Step 2: Quick Mode

For Simple skills or when the user chooses speed.

1. Ask one question: "What should this skill do? Describe it in a sentence or two."
2. Immediately generate a complete draft using intelligent defaults (see `references/skill-design-elements.md` for default templates).
3. Run `scripts/init_skill.py` to create the folder structure.
4. Write SKILL.md and any needed resources.
5. Validate with `scripts/quick_validate.py`.
6. Move to Step 5 (Test & Iterate).

**Intelligent defaults** fill in unspecified elements: description uses pushy template, steps are inferred from goal, prerequisites use generic defaults, errors use domain-top-3, boundaries use "not for unstructured text" generic fallback. These defaults are starting points — testing will expose what needs refinement.

## Step 3: Guided Mode

For Standard/Complex skills or when the user chooses thoroughness.

This is the core innovation: **grouped alignment with propose-and-confirm**. Read `references/guided-alignment.md` for the complete protocol, slot management rules, and per-group question templates. Summary below.

### 3.1 How It Works

Instead of asking the user to answer questions from scratch, **you propose specifics based on what they've already said, and they confirm or correct**. This minimizes user cognitive load while ensuring no critical element is missed.

### 3.2 Three Alignment Groups

| Group | Elements | Complexity filter |
|-------|----------|-------------------|
| **Group 1: Positioning** | name, description (trigger scenarios + near-miss negatives), goal | All levels |
| **Group 2: Process** | prerequisites, step-by-step workflow, code/scripts needed, output format | Standard+ |
| **Group 3: Robustness** | common errors & fixes, applicability boundaries, fallback path, completion checklist, worked example | Complex only |

**Branch adaptation**: Simple skills skip Groups 2-3 (use defaults). Standard skills do Groups 1-2. Complex skills do all three. Within each group, conditional slots may be skipped based on prior answers (e.g., if "no scripts needed", skip code_example slot).

### 3.3 Propose-and-Confirm Pattern

For each group, present proposals in batch (not one question at a time):

```
Agent: "【Group 1: Positioning】Based on what you said, I propose:

  name: csv-log-cleaner
  description: Clean and validate CSV log files, generate standardized reports with anomaly statistics. Use whenever the user mentions CSV cleaning, log validation, data normalization, anomaly detection in tabular data, or wants to process structured log files — even if they don't say 'skill'.
  goal: Transform raw CSV logs into cleaned data + anomaly report, with deterministic pass/fail verification.

  Any corrections? Say 'looks good' to continue, or point out what to change."
```

User replies → update confirmed slots → move to next group.

### 3.4 Slot State Management

Maintain an internal slot state throughout the conversation. Critical rules:

- **Never lose a confirmed slot.** Once the user confirms a value, it stays confirmed. Do not overwrite it in later rounds.
- **Never regress a confirmed slot to empty.** If the model proposes a new value for a confirmed slot, ignore the proposal.
- **Terminal states**: `confirmed`, `skipped`, `defaulted`. Once terminal, the slot is frozen.
- **Proposed states**: `proposed` (waiting for user confirmation). These can be overwritten by user corrections.

See `references/guided-alignment.md` for the full slot schema and merge logic.

### 3.5 Progress Summary

After each group, show a brief summary so the user sees progress:

```
✅ Group 1 complete: name, description, goal confirmed
📋 Group 2 next: prerequisites, workflow, scripts, output format
```

### 3.6 Exit Hatch

At any point, the user can say "just draft it" — skip remaining groups, fill with defaults, and generate immediately. The user can also say "go back to group X" to revisit.

## Step 4: Generate the Skill

After alignment (or after the quick-mode question), create the skill.

### 4.1 Determine Output Location

Unless the user specifies a location, create the skill in the environment's standard skills directory. Detect it from known skill paths in the current environment. If ambiguous, ask. Common patterns:
- `~/.claude/skills/` (Claude Code)
- `~/.codex/skills/` (Codex CLI)
- `workspace/.user_skills/` (generic workspace)
- Project-local `skills/` directory

### 4.2 Initialize Folder Structure

Run the init script:
```bash
python scripts/init_skill.py <skill-name> --path <target-directory>
```

This creates:
```
skill-name/
├── SKILL.md (template)
├── scripts/ (example)
├── references/ (example)
└── assets/ (example)
```

### 4.3 Write SKILL.md

Follow the structure in `references/skill-design-elements.md`. Key rules:
- **Frontmatter**: `name` + `description` only. Description must be pushy (include trigger scenarios, not just what it does).
- **Body**: Imperative form. Start with goal/overview, then workflow, then details. Keep under 500 lines.
- **Progressive Disclosure**: Core instructions in SKILL.md. Detailed references in `references/`. Executable code in `scripts/`. Output templates in `assets/`.
- **No extraneous files**: No README.md, CHANGELOG.md, INSTALLATION.md. The skill folder is the deliverable.

### 4.4 Add Bundled Resources

Based on alignment results:
- **scripts/**: If the same code would be rewritten repeatedly (e.g., PDF rotation, data pipeline). Test scripts by running them.
- **references/**: If detailed documentation is needed only in specific scenarios (e.g., API specs, domain schemas). Include a table of contents for files >100 lines.
- **assets/**: If the skill produces output using templates (e.g., brand templates, boilerplate code, fonts).

Delete any example files that aren't needed.

### 4.5 Validate

```bash
python scripts/quick_validate.py <skill-path>
```

Checks: SKILL.md exists, valid YAML frontmatter with name+description, naming conventions (hyphen-case, max 64 chars), description constraints (no angle brackets, max 1024 chars).

## Step 5: Test & Iterate (Cross-Framework)

Testing methodology that works across any agent framework — no subagents, no special viewer, no CLI dependencies required.

### 5.1 Create Test Prompts

Write 2-3 realistic test prompts — the kind a real user would actually say. Share with the user for confirmation.

Save to `<skill-name>-workspace/evals.json`:
```json
{
  "skill_name": "csv-log-cleaner",
  "evals": [
    {
      "id": 1,
      "prompt": "Clean this sales data CSV and tell me how many rows had issues",
      "expected_output": "Cleaned CSV file + anomaly count report",
      "files": ["test_data.csv"]
    }
  ]
}
```

### 5.2 Run Paired Comparison

For each test prompt, run twice:
- **With skill**: The agent has access to the skill
- **Without skill (baseline)**: The agent solves the same task without the skill

If the framework supports parallel execution (subagents, background tasks), run both simultaneously. Otherwise run sequentially.

Save outputs to `<skill-name>-workspace/iteration-1/eval-0/with_skill/` and `without_skill/`.

### 5.3 Evaluate Results

Present both outputs to the user side by side. Ask:
- "Does the with-skill output meet expectations?"
- "Where does it fail?"
- "Is it better than the baseline?"

For objectively verifiable skills (file transforms, code generation), also run programmatic checks:
- Does the output file exist?
- Does it pass format validation?
- Are key fields correct?

### 5.4 Iterate

Based on feedback:
1. Update SKILL.md or resources
2. Re-run test cases into `iteration-2/`
3. Compare with previous iteration
4. Repeat until satisfied or no meaningful progress

**Common iteration patterns**:
- If the agent ignores the skill → improve description (make it more pushy, add trigger scenarios)
- If the agent uses the skill but produces wrong output → add more specific details, examples, or error handling
- If the agent wastes time on unproductive steps → remove parts of the skill that cause overhead
- If all test cases independently write the same helper script → bundle that script into `scripts/`

## Core Design Principles

These principles apply to every skill you create. They are derived from empirical benchmarking (SkillsBench, 87 tasks, 18 model-harness configurations).

### 1. Concise is Key

The context window is a shared resource. Only add information the agent doesn't already have. Challenge every paragraph: "Does this justify its token cost?" Prefer concise examples over verbose explanations.

**Empirical finding**: Comprehensive documentation skills gain only +0.7pp, while compact focused skills gain +19-21.5pp. More is not better.

### 2. Set Appropriate Degrees of Freedom

Match specificity to task fragility:
- **High freedom** (text instructions): Multiple valid approaches, context-dependent decisions
- **Medium freedom** (pseudocode/parameterized scripts): Preferred pattern exists, some variation OK
- **Low freedom** (specific scripts, few params): Fragile operations, consistency critical, fixed sequence required

### 3. Details > Completeness

Write what the agent cannot infer: file format quirks, parsing pitfalls, calibrated default parameters, verifier-facing constraints. Do not write general knowledge the model already possesses.

**Empirical finding**: The highest-gain skills all contain verifier-facing details and domain-specific pitfalls that are not in model pretraining.

### 4. Boundaries and Fallbacks

Every skill should state:
- **When to use it** (in description)
- **When NOT to use it** (in body)
- **What to do if it fails** (lightweight fallback)

**Empirical finding**: 13 of 87 benchmark tasks showed negative skill gains. Root cause in all cases: the skill prescribed a single "correct" pipeline without applicability boundaries or fallback paths.

### 5. Progressive Disclosure

Three-level loading:
1. **Metadata** (name + description): always in context (~100 words)
2. **SKILL.md body**: loaded when skill triggers (<500 lines ideal)
3. **Bundled resources**: loaded as needed (unlimited; scripts execute without loading)

When SKILL.md approaches 500 lines, split details into `references/` with clear pointers about when to read them. Keep references one level deep.

### 6. Pushy Descriptions

Agents tend to under-trigger skills. The description should be slightly pushy: include not just what the skill does, but specific scenarios, phrases, and contexts that should trigger it — even if the user doesn't explicitly ask for a "skill".

Bad: "Process CSV files."
Good: "Clean and validate CSV log files, generate anomaly reports. Use whenever the user mentions CSV cleaning, log validation, data normalization, tabular data processing, or wants to detect anomalies in structured data — even if they don't say 'skill'."

## Cross-Framework Compatibility

This skill creator works across agent frameworks. See `references/cross-framework.md` for framework-specific adaptation notes. Key compatibility principles:

1. **No framework-specific dependencies in generated skills**: SKILL.md is plain Markdown + YAML. Scripts are standard Python/Bash. References are plain Markdown.
2. **No required subagents or special tools**: The guided alignment and testing work in any conversational interface.
3. **Framework-specific features are optional**: If the framework supports subagents, use them for parallel testing. If not, run sequentially. If it supports a viewer, use it. If not, present results in conversation.
4. **Output location is auto-detected**: The skill finds the framework's standard skills directory, or asks the user.

## Reference Files

Read these when needed:
- `references/guided-alignment.md` — Complete guided-mode protocol: slot schema, merge logic, per-group question templates, propose-and-confirm examples
- `references/skill-design-elements.md` — Skill design element definitions, intelligent defaults, SKILL.md structure templates, per-element writing guidance
- `references/cross-framework.md` — Framework-specific notes for Claude Code, Codex CLI, Gemini CLI, Coze, Dify, 百炼, 元器, and generic agents
- `references/workflows.md` — Sequential and conditional workflow patterns for skill bodies
- `references/output-patterns.md` — Template and example patterns for skill output formatting

## Scripts

- `scripts/init_skill.py` — Initialize a new skill folder with template structure
- `scripts/quick_validate.py` — Validate skill frontmatter and naming conventions

---

**Final reminder**: The goal is a skill that another agent instance will actually use and benefit from. Write for that agent — include what it doesn't know, omit what it already does, and always give it an exit when the skill's approach doesn't fit.
