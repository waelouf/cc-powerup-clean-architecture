# Contributing to dev-architect

Thank you for your interest in contributing to dev-architect! This guide will help you get started.

## How to Contribute

### Reporting Issues

If you find a bug or have a feature request:

1. Check if the issue already exists in [GitHub Issues](https://github.com/waelouf/dev-architect/issues)
2. If not, create a new issue with:
   - Clear title and description
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your environment details

### Contributing Code

1. **Fork the repository**
2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. **Make your changes**
4. **Test your changes**
5. **Commit with clear messages**
   ```bash
   git commit -m "Add: description of your changes"
   ```
6. **Push to your fork**
   ```bash
   git push origin feature/your-feature-name
   ```
7. **Open a Pull Request**

## Adding New Components

### Adding a Skill

1. Create a new `.md` file in the `skills/` directory
2. Follow this structure:

```markdown
---
name: skill-name
description: Brief description of what this skill does
---

[Skill instructions and guidance]
```

3. Update README.md to list the new skill
4. Test the skill locally

### Adding a Slash Command

1. Create a new `.md` file in the `commands/` directory
2. Follow this structure:

```markdown
---
name: command-name
description: What this command does
arguments:
  - name: arg-name
    description: Argument description
    required: true/false
---

[Command prompt/instructions]
```

3. Update README.md to document the new command
4. Test the command locally

### Adding a Subagent

1. Create a new `.md` file in the `subagents/` directory
2. Follow this structure:

```markdown
---
name: agent-name
description: What this agent specializes in
tools:
  - Read
  - Grep
  - Glob
model: sonnet/haiku/opus
---

[Agent instructions and behavior]
```

3. Update README.md to document the new subagent
4. Test the subagent

## Code Quality Guidelines

- **Clear naming**: Use descriptive names for files and components
- **Documentation**: Include clear descriptions and instructions
- **Focused scope**: Each component should have a single, well-defined purpose
- **Examples**: Provide examples where helpful
- **Testing**: Test your changes before submitting

## Commit Message Guidelines

Use clear, descriptive commit messages:

- `Add: new feature or component`
- `Fix: bug fix`
- `Update: improvements to existing feature`
- `Docs: documentation changes`
- `Refactor: code restructuring`

## Questions?

If you have questions about contributing, feel free to:

- Open an issue for discussion
- Reach out to the maintainer

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
