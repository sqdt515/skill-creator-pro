# Guided Alignment Protocol

Complete protocol for Guided Mode: slot management, group templates, propose-and-confirm examples, and iteration logic.

## Slot Schema

Each design element is a slot with this structure:

```json
{
  "name": "description",
  "value": null,
  "status": "pending",
  "group": 1,
  "required_for": ["complex", "standard", "simple"],
  "conditional_on": null
}
```

### Slot Statuses

| Status | Meaning | Frozen? |
|--------|---------|---------|
| `pending` | Not yet addressed | No |
| `proposed` | Agent proposed a value, awaiting user confirmation | No |
| `confirmed` | User confirmed the value | **Yes** |
| `skipped` | User explicitly said "not needed" | **Yes** |
| `defaulted` | Filled with intelligent default (Quick Mode or "just draft it") | No (can be refined in iteration) |

### Slot Definitions

#### Group 1: Positioning (all complexity levels)

| Slot | Description | Default |
|------|-------------|---------|
| `name` | Hyphen-case identifier, max 64 chars | Extracted from goal keywords |
| `description` | Pushy trigger description: what it does + when to use + near-miss scenarios | Template-filled from goal |
| `goal` | 1-2 sentence statement of what the skill enables | Inferred from user description |

#### Group 2: Process (Standard+ complexity)

| Slot | Description | Conditional |
|------|-------------|-------------|
| `prerequisites` | Environment, dependencies, input requirements | — |
| `steps` | Ordered list of workflow steps | — |
| `needs_scripts` | Whether bundled scripts are needed | — |
| `script_details` | What scripts do, key functions | Only if `needs_scripts` = yes |
| `output_format` | Expected output structure/format | — |
| `needs_references` | Whether reference docs are needed | — |
| `reference_details` | What references cover | Only if `needs_references` = yes |

#### Group 3: Robustness (Complex only)

| Slot | Description |
|------|-------------|
| `common_errors` | Top 3-5 errors + fixes |
| `boundaries` | When NOT to use this skill |
| `fallback` | What to do if the skill's approach fails |
| `checklist` | Completion verification checklist |
| `worked_example` | Input → output example pair |

## Merge Logic

```python
TERMINAL = {"confirmed", "skipped"}

def merge_slots(current: dict, proposed: dict) -> dict:
    """
    The model proposes. The code decides.
    Never lose or overwrite a confirmed slot.
    """
    out = dict(current)
    for name, proposal in proposed.items():
        if name not in current:
            out[name] = proposal
            continue
        current_slot = current[name]
        if current_slot["status"] in TERMINAL:
            # Frozen — do not overwrite
            continue
        # Proposed or pending — accept the proposal
        out[name] = proposal
    return out

def is_group_complete(slots: dict, group: int) -> bool:
    """All slots in this group are terminal (confirmed/skipped/defaulted)."""
    group_slots = [s for s in slots.values() if s["group"] == group]
    return all(s["status"] in TERMINAL or s["status"] == "defaulted" 
               for s in group_slots)

def next_pending_slot(slots: dict, group: int) -> dict | None:
    """Find the next pending/proposed slot in this group that isn't conditional-skipped."""
    for slot in slots.values():
        if slot["group"] != group:
            continue
        if slot["status"] in TERMINAL or slot["status"] == "defaulted":
            continue
        if slot["conditional_on"] and not condition_met(slot["conditional_on"], slots):
            continue
        return slot
    return None
```

## Group Templates

### Group 1: Positioning

**Proposal format:**

```
【Group 1: Positioning】Based on what you described, I propose:

  name: <hyphen-case-name>
  description: <pushy description with trigger scenarios>
  goal: <1-2 sentence goal>

Any corrections? Say "looks good" to continue, or point out what to change.
```

**Description proposal formula:**
```
<What it does in one clause>. <Key capabilities>. Use whenever the user mentions 
<keyword 1>, <keyword 2>, <keyword 3>, or wants to <action> — even if they don't 
explicitly ask for a "skill".
```

**Example:**
```
name: pdf-form-filler
description: Fill interactive PDF forms with data, validate field mappings, and 
generate completed documents. Use whenever the user mentions PDF forms, fillable 
fields, form filling, document automation, or wants to populate a PDF template with 
data — even if they don't say "skill".
goal: Take a fillable PDF and data source, map data to form fields, validate the 
mapping, and output a completed PDF with verification.
```

