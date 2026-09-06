# Cross-Framework Compatibility Guide

This skill creator is designed to work across any agent framework. This document covers framework-specific adaptations, detection strategies, and compatibility principles.

## Compatibility Principles

1. **Generated skills are framework-agnostic**: SKILL.md is plain Markdown + YAML frontmatter. Scripts are standard Python/Bash. References are plain Markdown. Any framework that reads files can use them.
2. **The creation workflow uses only conversational interaction**: Guided alignment, propose-and-confirm, and testing all work through natural language exchange. No special APIs required.
3. **Framework-specific features are optional accelerators**: Subagents, viewers, CLI tools — use them if available, fall back to sequential/conversational methods if not.
4. **Output location is auto-detected or user-specified**: Never hardcode a framework's skill directory.

---

## Framework Detection

At the start of skill creation, detect the current framework from environment signals:

| Signal | Framework |
|--------|-----------|
| `~/.claude/` exists, `claude` CLI available | Claude Code |
| `~/.codex/` exists, `codex` CLI available | Codex CLI (OpenAI) |
| `gemini` CLI available | Gemini CLI |
| Coze/扣子 web environment | Coze |
| Dify workspace | Dify |
| 百炼 (Model Studio) environment | Alibaba Cloud Model Studio |
| 元器 (Yuanqi) environment | Tencent Yuanqi |
| Generic workspace with `skills/` dir | Generic agent framework |

If detection is ambiguous, ask the user: "Which framework will this skill be used in? This affects where to save it and any framework-specific conventions."

---

## Framework-Specific Adaptations

### Claude Code

**Skill directory**: `~/.claude/skills/<skill-name>/`

**Available accelerators**:
- Subagents for parallel test execution (with-skill + baseline simultaneously)
- `claude -p` for automated description testing
- Skill auto-discovery from `~/.claude/skills/`

**Adaptation notes**:
- Claude Code uses progressive disclosure natively — the three-level loading system is built-in
- Frontmatter `name` must match directory name exactly
- `allowed-tools` frontmatter field is supported (restricts which tools the skill can use)
- Skills are auto-loaded; no manual installation needed

**Testing approach**: Use subagents to run with-skill and baseline in parallel. Save outputs to workspace. Present results directly or use a simple HTML viewer if available.

---

### Codex CLI (OpenAI)

**Skill directory**: `~/.codex/skills/<skill-name>/` or project-local `.codex/skills/`

**Available accelerators**:
- Subagents / background tasks for parallel execution
- Project-level skill override (`.codex/skills/` in project directory)

**Adaptation notes**:
- Codex CLI supports skills via the `skills/` directory convention
- Frontmatter format is compatible (name + description)
- Skills are discovered at startup

**Testing approach**: Run test cases sequentially or in background tasks. Compare outputs in conversation.

---

### Gemini CLI

**Skill directory**: Framework-specific; typically project-local `skills/` or configured path

**Available accelerators**:
- Gemini CLI has native agent capabilities
- May support skill/plugin loading depending on version

**Adaptation notes**:
- If the framework doesn't have a standard skills directory, ask the user where to place it
- The SKILL.md format is universally readable
- Scripts work regardless of framework

**Testing approach**: Sequential execution in conversation. Present with-skill vs baseline outputs for user comparison.

---

### Coze / 扣子

**Skill directory**: Coze uses "插件" (plugins) and "工作流" (workflows) rather than file-based skills. SKILL.md content can be adapted into:
- **插件描述**: The description + workflow becomes the plugin's function description and parameters
- **工作流**: The step-by-step workflow becomes a Coze workflow with nodes
- **知识库**: References become knowledge base entries

**Adaptation notes**:
- Coze is a low-code platform, not a file-based agent framework
- The guided alignment output (name, description, steps, inputs/outputs) maps directly to Coze's plugin/workflow configuration form
- Scripts may need to be converted to Coze workflow nodes (HTTP requests, code nodes)
- The "propose-and-confirm" alignment is especially valuable here because Coze users often don't know what to put in each configuration field

**Testing approach**: Use Coze's built-in preview/debug panel. Run test prompts in the preview and compare with/without the configured plugin/workflow.

---

### Dify

**Skill directory**: Dify uses "工具" (tools) and "工作流" (workflows). SKILL.md content maps to:
- **自定义工具**: Description + API schema becomes a custom tool
- **工作流/Chatflow**: Steps become workflow nodes; the guided alignment helps define each node's inputs/outputs
- **知识库**: References become knowledge documents

