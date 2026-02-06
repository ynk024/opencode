# OpenCode System Prompts Analysis

## Overview

This document provides a comprehensive analysis of all system prompts used in OpenCode and highlights the key features that make the coding workflow exceptional.

## System Prompts Inventory

### 1. Core Provider-Specific Prompts

OpenCode uses different prompts optimized for different AI providers:

#### **`anthropic.txt`** (Claude models)
- **Used for**: Claude models
- **Size**: ~8.2KB
- **Key Features**: 
  - Extensive TODO management system with TodoWrite tools
  - Detailed examples of task planning and execution
  - Strong emphasis on professional objectivity and technical accuracy
  - Clear tool usage policies with Task delegation
  - Code reference formatting with `file_path:line_number` pattern

#### **`beast.txt`** (GPT models)
- **Used for**: GPT-4, GPT-5, O1, O3 models
- **Size**: ~11KB
- **Key Features**:
  - Autonomous, iterative problem-solving approach
  - Mandatory internet research using WebFetch
  - "Keep going until complete" philosophy
  - Detailed workflow with 10-step process
  - Emphasis on thorough testing and validation
  - Memory system for storing conventions

#### **`gemini.txt`** (Gemini models)
- **Used for**: Google Gemini models
- **Size**: ~15.4KB
- **Key Features**:
  - Strict adherence to project conventions
  - Minimal, concise CLI-focused communication
  - Emphasis on security and safety
  - Clear separation of tools vs text communication
  - Strong focus on preventing unnecessary file creation

#### **`qwen.txt`** (Qwen and default)
- **Used for**: Qwen models and as fallback
- **Size**: ~9.7KB
- **Key Features**:
  - Ultra-concise responses (fewer than 4 lines)
  - Malware detection and refusal system
  - Minimal preamble/postamble
  - Focus on directness and brevity
  - Examples showing one-word answers when appropriate

#### **`codex_header.txt`** (Base instructions)
- **Used for**: All models (header)
- **Size**: ~7.4KB
- **Key Features**:
  - Editing constraints (ASCII preference, minimal comments)
  - Tool usage guidelines
  - Git and workspace hygiene rules
  - Frontend design principles (avoiding generic layouts)
  - Final answer structure and formatting guidelines

### 2. Agent Generation Prompt

#### **`agent/generate.txt`**
- **Purpose**: Meta-prompt for creating new agents
- **Key Features**:
  - Guides users through agent creation
  - Creates expert personas for agents
  - Defines comprehensive instruction architecture
  - Includes quality control mechanisms
  - Generates identifiers and whenToUse descriptions
  - Integrates with project-specific context (CLAUDE.md)

### 3. Specialized Prompts

#### **`plan.txt`**
- **Purpose**: Read-only planning mode
- **Key Features**:
  - Enforces ABSOLUTE no-edit constraint
  - Planning and delegation only
  - Research and analysis focus
  - Asks clarifying questions

#### **`max-steps.txt`**
- **Purpose**: Injected when step limit reached
- **Features**: Guides agent to summarize work and remaining tasks

#### **`build-switch.txt`**
- **Purpose**: Context when switching between agents
- **Size**: ~233 bytes (minimal)

### 4. Agent Configurations

Found in `.opencode/agent/` directory:

- **`docs.md`**: Technical documentation writing with relaxed, friendly tone
- **`triage.md`**: GitHub issue triage with label management and assignment rules
- **`duplicate-pr.md`**: Detecting duplicate PRs

## What Makes OpenCode's Workflow Exceptional

### 1. **Provider-Optimized Prompts**

Each AI model gets a prompt tailored to its strengths:
- **Claude**: Task management and structured planning
- **GPT**: Autonomous iteration and research
- **Gemini**: Convention adherence and security
- **Qwen**: Ultra-concise communication

This ensures optimal performance across different LLM providers.

### 2. **Parallel Tool Execution**

Emphasized across all prompts:
```
"Call multiple tools in parallel when there are no dependencies"
```
This dramatically improves performance by reducing sequential API calls.

### 3. **Task Delegation System**

The prompts heavily emphasize using the Task tool to:
- Launch specialized agents for specific tasks
- Keep main context clean
- Enable parallel work execution
- Separate concerns (explore, execute, review)

### 4. **Progressive Disclosure of Information**

Prompts encourage:
- Using explore agents for codebase questions
- Using task agents for command execution
- Only bringing necessary information into main context
- Minimizing token usage through smart delegation

### 5. **Strong Convention Following**

All prompts emphasize:
```
"NEVER assume a library is available"
"First check existing code patterns"
"Mimic code style, use existing libraries"
"Follow security best practices"
```

### 6. **Autonomous Yet Controlled**

Balance between:
- **Autonomous**: "Keep going until complete" (Beast mode)
- **Controlled**: Read-only planning mode, permission system
- **Safe**: Security checks, no secret exposure

