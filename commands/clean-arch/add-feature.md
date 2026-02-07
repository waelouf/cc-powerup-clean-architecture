---
description: Generate a complete CRUD feature across all Clean Architecture layers
arguments:
  - name: entity-name
    description: Name of the entity to generate (e.g., Product, Order, Customer)
    required: true
---

You are an expert .NET architect specializing in Clean Architecture. Use the `clean-architecture` skill to generate a complete CRUD feature for the entity: **{{entity-name}}**

## Your Task

Follow the workflow defined in the `clean-architecture` skill, Section B, Command `/clean-arch:add-feature`.

## Steps

1. **Gather Feature Details** using `AskUserQuestion`:
   - Properties (format: PropertyName:Type, PropertyName:Type)
   - Relationships to other entities (if any)
   - Operations needed (Create, Read, Update, Delete, Custom search)

2. **Generate Entity Class** (`src/ApplicationCore/Entities/{{entity-name}}.cs`):
   - Inherit from BaseEntity and implement IAggregateRoot
   - Properties with private setters
   - Private parameterless constructor for EF Core
   - Public constructor with Guard clauses
   - Business methods for state changes

3. **Generate EF Core Configuration** (`src/Infrastructure/Data/Config/{{entity-name}}Configuration.cs`):
   - Implement IEntityTypeConfiguration<{{entity-name}}>
   - Configure table name, property constraints
   - Configure relationships
   - Handle value objects with OwnsOne() if needed

4. **Update DbContext**:
   - Add `DbSet<{{entity-name}}> {{entity-name}}s { get; set; }`

5. **Generate Specifications** (`src/ApplicationCore/Specifications/{{entity-name}}Specifications.cs`):
   - {{entity-name}}ByIdSpec
   - {{entity-name}}ListPaginatedSpec
   - {{entity-name}}WithRelationsSpec (if has relationships)

6. **Generate Service Layer**:
   - Interface: `src/ApplicationCore/Interfaces/I{{entity-name}}Service.cs`
   - Implementation: `src/ApplicationCore/Services/{{entity-name}}Service.cs`
   - Methods: GetByIdAsync, ListAsync, CreateAsync, UpdateAsync, DeleteAsync

7. **Generate API Endpoints**:
   - If FastEndpoints: Create `src/API/Endpoints/{{entity-name}}Endpoints/` folder
     - {{entity-name}}GetByIdEndpoint.cs
     - {{entity-name}}CreateEndpoint.cs
     - {{entity-name}}UpdateEndpoint.cs
     - {{entity-name}}DeleteEndpoint.cs
     - {{entity-name}}ListEndpoint.cs
   - If Minimal APIs: Add endpoint mappings to Program.cs

8. **Generate Tests**:
   - Unit tests: `tests/UnitTests/ApplicationCore/Services/{{entity-name}}ServiceTests.cs`
   - Integration tests: `tests/IntegrationTests/Repositories/{{entity-name}}RepositoryTests.cs`

9. **Register Services**:
   - Update DI registration: `services.AddScoped<I{{entity-name}}Service, {{entity-name}}Service>()`

10. **Guide Migration Creation**:
    - Show command: `dotnet ef migrations add Add{{entity-name}} -p src/Infrastructure -s src/API`

11. **Provide Summary**:
    - List all generated files
    - Show next steps (run migration, test endpoints)

## Important

- Use the exact code patterns from the skill's Section C
- Ensure proper encapsulation (private setters, guard clauses)
- Follow repository and specification patterns
- Use Result pattern for service returns
- Include comprehensive tests
- Make entity name singular (Product not Products)

Execute this workflow interactively, generating all files with production-ready code.
