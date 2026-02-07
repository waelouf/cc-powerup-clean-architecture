# Skills

This directory contains Claude Code skills for the clean-architecture-powerup plugin.

## What are Skills?

Skills are reusable capabilities that Claude can invoke during conversations when relevant tasks arise. They provide specialized expertise and guidance for specific domains.

## Creating a New Skill

1. Create a new `.md` file in this directory
2. Use the following structure:

```markdown
---
name: skill-name
description: Brief description of what this skill does
---

[Your skill instructions here]

Define what the skill does, when it should be used, and how it should behave.
Include any specific guidelines, best practices, or areas of expertise.
```

## Skill Naming Conventions

- Use lowercase with hyphens: `my-skill-name`
- Be descriptive and specific
- Keep names concise but clear

## Example Skills

See `architecture-patterns.md` for a complete example of a well-structured skill.

## Best Practices

- **Clear Purpose**: Each skill should have a single, well-defined purpose
- **Detailed Instructions**: Provide comprehensive guidance on how the skill should work
- **Context Awareness**: Explain when and why the skill should be invoked
- **Examples**: Include examples where helpful
- **Scope**: Define the boundaries of the skill's expertise

## Testing Your Skill

After creating a skill, test it by:

1. Installing the plugin locally: `claude-code plugin link .`
2. Starting a conversation where the skill would be relevant
3. Verifying that Claude invokes the skill appropriately

For more information on creating skills, see the [CONTRIBUTING.md](../CONTRIBUTING.md) guide.
