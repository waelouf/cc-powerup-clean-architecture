# Slash Commands

This directory contains slash commands for the dev-architect plugin.

## What are Slash Commands?

Slash commands are user-invoked commands that start with `/` and expand into full prompts. They provide quick access to common workflows and tasks.

## Creating a New Slash Command

1. Create a new `.md` file in this directory
2. Use the following structure:

```markdown
---
name: command-name
description: What this command does
arguments:
  - name: argument-name
    description: What this argument is for
    required: true/false
---

[Your command prompt here]

Use {{argument-name}} to reference arguments in your prompt.
Use {{argument-name::default-value}} to provide a default value.
```

## Command Naming Conventions

- Use lowercase with hyphens: `my-command`
- Be descriptive and action-oriented
- Start with a verb when possible (e.g., `analyze-architecture`, `review-code`)

## Example Commands

See `analyze-architecture.md` for a complete example of a well-structured slash command.

## Best Practices

- **Clear Arguments**: Define what arguments your command accepts
- **Defaults**: Provide sensible defaults where appropriate
- **Instructions**: Write clear, specific instructions for what Claude should do
- **Tool Hints**: Suggest which tools to use (e.g., "Use the Explore agent to...")
- **Output Format**: Specify how results should be presented

## Using Arguments

Arguments can be referenced in your prompt using double curly braces:

- `{{path}}` - Required argument
- `{{path::.}}` - Optional argument with default value "."
- `{{format::json}}` - Optional argument with default "json"

## Testing Your Command

After creating a command, test it by:

1. Installing the plugin locally: `claude-code plugin link .`
2. Running the command: `/command-name arg1 arg2`
3. Verifying the output and behavior

For more information on creating slash commands, see the [CONTRIBUTING.md](../CONTRIBUTING.md) guide.
