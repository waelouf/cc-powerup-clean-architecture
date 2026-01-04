---
description: Scaffold a new Clean Architecture .NET solution with interactive wizard
arguments:
  - name: project-name
    description: Optional project name (will prompt if not provided)
    required: false
---

You are an expert .NET architect specializing in Clean Architecture. Use the `dotnet-clean-arch` skill to scaffold a new Clean Architecture solution.

## Your Task

Follow the workflow defined in the `dotnet-clean-arch` skill, Section B, Command `/dotnet-clean-arch:new`.

## Steps

1. **Gather Project Information** using `AskUserQuestion`:
   - Project name (default: "MySolution" if not provided as argument)
   - API style (FastEndpoints, Minimal APIs, or Controllers)
   - Database (SQL Server, PostgreSQL, or In-Memory)
   - Authentication setup (JWT, ASP.NET Identity, or None)

2. **Generate Solution Structure**:
   - Create directory structure (src/ApplicationCore, src/Infrastructure, src/API, tests/)
   - Generate .sln file and .csproj files
   - Configure project references (Infrastructure → ApplicationCore, API → both)

3. **Add NuGet Packages**:
   - ApplicationCore: Ardalis.GuardClauses, Ardalis.Specification, Ardalis.Result, MediatR
   - Infrastructure: EF Core packages, Ardalis.Specification.EntityFrameworkCore, Identity
   - API: FastEndpoints or Minimal API packages, AutoMapper, JWT Bearer
   - Tests: xUnit, NSubstitute, EF Core InMemory

4. **Generate Base Code Files**:
   - ApplicationCore: BaseEntity, IAggregateRoot, IRepository, IReadRepository
   - Infrastructure: AppDbContext, EfRepository, Dependencies.cs
   - API: Program.cs with DI, ServiceCollectionExtensions
   - Tests: Base test classes and builders directory

5. **Create README.md** with:
   - How to run the application
   - How to run tests
   - How to create migrations
   - Next steps (use /dotnet-clean-arch:add-feature)

6. **Provide Summary**:
   - List all created projects
   - Show next steps
   - Explain how to add first feature

## Important

- Use the exact code patterns from the skill's Section C (Embedded Code Patterns)
- Follow Clean Architecture dependency rules strictly
- Generate working, production-ready code
- Include comprehensive comments in generated files
- Make sure all dotnet commands are executable

Execute this workflow interactively, asking questions as needed and generating all files.
