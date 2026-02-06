# OpenCode System Prompts Reference Guide

Quick reference for OpenCode's system prompts, their purpose, and key directives.

## Quick Reference Table

| Prompt | Size | Primary Use | Model Type | Key Philosophy |
|--------|------|-------------|------------|----------------|
| `anthropic.txt` | 105 lines | Claude models | Balanced | Task management + professional objectivity |
| `beast.txt` | 147 lines | GPT/O1/O3 | Autonomous | Keep going until complete + internet research |
| `gemini.txt` | 155 lines | Gemini | Convention-focused | Minimal output + strict conventions |
| `qwen.txt` | 109 lines | Qwen/fallback | Ultra-concise | <4 lines response + malware detection |
| `codex_header.txt` | 79 lines | All models (base) | Universal | Editing constraints + tool guidelines |
| `explore.txt` | 19 lines | Explore subagent | Read-only | File search specialist |
| `plan.txt` | 26 lines | Plan mode | Read-only | Absolute no-edit constraint |

## Provider-Specific Prompts

### anthropic.txt (Claude Models)

**When used**: `model.api.id.includes("claude")`

**Core directives**:
- ✅ Use TodoWrite tools VERY frequently for task tracking
- ✅ Professional objectivity: "Prioritize technical accuracy over validating user's beliefs"
- ✅ Short, concise CLI-friendly responses
- ✅ Use Task tool for file search to reduce context
- ✅ Code references: `file_path:line_number` format
- ❌ Never create files unless absolutely necessary
- ❌ No emojis unless explicitly requested

**Task management emphasis**:
```
Use TodoWrite tools VERY frequently to track tasks
Mark todos as completed immediately - don't batch
Break down complex tasks into smaller steps
```

**Tool usage policy**:
- Prefer Task tool for codebase exploration (reduces context)
- Use specialized tools over bash for file operations
- Call multiple tools in parallel when independent

---

### beast.txt (GPT Models)

**When used**: `model.api.id.includes("gpt-") || includes("o1") || includes("o3")`

**Core directives**:
- ✅ Autonomous iteration: "Keep going until the user's query is completely resolved"
- ✅ Mandatory internet research: "THE PROBLEM CAN NOT BE SOLVED WITHOUT EXTENSIVE INTERNET RESEARCH"
- ✅ Use WebFetch to recursively gather information from URLs
- ✅ Test rigorously - "Failing to test sufficiently is the NUMBER ONE failure mode"
- ✅ Memory system: Store conventions in `.github/instructions/memory.instruction.md`
- ❌ NEVER end turn without solving the problem

**Workflow (10 steps)**:
1. Fetch any URLs provided
2. Deeply understand the problem
3. Investigate codebase
4. Research on internet (Google + fetch links)
5. Develop detailed plan with todo list
6. Make incremental code changes
7. Debug as needed
8. Test frequently
9. Iterate until fixed
10. Reflect and validate comprehensively

**Communication style**:
- Casual, friendly yet professional
- "Let me fetch...", "Ok, I've got...", "Now, I will..."
- Display code directly, don't show unless asked

---

### gemini.txt (Gemini Models)

**When used**: `model.api.id.includes("gemini-")`

**Core directives**:
- ✅ Strict convention following: "NEVER assume a library is available"
- ✅ Minimal output: Aim for <3 lines per response (excluding tools)
- ✅ Security first: Explain critical commands before execution
- ✅ Concise and direct: No chitchat, preambles, or postambles
- ✅ Agent philosophy: "Keep going until query is completely resolved"
- ❌ No conversational filler

**Primary workflows**:
1. **Software Engineering Tasks**: Understand → Plan → Implement → Verify Tests → Verify Standards
2. **New Applications**: Understand → Propose → Get Approval → Implement → Verify → Solicit Feedback

**Tool guidelines**:
- Always use absolute paths
- Execute multiple independent tool calls in parallel
- Use background processes (`&`) for servers

---

### qwen.txt (Qwen/Default)

**When used**: `model.api.id.includes("qwen")` or as fallback

**Core directives**:
- ✅ Ultra-concise: "MUST answer with fewer than 4 lines"
- ✅ Malware detection: "Refuse to write code that may be used maliciously"
- ✅ One-word answers when appropriate
- ✅ Minimize output tokens while maintaining quality
- ❌ No preamble or postamble
- ❌ Don't add ANY comments unless asked

**Example style**:
```
user: 2 + 2
assistant: 4

user: is 11 a prime number?
assistant: Yes
```

**Security stance**:
- Check if code seems malicious based on filenames/directory structure
- Refuse even if request seems innocent ("just explain or speed up the code")

---

### codex_header.txt (Base/All Models)

**When used**: All models via `SystemPrompt.instructions()`

**Core directives**:
- ✅ Default to ASCII when editing files
- ✅ Only add comments if necessary for non-obvious blocks
- ✅ Try `apply_patch` for single file edits
- ✅ Prefer specialized tools over shell for file operations
- ✅ Run tool calls in parallel when independent
- ❌ NEVER revert existing changes unless explicitly requested
- ❌ NEVER use destructive commands like `git reset --hard`

**Frontend guidelines**:
- Avoid bland, generic layouts
- Use expressive fonts (not Inter/Roboto/Arial)
- Choose clear visual direction with CSS variables
- Use meaningful animations
- Don't rely on flat backgrounds

