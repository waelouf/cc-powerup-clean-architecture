# dev-architect

Skills and tools for building well-architected software systems.

## Overview

dev-architect is a Claude Code plugin that provides expert guidance on software architecture, design patterns, and engineering best practices. With a comprehensive .NET Clean Architecture suite and general architecture tools, it helps you build robust, maintainable software systems faster.

**Perfect for:**
- .NET developers building Clean Architecture applications
- Teams migrating legacy code to modern patterns
- Architects seeking pattern guidance and code reviews
- Developers learning Clean Architecture and DDD

## Features

### .NET Clean Architecture Suite
- **Scaffold complete projects** with Clean Architecture, DDD, and Repository pattern
- **Generate CRUD features** across all layers in minutes
- **Migrate legacy code** to Clean Architecture with guided refactoring
- **Audit architecture** for violations and anti-patterns
- **Pattern library** with 14 proven examples from Microsoft's eShopOnWeb

### Architecture Guidance
- **Expert skills** for architecture patterns and design principles
- **Code review** with specialized subagents
- **Quick commands** for common architecture workflows

## Installation

### Install from GitHub

```bash
claude plugin marketplace add waelouf/dev-architect

claude plugin install dev-architect
```

### Install Locally

1. Clone this repository:
```bash
git clone https://github.com/waelouf/dev-architect.git
cd dev-architect
```

2. Link the plugin:
```bash
claude-code plugin link .
```

## Usage

### Skills

Skills can be invoked during conversations when relevant tasks arise. Available skills:

#### `dotnet-clean-arch`
Comprehensive skill for building .NET Clean Architecture monolithic applications with FastEndpoints, Repository pattern, and Domain-Driven Design.

**Key capabilities:**
- Scaffold complete Clean Architecture projects from scratch
- Generate full-stack CRUD features across all layers
- Migrate existing codebases to Clean Architecture
- Audit architecture for violations and anti-patterns
- Browse and copy proven patterns from Microsoft's eShopOnWeb

[Read full documentation →](skills/dotnet-clean-arch/README.md)

#### `architecture-patterns`
Expert guidance on software architecture patterns and design principles (SOLID, DRY, KISS, YAGNI, etc.).

### Slash Commands

Use slash commands for quick access to common workflows:

#### .NET Clean Architecture Commands

- `/dotnet-clean-arch:new` - Scaffold a new Clean Architecture solution with interactive wizard
- `/dotnet-clean-arch:add-feature <EntityName>` - Generate complete CRUD feature across all layers
- `/dotnet-clean-arch:migrate` - Migrate existing codebase to Clean Architecture with guided refactoring
- `/dotnet-clean-arch:audit` - Scan project for Clean Architecture violations and anti-patterns
- `/dotnet-clean-arch:patterns` - Browse and copy 14 proven patterns with interactive examples

#### Architecture Analysis

- `/analyze-architecture [path]` - Analyze codebase architecture and provide recommendations

### Subagents

Specialized agents for focused tasks:

- **`code-reviewer`** - Review code quality, architecture, and best practices with detailed feedback

## Quick Start Examples

### Building a .NET Clean Architecture API

```bash
# 1. Create a new Clean Architecture project
/dotnet-clean-arch:new

# 2. Add CRUD features
/dotnet-clean-arch:add-feature Product
/dotnet-clean-arch:add-feature Order
/dotnet-clean-arch:add-feature Customer

# 3. Verify architecture compliance
/dotnet-clean-arch:audit

# 4. Run the application
dotnet run --project src/API
```

### Analyzing an Existing Codebase

```bash
# Analyze architecture patterns
/analyze-architecture

# Check for Clean Architecture violations
/dotnet-clean-arch:audit

# Get migration guidance
/dotnet-clean-arch:migrate
```

### Learning Patterns

```bash
# Browse architecture patterns
/dotnet-clean-arch:patterns

# Get specific guidance
# (skill will be invoked automatically when discussing architecture)
```

## Structure

```
dev-architect/
├── skills/              # Skill definitions
├── commands/            # Slash command definitions
├── subagents/           # Subagent configurations
├── package.json         # Plugin metadata
├── README.md           # This file
├── LICENSE             # MIT License
└── CONTRIBUTING.md     # Contribution guidelines
```

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on how to contribute to this project.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

Created by [waelouf](https://github.com/waelouf)

## Support

If you encounter any issues or have questions, please [open an issue](https://github.com/waelouf/dev-architect/issues) on GitHub.
