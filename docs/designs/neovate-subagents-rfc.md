# RFC: Subagents System

**Status:** Draft
**Created:** 2025-11-21
**Last Updated:** 2025-11-21

## Abstract

This RFC proposes the design and implementation of a specialized subagent system for NeovateCode, inspired by Claude Code's subagent architecture. The system will enable delegation of specific tasks to specialized AI assistants that operate in isolated context windows, improving task specialization while preserving the main conversation's focus and efficiency.

## Table of Contents

1. [Product Research: Claude Code Subagents](#product-research-claude-code-subagents)
2. [Motivation](#motivation)
3. [Goals & Non-Goals](#goals--non-goals)
4. [Detailed Design](#detailed-design)
5. [Implementation Plan](#implementation-plan)
6. [Security Considerations](#security-considerations)
7. [Performance Considerations](#performance-considerations)
8. [Alternatives Considered](#alternatives-considered)
9. [Open Questions](#open-questions)
10. [References](#references)

---

## Product Research: Claude Code Subagents

### Overview

Claude Code implements a sophisticated subagent system that provides task-specific AI delegation with isolated contexts. According to their documentation: _"Each subagent operates in its own context, preventing pollution of the main conversation."_ This section provides a deep analysis of the existing implementation to inform our design decisions.

### Core Value Proposition

**Primary Benefits Observed:**

1. **Context Preservation**

   - Quote from docs: _"Each subagent operates in its own context, preventing pollution of the main conversation"_
   - Enables longer overall sessions by isolating specialized work
   - Main conversation remains focused and relevant

2. **Specialized Expertise**

   - Task-specific system prompts improve output quality
   - Fine-tuned for specific domains with higher success rates
   - Each agent can have deep, focused instructions

3. **Reusability & Collaboration**

   - Can be used across projects
   - Shareable with teams via version control
   - Standardizes workflows and best practices

4. **Flexible Security**
   - Customizable tool access levels per agent
   - Principle of least privilege enforcement
   - Reduces risk of unintended actions

### Feature Analysis

#### Core Capabilities

**1. Isolated Context Windows**

- Each subagent operates in a completely separate conversation context
- No access to main conversation history (except the passed task prompt)
- No cross-agent state sharing
- Prevents context pollution in main thread
- Clean-slate execution ensures predictability
- **Trade-off**: "Clean-slate invocations may add latency as subagents gather required context"

**2. Multi-Level Configuration System**

**Storage Locations & Priority:**

1. **Project-level** (highest priority) - `.claude/agents/*.md` for team-shared agents
2. **CLI-defined** - `--agents` JSON flag for session-specific configs
3. **User-level** - `~/.claude/agents/*.md` for personal agents
4. **Plugin-provided** (lowest priority) - Integrated via plugin manifests

**3. Configuration Format**

```markdown
---
name: identifier-name
description: Purpose and invocation criteria
tools: tool1, tool2 (optional)
model: sonnet/opus/haiku/inherit (optional)
---

System prompt defining role and behavior
```

**Field Specifications:**
| Field | Required | Details |
|-------|----------|---------|
| `name` | ✅ Yes | Lowercase with hyphens, unique identifier |
| `description` | ✅ Yes | Natural language purpose statement, triggers invocation |
| `tools` | ❌ No | Comma-separated list; inherits all if omitted |
| `model` | ❌ No | Defaults to configured subagent model setting |

**4. Tool Access Control**

**Two Configuration Options:**

Option 1: **Inherit All Tools (Default)**

- Omit the `tools` field entirely
- Agent receives all tools from main thread
- Includes MCP (Model Context Protocol) server tools
- Best for general-purpose agents

Option 2: **Granular Control**

- Specify tools as comma-separated list
- Only listed tools available to agent
- Best for security-sensitive agents

**MCP Integration:** Full support for Model Context Protocol tools, inherited by default

**MCP Tool Details:**

- **Tool name format**: `mcp__server__tool` (double underscore separators)
- **Validation timing**: Tool names validated at configuration load time, not runtime
- **Error handling**: Unknown tool names rejected with clear error messages
- **Discovery**: `/agents` command displays all available MCP tools during agent creation
- **Inheritance**: MCP tools automatically included when using "Inherit All Tools" option

**5. Model Selection**

**Model Options:**

- `sonnet`, `opus`, `haiku` - Specific models for cost/performance tuning
- `inherit` - Matches main conversation model
- Omit field - Uses default subagent model configuration

**6. Agent Nesting Prevention**

- **Critical safety mechanism** - Subagents cannot spawn other subagents
- Prevents infinite recursion and uncontrolled resource usage
- **Exception**: Special Plan agent for plan mode
- **Essential for**: System stability, cost control, predictable execution
- **Implementation**: Disable Task, TodoWrite, and TodoRead tools in subagent contexts
- **Security benefit**: Limits attack surface for prompt injection exploits

**7. Invocation Methods**

**Four Invocation Methods:**

1. **Automatic** - Claude matches tasks to agents based on descriptions
2. **Explicit** - User mentions agent by name ("use the code-reviewer subagent")
3. **@ Mention** - User types `@agent-name` in chat to invoke specific agent
   - Provides autocomplete/selection UI
   - Clear visual indicator of agent invocation
   - Familiar UX pattern from chat applications
4. **CLI** - Dynamic definition via `--agents` flag for testing/automation

#### Advanced Features

**1. Resumable Agent Execution**

- Each execution gets unique `agentId`
- Transcript stored as `agent-{agentId}.jsonl` in project directory
- Resume via `resume` parameter to continue previous sessions
- **Recording behavior**: Recording is disabled during resume to avoid duplicating messages
- **Agent types**: Both synchronous and asynchronous subagents can be resumed
- **Context restoration**: Full context from previous conversation is restored when resuming
- **Use case**: Long-running research, iterative refinement, particularly useful for analysis tasks that span multiple sessions

**2. Subagent Chaining**

- Sequence multiple subagents for complex workflows
- Results inform subsequent agent invocations

**3. Dynamic Selection**

- AI matches tasks to agents based on description similarity and context
- Documentation: _"Claude intelligently matches tasks to subagents based on context and description specificity"_

### Management Interface

**`/agents` Command** - Interactive interface for managing subagents:

- View all agents (built-in, user, project, plugin)
- Create new agents with AI assistance
- Edit existing agents (tools, prompts, configuration)
- Delete custom agents (built-in protected)
- Press 'e' to edit in external editor ($EDITOR)

**@ Mention in Chat** - Quick agent invocation surface:

- Type `@` to trigger agent autocomplete dropdown
- Shows available agents with descriptions
- Select agent to invoke with current context
- Visual badge/indicator shows which agent is handling the task

**Alternative:** Direct file editing in `.claude/agents/*.md` or `~/.claude/agents/*.md`

### Built-in Subagents

Claude Code ships with 4 built-in subagents:

**1. general-purpose** (Sonnet)

- General-purpose agent for complex tasks, searching code, multi-step execution
- Tools: All tools (\*)
- Use case: When main agent needs to delegate complex research or multi-step tasks

**2. statusline-setup** (Sonnet)

- Configures user's Claude Code status line setting
- Tools: Read, Edit
- Use case: Status line customization

**3. Explore** (Haiku)

- Fast agent for exploring codebases
- Find files by patterns, search code for keywords, answer codebase questions
- **Thoroughness levels**: "quick", "medium", "very thorough" - configurable depth of exploration
- Tools: All tools
- **Introduced**: v2.0.17
- **Context efficiency**: Haiku-powered for reduced cost on frequent codebase searches
- Use case: Quick codebase exploration without high cost, optimal for repeated exploration tasks

**4. Plan** (Sonnet)

- Specialized for plan mode codebase research
- Tools: Read, Glob, Grep, Bash
- **Nesting exception**: Only subagent that works in plan mode, special architectural exception to nesting prevention
- **Resumption capabilities**: Added in v2.0.28, can resume long-running research sessions
- **Context isolation**: Operates in separate context even during plan mode to preserve main conversation
- Use case: Research during planning phase, exploratory analysis before implementation

### Best Practices

1. _"Generate initial subagents with Claude, then customize"_
2. _"Design single-responsibility subagents rather than multipurpose ones"_
3. _"Write detailed, specific system prompts with examples"_
4. _"Limit tool access to necessary functions only"_
5. _"Version control project subagents for team collaboration"_
6. _"Include action-oriented descriptions like 'use PROACTIVELY' for better auto-delegation"_

### Performance Characteristics

**Benefit:** _"Subagents preserve main context for longer overall sessions"_

**Cost:** _"Clean-slate invocations may add latency as subagents gather required context"_

**Optimizations:** Use Haiku for simple tasks, provide context in task description, use resumable agents for iterative work

### Key Design Patterns

**1. Hierarchical Configuration with Priority Override**

- Multiple sources (Project → CLI → User → Plugin) with clear precedence
- Enables collaboration (project), testing (CLI override), personalization (user), extension (plugin)

**2. Context Isolation by Default**

- Each subagent gets separate conversation context
- Prevents pollution, enables longer main sessions
- **Trade-off:** Cold start cost, mitigated by resumable agents

**3. Secure by Default, Flexible When Needed**

- Tool inheritance as default (reduces config burden)
- Explicit restriction available for security-sensitive agents

**4. Automatic with Manual Override**

- Auto-delegation based on description matching
- Explicit invocation always available for control

**5. File-Based Configuration**

- Markdown + YAML frontmatter in `.claude/agents/`
- Git-compatible, human-readable, no database needed

### Trade-offs Analysis

**Benefits:**

- ✅ Context preservation for longer sessions
- ✅ Specialized expertise (_"Fine-tuned for specific domains with higher success rates"_)
- ✅ Reusability & team collaboration
- ✅ Flexible security via tool permissions

**Costs:**

- ❌ Cold start latency (mitigated by resumable agents, Haiku model)
- ❌ No persistent state (mitigated by transcript storage)
- ❌ Management complexity (mitigated by `/agents` UI, AI-assisted creation)
- ❌ Risk of over-delegation (mitigated by explicit invocation option)

### Implementation Insights & Lessons

1. **Simplicity enables adoption** - Minimal required fields (name + description), optional complexity
2. **Multiple scopes with clear priority required** - Project/CLI/User/Plugin hierarchy essential
3. **Tool inheritance right default for dev tools** - Unlike security tools, most agents need broad access
4. **Resumption > persistent memory** - Simpler, more predictable than stateful agents
5. **Strategic limitations improve architecture** - Nesting prevention simplifies implementation
6. **File-based + Interactive UI both needed** - Files for git, UI for ease of use
7. **AI task-matching > regex rules** - Natural language descriptions more flexible
8. **Concrete examples as important as API docs** - Example patterns lower barrier to entry

---

## Motivation

NeovateCode needs a subagent system to enable task delegation with isolated contexts and specialized expertise. Key benefits include:

- **Context preservation** - Main conversation stays focused, enabling longer sessions
- **Task specialization** - Domain-specific prompts improve output quality
- **Security controls** - Granular tool permissions per agent
- **Team collaboration** - Shareable configurations standardize workflows

This follows the proven pattern established by Claude Code's subagent architecture.

---

## Goals & Non-Goals

### Goals

#### MVP: Task Tools with Built-in Subagents

- ✅ Implement Task tool interface for LLM invocation
- ✅ Build agent executor with isolated context windows
- ✅ Implement message history management for isolated conversations
- ✅ Create tool resolver and permission system
- ✅ Build nesting prevention mechanism (core security feature)
  - Filter out Task, TodoWrite, and TodoRead tools in subagent contexts
  - Prevent subagents from spawning other subagents
  - Prevent subagents from managing main conversation todos
- ✅ Ship two built-in agents: `general-proposal` and `explore`

#### Phase 1: Custom Subagent Support

- ✅ Implement configuration file parser (YAML frontmatter + Markdown)
- ✅ Build multi-level configuration loader (project and user directories)
- ✅ Create agent registry with priority resolution
- ✅ Support per-agent model selection
- ✅ Enable custom agents to coexist with built-in agents

#### Phase 2: Interactive Agent Management

- ✅ Implement `/agents` command interface
- ✅ Build agent CRUD operations (Create, Read, Update, Delete)
- ✅ Add AI-assisted agent generation
- ✅ Implement auto-delegation with task matching
- ✅ Implement @ mention surface for agent selection in chat

### Non-Goals (Future Features)

- ⏳ Resumable agent execution - Transcript storage and resume capabilities
- ⏳ Agent chaining - Sequential composition of multiple agents
- ⏳ Persistent agent state - Agents maintaining state across invocations
- ⏳ Config file watching - Watch project-level config files for automatic reload on changes

---

## Detailed Design

### Architecture Overview

The NeovateCode subagent system follows a hierarchical architecture with clear separation between configuration loading, task delegation, and execution. The system ensures isolated execution contexts while maintaining centralized control over agent lifecycle and security.

#### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           NeovateCode CLI                                    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        Main Conversation Thread                        │  │
│  │                                                                        │  │
│  │  ┌──────────────────┐         ┌─────────────────────────────────┐    │  │
│  │  │  User Request    │────────▶│   Main AI Agent (LLM)           │    │  │
│  │  └──────────────────┘         │  - Process user request         │    │  │
│  │                               │  - Decide task delegation       │    │  │
│  │                               │  - Call Task tool when needed   │    │  │
│  │                               │  - Access to all tools          │    │  │
│  │                               └────────────┬────────────────────┘    │  │
│  │                                            │                          │  │
│  │                                            │ Task Tool Call           │  │
│  │                                            │ {subagent_type,          │  │
│  │                                            │  prompt, model}          │  │
│  │                                            ▼                          │  │
│  │  ┌─────────────────────────────────────────────────────────────┐     │  │
│  │  │              Task Tool (Agent Executor)                      │     │  │
│  │  │                                                              │     │  │
│  │  │  1. Lookup subagent config by subagent_type                 │     │  │
│  │  │  2. Check execution stack (nesting prevention)              │     │  │
│  │  │  3. Filter tools based on agent config                      │     │  │
│  │  │  4. Select model (config/inherit/default)                   │     │  │
│  │  │  5. Create isolated LLM context                             │     │  │
│  │  │  6. Execute subagent task                                   │     │  │
│  │  │  7. Store transcript (optional)                             │     │  │
│  │  │  8. Return result to main agent                             │     │  │
│  │  └─────────────────────────────────────────┬───────────────────┘     │  │
│  └────────────────────────────────────────────┼─────────────────────────┘  │
└─────────────────────────────────────────────┬─┼─────────────────────────────┘
                                              │ │
                     ┌────────────────────────┘ └────────────────────┐
                     │                                                │
         ┌───────────▼───────────┐                      ┌────────────▼────────────┐
         │   Subagent Context A  │                      │   Subagent Context B    │
         │  (e.g., "explore")    │                      │  (e.g., "general")      │
         │  ┌─────────────────┐  │                      │  ┌─────────────────┐   │
         │  │ LLM Thread      │  │                      │  │ LLM Thread      │   │
         │  │ - Fresh context │  │                      │  │ - Fresh context │   │
         │  │ - No history    │  │                      │  │ - No history    │   │
         │  │ - Only prompt   │  │                      │  │ - Only prompt   │   │
         │  └─────────────────┘  │                      │  └─────────────────┘   │
         │  ┌─────────────────┐  │                      │  ┌─────────────────┐   │
         │  │ Tool Subset     │  │                      │  │ Tool Subset     │   │
         │  │ (configured)    │  │                      │  │ (all tools)     │   │
         │  │ ❌ No Task     │  │                      │  │ ❌ No Task     │   │
         │  │ ❌ No TodoWrite│  │                      │  │ ❌ No TodoWrite│   │
         │  │ ❌ No TodoRead │  │                      │  │ ❌ No TodoRead │   │
         │  └─────────────────┘  │                      │  └─────────────────┘   │
         │  ┌─────────────────┐  │                      │  ┌─────────────────┐   │
         │  │ System Prompt   │  │                      │  │ System Prompt   │   │
         │  │ (from config)   │  │                      │  │ (from config)   │   │
         │  └─────────────────┘  │                      │  └─────────────────┘   │
         └───────────┬───────────┘                      └────────────┬───────────┘
                     │                                                │
                     │ Nesting prevention: Task, TodoWrite & TodoRead│ Nesting prevention: Task, TodoWrite & TodoRead
                     │ automatically filtered out                     │ automatically filtered out
                     │                                                │
                     ▼                                                ▼
              Result returned                                  Result returned
              to main agent                                    to main agent
```

#### Configuration Loading Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Configuration Sources (Priority Order)                │
└─────────────────────────────────────────────────────────────────────────────┘

   1️⃣ PROJECT LEVEL (Highest Priority)
   ┌──────────────────────────────────────────────────────────────────────┐
   │  Merge within project level:                                         │
   │                                                                       │
   │  Step 1: Load .neovate/agents/*.md                                   │
   │  ┌──────────────────────────┐                                        │
   │  │ .neovate/agents/         │                                        │
   │  │ ├─ code-reviewer.md      │  Loaded first                         │
   │  │ ├─ debugger.md           │                                        │
   │  │ └─ data-scientist.md     │                                        │
   │  └──────────────────────────┘                                        │
   │                                                                       │
   │  Step 2: Load .neovate/config.json (agents field)                    │
   │  ┌──────────────────────────┐                                        │
   │  │ .neovate/config.json     │                                        │
   │  │ {                        │  Merges and OVERRIDES                 │
   │  │   "agents": {            │  any matching names                   │
   │  │     "debugger": {...}    │  from Step 1                          │
   │  │   }                      │                                        │
   │  │ }                        │                                        │
   │  └──────────────────────────┘                                        │
   │                                                                       │
   │  Result: Merged project-level agents                                 │
   └──────────────────────────────────────────────────────────────────────┘

   2️⃣ USER LEVEL (Lower Priority - fallback for missing agents)
   ┌──────────────────────────────────────────────────────────────────────┐
   │  Merge within user level:                                            │
   │                                                                       │
   │  Step 1: Load ~/.neovate/agents/*.md                                 │
   │  ┌──────────────────────────┐                                        │
   │  │ ~/.neovate/agents/       │                                        │
   │  │ ├─ personal.md           │  Loaded first                         │
   │  │ └─ my-helper.md          │                                        │
   │  └──────────────────────────┘                                        │
   │                                                                       │
   │  Step 2: Load ~/.neovate/config.json (agents field)                  │
   │  ┌──────────────────────────┐                                        │
   │  │ ~/.neovate/config.json   │                                        │
   │  │ {                        │  Merges and OVERRIDES                 │
   │  │   "agents": {            │  any matching names                   │
   │  │     "personal": {...}    │  from Step 1                          │
   │  │   }                      │                                        │
   │  │ }                        │                                        │
   │  └──────────────────────────┘                                        │
   │                                                                       │
   │  Result: Merged user-level agents                                    │
   └──────────────────────────────────────────────────────────────────────┘

                     │
                     ▼
        ┌────────────────────────────────────────────────────────┐
        │  Configuration Loader                                   │
        │                                                         │
        │  1. Load user-level agents (files + config.json)        │
        │     - Merge: config.json overrides file-based           │
        │                                                         │
        │  2. Load project-level agents (files + config.json)     │
        │     - Merge: config.json overrides file-based           │
        │                                                         │
        │  3. Final merge: project overrides user                 │
        │     - Only for agents with same name                    │
        │     - User agents without conflicts remain              │
        │                                                         │
        │  4. Validate all configs                                │
        │  5. Cache parsed configs                                │
        └────────────┬───────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │     Agent Registry          │
        │  Map<name, AgentConfig>     │
        │  - Merged from both levels  │
        │  - Project overrides user   │
        │  - Config overrides files   │
        └─────────────────────────────┘
```

#### Agent Invocation Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Agent Invocation Process                            │
└─────────────────────────────────────────────────────────────────────────────┘

    User Request
         │
         ▼
   ┌──────────────────────────────────────────────────────────┐
   │           Main AI Agent (LLM)                            │
   │           Makes delegation decision internally           │
   │                                                           │
   │           If delegates → Calls Task tool with:           │
   │             - subagent_type: "explore"                   │
   │             - prompt: "Search for authentication code"   │
   │             - model: "haiku" (optional)                  │
   └───────────────────────┬──────────────────────────────────┘
                           │
                           │ Task tool invoked
                           ▼
   ┌──────────────────────────────────────────────────────────┐
   │              Task Tool Execution Pipeline                 │
   │                                                           │
   │  1. Lookup agent config by subagent_type                 │
   │     └─ Load from registry (built-in or custom)           │
   │                                                           │
   │  2. Resolve tool permissions                              │
   │     ├─ If tools specified in config → Filter to subset   │
   │     ├─ If not specified → Inherit all from main          │
   │     └─ Always filter out: Task, TodoWrite (nesting)      │
   │                                                           │
   │  3. Select model                                          │
   │     ├─ Tool call param "model" (highest priority)        │
   │     ├─ Agent config model (sonnet/opus/haiku)            │
   │     ├─ "inherit" → Use main conversation model           │
   │     └─ Unspecified → Use default subagent model          │
   │                                                           │
   │  4. Create isolated LLM context                           │
   │     ├─ Fresh conversation thread                         │
   │     ├─ System prompt from agent config                   │
   │     ├─ User message = prompt parameter                   │
   │     └─ No access to main conversation history            │
   │                                                           │
   │  5. Execute subagent                                      │
   │     └─ Tool calls filtered through permission system     │
   │                                                           │
   │  6. Store transcript (optional)                           │
   │     └─ agent-{agentId}.jsonl                             │
   │                                                           │
   │  7. Return result to main agent                           │
   └───────────────────────────────────────────────────────────┘
```

#### Nesting Prevention Mechanism

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Tool Filtering (Nesting Prevention)                       │
└─────────────────────────────────────────────────────────────────────────────┘

Main Conversation:                     Subagent Context:
┌──────────────────┐                   ┌──────────────────────────────┐
│ Available Tools: │                   │ Available Tools (filtered):  │
│ - Read           │                   │ - Read                       │
│ - Write          │                   │ - Write                      │
│ - Grep           │                   │ - Grep                       │
│ - Glob           │                   │ - Glob                       │
│ - Bash           │                   │ - Bash                       │
│ - Task ✅        │                   │ - Task ❌ (filtered out)     │
│ - TodoWrite ✅   │                   │ - TodoWrite ❌ (filtered out)│
│ - TodoRead ✅    │                   │ - TodoRead ❌ (filtered out) │
│ - ...            │                   │ - ...                        │
└──────────────────┘                   └──────────────────────────────┘
        │                                      │
        │ Can invoke subagents                 │ Cannot invoke subagents
        │ via Task tool                        │ Task tool not available
        ▼                                      ▼
┌──────────────────┐                   ┌──────────────────────────────┐
│ Task(            │                   │ If LLM tries to call Task:   │
│   type:"explore",│                   │ ❌ Tool not found error      │
│   prompt:"..."   │                   │                              │
│ ) ✅ Success     │                   │ LLM must work with available │
└──────────────────┘                   │ tools only (no delegation)   │
                                       └──────────────────────────────┘

Implementation:
  1. When creating subagent context, filter out Task and TodoWrite tools
  2. LLM system prompt indicates these tools are unavailable
  3. No additional runtime checking needed - tools simply don't exist
  4. Simple and foolproof: can't call what doesn't exist
```

#### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Data Flow                                       │
└─────────────────────────────────────────────────────────────────────────────┘

User Input ──────┐
                 │
Configuration ───┼──▶ ┌──────────────────┐
Files            │    │  Agent Registry   │
                 │    │  (in-memory)      │
                 │    │  - Built-in       │
                 │    │  - Custom         │
                 │    └─────────┬─────────┘
                 │              │
                 ▼              │ Available to
            ┌─────────────────────────┐
            │   Main AI Agent (LLM)   │
            │   - Process request     │
            │   - Decide delegation   │◀─── Conversation
            │   - Access registry     │     History
            └────────┬────────────────┘
                     │
                     │ If delegate → Task tool call
                     │ {subagent_type, prompt, model}
                     ▼
            ┌─────────────────────────┐
            │  Task Tool              │
            │  (Agent Executor)       │
            │  Inputs:                │
            │  - subagent_type        │
            │  - prompt               │
            │  - model (optional)     │
            │  - Agent registry       │
            │  - Execution stack      │
            └────────┬────────────────┘
                     │
                     │ Lookup config
                     ▼
            ┌─────────────────────────┐
            │  Agent Config           │
            │  {name, tools, model,   │
            │   systemPrompt}         │
            └────────┬────────────────┘
                     │
         ┌───────────┴────────────┐
         │                        │
         ▼                        ▼
┌──────────────────┐    ┌──────────────────┐
│ LLM API          │    │ Transcript Store │
│ (Subagent)       │    │ agent-{id}.jsonl │
│ - Model request  │    │ (optional)       │
│ - System prompt  │    │                  │
│ - User prompt    │    │                  │
│ - Streaming resp │    │                  │
└────────┬─────────┘    └──────────────────┘
         │
         │ Tool calls (filtered)
         ▼
┌──────────────────┐
│ Tool Executor    │
│ (permission      │
│  filtered)       │
│ - Read, Write    │
│ - Grep, Glob     │
│ - Bash, etc.     │
└────────┬─────────┘
         │ Tool results
         ▼
    Subagent output
         │
         ▼
┌──────────────────┐
│ Task Tool Result │
│ (returned to     │
│  main agent)     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Main AI Agent    │
│ (continues)      │
└──────────────────┘
```

### Core Components

#### 1. Task Tool Implementation

**System Prompt Integration**

- Main agent's system prompt lists available subagents with descriptions
- Includes delegation guidelines (when to use Task tool)

**Task Tool Parameters**

- **subagent_type**: Name of subagent to invoke (required)
- **prompt**: Task description for subagent (required)
- **description**: Short description for UI (optional)
- **model**: "sonnet" | "opus" | "haiku" - override model (optional)

**Execution Flow**

1. Lookup agent config from registry
2. Validate agent exists
3. Resolve model (param > config > inherit > default)
4. Execute via AgentExecutor with filtered tools
5. Return string result

**Return Result**

- String: subagent's final response or formatted error message

**Delegation Logic**
Main agent delegates when:

- Complex multi-step tasks needing isolated context
- Explicit user request (e.g., "use explore agent")
- Agent description contains "PROACTIVELY" keyword

#### 2. Agent Executor & Context Isolation

**Isolated Context**

- Fresh LLM thread per execution (no access to main conversation history)
- Separate message history per subagent
- Tools filtered based on config and subagent status
- Stateless (no persistence across invocations)

**Message History**

- Includes: system prompt, task prompt, responses, tool calls/results
- No cross-contamination with main conversation
- Discarded after execution completes

**Context Cleanup**

- Message history discarded
- LLM thread terminated
- Resources cleaned up

**Intermediate Messages**

- Streaming responses supported
- Tool calls handled transparently
- Progress updates to user (optional)

**Error Handling**

- Agent not found → error to main agent
- Tool errors → subagent handles/retries
- LLM API errors → propagate formatted error
- Validation errors → clear error messages

**Model Selection**
Priority: param > config > inherit > default

#### 3. Tool Permission System

**Tool Resolver**

- Config specifies `tools` → use explicit subset
- Config omits `tools` → inherit all from main (default)

**Tool Filtering for Subagents (Nesting Prevention)**

**CRITICAL**: When `isSubagent = true`, automatically remove:

- **Task** - Prevents nesting (subagents can't spawn subagents)
- **TodoWrite** - Can't modify main conversation todos
- **TodoRead** - Can't read main conversation todos

**Why This Works:**

- Simple: Task tool doesn't exist in subagent's tool list
- No runtime checks: filtered at tool resolution time
- Stateless: applied fresh each execution

**Why Essential:**

- Prevents infinite recursion and exponential costs
- Predictable flat delegation model
- Limits attack surface for prompt injection

**Permission Approval**

- Same approval rules as main conversation
- User controls which tools require approval

**Tool Validation**

- Validated at config load time
- Unknown tool names rejected with clear errors

**MCP Tool Support**

- Format: `mcp__server__tool`
- Inherited by default
- Subject to same filtering rules

#### 4. Agent Configuration System

**Configuration Schema**

- **name**: Unique identifier (required)
- **description**: Invocation trigger (required)
- **systemPrompt**: Agent's role/behavior (required)
- **tools**: Array or omit to inherit (optional)
- **model**: "sonnet" | "opus" | "haiku" | "inherit" (optional)

**Configuration Sources**

1. **File-based** (.md): YAML frontmatter + Markdown body
   - `~/.neovate/agents/*.md` or `.neovate/agents/*.md`
2. **JSON-based** (config.json): `agents` field
   - `~/.neovate/config.json` or `.neovate/config.json`

**Priority & Merge Strategy**

1. Load USER-level: files + config.json (config.json overrides files)
2. Load PROJECT-level: files + config.json (config.json overrides files)
3. Final merge: PROJECT overrides USER (for matching names only)

**Agent Registry**

- Central registry: `Map<name, AgentConfig>`
- Built at startup
- Task tool queries by name
- Includes built-in + custom agents

**Validation**

- Required fields: name, description, systemPrompt
- Tool names validated against available tools
- Agent names must be unique
- Invalid configs rejected with errors

#### 5. Management Interface

**`/agents` Command**

```
/agents
  ├── View All Agents (built-in, user, project)
  ├── Create (scope: project/user, AI-assisted or manual)
  ├── Edit (update fields, system prompt, tools, model)
  └── Delete (custom only, built-in protected)
```

**CRUD Operations**

- **Create**: AI-assisted generation or manual edit
- **Read**: View details and source
- **Update**: Edit in $EDITOR or inline
- **Delete**: Remove custom agents only

**External Editor**

- Press 'e' to open in $EDITOR
- Edit .md or config.json directly
- Changes reload on next invocation

### File Structure

```
Project Structure:
.neovate/
  config.json              # Centralized config with agents field
  agents/                  # Optional: file-based agents
    code-reviewer.md
    debugger.md
    data-scientist.md

User Structure:
~/.neovate/
  config.json              # Personal config with agents field
  agents/                  # Optional: personal file-based agents
    personal-assistant.md
    note-taker.md
```

**Priority Rules:**

1. **Project-level overrides user-level** (for agents with the same name)
2. **Within each level**, config.json overrides file-based (for agents with the same name)
3. **Agents with different names** from both levels are merged (no conflict)

**Example Scenario:**

```
User level has:
  - ~/.neovate/agents/personal.md        → "personal" agent
  - ~/.neovate/agents/debugger.md        → "debugger" agent (v1)
  - ~/.neovate/config.json               → "helper" agent

Project level has:
  - .neovate/agents/code-reviewer.md     → "code-reviewer" agent
  - .neovate/agents/debugger.md          → "debugger" agent (v2)
  - .neovate/config.json                 → "debugger" agent (v3)

Final result:
  - "personal" agent       (from user file)
  - "helper" agent         (from user config.json)
  - "code-reviewer" agent  (from project file)
  - "debugger" agent       (from project config.json v3) ← wins!
    - Project overrides user
    - Within project, config.json overrides file
```

**Example .neovate/config.json with agents:**

```json
{
  "agents": {
    "code-reviewer": {
      "description": "Expert code reviewer. Use proactively after code changes.",
      "prompt": "You are a senior code reviewer specializing in code quality, security, and best practices.\n\nWhen invoked, review code changes and provide constructive feedback...",
      "tools": ["Read", "Grep", "Glob", "Bash"],
      "model": "inherit"
    },
    "test-runner": {
      "description": "Runs tests and analyzes failures",
      "prompt": "You are a test automation expert...",
      "tools": ["Read", "Bash", "Grep"],
      "model": "haiku"
    }
  },
  "defaultModel": "sonnet",
  "otherSettings": "..."
}
```

### Agent Definition Example

```markdown
---
name: code-reviewer
description: Expert code reviewer. Use proactively after code changes to review quality, security, and best practices.
tools: Read, Grep, Glob, Bash
---

You are a senior code reviewer specializing in code quality, security, and best practices.

...
```

---

## Security Considerations

### Tool Permission Enforcement

**Risk:** Agents could access tools they shouldn't have
**Mitigation:**

- Validate tool permissions before each tool call
- Deny-by-default for unspecified tools
- Audit log of tool usage per agent

### Configuration Injection

**Risk:** Malicious YAML could execute code
**Mitigation:**

- Use safe YAML parser (no custom tags)
- Validate all configuration values
- Sanitize file paths

### Prompt Injection via Agent Descriptions

**Risk:** Crafted descriptions could manipulate delegation
**Mitigation:**

- Treat descriptions as data, not instructions
- Use structured matching, not prompt concatenation
- Rate matching algorithm's reasoning

---

## Performance Considerations

### Latency Concerns

**Cold Start Overhead:**

- Each agent invocation starts with empty context
- No warm state to leverage
- Fresh LLM conversation initialization

**Mitigation Strategies:**

- Use faster models (Haiku) for simple agents
- Optimize prompt length in system prompt
- Cache parsed configurations
- Parallel agent discovery on startup

### Context Window Management

**Main Conversation Preservation:**

- Agents help preserve main context by isolating work
- Enables longer overall sessions
- Reduces need for conversation resets

**Trade-off:**

- Agent results returned to main → still consumes main context
- Need concise result summarization

### Storage Concerns

**Transcript Accumulation:**

- Each agent execution creates transcript file
- Can accumulate over time

**Mitigation:**

- Implement transcript TTL and auto-cleanup
- Add transcript compression
- Provide cleanup command

### Configuration Loading

**Startup Performance:**

- Must scan multiple directories
- Parse YAML + Markdown for each agent
- Can be slow with many agents

**Optimization:**

- Lazy loading (load on first use)
- Cache parsed configurations
- Watch files for changes (hot reload)
- Parallel parsing

---

## Open Questions

### Technical Questions

1. **How do we handle agent execution timeouts?**

   - Should we set max execution time per agent?
   - How do we gracefully terminate long-running agents?
   - Should users be able to configure timeouts?

2. **Should we treat the main conversation as a first-class agent?**

   - Main conversation could be an agent with its own configuration
   - Would unify the architecture (all conversations are agents)
   - Enables features like: main agent model selection, tool restrictions, custom system prompts
   - Trade-off: Added complexity vs architectural consistency

### Product Questions

1. **What built-in agents should we ship with?**

- Explore Agent
- General Proposal

2. **How do we measure agent effectiveness?**

   - Success metrics?
   - User feedback mechanism?
   - A/B testing different agent prompts?

3. **How do we handle breaking changes?**
   - Version agents?
   - Schema versioning?
   - Migration tools?

### User Experience Questions

1. **How do users discover which agents are available?**

   - Better than just `/agents` list?
   - Context-aware suggestions?
   - In-conversation hints?

2. **How do we communicate agent activity to users?**

   - Show "Agent X is working..."?
   - Progress indicators?
   - Real-time output streaming?

3. **Should users approve agent invocations?**

   - Auto-invoke vs ask first?
   - Per-agent settings?
   - Global toggle?

4. **How do we handle agent failures gracefully?**
   - Fallback to main conversation?
   - Retry logic?
   - User notification?

---

## References

### External References

1. **Claude Code Documentation**

   - Subagents guide: https://code.claude.com/docs/en/subagents
   - Plugin system: https://code.claude.com/docs/en/plugins
   - CLI reference: https://code.claude.com/docs/en/cli-reference

---

## Appendix

### Appendix A: Configuration Examples

**Minimal Agent:**

```yaml
---
name: simple-agent
description: A minimal agent for testing
---
You are a helpful assistant.
```

**Full-Featured Agent:**

```yaml
---
name: advanced-agent
description: Advanced agent with all options. Use for complex analysis tasks.
tools: Read, Write, Grep, Glob, Bash
model: sonnet
---
# Advanced Agent System Prompt

You are a senior software architect with expertise in system design.

## Your Capabilities
- Analyze complex codebases
- Design scalable architectures
- Review technical designs

## Your Process
1. Understand the requirements thoroughly
2. Analyze existing code and patterns
3. Propose solutions with trade-offs
4. Provide implementation guidance

## Output Format
Always structure your responses with:
- Executive Summary
- Detailed Analysis
- Recommendations
- Implementation Steps
- Risks & Mitigation
```

### Appendix B: Configuration Examples

**Define agents in .neovate/config.json:**

```json
{
  "agents": {
    "reviewer": {
      "description": "Code reviewer for quality and security",
      "prompt": "You are an expert code reviewer. Review code for quality, security, and best practices.",
      "tools": ["Read", "Grep", "Glob"],
      "model": "haiku"
    },
    "test-analyst": {
      "description": "Analyzes test failures and suggests fixes",
      "prompt": "You are a test debugging expert. Analyze test failures and provide actionable fixes.",
      "tools": ["Read", "Bash", "Grep"],
      "model": "sonnet"
    }
  }
}
```

**Define agents in ~/.neovate/config.json (personal):**

```json
{
  "agents": {
    "my-helper": {
      "description": "My personal coding assistant",
      "prompt": "You are my personal assistant. Help with tasks using my preferred coding style.",
      "model": "inherit"
    }
  }
}
```

### Appendix C: Comparison Matrix

| Feature            | Claude Code | NeovateCode (Proposed) | Notes                              |
| ------------------ | ----------- | ---------------------- | ---------------------------------- |
| Isolated contexts  | ✅          | ✅                     | Core feature                       |
| Multi-level config | ✅          | ✅                     | Project/User (files + config.json) |
| CLI config         | ✅          | ⏳ Future              | Not supported in NeovateCode       |
| Plugin agents      | ✅          | ⏳ Future              | Not supported in NeovateCode       |
| config.json agents | ❌          | ✅                     | NeovateCode addition               |
| Tool permissions   | ✅          | ✅                     | Inherit or specify                 |
| Resumable agents   | ✅          | ⏳ Future              | Planned, not MVP                   |
| Nesting prevention | ✅          | ✅                     | Core security feature              |
| MCP integration    | ✅          | ✅                     | Full support                       |
| `/agents` command  | ✅          | ✅                     | CRUD interface                     |
| Visual builder     | ❌          | ❌                     | Not planned                        |
| Cloud marketplace  | ❌          | ❌                     | Local-first                        |

---
