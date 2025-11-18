# RFC: NeovateCode Subagents System

**Status:** Draft
**Author:** [Your Name]
**Created:** 2025-11-17
**Last Updated:** 2025-11-17

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

**6. Invocation Methods**

**Three Invocation Methods:**

1. **Automatic** - Claude matches tasks to agents based on descriptions
2. **Explicit** - User mentions agent by name ("use the code-reviewer subagent")
3. **CLI** - Dynamic definition via `--agents` flag for testing/automation

#### Advanced Features

**1. Resumable Agent Execution**

- Each execution gets unique `agentId`
- Transcript stored as `agent-{agentId}.jsonl` in project directory
- Resume via `resume` parameter to continue previous sessions
- **Recording behavior**: Recording is disabled during resume to avoid duplicating messages
- **Agent types**: Both synchronous and asynchronous subagents can be resumed
- **Context restoration**: Full context from previous conversation is restored when resuming
- **Use case**: Long-running research, iterative refinement, particularly useful for analysis tasks that span multiple sessions

**2. Agent Nesting Prevention**

- Subagents cannot spawn other subagents
- Prevents infinite recursion
- **Exception**: Special Plan agent for plan mode

**3. Subagent Chaining**

- Sequence multiple subagents for complex workflows
- Results inform subsequent agent invocations

**4. Dynamic Selection**

- AI matches tasks to agents based on description similarity and context
- Documentation: _"Claude intelligently matches tasks to subagents based on context and description specificity"_

### Management Interface

**`/agents` Command** - Interactive interface for managing subagents:

- View all agents (built-in, user, project, plugin)
- Create new agents with AI assistance
- Edit existing agents (tools, prompts, configuration)
- Delete custom agents (built-in protected)
- Press 'e' to edit in external editor ($EDITOR)

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
- ✅ Build nesting prevention mechanism
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
- ✅ Add external editor integration ($EDITOR)

### Non-Goals

#### Out of Scope for Initial Release

- ❌ Resumable agents with transcript storage
- ❌ CLI `--agents` flag for dynamic configuration
- ❌ Plugin-provided agents
- ❌ Agent chaining for complex workflows
- ❌ Cloud marketplace or agent sharing platform
- ❌ Visual agent builder
- ❌ Agent analytics and effectiveness metrics
- ❌ Persistent agent memory across invocations

---

## Detailed Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Main Conversation                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Agent Dispatcher                                      │ │
│  │  - Task matching                                       │ │
│  │  - Agent selection                                     │ │
│  │  - Priority resolution                                 │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼──────┐ ┌────▼─────┐ ┌──────▼───────┐
│  Subagent A  │ │ Subagent B│ │  Subagent C  │
│              │ │           │ │              │
│ Context A    │ │ Context B │ │  Context C   │
│ Tools A      │ │ Tools B   │ │  Tools C     │
│ Model A      │ │ Model B   │ │  Model C     │
└──────────────┘ └───────────┘ └──────────────┘
```

### Core Components

#### 1. Agent Configuration System

**Configuration Loader**

- Scans directories: CLI args, `.neovate/agents/`, `~/.neovate/agents/`, plugins
- Parses YAML frontmatter + Markdown body
- Validates required fields (name, description)
- Merges configurations with priority ordering
- Caches parsed configurations for performance

**Configuration Schema**

```typescript
interface AgentConfig {
  name: string; // Required: unique identifier
  description: string; // Required: invocation trigger description
  tools?: string[]; // Optional: specific tools or inherit all
  model?: "sonnet" | "opus" | "haiku" | "inherit"; // Optional: model selection
  systemPrompt: string; // Markdown body content
}
```

**Priority Resolution**

```typescript
enum ConfigSource {
  PROJECT = 4, // Highest priority
  CLI = 3,
  USER = 2,
  PLUGIN = 1, // Lowest priority
}

function resolveAgent(name: string): AgentConfig {
  const candidates = findAllAgents(name);
  return candidates.sort((a, b) => b.priority - a.priority)[0];
}
```

#### 2. Agent Dispatcher

**Task Matching Algorithm**

```typescript
interface TaskMatch {
  agent: AgentConfig;
  score: number;
  reason: string;
}