### 7. **Context Management**

Multiple strategies:
- **Specialized agents**: Docs, triage, review agents
- **Mode switching**: Build vs Plan modes
- **Child sessions**: Subagents work in separate contexts
- **Memory system**: Store project conventions

### 8. **Output Quality Control**

- **Concise communication**: CLI-friendly, minimal tokens
- **Code references**: `file_path:line_number` format
- **Structured responses**: Bullets, headers, code blocks
- **No unnecessary output**: Skip summaries unless requested

### 9. **Iterative Refinement**

Prompts encourage:
- Test after changes
- Verify with linters/builds
- Fix and re-test
- Handle edge cases
- Rigorous validation

### 10. **Professional Objectivity**

From Anthropic prompt:
```
"Prioritize technical accuracy and truthfulness over validating 
the user's beliefs. Focus on facts and problem-solving."
```

This prevents unhelpful agreement and promotes honest feedback.

### 11. **Security-First Approach**

All prompts include:
- Never expose/log secrets
- Apply security best practices
- Explain destructive commands
- Detect and refuse malware-related work

### 12. **Smart File Operations**

- Prefer Edit over Create
- Use specialized tools (not bash cat/sed)
- Read before editing
- Minimal file changes
- Don't create unnecessary files

### 13. **Proactive Problem Solving**

Prompts encourage:
- Creating .env files when needed
- Installing dependencies
- Running appropriate tests
- Fixing related issues
- Following up on natural next steps

### 14. **Rich Tool Ecosystem**

Prompts integrate:
- File operations (read, edit, write, glob, grep)
- Shell commands (bash)
- Git operations
- Web fetching
- TODO management
- Task delegation
- Memory storage

### 15. **Meta-Programming Capability**

The agent generation system allows:
- Creating new specialized agents
- Defining custom workflows
- Adapting to project needs
- Learning from project context (CLAUDE.md)

## Prompt Engineering Patterns

### Pattern 1: Clear Constraints

Every prompt starts with hard constraints:
```
"You are OpenCode, the best coding agent on the planet."
"IMPORTANT: You must NEVER generate or guess URLs"
```

### Pattern 2: Structured Workflows

Multi-step workflows clearly defined:
1. Understand
2. Plan
3. Implement
4. Verify
5. Iterate

### Pattern 3: Rich Examples

All prompts include extensive examples showing:
- Good vs bad practices
- Expected communication style
- Tool usage patterns
- Edge case handling

### Pattern 4: Context-Aware Behavior

```xml
<system-reminder> tags contain useful information
Tool results may include additional context
```

### Pattern 5: Gradual Complexity

Simple tasks → concise responses
Complex tasks → detailed planning and execution

## Technical Implementation

### Prompt Selection Logic (from `system.ts`)

```typescript
if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
if (model.api.id.includes("gpt-") || ...) return [PROMPT_BEAST]
if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
return [PROMPT_ANTHROPIC_WITHOUT_TODO]
```

### Environment Context Injection

Every prompt gets enriched with:
- Model ID and provider
- Working directory
- Git repo status
- Platform information
- Current date

## Key Differentiators from Other Coding Agents

1. **Provider-specific optimization** - Not one-size-fits-all
2. **Parallel execution emphasis** - Speed through concurrency
3. **Hierarchical agent system** - Primary + subagents
4. **Mode-based behavior** - Build vs Plan vs custom
5. **Context hygiene** - Keep main context clean via delegation
6. **Meta-programming** - Generate new agents on demand
7. **Convention discovery** - Learn from codebase, not assumptions
8. **Professional tone** - Honest, objective, non-validating
9. **Security-first** - Multiple safeguards built in
10. **CLI-optimized** - Concise, scannable, actionable

## Recommendations for Other Projects

1. **Optimize per provider**: Don't use same prompt for all models
2. **Enable parallelism**: Dramatically improves performance
3. **Implement task delegation**: Keeps context clean and focused
4. **Use structured workflows**: Guide agents through complex tasks
5. **Provide rich examples**: Show don't just tell
6. **Balance autonomy with control**: Modes for different safety levels
7. **Emphasize convention following**: Better code quality
8. **Include security from start**: Not an afterthought
9. **Optimize for output medium**: CLI vs chat vs IDE
10. **Enable meta-programming**: Let users create custom agents

## Conclusion

OpenCode's system prompts are exceptional because they:

- **Optimize for each AI provider's strengths**
- **Enable massive parallelism** through concurrent tool calls
- **Maintain clean context** through hierarchical agent delegation
- **Follow project conventions** rigorously
- **Balance autonomy with safety** through modes and permissions
- **Emphasize quality** through testing and verification
- **Communicate efficiently** for CLI environments
- **Support customization** through agent generation
- **Prioritize security** throughout
- **Focus on facts** over validation

The combination of these factors creates a coding workflow that is fast, reliable, secure, and highly adaptable to different use cases and coding styles.
