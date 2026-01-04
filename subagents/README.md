# Subagents

This directory contains subagent configurations for the dev-architect plugin.

## What are Subagents?

Subagents are specialized AI agents designed for specific tasks. They can be launched using the Task tool and run autonomously with access to specific tools and instructions.

## Creating a New Subagent

1. Create a new `.md` file in this directory
2. Use the following structure:

```markdown
---
name: agent-name
description: What this agent specializes in
tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Edit
  - Write
model: sonnet/haiku/opus
---

[Your agent instructions here]

Define the agent's role, capabilities, and behavior.
Specify how it should approach tasks and what output format to use.
```

## Agent Naming Conventions

- Use lowercase with hyphens: `my-agent`
- Be descriptive and specific about the agent's specialty
- Consider using role-based names (e.g., `code-reviewer`, `test-generator`)

## Available Tools

Common tools you can grant to subagents:

- `Read` - Read files
- `Write` - Create new files
- `Edit` - Modify existing files
- `Grep` - Search file contents
- `Glob` - Find files by pattern
- `Bash` - Execute shell commands
- `WebFetch` - Fetch web content
- `WebSearch` - Search the web

## Model Selection

- `haiku` - Fast, cost-effective for simple tasks
- `sonnet` - Balanced performance (recommended for most tasks)
- `opus` - Most capable for complex reasoning

## Example Subagents

See `code-reviewer.md` for a complete example of a well-structured subagent.

## Best Practices

- **Focused Scope**: Each agent should specialize in a specific domain or task type
- **Clear Instructions**: Provide detailed guidance on how the agent should work
- **Minimal Tools**: Only grant tools the agent actually needs
- **Output Format**: Specify how the agent should structure its output
- **Model Choice**: Use the least capable model that can handle the task
- **Checklists**: Provide checklists or frameworks for systematic work

## Testing Your Subagent

After creating a subagent, test it by:

1. Installing the plugin locally: `claude-code plugin link .`
2. Using the Task tool to invoke it in a conversation
3. Verifying the agent's behavior and output quality

For more information on creating subagents, see the [CONTRIBUTING.md](../CONTRIBUTING.md) guide.
