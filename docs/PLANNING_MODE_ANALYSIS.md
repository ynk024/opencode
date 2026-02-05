# Planning Mode Analysis: Choice-Based Clarification Questions

## Overview

This document provides a comprehensive analysis of how the planning mode works in OpenCode, with a specific focus on choice-based clarification questions that allow the AI agent to interact with users during the planning phase.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Question System Flow](#question-system-flow)
3. [Planning Mode Configuration](#planning-mode-configuration)
4. [Choice-Based Questions](#choice-based-questions)
5. [UI Components](#ui-components)
6. [Code Flow Diagrams](#code-flow-diagrams)

---

## Architecture Overview

The planning mode in OpenCode is built on a modular architecture with several key components:

### Core Components

1. **Question System** (`packages/opencode/src/question/index.ts`)
   - Manages asking and answering questions
   - Handles question state and lifecycle
   - Publishes events via the Bus system

2. **Planning Tools** (`packages/opencode/src/tool/plan.ts`)
   - `PlanEnterTool`: Switches to plan mode
   - `PlanExitTool`: Exits plan mode and switches to build agent

3. **Question Tool** (`packages/opencode/src/tool/question.ts`)
   - Generic tool for asking questions during execution
   - Used by all agents (when permitted)

4. **Agent Configuration** (`packages/opencode/src/agent/agent.ts`)
   - Defines agent behaviors and permissions
   - Configures which tools are available to each agent

---

## Question System Flow

### 1. Question Definition Schema

The question system uses Zod schemas to define structure:

```typescript
// packages/opencode/src/question/index.ts

export const Option = z.object({
  label: z.string().describe("Display text (1-5 words, concise)"),
  description: z.string().describe("Explanation of choice"),
})

export const Info = z.object({
  question: z.string().describe("Complete question"),
  header: z.string().describe("Very short label (max 30 chars)"),
  options: z.array(Option).describe("Available choices"),
  multiple: z.boolean().optional().describe("Allow selecting multiple choices"),
  custom: z.boolean().optional().describe("Allow typing a custom answer (default: true)"),
})
```

**Key Fields:**
- `question`: The full question text shown to the user
- `header`: Short label for tabs (when multiple questions)
- `options`: Array of choice objects with label and description
- `multiple`: Allow multi-select (default: false)
- `custom`: Allow custom text input (default: true)

### 2. Asking a Question

Questions are asked using the `Question.ask()` function:

```typescript
const answers = await Question.ask({
  sessionID: ctx.sessionID,
  questions: [
    {
      question: "Would you like to switch to plan mode?",
      header: "Plan Mode",
      custom: false,
      options: [
        { label: "Yes", description: "Switch to plan agent for research and planning" },
        { label: "No", description: "Stay with build agent to continue making changes" },
      ],
    },
  ],
  tool: ctx.callID ? { messageID: ctx.messageID, callID: ctx.callID } : undefined,
})
```

**Process:**
1. Generate unique question ID
2. Store question in pending state
3. Publish `question.asked` event via Bus
4. Return a Promise that resolves when answered
5. Promise blocks tool execution until user responds

### 3. Question State Management

The question system maintains state via `Instance.state()`:

```typescript
const pending: Record<string, {
  info: Request
  resolve: (answers: Answer[]) => void
  reject: (e: any) => void
}> = {}
```

- Questions are stored by their unique ID
- Each has resolver/rejector callbacks
- State is cleared when answered or rejected

### 4. Event Flow

```
Agent Tool Call
    ↓
Question.ask()
    ↓
Store in pending state
    ↓
Publish "question.asked" event
    ↓
UI receives event via sync
    ↓
User interacts with UI
    ↓
UI calls SDK reply/reject
    ↓
Question.reply() or Question.reject()
    ↓
Resolve/reject Promise
    ↓
Tool execution continues
```

---

## Planning Mode Configuration

### Agent Permissions

The plan agent has specific permission rules defined in `packages/opencode/src/agent/agent.ts`:

```typescript
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      question: "allow",           // Can ask questions
      plan_exit: "allow",          // Can exit plan mode
      external_directory: {
        [path.join(Global.Path.data, "plans", "*")]: "allow",
      },
      edit: {
        "*": "deny",               // No edits by default
        [path.join(".opencode", "plans", "*.md")]: "allow",  // Except plan files
      },
    }),
  ),
  mode: "primary",
  native: true,
}
```

**Key Restrictions:**
- No file editing except plan markdown files
- Can use question tool
- Can use plan_exit tool
- Read-only operations allowed (grep, list, read, etc.)

### System Prompt

The plan mode has a specific system reminder (`packages/opencode/src/session/prompt/plan.txt`):

```
CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. STRICTLY FORBIDDEN:
ANY file edits, modifications, or system changes.
```

**Responsibilities:**
- Think, read, search, and construct well-formed plans
- Ask clarifying questions
- Research and analyze without making changes
- Present comprehensive plans before implementation

---

## Choice-Based Questions

### Planning Mode Questions

#### 1. Plan Enter Tool

**Location:** `packages/opencode/src/tool/plan.ts` - `PlanEnterTool`

**Purpose:** Ask user to switch to plan mode

**Question Structure:**
```typescript
{
  question: `Would you like to switch to the plan agent and create a plan saved to ${plan}?`,
  header: "Plan Mode",
  custom: false,  // No custom answers allowed
  options: [
    { label: "Yes", description: "Switch to plan agent for research and planning" },
    { label: "No", description: "Stay with build agent to continue making changes" },
  ],
}
```

**Behavior:**
- If "Yes": Creates new user message with agent="plan", switches mode
- If "No": Throws `Question.RejectedError`, tool call fails

#### 2. Plan Exit Tool

**Location:** `packages/opencode/src/tool/plan.ts` - `PlanExitTool`

**Purpose:** Ask user to switch from plan to build mode

**Question Structure:**
```typescript
{
  question: `Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?`,
  header: "Build Agent",
  custom: false,  // No custom answers allowed
  options: [
    { label: "Yes", description: "Switch to build agent and start implementing the plan" },
    { label: "No", description: "Stay with plan agent to continue refining the plan" },
  ],
}
```

**Behavior:**
- If "Yes": Creates new user message with agent="build", switches mode
- If "No": Throws `Question.RejectedError`, stays in plan mode

### Generic Question Tool

**Location:** `packages/opencode/src/tool/question.ts`

The question tool is available to all agents (when permitted) and allows asking arbitrary questions:

```typescript
export const QuestionTool = Tool.define("question", {
  description: DESCRIPTION,
  parameters: z.object({
    questions: z.array(Question.Info.omit({ custom: true }))
  }),
  async execute(params, ctx) {
    const answers = await Question.ask({
      sessionID: ctx.sessionID,
      questions: params.questions,
      tool: ctx.callID ? { messageID: ctx.messageID, callID: ctx.callID } : undefined,
    })
    // Returns formatted answers
  },
})
```

**Features:**
- Can ask multiple questions in one call
- Supports single or multi-select
- Default enables custom answers (unless custom: false)
- Returns array of answers (each answer is array of labels)

**Tool Description** (`packages/opencode/src/tool/question.txt`):
```
Use this tool when you need to ask the user questions during execution:
1. Gather user preferences or requirements
2. Clarify ambiguous instructions
3. Get decisions on implementation choices
4. Offer choices to the user about what direction to take
```

### Answer Format

Questions return answers as an array of label arrays:

```typescript
// Single question, single select
answers = [["Yes"]]

// Multiple questions
answers = [["Build"], ["Dev"]]

// Single question, multi-select
answers = [["Option 1", "Option 2", "Custom answer"]]
```

---

## UI Components

### 1. Question Prompt Component

**Location:** `packages/ui/src/components/message-part.tsx` - `QuestionPrompt`

**Responsibilities:**
- Render question UI with options
- Handle user interaction (clicks, custom input)
- Support single/multi-select modes
- Support multiple questions with tabs
- Submit answers to SDK

**Key Features:**

#### Single vs Multi-Question Mode

```typescript
const single = createMemo(() => 
  questions().length === 1 && questions()[0]?.multiple !== true
)
```

- **Single mode**: Auto-submit on selection
- **Multi mode**: Show tabs, review screen, manual submit

#### Option Selection

```typescript
function pick(answer: string, custom: boolean = false) {
  const answers = [...store.answers]
  answers[store.tab] = [answer]
  setStore("answers", answers)
  
  if (single()) {
    // Auto-submit for single questions
    data.replyToQuestion?.({
      requestID: props.request.id,
      answers: [[answer]],
    })
    return
  }
  // Move to next question
  setStore("tab", store.tab + 1)
}
```

#### Custom Answer Input

When `custom` is not explicitly `false`, users can type their own answer:

```typescript
<button 
  data-slot="question-option"
  onClick={() => selectOption(options().length)}
>
  <span data-slot="option-label">
    {i18n.t("ui.messagePart.option.typeOwnAnswer")}
  </span>
</button>
```

Shows input form when clicked, allows text entry.

### 2. Question Display (Completed)

**Location:** `packages/ui/src/components/message-part.tsx` - Tool Registry

Once answered, questions are displayed in a collapsed format:

```typescript
ToolRegistry.register({
  name: "question",
  render(props) {
    const answers = createMemo(() => (props.metadata.answers ?? []) as QuestionAnswer[])
    const completed = createMemo(() => answers().length > 0)
    
    return (
      <BasicTool {...props} defaultOpen={completed()}>
        <For each={questions()}>
          {(q, i) => {
            const answer = () => answers()[i()] ?? []
            return (
              <div>
                <div>{q.question}</div>
                <div>{answer().join(", ") || "Unanswered"}</div>
              </div>
            )
          }}
        </For>
      </BasicTool>
    )
  },
})
```

### 3. Data Synchronization

**Location:** `packages/app/src/context/global-sync.tsx`

The app maintains sync with backend state:

```typescript
data: {
  question?: {
    [sessionID: string]: QuestionRequest[]
  }
}
```

Questions are stored per session and rendered when pending.

**Location:** `packages/app/src/pages/directory-layout.tsx`

SDK methods are connected to UI callbacks:

```typescript
const replyToQuestion = (input: { requestID: string; answers: QuestionAnswer[] }) =>
  sdk.client.question.reply(input)

const rejectQuestion = (input: { requestID: string }) => 
  sdk.client.question.reject(input)
```

---

## Code Flow Diagrams

### Complete Question Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent Execution Context                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Tool Call (plan_enter / plan_exit / question)                  │
│  - Prepares question data                                        │
│  - Calls Question.ask()                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Question.ask() - packages/opencode/src/question/index.ts       │
│  1. Generate unique question ID (Identifier.ascending)           │
│  2. Create Promise with resolve/reject                           │
│  3. Store in pending state                                       │
│  4. Publish Bus.Event.Asked                                      │
│  5. Return Promise (blocks tool execution)                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Bus System                                                      │
│  - Broadcasts question.asked event                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  SDK/WebSocket Sync                                              │
│  - Event propagated to client                                    │
│  - Updates store.question[sessionID]                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  UI Rendering - packages/ui/src/components/session-turn.tsx     │
│  - Detects new question in store                                 │
│  - Renders QuestionPrompt component                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  QuestionPrompt Component                                        │
│  packages/ui/src/components/message-part.tsx                     │
│  - Displays question text                                        │
│  - Renders option buttons                                        │
│  - Handles custom input (if enabled)                             │
│  - Manages multi-question tabs                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  User Interaction                                                │
│  - Clicks option button                                          │
│  - OR types custom answer                                        │
│  - OR dismisses question                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────┴─────────┐
                    ↓                   ↓
    ┌───────────────────────┐   ┌──────────────────┐
    │  User Selected Answer │   │  User Dismissed  │
    └───────────────────────┘   └──────────────────┘
                    ↓                   ↓
    ┌───────────────────────┐   ┌──────────────────────────┐
    │ replyToQuestion()     │   │ rejectQuestion()         │
    │ sdk.client.question   │   │ sdk.client.question      │
    │   .reply({            │   │   .reject({requestID})   │
    │     requestID,        │   └──────────────────────────┘
    │     answers           │               ↓
    │   })                  │   ┌──────────────────────────┐
    └───────────────────────┘   │ Question.reject()        │
                    ↓            │ - Remove from pending    │
    ┌───────────────────────┐   │ - Publish rejected event │
    │ Question.reply()      │   │ - Call reject()          │
    │ - Remove from pending │   └──────────────────────────┘
    │ - Publish replied evt │               ↓
    │ - Call resolve()      │   ┌──────────────────────────┐
    └───────────────────────┘   │ Promise Rejected         │
                    ↓            │ throws RejectedError     │
    ┌───────────────────────┐   └──────────────────────────┘
    │ Promise Resolved      │               ↓
    │ answers returned      │   ┌──────────────────────────┐
    └───────────────────────┘   │ Tool Execution Fails     │
                    ↓            │ Agent handles error      │
    ┌───────────────────────┐   └──────────────────────────┘
    │ Tool Execution        │
    │ Continues             │
    │ - Process answers     │
    │ - Take action         │
    │ - Return result       │
    └───────────────────────┘
                    ↓
    ┌───────────────────────┐
    │ Agent Continues       │
    │ Execution             │
    └───────────────────────┘
```

### Plan Mode Transition Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      Build Agent Active                          │
│  User: "Let's create a plan first"                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Agent decides to call plan_enter tool                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  PlanEnterTool.execute()                                        │
│  - Asks: "Switch to plan agent?"                                │
│  - Options: Yes / No                                             │
│  - custom: false (strict choice)                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    [User Question Flow]
                              ↓
                    ┌─────────┴─────────┐
                    ↓                   ↓
        ┌───────────────────┐   ┌───────────────────┐
        │  User: "Yes"      │   │  User: "No"       │
        └───────────────────┘   └───────────────────┘
                    ↓                   ↓
        ┌───────────────────┐   ┌───────────────────┐
        │ Create User Msg   │   │ Throw Rejected    │
        │ agent: "plan"     │   │ Stay in build     │
        │ text: "Switch to  │   └───────────────────┘
        │  plan mode..."    │
        └───────────────────┘
                    ↓
        ┌───────────────────┐
        │ Session Updated   │
        │ - New message     │
        │ - Agent: plan     │
        └───────────────────┘
                    ↓
        ┌───────────────────┐
        │ Plan Agent Active │
        │ - Read-only mode  │
        │ - Can ask Qs      │
        │ - Creates plan    │
        └───────────────────┘
                    ↓
        User: "Plan complete"
                    ↓
        ┌───────────────────┐
        │ plan_exit called  │
        └───────────────────┘
                    ↓
        ┌───────────────────┐
        │ Ask: "Switch to   │
        │ build agent?"     │
        │ Yes / No          │
        └───────────────────┘
                    ↓
            [User Question Flow]
                    ↓
        ┌───────────────────┐
        │ If Yes: Switch to │
        │ build agent       │
        │ agent: "build"    │
        └───────────────────┘
                    ↓
        ┌───────────────────┐
        │ Build Agent Active│
        │ Can edit files    │
        │ Implements plan   │
        └───────────────────┘
```

---

## Key Insights

### 1. Blocking Behavior

Questions **block tool execution** by returning a Promise that only resolves when answered. This ensures:
- Agent waits for user input
- No race conditions
- Sequential flow control

### 2. Custom Answers

By default, `custom: true` allows users to type their own answer. This can be disabled for strict binary choices:

```typescript
custom: false  // Only predefined options allowed
```

### 3. Multi-Select Support

Questions can allow multiple selections:

```typescript
multiple: true  // User can pick multiple options
```

Answers for multi-select questions are still arrays:
```typescript
[["Option 1", "Option 2", "Custom"]]
```

### 4. Error Handling

When users dismiss/reject a question:
- Promise rejects with `Question.RejectedError`
- Tool execution fails gracefully
- Agent can catch and handle the rejection

### 5. State Isolation

Questions are stored per-session:
- Each session has independent question queue
- Questions don't interfere between sessions
- Cleanup happens on session end

### 6. Permission Control

Agents can have question permission:
```typescript
permission: {
  question: "allow" | "deny" | "ask"
}
```

- `build` agent: `question: "allow"`
- `plan` agent: `question: "allow"`
- `explore` agent: `question: "deny"` (fast, no interaction)

---

## Summary

The planning mode's choice-based clarification system is a sophisticated event-driven architecture that:

1. **Blocks agent execution** until user responds
2. **Provides type-safe** question/answer schemas
3. **Supports flexible UI** with single/multi-select, custom input, tabs
4. **Integrates seamlessly** with agent permissions
5. **Maintains clean state** with proper lifecycle management
6. **Enables rich interactions** during planning phase

The key innovation is using **Promises as synchronization primitives** between the agent execution context and the UI layer, coordinated through a central Bus event system and WebSocket synchronization.

This allows the AI agent to pause execution, ask clarifying questions with predefined options, and resume based on user choices - all while maintaining a clean separation between the backend logic and frontend UI components.