function matchTask(userRequest: string, agents: AgentConfig[]): TaskMatch[] {
  return agents
    .map((agent) => ({
      agent,
      score: calculateMatchScore(userRequest, agent.description),
      reason: explainMatch(userRequest, agent.description),
    }))
    .sort((a, b) => b.score - a.score);
}
```

**Invocation Decision Tree**

1. Check for explicit agent name mention → use specified agent
2. Check for proactive keywords in agent descriptions → auto-invoke
3. Analyze task type vs agent descriptions → score and select best match
4. If no good match → execute in main conversation

#### 3. Context Isolation Manager

**Isolated Context Implementation**

- Each agent gets fresh LLM conversation thread
- No access to main conversation history (except passed task prompt)
- Separate message history per agent execution
- Clean up on completion

**Nesting Prevention**

```typescript
class AgentExecutor {
  private static executionStack: string[] = [];

  execute(agent: AgentConfig, task: string): Promise<string> {
    if (AgentExecutor.executionStack.length > 0) {
      throw new Error("Agent nesting not allowed");
    }

    AgentExecutor.executionStack.push(agent.name);
    try {
      return this.doExecute(agent, task);
    } finally {
      AgentExecutor.executionStack.pop();
    }
  }
}
```

#### 4. Tool Permission System

**Tool Resolver**

```typescript
function resolveTools(agent: AgentConfig, mainTools: Tool[]): Tool[] {
  if (!agent.tools || agent.tools.length === 0) {
    // Inherit all tools from main thread
    return mainTools;
  }

  // Filter to specified tools only
  return mainTools.filter((tool) => agent.tools.includes(tool.name));
}
```

**Permission Validation**

- Validate tool names at configuration load time
- Reject unknown tool names with clear error
- Support MCP tool name format: `mcp__server__tool`

#### 5. Resumable Execution

**Transcript Storage**

```typescript
interface AgentTranscript {
  agentId: string;
  agentName: string;
  model: string;
  tools: string[];
  createdAt: string;
  messages: Message[];
}

class TranscriptManager {
  save(agentId: string, transcript: AgentTranscript): void {
    const path = `agent-${agentId}.jsonl`;
    fs.writeFileSync(path, JSON.stringify(transcript));
  }

  load(agentId: string): AgentTranscript | null {
    const path = `agent-${agentId}.jsonl`;
    if (!fs.existsSync(path)) return null;
    return JSON.parse(fs.readFileSync(path, "utf-8"));
  }
}
```

**Resume Flow**

1. User requests resume with `agentId`
2. Load transcript from `agent-{agentId}.jsonl`
3. Restore agent configuration and messages
4. Continue conversation with new task
5. Append to existing transcript

#### 6. Management Interface

**`/agents` Command Structure**

```
/agents
  ├── View All Agents
  ├── Create New Agent
  │   ├── Choose scope (project/user)
  │   ├── Generate with AI
  │   └── Manual creation
  ├── Edit Agent
  │   ├── Update configuration
  │   ├── Edit system prompt
  │   └── Modify tools
  └── Delete Agent
```

### File Structure

```
Project Structure:
.neovate/
  agents/
    code-reviewer.md
    debugger.md
    data-scientist.md

User Structure:
~/.neovate/
  agents/
    personal-assistant.md
    note-taker.md

Plugin Structure:
~/.neovate/
  plugins/
    security-plugin/
      agents/
        security-scanner.md
```

### Agent Definition Example

```markdown
---
name: code-reviewer
description: Expert code reviewer. Use proactively after code changes to review quality, security, and best practices.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer specializing in code quality, security, and best practices.

## Your Role

When invoked, you will review code changes and provide constructive feedback.

## Review Process

1. Run `git diff` to see recent changes
2. Focus on modified files and their context
3. Begin review immediately without asking permission

## Review Checklist