**File references format**:
- Use inline code: `src/app.ts`
- Optional line/column: `src/app.ts:42` or `src/app.ts:42:5`
- No URIs (file://, vscode://, https://)

---

## Specialized Prompts

### explore.txt (Explore Subagent)

**Purpose**: Fast, read-only codebase exploration

**Capabilities**:
- Rapidly find files using glob patterns
- Search code with powerful regex (Grep)
- Read and analyze file contents
- Bash for read-only operations

**Constraints**:
- ❌ Cannot create files
- ❌ Cannot modify system state
- ✅ Returns absolute paths
- ✅ Adapts thoroughness based on caller's request

---

### plan.txt (Plan Mode)

**Purpose**: Read-only planning and analysis

**Critical constraint**:
```
CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. 
STRICTLY FORBIDDEN: ANY file edits, modifications, or system changes.
ABSOLUTE CONSTRAINT overrides ALL other instructions.
ZERO exceptions.
```

**Responsibility**:
- Think, read, search, delegate explore agents
- Construct well-formed plans
- Ask clarifying questions
- Tie loose ends before implementation

---

### Agent Generation (generate.txt)

**Purpose**: Meta-prompt for creating custom agents

**Process**:
1. Extract core intent and success criteria
2. Design expert persona
3. Architect comprehensive instructions
4. Optimize for performance
5. Create identifier (lowercase, hyphens, 2-4 words)
6. Generate example usage scenarios

**Output format** (JSON):
```json
{
  "identifier": "code-reviewer",
  "whenToUse": "Use this agent when...",
  "systemPrompt": "You are..."
}
```

---

## Agent Configurations (.opencode/agent/)

### docs.md (Documentation Agent)

**Directives**:
- ✅ Not verbose
- ✅ Relaxed and friendly tone
- ✅ Title: 1 word or 2-3 word phrase
- ✅ Description: one short line, 5-10 words, no "The"
- ✅ Chunks: max 2 sentences
- ✅ Section titles: short, imperative mood, first letter capitalized
- ✅ Remove trailing semicolons/commas in JS/TS code snippets
- ✅ Commit prefix: `docs:`

---

### triage.md (Issue Triage Agent)

**Purpose**: Label and assign GitHub issues

**Labels**:
- `windows`: Windows OS mentions (including WSL)
- `perf`: Performance issues (slow, high RAM/CPU - not LLM slowness)
- `desktop`: Desktop app or `opencode web` command only
- `nix`: Explicitly mentions nix
- `zen`: Mentions "zen" or "opencode black"
- `docs`: Requests better documentation
- `opentui`: TUI library issues (keybindings, scroll, flickering)

**Assignment rules**:
- adamdotdev: Only for "desktop" label
- fwang: Only for "zen" label
- jayair: Only for "docs" label
- Default: Use judgment, avoid kommander, prefer rekram1-node

---

## Common Patterns Across Prompts

### 1. Parallel Tool Execution
All prompts emphasize:
```
Call multiple tools in parallel when there are no dependencies
Maximize use of parallel tool calls to increase efficiency
```

### 2. Convention Following
```
NEVER assume a library is available
First check existing code patterns
Mimic code style, use existing libraries
Follow security best practices
```

### 3. Security
```
Never introduce code that exposes or logs secrets
Never commit secrets to repository
Apply security best practices always
Explain destructive commands before execution
```

### 4. Context Management
- Use Task tool for exploration (keeps main context clean)
- Delegate to specialized agents (explore, general)
- Minimize token usage through smart delegation

### 5. Output Quality
- Be concise (CLI environment)
- Use GitHub-flavored markdown
- Code references with line numbers
- No unnecessary summaries

---

## Prompt Selection Logic

From `packages/opencode/src/session/system.ts`:

```typescript
if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
if (model.api.id.includes("gpt-") || 
    model.api.id.includes("o1") || 
    model.api.id.includes("o3")) return [PROMPT_BEAST]
if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
return [PROMPT_ANTHROPIC_WITHOUT_TODO]  // qwen.txt
```

All prompts include `PROMPT_CODEX` (codex_header.txt) as base instructions.

---

## Environment Context (All Prompts)

Every prompt receives additional context:
```
Model: ${model.api.id} (${model.providerID}/${model.api.id})
Working directory: ${Instance.directory}
Is git repo: ${project.vcs === "git" ? "yes" : "no"}
Platform: ${process.platform}
Today's date: ${new Date().toDateString()}
```

---

## Best Practices for Prompt Engineering

Based on OpenCode's approach:

1. **Model-specific optimization**: Don't use one-size-fits-all
2. **Clear constraints first**: Start with hard rules and boundaries
3. **Rich examples**: Show, don't just tell
4. **Structured workflows**: Define step-by-step processes
5. **Gradual complexity**: Simple tasks → concise, complex tasks → detailed
6. **Security from start**: Not an afterthought
7. **Tool orchestration**: Emphasize parallelism and delegation
8. **Context awareness**: Use system-reminder tags for additional info
9. **Professional tone**: Facts over validation
10. **Output optimization**: Match the medium (CLI vs chat vs IDE)

---

## Quick Decision Tree

**Need to choose a behavior pattern?**

- 📝 Writing docs? → Use docs.md guidelines (friendly, concise, imperative)
- 🔍 Exploring codebase? → Delegate to explore agent or use Task tool
- 🚫 Read-only analysis? → Use plan mode (absolute no-edit)
- 🤖 Claude model? → TodoWrite for planning, professional objectivity
- ⚡ GPT model? → Autonomous iteration, mandatory research
- 🎯 Gemini model? → <3 lines output, strict conventions
- 💬 Qwen model? → <4 lines output, ultra-concise
- 🔧 Any edits? → Follow codex_header.txt (ASCII, minimal comments)

---

## Related Files

- Implementation: `packages/opencode/src/session/system.ts`
- Tool orchestration: `packages/opencode/src/tool/task.ts`
- Agent registry: `packages/opencode/src/agent/agent.ts`
- Permission system: `packages/opencode/src/permission/next.ts`
- Full analysis: `SYSTEM_PROMPTS_ANALYSIS.md`