**Adaptation notes**:
- Dify's User Input node is conceptually similar to skill prerequisites — the guided alignment's "prerequisites" slot maps directly
- Dify's Human Input node can be used for the propose-and-confirm flow if building an automated skill-creation workflow
- Conditional logic in workflows maps to the "conditional workflow" pattern

**Testing approach**: Use Dify's workflow preview. Run test cases and inspect outputs. Compare with/without the tool/workflow.

---

### 阿里云百炼 (Model Studio)

**Skill directory**: 百炼 uses "插件" and "工作流" in the Agent 2.0 framework. SKILL.md maps to:
- **插件**: Description + function definition
- **工作流**: Steps become visual workflow nodes
- **知识库**: References become knowledge base entries

**Adaptation notes**:
- 百炼's Agent 2.0 supports plugin + workflow + knowledge composition
- The guided alignment's structured output (name, description, prerequisites, steps, output) maps to the agent configuration form
- 通义晓蜜's visual流程技能编排 supports the conditional workflow pattern

**Testing approach**: Use 百炼's built-in debug dialog. Run test prompts and compare outputs.

---

### 腾讯元器 (Yuanqi)

**Skill directory**: 元器 uses "插件" and "工作流" configuration. SKILL.md maps to:
- **插件配置**: Description + function parameters
- **工作流**: Steps become workflow nodes
- **知识库**: References become knowledge entries

**Adaptation notes**:
- 元器's creation flow (选类型→填名称简介→人设→扩展能力→调试) is a form-based approach
- The guided alignment can pre-fill each form field with proposed values, reducing user effort
- 元器 supports "手动输入" workflow creation for complex logic — the step-by-step alignment maps directly

**Testing approach**: Use 元器's preview debug panel. Run test cases and compare.

---

### Generic / Unknown Framework

**Skill directory**: Ask the user, or use a project-local `skills/` directory.

**Adaptation notes**:
- The SKILL.md format is universally readable as documentation
- Scripts work in any environment with Python/Bash
- If the framework doesn't support auto-discovery, the user can manually reference the SKILL.md content

**Testing approach**: Sequential execution in conversation. Present with-skill vs baseline outputs for user comparison. Use programmatic checks (file existence, format validation) where possible.

---

## Feature Availability Matrix

| Feature | Claude Code | Codex CLI | Gemini CLI | Coze | Dify | 百炼 | 元器 | Generic |
|---------|-------------|-----------|------------|------|------|------|------|---------|
| File-based skills | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ |
| Subagents/parallel | ✅ | ✅ | ⚠️ | ❌ | ⚠️ | ❌ | ❌ | ❌ |
| Auto skill discovery | ✅ | ✅ | ⚠️ | N/A | N/A | N/A | N/A | ⚠️ |
| Built-in test/debug | ⚠️ | ⚠️ | ⚠️ | ✅ | ✅ | ✅ | ✅ | ❌ |
| HTML viewer | ⚠️ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ |
| CLI automation | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ⚠️ |

✅ = native support | ⚠️ = partial/workaround | ❌ = not available | N/A = different paradigm

---

## Fallback Strategies

When a framework lacks a feature, use these fallbacks:

### No subagents / no parallel execution
→ Run with-skill and baseline sequentially. Save both outputs. Present side by side in conversation.

### No HTML viewer
→ Present results as text in conversation. For file outputs, show file paths and key content excerpts. Use markdown tables for comparison.

### No built-in test framework
→ Create test prompts manually. Run them in conversation. Use programmatic validation (file existence, JSON parsing, format checks) where possible.

### No standard skill directory
→ Ask the user where to save. Suggest project-local `skills/` as default. Provide the SKILL.md content inline if the user can't use files.

### No CLI for automated testing
→ Manual testing in conversation. Ask the user to verify outputs. Use qualitative feedback rather than quantitative metrics.

---

## Generated Skill Portability

A skill created with this tool can be used across frameworks because:

1. **SKILL.md** is plain Markdown + YAML — readable by any framework or human
2. **scripts/** are standard Python/Bash — executable in any environment with the runtime
3. **references/** are plain Markdown — usable as documentation or knowledge base entries
4. **assets/** are standard files — usable as templates or resources

To adapt a file-based skill to a low-code platform (Coze/Dify/百炼/元器):
- `description` → plugin/tool function description
- `prerequisites` → input fields / user input node
- `workflow steps` → workflow nodes
- `output format` → output node configuration
- `references/` → knowledge base documents
- `scripts/` → code nodes or HTTP tool nodes (may require rewriting)

The guided alignment's structured output makes this adaptation straightforward because each element is already isolated and confirmed.