### Group 2: Process

**Proposal format:**

```
【Group 2: Process】Based on the confirmed positioning, I propose:

  Prerequisites:
  - <env/dependency 1>
  - <env/dependency 2>

  Workflow:
  1. <step 1>
  2. <step 2>
  3. <step 3>

  Bundled scripts needed: <yes/no — if yes, what they do>
  Output format: <expected output structure>

  Any corrections? Say "looks good" to continue to Group 3, or point out changes.
```

**Step proposal rules:**
- Each step is a single action with expected outcome
- Include "how" only when non-obvious (specific commands, API calls)
- 3-7 steps is typical; more suggests the skill should be split

### Group 3: Robustness

**Proposal format:**

```
【Group 3: Robustness】Final group. I propose:

  Common errors & fixes:
  1. <error>: <fix>
  2. <error>: <fix>
  3. <error>: <fix>

  When NOT to use: <boundary statement>
  Fallback if approach fails: <lightweight alternative>

  Completion checklist:
  - [ ] <check 1>
  - [ ] <check 2>

  Worked example:
  Input: <sample input>
  Output: <sample output>

  Any corrections? Say "looks good" and I'll generate the skill.
```

**Common errors proposal method:**
Based on the domain and workflow, predict the top 3 failures:
1. Environment/dependency issues (wrong version, missing library)
2. Input format issues (unexpected encoding, malformed data)
3. Process-specific issues (API rate limits, file locks, edge cases in the workflow)

**Boundary statement formula:**
```
Do not use this skill for <what it's not for>. 
For those cases, <what to do instead>.
```

**Fallback statement formula:**
```
If <failure condition>, do not continue with this workflow. 
Instead, <lightweight alternative: report error / try simpler approach / ask user>.
```

## Handling User Responses

### Confirmation
User says "looks good", "yes", "fine", "没问题", "可以" → mark all proposed slots in current group as `confirmed`.

### Correction
User says "change X to Y", "X should be Z", "把X改成Y" → update that slot's value, mark `confirmed`. Other proposed slots remain `proposed` — re-ask if needed or infer confirmation.

### Partial response
User addresses only some slots → mark addressed ones `confirmed`, leave others `proposed`. Re-present unconfirmed slots with: "You confirmed X and Y. For Z, I proposed <value> — is that OK?"

### "Skip this" / "不需要"
User says "skip", "not needed", "不用" → mark slot as `skipped`.

### "Just draft it" / "直接生成吧"
User wants to skip remaining alignment → mark all pending slots as `defaulted` (fill with intelligent defaults), proceed to generation immediately.

### "Go back" / "回到上一组"
User wants to revisit a previous group → switch to that group. Previously confirmed slots stay confirmed but can be explicitly corrected (user says "change the name to X" → update even though confirmed).

## Complexity-Based Group Filtering

```python
def active_groups(complexity: str) -> list[int]:
    if complexity == "simple":
        return [1]  # Only positioning; rest use defaults
    elif complexity == "standard":
        return [1, 2]  # Positioning + process
    else:  # complex
        return [1, 2, 3]  # All groups
```

Within a group, conditional slots are skipped if their condition isn't met:
- If `needs_scripts` = no/skipped → skip `script_details`
- If `needs_references` = no/skipped → skip `reference_details`
- If user says "pure text processing, no dependencies" → simplify `prerequisites`

## Progress Tracking

After each group completes, output:

```
✅ Group <N> complete: <list of confirmed slots>
📋 Next: Group <N+1> (<elements>)
```

When all groups complete:

```
✅ All alignment complete. Generating skill now...
```

## Anti-Patterns

1. **Don't ask open-ended questions.** "What are the prerequisites?" forces the user to invent. Instead: "I propose prerequisites: Python 3.10+, pandas. Correct?"

2. **Don't ask one slot per turn.** Batch 2-4 related slots per message. One turn per group is the target.

3. **Don't re-propose confirmed slots.** Once confirmed, frozen. Only change if user explicitly says "change X".

4. **Don't skip Group 1 for any complexity.** Name, description, and goal are always needed.

5. **Don't make the user read walls of text.** Keep proposals concise. Use bullet points. Bold the slot names.

6. **Don't forget the exit hatch.** Always remind (implicitly or explicitly) that the user can say "just draft it" at any time.
