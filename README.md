# Clean Architecture Powerup

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/waelouf/cc-powerup-clean-architecture)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![.NET](https://img.shields.io/badge/.NET-10-512BD4.svg)](https://dotnet.microsoft.com/)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-orange.svg)](https://claude.ai/code)
[![Powerup](https://img.shields.io/badge/Claude_Code-Powerup-purple.svg)](https://github.com/waelouf/cc-powerup-clean-architecture)

> Part of **Claude Code Powerups** - A collection of powerful plugins for supercharging your development workflow

An opinionated toolkit for building production-ready **.NET Clean Architecture applications** with FastEndpoints, Repository pattern, and Domain-Driven Design.

## Overview

Clean Architecture Powerup is a comprehensive Claude Code plugin that helps you build, migrate, and maintain .NET applications following Clean Architecture principles. Based on Microsoft's eShopOnWeb reference application, it provides expert guidance, code generation, and architecture validation.

> **⚡ Opinionated Skill**: This is an opinionated toolkit specifically designed for **.NET/C# applications**. It prescribes a specific approach to Clean Architecture using FastEndpoints, Repository pattern, Specification pattern, and Domain-Driven Design principles. If you're looking for language-agnostic architecture guidance or other tech stacks, this plugin may not be the best fit.

**Perfect for:**
- .NET developers building Clean Architecture applications
- Teams migrating legacy code to modern architecture patterns
- Architects seeking pattern guidance and code reviews
- Developers learning Clean Architecture and Domain-Driven Design

## Features

### 🏗️ Project Scaffolding
Scaffold complete Clean Architecture solutions from scratch with interactive wizard:
- Choose your API style: FastEndpoints, Minimal APIs, or Controllers
- Select database: SQL Server, PostgreSQL, or In-Memory
- Configure authentication: JWT, ASP.NET Identity, or None
- Auto-configure project structure, dependencies, and references

### ⚡ Feature Generation
Generate complete CRUD features across all layers in minutes:
- Domain entities with proper encapsulation and business logic
- Repository interfaces and implementations
- Specification pattern for complex queries
- Service layer with Result pattern
- API endpoints with validation
- Unit and integration tests

### 🔄 Legacy Migration
Migrate existing codebases to Clean Architecture with guided refactoring:
- Architecture analysis and violation detection
- Layer separation guidance
- Dependency inversion recommendations
- Step-by-step migration plan

### 🔍 Architecture Auditing
Scan your project for architecture violations and anti-patterns:
- Dependency rule violations
- Missing abstractions
- Improper layer coupling
- Anti-patterns and code smells
- Actionable fix suggestions

### 📚 Pattern Library
Browse and copy 14 proven Clean Architecture patterns:
- Repository Pattern with generic base
- Specification Pattern for queries
- Domain Events with MediatR
- Result Pattern for operation outcomes
- Guard Clauses for validation
- Test Data Builders
- And more...

## Installation

### From GitHub

```bash
claude plugin marketplace add waelouf/cc-powerup-clean-architecture

claude plugin install clean-architecture-powerup
```

### Local Development

1. Clone this repository:
```bash
git clone https://github.com/waelouf/cc-powerup-clean-architecture.git
cd cc-powerup-clean-architecture
```

2. Link the plugin:
```bash
claude-code plugin link .
```

## Usage

### Slash Commands

The plugin provides 5 powerful slash commands:

#### `/clean-arch:new` - Scaffold New Project
Create a new Clean Architecture solution with interactive wizard.

```bash
/clean-arch:new
# or with project name
/clean-arch:new MyProject
```

#### `/clean-arch:add-feature <EntityName>` - Generate CRUD Feature
Generate a complete CRUD feature across all layers.

```bash
/clean-arch:add-feature Product
/clean-arch:add-feature Order
/clean-arch:add-feature Customer
```

#### `/clean-arch:migrate` - Migrate Legacy Code
Analyze and migrate existing codebase to Clean Architecture.

```bash
/clean-arch:migrate
```

#### `/clean-arch:audit` - Audit Architecture
Scan for violations and anti-patterns with actionable fixes.

```bash
/clean-arch:audit
```

#### `/clean-arch:patterns` - Browse Patterns
Interactive pattern library with 14 proven examples.

```bash
/clean-arch:patterns
```

### Automatic Skill Invocation

The `clean-architecture` skill is automatically invoked during conversations when you discuss Clean Architecture topics:

- Architecture design and planning
- Code reviews for Clean Architecture projects
- Troubleshooting architecture issues
- Learning Clean Architecture concepts

## Quick Start Example

```bash
# 1. Create a new Clean Architecture project
/clean-arch:new MyECommerceApp

# 2. Generate CRUD features
/clean-arch:add-feature Product
/clean-arch:add-feature Order
/clean-arch:add-feature Customer

# 3. Verify architecture compliance
/clean-arch:audit

# 4. Run the application
cd MyECommerceApp
dotnet run --project src/API
```

## Architecture

This plugin implements a three-layer Clean Architecture:

```
┌─────────────────────────────────────────┐
│         Presentation (API/Web)          │  ← User Interface
│  Endpoints, Controllers, ViewModels     │
└────────────┬────────────────────────────┘
             │ depends on ↓
┌────────────▼────────────────────────────┐
│        Application Core (Domain)        │  ← Business Logic
│  Entities, Interfaces, Services,        │
│  Specifications, Domain Events          │
└────────────┬────────────────────────────┘
             │ depends on ↓
┌────────────▼────────────────────────────┐
│         Infrastructure (Data)           │  ← External Concerns
│  DbContext, Repositories, Identity,     │
│  Email, File System, External APIs      │
└──────────────────────────────────────────┘
```

**Key Principle:** Dependencies point INWARD. The Application Core has NO dependencies on external layers.

## Project Structure

```
MySolution/
├── src/
│   ├── ApplicationCore/       # Domain layer (no external dependencies)
│   │   ├── Entities/         # Domain entities & aggregates
│   │   ├── Interfaces/       # Service & repository contracts
│   │   ├── Services/         # Business logic
│   │   ├── Specifications/   # Query objects
│   │   └── Events/           # Domain events
│   │
│   ├── Infrastructure/        # Data access & external concerns
│   │   ├── Data/             # EF Core DbContext, migrations
│   │   └── Identity/         # ASP.NET Identity
│   │
│   └── API/                   # Presentation layer
│       ├── Endpoints/        # API endpoints
│       └── Program.cs        # Application startup
│
└── tests/
    ├── UnitTests/            # Fast, isolated tests
    └── IntegrationTests/     # Database integration tests
```

## Technologies & Patterns

- **.NET 10** - Latest .NET runtime
- **FastEndpoints** - High-performance endpoint routing
- **Entity Framework Core** - ORM and data access
- **Repository Pattern** - Data access abstraction
- **Specification Pattern** - Complex query encapsulation
- **Domain-Driven Design** - Business logic modeling
- **Result Pattern** - Operation outcome handling
- **Guard Clauses** - Input validation
- **MediatR** - Domain events and CQRS
- **xUnit** - Unit and integration testing

## Documentation

- [CLAUDE.md](CLAUDE.md) - Plugin architecture and development guide
- [CONTRIBUTING.md](CONTRIBUTING.md) - Contribution guidelines
- [skills/clean-architecture/SKILL.md](skills/clean-architecture/SKILL.md) - Complete skill documentation
- [Microsoft eShopOnWeb](https://github.com/dotnet-architecture/eShopOnWeb) - Reference architecture

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on how to contribute.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

Created by [Wael Mansour](https://github.com/waelouf)

Part of the **Claude Code Powerups** collection.

## Support

If you encounter any issues or have questions:
- Open an issue on [GitHub](https://github.com/waelouf/cc-powerup-clean-architecture/issues)
- Check the [skill documentation](skills/clean-architecture/SKILL.md)
- Run `/clean-arch:patterns` for interactive examples

## Related Powerups

Looking for more Claude Code Powerups? Check out the full collection at the Claude Code Powerups marketplace.