- Code is simple and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling implemented
- No exposed secrets or API keys
- Input validation in place
- Good test coverage
- Performance considerations addressed

## Output Format

Provide feedback organized by priority:

### Critical Issues (must fix)

- [Issue with specific line reference and fix suggestion]

### Warnings (should fix)

- [Issue with explanation]

### Suggestions (consider improving)

- [Suggestion with rationale]

Always include specific code examples of how to fix issues.
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

### Transcript Data Leakage

**Risk:** Sensitive data in transcripts persisted to disk
**Mitigation:**

- Store transcripts in secure location with proper permissions
- Add option to disable transcript persistence
- Auto-cleanup old transcripts
- Warn users about sensitive data

### Agent Impersonation

**Risk:** Malicious agent with same name overrides legitimate one
**Mitigation:**

- Clear priority system (project > user)
- Show source in `/agents` interface
- Require confirmation for non-project agents in shared environments

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

2. **What's the transcript cleanup strategy?**

   - Default TTL for transcripts?
   - Automatic cleanup vs manual?
   - Compress vs delete?

3. **How do we handle agent version migrations?**

   - If agent schema changes, how to migrate existing agents?
   - Backward compatibility guarantees?
   - Deprecation process?

4. **Should agents be able to call main conversation's context?**

   - Currently fully isolated, but should agents read-only access main history?
   - Security implications?
   - Use cases?

5. **How do we prevent agent description spam?**
   - Can users create agents with overly aggressive "PROACTIVELY" descriptions?
   - Rate limiting on proactive invocations?
   - User controls for auto-delegation?

### Product Questions

1. **What built-in agents should we ship with?**

   - Plan agent (confirmed)
   - Code reviewer?
   - Debugger?
   - Test runner?
   - Documentation writer?

2. **How do we measure agent effectiveness?**

   - Success metrics?
   - User feedback mechanism?
   - A/B testing different agent prompts?

3. **Should we support agent templates?**

   - Pre-built templates users can customize?
   - Gallery of community agents?
   - Import/export format?

4. **How do we handle breaking changes?**
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

2. **Similar Systems**

   - OpenAI Assistants API
   - LangChain Agents
   - AutoGPT multi-agent systems

3. **Design Patterns**
   - Delegation Pattern
   - Strategy Pattern
   - Chain of Responsibility Pattern

### Internal References

1. **Related RFCs**

   - RFC-002: NeovateCode Plugin System (planned)
   - RFC-003: Tool Permission System (planned)
   - RFC-004: Context Management (planned)

2. **Codebase References**
   - Tool system architecture: `src/tools/`
   - Configuration loader: `src/config/`
   - LLM interface: `src/llm/`

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

### Appendix B: CLI Usage Examples

**Define agent via CLI:**

```bash
neovate --agents '{
  "reviewer": {
    "description": "Code reviewer",
    "prompt": "Review code for quality",
    "tools": ["Read", "Grep"],
    "model": "haiku"
  }
}'
```

**Resume agent session:**

```bash
# After agent completes and returns agentId "abc123"
neovate --resume-agent abc123
```

### Appendix C: Comparison Matrix

| Feature            | Claude Code | NeovateCode (Proposed) | Notes                   |
| ------------------ | ----------- | ---------------------- | ----------------------- |
| Isolated contexts  | ✅          | ✅                     | Core feature            |
| Multi-level config | ✅          | ✅                     | Project/CLI/User/Plugin |
| Tool permissions   | ✅          | ✅                     | Inherit or specify      |
| Resumable agents   | ✅          | ✅                     | With transcripts        |
| Agent chaining     | ✅          | ✅                     | Sequential execution    |
| Nesting prevention | ✅          | ✅                     | Built-in safeguard      |
| MCP integration    | ✅          | ✅                     | Full support            |
| `/agents` command  | ✅          | ✅                     | CRUD interface          |
| Visual builder     | ❌          | ❌                     | Not planned             |
| Cloud marketplace  | ❌          | ❌                     | Local-first             |

---

**Document Version:** 1.0
**Next Review Date:** 2025-12-01
