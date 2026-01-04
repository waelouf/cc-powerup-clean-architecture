---
skill: dotnet-clean-arch
description: Comprehensive skill for building .NET Clean Architecture monolithic applications with FastEndpoints, Repository pattern, and Domain-Driven Design
version: 1.0.0
author: Generated from eShopOnWeb reference application
---

# .NET Clean Architecture Skill

This skill helps you build, migrate, and maintain .NET monolithic applications following Clean Architecture principles and Domain-Driven Design patterns, based on Microsoft's eShopOnWeb reference application.

## Commands

### `/dotnet-clean-arch:new`
**Interactive project scaffolding wizard**
Scaffold a new Clean Architecture solution from scratch with your choice of API style, database, and testing setup.

### `/dotnet-clean-arch:migrate`
**Brownfield migration assistant**
Analyze and migrate existing codebases to Clean Architecture, guiding you through layer separation and refactoring.

### `/dotnet-clean-arch:add-feature <EntityName>`
**Feature generator**
Generate a complete CRUD feature across all layers: entity, repository, specifications, service, API endpoints, and tests.

### `/dotnet-clean-arch:audit`
**Architecture validator**
Scan your project for architecture violations, dependency issues, and anti-patterns with actionable fix suggestions.

### `/dotnet-clean-arch:patterns`
**Pattern library browser**
Browse and copy proven Clean Architecture patterns: Repository, Specification, Domain Events, DI, Testing, and more.

## Prerequisites

- .NET 10 SDK or later
- Basic understanding of C#, ASP.NET Core, and Entity Framework Core
- Familiarity with SOLID principles

---

# Section A: Clean Architecture Overview

## What is Clean Architecture?

Clean Architecture is an architectural pattern that separates concerns into distinct layers with clear dependency rules:

```
┌─────────────────────────────────────────┐
│         Presentation (API/Web)          │  ← User Interface
│  Controllers/Endpoints, Pages, ViewModels│
└────────────┬────────────────────────────┘
             │ depends on ↓
┌────────────▼────────────────────────────┐
│        Application Core (Domain)         │  ← Business Logic
│  Entities, Interfaces, Services,        │
│  Specifications, Domain Events           │
└────────────┬────────────────────────────┘
             │ depends on ↓
┌────────────▼────────────────────────────┐
│         Infrastructure (Data)            │  ← External Concerns
│  DbContext, Repositories, Identity,     │
│  Email, File System, External APIs      │
└──────────────────────────────────────────┘
```

**Key Principle:** Dependencies point INWARD. The Application Core has NO dependencies on external layers.

## Project Structure

```
MySolution/
├── src/
│   ├── ApplicationCore/              # Domain layer (no dependencies)
│   │   ├── Entities/                 # Domain entities & aggregates
│   │   ├── Interfaces/               # Service & repository contracts
│   │   ├── Services/                 # Business logic
│   │   ├── Specifications/           # Query objects
│   │   ├── Events/                   # Domain events
│   │   └── Exceptions/               # Domain exceptions
│   │
│   ├── Infrastructure/               # Data access & external concerns
│   │   ├── Data/                     # EF Core DbContext, migrations
│   │   │   ├── Config/               # Entity configurations
│   │   │   └── Migrations/           # EF migrations
│   │   ├── Identity/                 # ASP.NET Identity
│   │   └── Services/                 # Infrastructure services (email, etc.)
│   │
│   └── API/                          # Presentation layer (FastEndpoints)
│       ├── Endpoints/                # API endpoints
│       │   ├── ProductEndpoints/
│       │   ├── OrderEndpoints/
│       │   └── AuthEndpoints/
│       ├── Extensions/               # DI registration
│       └── Program.cs                # Application startup
│
└── tests/
    ├── UnitTests/                    # Fast, isolated tests
    │   ├── ApplicationCore/
    │   └── Builders/                 # Test data builders
    ├── IntegrationTests/             # Repository + DB tests
    └── FunctionalTests/              # API end-to-end tests
```

## Dependency Rules

1. **ApplicationCore** → No dependencies (pure domain logic)
2. **Infrastructure** → References ApplicationCore only
3. **API/Web** → References ApplicationCore and Infrastructure
4. **Tests** → Can reference any project

---

# Section B: Command Implementation Guides

## Command: `/dotnet-clean-arch:new`

When the user invokes this command, follow this workflow:

### Step 1: Gather Project Information

Use `AskUserQuestion` to gather:

```xml
<question 1>
Question: "What is your project name?"
Header: "Project Name"
Options:
- Enter custom name (text input)
Default: MySolution
</question>

<question 2>
Question: "Which API style do you prefer?"
Header: "API Style"
Options:
- FastEndpoints (Recommended for new projects)
- Minimal APIs (Lightweight, built-in)
- Controllers (Traditional MVC)
</question>

<question 3>
Question: "Which database will you use?"
Header: "Database"
Options:
- SQL Server (LocalDB for development)
- PostgreSQL
- In-Memory (for testing/prototyping)
</question>

<question 4>
Question: "Include authentication setup?"
Header: "Authentication"
Options:
- Yes, with JWT Bearer authentication
- Yes, with ASP.NET Identity
- No, I'll add it later
</question>
```

### Step 2: Generate Solution Structure

Create the directory structure and solution file:

```bash
mkdir -p src/ApplicationCore src/Infrastructure src/API tests/UnitTests tests/IntegrationTests

dotnet new sln -n {ProjectName}
dotnet new classlib -n ApplicationCore -o src/ApplicationCore
dotnet new classlib -n Infrastructure -o src/Infrastructure
dotnet new webapi -n API -o src/API
dotnet new xunit -n UnitTests -o tests/UnitTests
dotnet new xunit -n IntegrationTests -o tests/IntegrationTests

dotnet sln add src/ApplicationCore/ApplicationCore.csproj
dotnet sln add src/Infrastructure/Infrastructure.csproj
dotnet sln add src/API/API.csproj
dotnet sln add tests/UnitTests/UnitTests.csproj
dotnet sln add tests/IntegrationTests/IntegrationTests.csproj
```

### Step 3: Configure Project References

```bash
# Infrastructure depends on ApplicationCore
dotnet add src/Infrastructure/Infrastructure.csproj reference src/ApplicationCore/ApplicationCore.csproj

# API depends on both
dotnet add src/API/API.csproj reference src/ApplicationCore/ApplicationCore.csproj
dotnet add src/API/API.csproj reference src/Infrastructure/Infrastructure.csproj

# Tests depend on what they test
dotnet add tests/UnitTests/UnitTests.csproj reference src/ApplicationCore/ApplicationCore.csproj
dotnet add tests/IntegrationTests/IntegrationTests.csproj reference src/Infrastructure/Infrastructure.csproj
```

### Step 4: Add NuGet Packages

**ApplicationCore** (domain layer - minimal dependencies):
```bash
cd src/ApplicationCore
dotnet add package Ardalis.GuardClauses
dotnet add package Ardalis.Specification
dotnet add package Ardalis.Result
dotnet add package MediatR
```

**Infrastructure** (data access):
```bash
cd ../Infrastructure
dotnet add package Microsoft.EntityFrameworkCore.SqlServer  # or .Npgsql for PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.InMemory
dotnet add package Ardalis.Specification.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

**API** (presentation):
```bash
cd ../API

# If FastEndpoints chosen:
dotnet add package FastEndpoints
dotnet add package FastEndpoints.Swagger

# If Minimal APIs chosen: (no extra package needed)

# Common packages:
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

**Tests**:
```bash
cd ../../tests/UnitTests
dotnet add package NSubstitute
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package coverlet.collector

cd ../IntegrationTests
dotnet add package Microsoft.EntityFrameworkCore.InMemory
dotnet add package xunit
```

### Step 5: Generate Base Code Files

Create the following files using the patterns in Section D:

**ApplicationCore**:
- `Entities/BaseEntity.cs`
- `Interfaces/IAggregateRoot.cs`
- `Interfaces/IRepository.cs`
- `Interfaces/IReadRepository.cs`

**Infrastructure**:
- `Data/AppDbContext.cs`
- `Data/EfRepository.cs`
- `Dependencies.cs` (DI registration)

**API**:
- `Program.cs` with DI setup
- `Extensions/ServiceCollectionExtensions.cs`

**Tests**:
- `UnitTests/Builders/` directory
- `IntegrationTests/` with base test class

### Step 6: Create README

Generate a README.md with:
- Project overview
- How to run (`dotnet run --project src/API`)
- How to run tests (`dotnet test`)
- How to add migrations
- Architecture overview

### Step 7: Inform User

Provide summary:
```
✅ Created Clean Architecture solution: {ProjectName}

Projects:
- ApplicationCore: Domain layer with entities and business logic
- Infrastructure: Data access with EF Core and {database}
- API: {API style} endpoints
- UnitTests: Fast, isolated tests with NSubstitute
- IntegrationTests: Repository tests with in-memory database

Next steps:
1. cd src/API && dotnet run
2. Browse to https://localhost:5001 (or displayed URL)
3. Use /dotnet-clean-arch:add-feature to create your first entity
```

---

## Command: `/dotnet-clean-arch:migrate`

When user invokes this command for an existing project:

### Step 1: Analyze Current Project

Use Glob and Grep to discover:

```bash
# Find existing controllers
Glob: **/*Controller.cs

# Find DbContext files
Glob: **/*Context.cs or **/*DbContext.cs

# Find entity models
Grep: "public class.*\{" in Models/ or Entities/ directories

# Find data access code
Grep: "_context\." to find direct DbContext usage

# Check current project structure
ls -R src/
```

### Step 2: Create Layer Mapping Report

Generate a markdown report:

```markdown
# Migration Analysis Report

## Current Structure
- Controllers found: 15 files
- DbContext usage: Direct in 12 controllers (⚠️ violation)
- Entity models: 8 classes in /Models
- Business logic: Mixed in controllers (⚠️ violation)

## Recommended Migration Path

### Phase 1: Extract Domain Layer (ApplicationCore)
Move these files to ApplicationCore/Entities/:
- Models/Product.cs → Entities/Product.cs
- Models/Order.cs → Entities/OrderAggregate/Order.cs
- Models/Customer.cs → Entities/Customer.cs

### Phase 2: Create Repository Layer (Infrastructure)
Extract data access from controllers:
- ProductController.cs (lines 45-67) → ProductRepository.cs
- OrderController.cs (lines 23-89) → OrderRepository.cs

### Phase 3: Create Service Layer (ApplicationCore)
Extract business logic:
- ProductController.CalculateDiscount() → ProductService.cs
- OrderController.ProcessOrder() → OrderService.cs

### Phase 4: Refactor Controllers → Endpoints
Convert controllers to thin endpoints:
- ProductController → ProductEndpoints/
- OrderController → OrderEndpoints/
```

### Step 3: Interactive Migration Steps

Guide user through each file:

```
📁 Migrating Product.cs...

Current location: src/Models/Product.cs
Target location: src/ApplicationCore/Entities/Product.cs

Changes needed:
1. Make setters private for encapsulation
2. Add Guard clauses in constructor
3. Implement IAggregateRoot interface

[Show diff]
- public decimal Price { get; set; }
+ public decimal Price { get; private set; }

+   public Product(string name, decimal price)
+   {
+       Guard.Against.NullOrEmpty(name, nameof(name));
+       Guard.Against.NegativeOrZero(price, nameof(price));
+       Name = name;
+       Price = price;
+   }

Apply changes? (y/n)
```

### Step 4: Generate Missing Abstractions

Create interfaces and implementations that don't exist:

```csharp
// Generate IProductRepository based on discovered usage
public interface IProductRepository : IRepository<Product>
{
    // Discovered from controller analysis:
    Task<IEnumerable<Product>> GetProductsByCategory(int categoryId);
    Task<Product> GetByIdWithReviews(int id);
}

// Generate ProductService based on business logic extraction
public interface IProductService
{
    Task<Result<Product>> CreateProduct(string name, decimal price);
    Task<Result<decimal>> CalculateDiscount(int productId, int quantity);
}
```

### Step 5: Update DI Registration

Show user how to update `Program.cs`:

```csharp
// Old:
builder.Services.AddDbContext<AppDbContext>();

// New (Clean Architecture):
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
builder.Services.AddScoped<IProductService, ProductService>();
```

### Step 6: Create Migration Checklist

Generate TODO checklist:

```markdown
## Migration Checklist

- [ ] Phase 1: Domain Layer
  - [ ] Move Product.cs → ApplicationCore/Entities/
  - [ ] Move Order.cs → ApplicationCore/Entities/OrderAggregate/
  - [ ] Add IAggregateRoot to entities
  - [ ] Make setters private

- [ ] Phase 2: Repository Layer
  - [ ] Create IRepository<T> interface
  - [ ] Create EfRepository<T> implementation
  - [ ] Replace direct DbContext usage in controllers

- [ ] Phase 3: Service Layer
  - [ ] Extract ProductService
  - [ ] Extract OrderService
  - [ ] Move business logic from controllers

- [ ] Phase 4: Refactor Presentation
  - [ ] Convert ProductController → ProductEndpoints
  - [ ] Convert OrderController → OrderEndpoints
  - [ ] Remove business logic from endpoints

- [ ] Phase 5: Testing
  - [ ] Add unit tests for services
  - [ ] Add integration tests for repositories
  - [ ] Add functional tests for endpoints
```

---

## Command: `/dotnet-clean-arch:add-feature <EntityName>`

When user runs `/dotnet-clean-arch:add-feature Product`:

### Step 1: Gather Feature Details

Ask questions about the entity:

```xml
<question 1>
Question: "What properties does {EntityName} have?"
Header: "Properties"
Format: "PropertyName:Type, PropertyName:Type"
Example: "Name:string, Price:decimal, Description:string, Stock:int"
</question>

<question 2>
Question: "Does {EntityName} have relationships to other entities?"
Header: "Relationships"
Options:
- No relationships
- Yes, specify relationships (text input)
Example: "Category:Many-to-One, Reviews:One-to-Many"
</question>

<question 3>
Question: "What operations do you need?"
Header: "Operations"
MultiSelect: true
Options:
- Create (POST)
- Read by ID (GET /{id})
- List all with pagination (GET /)
- Update (PUT /{id})
- Delete (DELETE /{id})
- Custom search/filter
</question>
```

### Step 2: Generate Entity Class

Create `src/ApplicationCore/Entities/{EntityName}.cs`:

```csharp
using Ardalis.GuardClauses;

namespace {ProjectName}.ApplicationCore.Entities;

public class {EntityName} : BaseEntity, IAggregateRoot
{
    // Properties with private setters
    {for each property}
    public {PropertyType} {PropertyName} { get; private set; }
    {end for}

    // EF Core requires parameterless constructor
    #pragma warning disable CS8618
    private {EntityName}() { }

    // Public constructor with guards
    public {EntityName}({parameters})
    {
        {for each property}
        Guard.Against.{ValidationRule}({propertyName}, nameof({propertyName}));
        {PropertyName} = {propertyName};
        {end for}
    }

    // Business methods
    public void Update{EntityName}({parameters})
    {
        {validation and update logic}
    }
}
```

### Step 3: Generate EF Core Configuration

Create `src/Infrastructure/Data/Config/{EntityName}Configuration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace {ProjectName}.Infrastructure.Data.Config;

public class {EntityName}Configuration : IEntityTypeConfiguration<{EntityName}>
{
    public void Configure(EntityTypeBuilder<{EntityName}> builder)
    {
        builder.ToTable("{EntityName}s");

        builder.Property(e => e.{PropertyName})
            .IsRequired()
            .HasMaxLength({length});

        {for decimal properties}
        builder.Property(e => e.{PropertyName})
            .HasColumnType("decimal(18,2)");
        {end for}

        {for relationships}
        builder.HasOne(e => e.{RelatedEntity})
            .WithMany()
            .HasForeignKey(e => e.{RelatedEntity}Id);
        {end for}
    }
}
```

### Step 4: Update DbContext

Add DbSet to `src/Infrastructure/Data/AppDbContext.cs`:

```csharp
public DbSet<{EntityName}> {EntityName}s { get; set; }
```

### Step 5: Generate Specifications

Create `src/ApplicationCore/Specifications/{EntityName}Specifications.cs`:

```csharp
using Ardalis.Specification;

namespace {ProjectName}.ApplicationCore.Specifications;

// Get by ID specification
public class {EntityName}ByIdSpec : Specification<{EntityName}>
{
    public {EntityName}ByIdSpec(int id)
    {
        Query.Where(e => e.Id == id);
    }
}

// List with pagination
public class {EntityName}ListPaginatedSpec : Specification<{EntityName}>
{
    public {EntityName}ListPaginatedSpec(int skip, int take)
    {
        Query.Skip(skip).Take(take);
    }
}

{if has relationships}
// Get with related entities
public class {EntityName}WithRelationsSpec : Specification<{EntityName}>
{
    public {EntityName}WithRelationsSpec(int id)
    {
        Query
            .Where(e => e.Id == id)
            {for each relationship}
            .Include(e => e.{RelatedEntity})
            {end for};
    }
}
{end if}
```

### Step 6: Generate Service Layer

Create `src/ApplicationCore/Interfaces/I{EntityName}Service.cs`:

```csharp
using Ardalis.Result;

namespace {ProjectName}.ApplicationCore.Interfaces;

public interface I{EntityName}Service
{
    Task<Result<{EntityName}>> GetByIdAsync(int id);
    Task<Result<IEnumerable<{EntityName}>>> ListAsync(int page, int pageSize);
    Task<Result<{EntityName}>> CreateAsync({parameters});
    Task<Result> UpdateAsync(int id, {parameters});
    Task<Result> DeleteAsync(int id);
}
```

Create `src/ApplicationCore/Services/{EntityName}Service.cs`:

```csharp
using Ardalis.GuardClauses;
using Ardalis.Result;

namespace {ProjectName}.ApplicationCore.Services;

public class {EntityName}Service : I{EntityName}Service
{
    private readonly IRepository<{EntityName}> _repository;

    public {EntityName}Service(IRepository<{EntityName}> repository)
    {
        _repository = repository;
    }

    public async Task<Result<{EntityName}>> GetByIdAsync(int id)
    {
        var spec = new {EntityName}ByIdSpec(id);
        var entity = await _repository.FirstOrDefaultAsync(spec);

        if (entity == null)
            return Result<{EntityName}>.NotFound();

        return Result<{EntityName}>.Success(entity);
    }

    public async Task<Result<IEnumerable<{EntityName}>>> ListAsync(int page, int pageSize)
    {
        var spec = new {EntityName}ListPaginatedSpec(page * pageSize, pageSize);
        var entities = await _repository.ListAsync(spec);
        return Result<IEnumerable<{EntityName}>>.Success(entities);
    }

    public async Task<Result<{EntityName}>> CreateAsync({parameters})
    {
        var entity = new {EntityName}({constructorParams});
        await _repository.AddAsync(entity);
        return Result<{EntityName}>.Success(entity);
    }

    // ... other methods
}
```

### Step 7: Generate API Endpoints

**If FastEndpoints:**

Create `src/API/Endpoints/{EntityName}Endpoints/{EntityName}GetByIdEndpoint.cs`:

```csharp
using FastEndpoints;

namespace {ProjectName}.API.Endpoints.{EntityName}Endpoints;

public class {EntityName}GetByIdEndpoint : Endpoint<GetByIdRequest, GetByIdResponse>
{
    private readonly I{EntityName}Service _service;

    public {EntityName}GetByIdEndpoint(I{EntityName}Service service)
    {
        _service = service;
    }

    public override void Configure()
    {
        Get("api/{entityName}s/{id}");
        AllowAnonymous(); // or add auth: Roles("Admin")
    }

    public override async Task HandleAsync(GetByIdRequest req, CancellationToken ct)
    {
        var result = await _service.GetByIdAsync(req.Id);

        if (result.IsSuccess)
            await SendOkAsync(new GetByIdResponse { {EntityName} = result.Value }, ct);
        else
            await SendNotFoundAsync(ct);
    }
}

public record GetByIdRequest
{
    public int Id { get; init; }
}

public record GetByIdResponse
{
    public {EntityName} {EntityName} { get; init; }
}
```

**If Minimal APIs:**

Add to `Program.cs`:

```csharp
// GET /api/{entityName}s/{id}
app.MapGet("/api/{entityName}s/{id}", async (int id, I{EntityName}Service service) =>
{
    var result = await service.GetByIdAsync(id);
    return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound();
});

// POST /api/{entityName}s
app.MapPost("/api/{entityName}s", async ({EntityName}CreateDto dto, I{EntityName}Service service) =>
{
    var result = await service.CreateAsync(dto.{Property1}, dto.{Property2});
    return result.IsSuccess
        ? Results.Created($"/api/{entityName}s/{result.Value.Id}", result.Value)
        : Results.BadRequest(result.Errors);
});

// ... other endpoints
```

### Step 8: Generate Tests

Create `tests/UnitTests/ApplicationCore/Services/{EntityName}ServiceTests.cs`:

```csharp
using NSubstitute;
using Xunit;

namespace UnitTests.ApplicationCore.Services;

public class {EntityName}ServiceTests
{
    private readonly IRepository<{EntityName}> _mockRepo;
    private readonly {EntityName}Service _service;

    public {EntityName}ServiceTests()
    {
        _mockRepo = Substitute.For<IRepository<{EntityName}>>();
        _service = new {EntityName}Service(_mockRepo);
    }

    [Fact]
    public async Task GetByIdAsync_ReturnsNotFound_WhenEntityDoesNotExist()
    {
        // Arrange
        _mockRepo.FirstOrDefaultAsync(Arg.Any<Specification<{EntityName}>>())
            .Returns(({EntityName})null);

        // Act
        var result = await _service.GetByIdAsync(999);

        // Assert
        Assert.True(result.IsNotFound());
    }

    [Fact]
    public async Task CreateAsync_CreatesEntity_WithValidData()
    {
        // Arrange
        var entity = new {EntityName}({constructorArgs});
        _mockRepo.AddAsync(Arg.Any<{EntityName}>()).Returns(entity);

        // Act
        var result = await _service.CreateAsync({createArgs});

        // Assert
        Assert.True(result.IsSuccess);
        await _mockRepo.Received(1).AddAsync(Arg.Any<{EntityName}>());
    }
}
```

Create `tests/IntegrationTests/Repositories/{EntityName}RepositoryTests.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Xunit;

namespace IntegrationTests.Repositories;

public class {EntityName}RepositoryTests
{
    private readonly AppDbContext _context;
    private readonly EfRepository<{EntityName}> _repository;

    public {EntityName}RepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;
        _context = new AppDbContext(options);
        _repository = new EfRepository<{EntityName}>(_context);
    }

    [Fact]
    public async Task AddAsync_PersistsEntity()
    {
        // Arrange
        var entity = new {EntityName}({constructorArgs});

        // Act
        await _repository.AddAsync(entity);
        await _context.SaveChangesAsync();

        // Assert
        var retrieved = await _repository.GetByIdAsync(entity.Id);
        Assert.NotNull(retrieved);
        Assert.Equal(entity.{Property}, retrieved.{Property});
    }
}
```

### Step 9: Create Migration

Guide user to create EF migration:

```bash
cd src/Infrastructure
dotnet ef migrations add Add{EntityName} -s ../API/API.csproj
```

### Step 10: Register Services

Update `src/API/Extensions/ServiceCollectionExtensions.cs`:

```csharp
services.AddScoped<I{EntityName}Service, {EntityName}Service>();
```

### Step 11: Summary Report

```
✅ Feature '{EntityName}' created successfully!

Files generated:
📁 ApplicationCore/
   ├── Entities/{EntityName}.cs
   ├── Interfaces/I{EntityName}Service.cs
   ├── Services/{EntityName}Service.cs
   └── Specifications/{EntityName}Specifications.cs

📁 Infrastructure/
   └── Data/Config/{EntityName}Configuration.cs

📁 API/
   └── Endpoints/{EntityName}Endpoints/ (3 endpoints)

📁 Tests/
   ├── UnitTests/ApplicationCore/Services/{EntityName}ServiceTests.cs
   └── IntegrationTests/Repositories/{EntityName}RepositoryTests.cs

Next steps:
1. Run migration: dotnet ef database update --project src/Infrastructure --startup-project src/API
2. Test endpoints: dotnet run --project src/API
3. Run tests: dotnet test
```

---

## Command: `/dotnet-clean-arch:audit`

### Step 1: Scan Project Structure

Use Glob to discover:

```bash
Glob: src/ApplicationCore/**/*.cs
Glob: src/Infrastructure/**/*.cs
Glob: src/API/**/*.cs
```

### Step 2: Check Dependency Violations

Use Grep to find violations:

```bash
# ApplicationCore should NOT reference Infrastructure or API
Grep in src/ApplicationCore/: "using.*Infrastructure"
Grep in src/ApplicationCore/: "using.*API"

# Result: ⚠️ VIOLATION if found
```

### Step 3: Check Repository Pattern Usage

```bash
# Controllers/Endpoints should NOT use DbContext directly
Grep in src/API/: "_context\.|DbContext"

# Should use IRepository instead
Grep in src/API/: "IRepository<"
```

### Step 4: Check for Business Logic in Controllers

```bash
# Look for business logic patterns in endpoints
Grep in src/API/Endpoints/: "if.*\.Price|for.*Items|while"
Grep in src/API/Controllers/: "Calculate|Validate|Process"

# These should be in services, not endpoints
```

### Step 5: Check Aggregate Root Design

```csharp
// Find entities without IAggregateRoot that are used in IRepository
Grep: "IRepository<" → extract type → check if implements IAggregateRoot

// Find entities with public setters (should be private)
Grep in Entities/: "{ get; set; }"
```

### Step 6: Generate Audit Report

```markdown
# Architecture Audit Report

## ✅ Passing Checks (3)
- ApplicationCore has no external dependencies
- All repositories use IRepository<T> interface
- Entities use private setters for encapsulation

## ⚠️ Violations Found (2)

### 1. Direct DbContext Usage in Endpoint
**Severity:** HIGH
**Location:** src/API/Endpoints/OrderEndpoints/CreateOrderEndpoint.cs:45

**Issue:**
```csharp
private readonly AppDbContext _context; // ❌ Should use IRepository

public override async Task HandleAsync(CreateOrderRequest req)
{
    var order = await _context.Orders.FindAsync(req.Id); // ❌ Direct EF query
}
```

**Fix:**
```csharp
private readonly IRepository<Order> _repository; // ✅ Use repository

public override async Task HandleAsync(CreateOrderRequest req)
{
    var order = await _repository.GetByIdAsync(req.Id); // ✅ Use repository method
}
```

### 2. Business Logic in Endpoint
**Severity:** MEDIUM
**Location:** src/API/Endpoints/ProductEndpoints/CreateProductEndpoint.cs:32

**Issue:**
```csharp
// Discount calculation logic in endpoint ❌
if (req.Quantity > 10)
{
    price = price * 0.9m; // 10% discount
}
```

**Fix:**
Move to service layer:
```csharp
// In IProductService
Task<Result<decimal>> CalculatePrice(int productId, int quantity);

// In ProductService
public async Task<Result<decimal>> CalculatePrice(int productId, int quantity)
{
    var product = await _repository.GetByIdAsync(productId);
    var price = product.Price;

    if (quantity > 10)
        price *= 0.9m; // Apply volume discount

    return Result<decimal>.Success(price);
}
```

## 📊 Summary
- Total files scanned: 47
- Violations found: 2
- Passing checks: 3
- Recommended priority: Fix HIGH severity issues first

## 🔧 Quick Fix Commands
1. Refactor OrderEndpoint to use repository
2. Extract price calculation to ProductService
```

---

## Command: `/dotnet-clean-arch:patterns`

When user runs this command, show interactive menu:

```
📚 Clean Architecture Pattern Library

Select a pattern to view:

1. Repository Pattern
2. Specification Pattern
3. Domain Events with MediatR
4. Service Layer Pattern
5. FastEndpoints CRUD
6. Minimal API CRUD
7. Entity Aggregate Design
8. Value Objects
9. EF Core Configuration
10. DI Registration
11. Unit Testing with Mocks
12. Integration Testing
13. Test Data Builders

Enter number (or 'q' to quit):
```

When user selects a pattern, show the code example from Section D and offer:
```
Options:
(v) View full implementation
(c) Copy to clipboard
(f) Create file in current project
(b) Back to menu
```

---

# Section C: Embedded Code Patterns

This section contains complete, production-ready code patterns extracted from Microsoft's eShopOnWeb reference application.

## Pattern 1: Base Entity & Aggregate Root

**File:** `ApplicationCore/Entities/BaseEntity.cs`

```csharp
namespace YourProject.ApplicationCore.Entities;

/// <summary>
/// Base class for all entities with integer primary key.
/// Can be modified to BaseEntity<T> for different key types.
/// </summary>
public abstract class BaseEntity
{
    /// <summary>
    /// Entity primary key. Virtual allows EF Core to override for lazy loading.
    /// Protected setter enforces immutability after creation.
    /// </summary>
    public virtual int Id { get; protected set; }
}
```

**File:** `ApplicationCore/Interfaces/IAggregateRoot.cs`

```csharp
namespace YourProject.ApplicationCore.Interfaces;

/// <summary>
/// Marker interface for aggregate roots in DDD.
/// Only aggregate roots can be accessed through repositories.
/// </summary>
public interface IAggregateRoot
{
}
```

**Usage Example:**

```csharp
public class Product : BaseEntity, IAggregateRoot
{
    // Only aggregate roots implement this interface
    // Child entities (like OrderItem) do NOT implement IAggregateRoot
}
```

---

## Pattern 2: Repository Pattern

**File:** `ApplicationCore/Interfaces/IRepository.cs`

```csharp
using Ardalis.Specification;

namespace YourProject.ApplicationCore.Interfaces;

/// <summary>
/// Repository for write operations. Only aggregate roots can use this.
/// Inherits from Ardalis.Specification IRepositoryBase for rich query capabilities.
/// </summary>
public interface IRepository<T> : IRepositoryBase<T> where T : class, IAggregateRoot
{
    // Methods inherited from IRepositoryBase:
    // - GetByIdAsync(int id)
    // - AddAsync(T entity)
    // - UpdateAsync(T entity)
    // - DeleteAsync(T entity)
    // - FirstOrDefaultAsync(ISpecification<T> specification)
    // - ListAsync(ISpecification<T> specification)
    // - CountAsync(ISpecification<T> specification)
}
```

**File:** `ApplicationCore/Interfaces/IReadRepository.cs`

```csharp
using Ardalis.Specification;

namespace YourProject.ApplicationCore.Interfaces;

/// <summary>
/// Read-only repository for queries. Use for CQRS read models.
/// </summary>
public interface IReadRepository<T> : IReadRepositoryBase<T> where T : class, IAggregateRoot
{
    // Read-only methods:
    // - GetByIdAsync(int id)
    // - FirstOrDefaultAsync(ISpecification<T> specification)
    // - ListAsync(ISpecification<T> specification)
    // - CountAsync(ISpecification<T> specification)
}
```

**File:** `Infrastructure/Data/EfRepository.cs`

```csharp
using Ardalis.Specification.EntityFrameworkCore;
using YourProject.ApplicationCore.Interfaces;
using Microsoft.EntityFrameworkCore;

namespace YourProject.Infrastructure.Data;

/// <summary>
/// Generic EF Core repository implementation.
/// Thin wrapper around Ardalis.Specification RepositoryBase.
/// </summary>
public class EfRepository<T> : RepositoryBase<T>, IReadRepository<T>, IRepository<T>
    where T : class, IAggregateRoot
{
    public EfRepository(AppDbContext dbContext) : base(dbContext)
    {
    }
}
```

**Usage in Service:**

```csharp
public class ProductService : IProductService
{
    private readonly IRepository<Product> _repository;

    public ProductService(IRepository<Product> repository)
    {
        _repository = repository;
    }

    public async Task<Product> GetByIdAsync(int id)
    {
        // Simple primary key lookup
        return await _repository.GetByIdAsync(id);
    }

    public async Task<Product> GetByIdWithDetailsAsync(int id)
    {
        // Complex query with specification
        var spec = new ProductWithDetailsSpec(id);
        return await _repository.FirstOrDefaultAsync(spec);
    }

    public async Task<IEnumerable<Product>> ListAsync(int page, int pageSize)
    {
        // Paginated query
        var spec = new ProductListPaginatedSpec(page * pageSize, pageSize);
        return await _repository.ListAsync(spec);
    }

    public async Task<Product> CreateAsync(string name, decimal price)
    {
        var product = new Product(name, price);
        await _repository.AddAsync(product);
        return product;
    }

    public async Task UpdateAsync(Product product)
    {
        await _repository.UpdateAsync(product);
    }

    public async Task DeleteAsync(Product product)
    {
        await _repository.DeleteAsync(product);
    }
}
```

---

## Pattern 3: Specification Pattern

The Specification pattern encapsulates query logic in reusable, testable objects.

**Simple Specification - Filter by Property:**

```csharp
using Ardalis.Specification;

namespace YourProject.ApplicationCore.Specifications;

public class ProductByNameSpec : Specification<Product>
{
    public ProductByNameSpec(string name)
    {
        Query.Where(p => p.Name == name);
    }
}
```

**Specification with Multiple IDs:**

```csharp
public class ProductsByIdsSpec : Specification<Product>
{
    public ProductsByIdsSpec(params int[] ids)
    {
        Query.Where(p => ids.Contains(p.Id));
    }
}
```

**Specification with Eager Loading (Include):**

```csharp
public class ProductWithCategorySpec : Specification<Product>
{
    public ProductWithCategorySpec(int productId)
    {
        Query
            .Where(p => p.Id == productId)
            .Include(p => p.Category);
    }
}
```

**Specification with Multiple Includes:**

```csharp
public class OrderWithItemsSpec : Specification<Order>
{
    public OrderWithItemsSpec(int orderId)
    {
        Query
            .Where(o => o.Id == orderId)
            .Include(o => o.OrderItems)
                .ThenInclude(i => i.Product);
    }
}
```

**Specification with Optional Filters:**

```csharp
public class ProductFilterSpec : Specification<Product>
{
    public ProductFilterSpec(int? categoryId, decimal? minPrice, decimal? maxPrice)
    {
        Query.Where(p =>
            (!categoryId.HasValue || p.CategoryId == categoryId) &&
            (!minPrice.HasValue || p.Price >= minPrice) &&
            (!maxPrice.HasValue || p.Price <= maxPrice));
    }
}
```

**Specification with Pagination:**

```csharp
public class ProductListPaginatedSpec : Specification<Product>
{
    public ProductListPaginatedSpec(int skip, int take, int? categoryId = null)
    {
        if (take == 0) take = int.MaxValue; // No limit

        Query
            .Where(p => !categoryId.HasValue || p.CategoryId == categoryId)
            .OrderBy(p => p.Name)
            .Skip(skip)
            .Take(take);
    }
}
```

**Specification with Ordering:**

```csharp
public class ProductsSortedByPriceSpec : Specification<Product>
{
    public ProductsSortedByPriceSpec(bool descending = false)
    {
        if (descending)
            Query.OrderByDescending(p => p.Price);
        else
            Query.OrderBy(p => p.Price);
    }
}
```

**Testing Specifications:**

```csharp
public class ProductFilterSpecTests
{
    [Fact]
    public void FiltersProductsByCategory()
    {
        // Arrange
        var spec = new ProductFilterSpec(categoryId: 1, null, null);
        var products = GetTestProducts();

        // Act - Specifications have an Evaluate method for testing
        var result = spec.Evaluate(products).ToList();

        // Assert
        Assert.All(result, p => Assert.Equal(1, p.CategoryId));
    }

    private List<Product> GetTestProducts()
    {
        return new List<Product>
        {
            new Product("Product 1", 10m) { CategoryId = 1 },
            new Product("Product 2", 20m) { CategoryId = 2 },
            new Product("Product 3", 30m) { CategoryId = 1 }
        };
    }
}
```

---

## Pattern 4: Entity Aggregate Design

**Simple Aggregate Root:**

```csharp
using Ardalis.GuardClauses;
using YourProject.ApplicationCore.Interfaces;

namespace YourProject.ApplicationCore.Entities;

public class Product : BaseEntity, IAggregateRoot
{
    // Properties with private setters for encapsulation
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public string Description { get; private set; }
    public int CategoryId { get; private set; }

    // Navigation property (can be null until loaded)
    public Category? Category { get; private set; }

    // EF Core requires parameterless constructor (private to hide from public API)
    #pragma warning disable CS8618 // Non-nullable field must contain a non-null value
    private Product() { }
    #pragma warning restore CS8618

    // Public constructor with validation
    public Product(string name, decimal price, string description = "")
    {
        Guard.Against.NullOrEmpty(name, nameof(name));
        Guard.Against.NegativeOrZero(price, nameof(price));

        Name = name;
        Price = price;
        Description = description ?? string.Empty;
    }

    // Business methods for state changes
    public void UpdatePrice(decimal newPrice)
    {
        Guard.Against.NegativeOrZero(newPrice, nameof(newPrice));
        Price = newPrice;
    }

    public void UpdateDetails(string name, string description)
    {
        Guard.Against.NullOrEmpty(name, nameof(name));
        Name = name;
        Description = description ?? string.Empty;
    }

    public void AssignToCategory(int categoryId)
    {
        Guard.Against.NegativeOrZero(categoryId, nameof(categoryId));
        CategoryId = categoryId;
    }
}
```

**Complex Aggregate with Collection:**

```csharp
using Ardalis.GuardClauses;

namespace YourProject.ApplicationCore.Entities.OrderAggregate;

public class Order : BaseEntity, IAggregateRoot
{
    public string BuyerId { get; private set; }
    public DateTimeOffset OrderDate { get; private set; } = DateTimeOffset.Now;
    public Address ShipToAddress { get; private set; }

    // Private backing field for collection
    private readonly List<OrderItem> _orderItems = new List<OrderItem>();

    // Public read-only access via AsReadOnly() wrapper (more efficient than ToList())
    public IReadOnlyCollection<OrderItem> OrderItems => _orderItems.AsReadOnly();

    #pragma warning disable CS8618
    private Order() { }
    #pragma warning restore CS8618

    public Order(string buyerId, Address shipToAddress, List<OrderItem> items)
    {
        Guard.Against.NullOrEmpty(buyerId, nameof(buyerId));
        Guard.Against.Null(shipToAddress, nameof(shipToAddress));
        Guard.Against.NullOrEmpty(items, nameof(items));

        BuyerId = buyerId;
        ShipToAddress = shipToAddress;
        _orderItems.AddRange(items);
    }

    // Aggregate root controls how items are added
    public void AddItem(int productId, string productName, decimal unitPrice, int quantity)
    {
        var existingItem = _orderItems.FirstOrDefault(i => i.ProductId == productId);

        if (existingItem != null)
        {
            existingItem.AddQuantity(quantity);
        }
        else
        {
            _orderItems.Add(new OrderItem(productId, productName, unitPrice, quantity));
        }
    }

    // Business logic methods
    public decimal Total()
    {
        return _orderItems.Sum(i => i.UnitPrice * i.Quantity);
    }

    public void RemoveItem(int orderItemId)
    {
        var item = _orderItems.FirstOrDefault(i => i.Id == orderItemId);
        if (item != null)
        {
            _orderItems.Remove(item);
        }
    }
}
```

**Child Entity (Not an Aggregate Root):**

```csharp
namespace YourProject.ApplicationCore.Entities.OrderAggregate;

// OrderItem is a child entity, NOT an aggregate root
// It's only accessible through the Order aggregate
public class OrderItem : BaseEntity
{
    public int ProductId { get; private set; }
    public string ProductName { get; private set; }
    public decimal UnitPrice { get; private set; }
    public int Quantity { get; private set; }

    #pragma warning disable CS8618
    private OrderItem() { }
    #pragma warning restore CS8618

    public OrderItem(int productId, string productName, decimal unitPrice, int quantity)
    {
        Guard.Against.NegativeOrZero(productId, nameof(productId));
        Guard.Against.NullOrEmpty(productName, nameof(productName));
        Guard.Against.NegativeOrZero(unitPrice, nameof(unitPrice));
        Guard.Against.NegativeOrZero(quantity, nameof(quantity));

        ProductId = productId;
        ProductName = productName;
        UnitPrice = unitPrice;
        Quantity = quantity;
    }

    public void AddQuantity(int quantity)
    {
        Guard.Against.NegativeOrZero(quantity, nameof(quantity));
        Quantity += quantity;
    }

    public void SetQuantity(int quantity)
    {
        Guard.Against.Negative(quantity, nameof(quantity));
        Quantity = quantity;
    }
}
```

---

## Pattern 5: Value Objects

Value objects are immutable objects defined by their attributes, not identity.

```csharp
namespace YourProject.ApplicationCore.Entities;

/// <summary>
/// Value object - no Id, immutable, compared by value.
/// Configured with OwnsOne() in EF Core.
/// </summary>
public class Address
{
    public string Street { get; private set; }
    public string City { get; private set; }
    public string State { get; private set; }
    public string Country { get; private set; }
    public string ZipCode { get; private set; }

    #pragma warning disable CS8618
    private Address() { }
    #pragma warning restore CS8618

    public Address(string street, string city, string state, string country, string zipCode)
    {
        Guard.Against.NullOrEmpty(street, nameof(street));
        Guard.Against.NullOrEmpty(city, nameof(city));
        Guard.Against.NullOrEmpty(state, nameof(state));
        Guard.Against.NullOrEmpty(country, nameof(country));
        Guard.Against.NullOrEmpty(zipCode, nameof(zipCode));

        Street = street;
        City = city;
        State = state;
        Country = country;
        ZipCode = zipCode;
    }

    // Value objects can have behavior
    public string GetFullAddress()
    {
        return $"{Street}, {City}, {State} {ZipCode}, {Country}";
    }
}
```

**Snapshot Value Object Pattern:**

```csharp
/// <summary>
/// Represents a snapshot of product info at order time.
/// If the product changes later, the order's record stays unchanged.
/// </summary>
public class ProductSnapshot
{
    public int ProductId { get; private set; }
    public string ProductName { get; private set; }
    public string PictureUri { get; private set; }

    #pragma warning disable CS8618
    private ProductSnapshot() { }
    #pragma warning restore CS8618

    public ProductSnapshot(int productId, string productName, string pictureUri)
    {
        Guard.Against.NegativeOrZero(productId, nameof(productId));
        Guard.Against.NullOrEmpty(productName, nameof(productName));

        ProductId = productId;
        ProductName = productName;
        PictureUri = pictureUri ?? string.Empty;
    }
}
```

---

## Pattern 6: EF Core Configuration

**Entity Configuration with Owned Type:**

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace YourProject.Infrastructure.Data.Config;

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        // Configure private collection for EF Core
        var navigation = builder.Metadata.FindNavigation(nameof(Order.OrderItems));
        navigation?.SetPropertyAccessMode(PropertyAccessMode.Field);

        builder.Property(o => o.BuyerId)
            .IsRequired()
            .HasMaxLength(256);

        // Owned type (value object) - stored as columns in Orders table
        builder.OwnsOne(o => o.ShipToAddress, address =>
        {
            address.WithOwner(); // Required

            address.Property(a => a.Street)
                .HasMaxLength(180)
                .IsRequired();

            address.Property(a => a.City)
                .HasMaxLength(100)
                .IsRequired();

            address.Property(a => a.State)
                .HasMaxLength(60);

            address.Property(a => a.Country)
                .HasMaxLength(90)
                .IsRequired();

            address.Property(a => a.ZipCode)
                .HasMaxLength(18)
                .IsRequired();
        });

        builder.Navigation(o => o.ShipToAddress).IsRequired();
    }
}
```

**Entity Configuration with Hi-Lo ID Generation:**

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");

        // Hi-Lo pattern for efficient, gapless ID generation
        builder.Property(p => p.Id)
            .UseHiLo("product_hilo")
            .IsRequired();

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(p => p.Price)
            .IsRequired()
            .HasColumnType("decimal(18,2)"); // Precision for money

        builder.Property(p => p.Description)
            .HasMaxLength(500);

        // Foreign key relationship
        builder.HasOne(p => p.Category)
            .WithMany()
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict); // Prevent cascade delete
    }
}
```

**DbContext with Auto-Discovery:**

```csharp
using Microsoft.EntityFrameworkCore;
using System.Reflection;

namespace YourProject.Infrastructure.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Auto-discover all IEntityTypeConfiguration implementations in this assembly
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

---

## Pattern 7: Service Layer

**Service Interface:**

```csharp
using Ardalis.Result;

namespace YourProject.ApplicationCore.Interfaces;

public interface IProductService
{
    Task<Result<Product>> GetByIdAsync(int id);
    Task<Result<IEnumerable<Product>>> ListAsync(int page, int pageSize);
    Task<Result<Product>> CreateAsync(string name, decimal price, string description);
    Task<Result> UpdateAsync(int id, string name, decimal price, string description);
    Task<Result> DeleteAsync(int id);
    Task<Result<decimal>> CalculateTotalPrice(int productId, int quantity);
}
```

**Service Implementation with Repository & Specifications:**

```csharp
using Ardalis.GuardClauses;
using Ardalis.Result;

namespace YourProject.ApplicationCore.Services;

public class ProductService : IProductService
{
    private readonly IRepository<Product> _repository;
    private readonly IAppLogger<ProductService> _logger;

    public ProductService(
        IRepository<Product> repository,
        IAppLogger<ProductService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<Result<Product>> GetByIdAsync(int id)
    {
        var spec = new ProductByIdSpec(id);
        var product = await _repository.FirstOrDefaultAsync(spec);

        if (product == null)
        {
            _logger.LogWarning("Product {ProductId} not found", id);
            return Result<Product>.NotFound();
        }

        return Result<Product>.Success(product);
    }

    public async Task<Result<IEnumerable<Product>>> ListAsync(int page, int pageSize)
    {
        Guard.Against.Negative(page, nameof(page));
        Guard.Against.NegativeOrZero(pageSize, nameof(pageSize));

        var spec = new ProductListPaginatedSpec(page * pageSize, pageSize);
        var products = await _repository.ListAsync(spec);

        return Result<IEnumerable<Product>>.Success(products);
    }

    public async Task<Result<Product>> CreateAsync(string name, decimal price, string description)
    {
        // Check for duplicates using specification
        var nameSpec = new ProductByNameSpec(name);
        var existingCount = await _repository.CountAsync(nameSpec);

        if (existingCount > 0)
        {
            return Result<Product>.Error($"Product with name '{name}' already exists");
        }

        var product = new Product(name, price, description);
        await _repository.AddAsync(product);

        _logger.LogInformation("Created product {ProductId}: {ProductName}", product.Id, product.Name);

        return Result<Product>.Success(product);
    }

    public async Task<Result> UpdateAsync(int id, string name, decimal price, string description)
    {
        var product = await _repository.GetByIdAsync(id);

        if (product == null)
            return Result.NotFound();

        product.UpdateDetails(name, description);
        product.UpdatePrice(price);

        await _repository.UpdateAsync(product);

        return Result.Success();
    }

    public async Task<Result> DeleteAsync(int id)
    {
        var product = await _repository.GetByIdAsync(id);

        if (product == null)
            return Result.NotFound();

        await _repository.DeleteAsync(product);

        _logger.LogInformation("Deleted product {ProductId}", id);

        return Result.Success();
    }

    public async Task<Result<decimal>> CalculateTotalPrice(int productId, int quantity)
    {
        var product = await _repository.GetByIdAsync(productId);

        if (product == null)
            return Result<decimal>.NotFound();

        var total = product.Price * quantity;

        // Business rule: Volume discount
        if (quantity >= 10)
        {
            total *= 0.9m; // 10% discount
        }

        return Result<decimal>.Success(total);
    }
}
```

**Service with Multiple Repositories:**

```csharp
public class OrderService : IOrderService
{
    private readonly IRepository<Order> _orderRepository;
    private readonly IRepository<Product> _productRepository;
    private readonly IRepository<Basket> _basketRepository;
    private readonly IMediator _mediator;

    public OrderService(
        IRepository<Order> orderRepository,
        IRepository<Product> productRepository,
        IRepository<Basket> basketRepository,
        IMediator mediator)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _basketRepository = basketRepository;
        _mediator = mediator;
    }

    public async Task<Result<Order>> CreateOrderFromBasketAsync(int basketId, Address shippingAddress)
    {
        // Get basket with items
        var basketSpec = new BasketWithItemsSpec(basketId);
        var basket = await _basketRepository.FirstOrDefaultAsync(basketSpec);

        if (basket == null)
            return Result<Order>.NotFound("Basket not found");

        if (!basket.Items.Any())
            return Result<Order>.Error("Basket is empty");

        // Get all products referenced in basket
        var productIds = basket.Items.Select(i => i.ProductId).ToArray();
        var productSpec = new ProductsByIdsSpec(productIds);
        var products = await _productRepository.ListAsync(productSpec);

        // Create order items
        var orderItems = basket.Items.Select(basketItem =>
        {
            var product = products.First(p => p.Id == basketItem.ProductId);
            return new OrderItem(
                product.Id,
                product.Name,
                basketItem.UnitPrice,
                basketItem.Quantity);
        }).ToList();

        // Create order
        var order = new Order(basket.BuyerId, shippingAddress, orderItems);
        await _orderRepository.AddAsync(order);

        // Publish domain event
        await _mediator.Publish(new OrderCreatedEvent(order));

        return Result<Order>.Success(order);
    }
}
```

---

## Pattern 8: Domain Events with MediatR

**Domain Event:**

```csharp
using Ardalis.SharedKernel;

namespace YourProject.ApplicationCore.Events;

public class OrderCreatedEvent : DomainEventBase
{
    public Order Order { get; init; }

    public OrderCreatedEvent(Order order)
    {
        Order = order;
    }
}
```

**Event Handler:**

```csharp
using MediatR;

namespace YourProject.ApplicationCore.Handlers;

public class OrderCreatedHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly ILogger<OrderCreatedHandler> _logger;
    private readonly IEmailSender _emailSender;

    public OrderCreatedHandler(
        ILogger<OrderCreatedHandler> logger,
        IEmailSender emailSender)
    {
        _logger = logger;
        _emailSender = emailSender;
    }

    public async Task Handle(OrderCreatedEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Order {OrderId} created by {BuyerId}",
            notification.Order.Id,
            notification.Order.BuyerId);

        // Send confirmation email
        await _emailSender.SendEmailAsync(
            notification.Order.BuyerId,
            "Order Confirmation",
            $"Your order #{notification.Order.Id} has been placed. Total: {notification.Order.Total():C}");
    }
}
```

**Publishing Events:**

```csharp
// In service:
var order = new Order(buyerId, address, items);
await _repository.AddAsync(order);

// Publish event
await _mediator.Publish(new OrderCreatedEvent(order), cancellationToken);
```

---

## Pattern 9: FastEndpoints API

**Base Request/Response Classes:**

```csharp
namespace YourProject.API.Endpoints;

public abstract class BaseMessage
{
    protected Guid _correlationId = Guid.NewGuid();
    public Guid CorrelationId() => _correlationId;
}

public abstract class BaseRequest : BaseMessage { }

public abstract class BaseResponse : BaseMessage
{
    public BaseResponse(Guid correlationId)
    {
        _correlationId = correlationId;
    }

    public BaseResponse() { }
}
```

**GET Endpoint:**

```csharp
using FastEndpoints;
using Microsoft.AspNetCore.Http.HttpResults;

namespace YourProject.API.Endpoints.ProductEndpoints;

public class ProductGetByIdEndpoint : Endpoint<GetByIdRequest, Results<Ok<GetByIdResponse>, NotFound>>
{
    private readonly IProductService _service;

    public ProductGetByIdEndpoint(IProductService service)
    {
        _service = service;
    }

    public override void Configure()
    {
        Get("api/products/{id}");
        AllowAnonymous();
        Description(d => d
            .Produces<GetByIdResponse>(200)
            .Produces(404)
            .WithTags("Products"));
    }

    public override async Task<Results<Ok<GetByIdResponse>, NotFound>> ExecuteAsync(
        GetByIdRequest request,
        CancellationToken ct)
    {
        var result = await _service.GetByIdAsync(request.Id);

        if (!result.IsSuccess)
            return TypedResults.NotFound();

        var response = new GetByIdResponse(request.CorrelationId())
        {
            Product = result.Value
        };

        return TypedResults.Ok(response);
    }
}

public record GetByIdRequest
{
    public int Id { get; init; }
}

public class GetByIdResponse : BaseResponse
{
    public GetByIdResponse(Guid correlationId) : base(correlationId) { }
    public GetByIdResponse() { }

    public Product Product { get; init; }
}
```

**POST Endpoint with Validation:**

```csharp
public class ProductCreateEndpoint : Endpoint<CreateRequest, CreateResponse>
{
    private readonly IProductService _service;

    public ProductCreateEndpoint(IProductService service)
    {
        _service = service;
    }

    public override void Configure()
    {
        Post("api/products");
        Roles("Admin", "ProductManager");
        AuthSchemes(JwtBearerDefaults.AuthenticationScheme);
        Description(d => d
            .Produces<CreateResponse>(201)
            .Produces(400)
            .WithTags("Products"));
    }

    public override async Task HandleAsync(CreateRequest req, CancellationToken ct)
    {
        var result = await _service.CreateAsync(req.Name, req.Price, req.Description);

        if (!result.IsSuccess)
        {
            await SendErrorsAsync(cancellation: ct);
            return;
        }

        var response = new CreateResponse(req.CorrelationId())
        {
            Product = result.Value
        };

        await SendCreatedAtAsync<ProductGetByIdEndpoint>(
            new { Id = result.Value.Id },
            response,
            cancellation: ct);
    }
}

public record CreateRequest : BaseRequest
{
    public string Name { get; init; }
    public decimal Price { get; init; }
    public string Description { get; init; }
}

public class CreateResponse : BaseResponse
{
    public CreateResponse(Guid correlationId) : base(correlationId) { }
    public CreateResponse() { }

    public Product Product { get; init; }
}
```

**LIST Endpoint with Pagination:**

```csharp
public class ProductListEndpoint : Endpoint<ListRequest, ListResponse>
{
    private readonly IProductService _service;

    public ProductListEndpoint(IProductService service)
    {
        _service = service;
    }

    public override void Configure()
    {
        Get("api/products");
        AllowAnonymous();
    }

    public override async Task<ListResponse> ExecuteAsync(ListRequest req, CancellationToken ct)
    {
        var result = await _service.ListAsync(req.Page, req.PageSize);

        var response = new ListResponse(req.CorrelationId())
        {
            Products = result.Value.ToList(),
            Page = req.Page,
            PageSize = req.PageSize,
            TotalCount = result.Value.Count() // In real app, get from service
        };

        return response;
    }
}

public record ListRequest : BaseRequest
{
    public int Page { get; init; } = 0;
    public int PageSize { get; init; } = 10;
}

public class ListResponse : BaseResponse
{
    public ListResponse(Guid correlationId) : base(correlationId) { }
    public ListResponse() { }

    public List<Product> Products { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalCount { get; init; }
}
```

**UPDATE Endpoint:**

```csharp
public class ProductUpdateEndpoint : Endpoint<UpdateRequest, Results<Ok, NotFound>>
{
    private readonly IProductService _service;

    public ProductUpdateEndpoint(IProductService service)
    {
        _service = service;
    }

    public override void Configure()
    {
        Put("api/products/{id}");
        Roles("Admin", "ProductManager");
    }

    public override async Task<Results<Ok, NotFound>> ExecuteAsync(UpdateRequest req, CancellationToken ct)
    {
        var result = await _service.UpdateAsync(req.Id, req.Name, req.Price, req.Description);

        if (!result.IsSuccess)
            return TypedResults.NotFound();

        return TypedResults.Ok();
    }
}

public record UpdateRequest : BaseRequest
{
    public int Id { get; init; }
    public string Name { get; init; }
    public decimal Price { get; init; }
    public string Description { get; init; }
}
```

**DELETE Endpoint:**

```csharp
public class ProductDeleteEndpoint : Endpoint<DeleteRequest, Results<NoContent, NotFound>>
{
    private readonly IProductService _service;

    public ProductDeleteEndpoint(IProductService service)
    {
        _service = service;
    }

    public override void Configure()
    {
        Delete("api/products/{id}");
        Roles("Admin");
    }

    public override async Task<Results<NoContent, NotFound>> ExecuteAsync(DeleteRequest req, CancellationToken ct)
    {
        var result = await _service.DeleteAsync(req.Id);

        if (!result.IsSuccess)
            return TypedResults.NotFound();

        return TypedResults.NoContent();
    }
}

public record DeleteRequest
{
    public int Id { get; init; }
}
```

---

## Pattern 10: Minimal APIs

For users who prefer Minimal APIs over FastEndpoints:

**Program.cs with Minimal APIs:**

```csharp
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
builder.Services.AddScoped<IProductService, ProductService>();

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Product endpoints
var products = app.MapGroup("/api/products").WithTags("Products");

products.MapGet("/", async (IProductService service, int page = 0, int pageSize = 10) =>
{
    var result = await service.ListAsync(page, pageSize);
    return Results.Ok(result.Value);
});

products.MapGet("/{id}", async (int id, IProductService service) =>
{
    var result = await service.GetByIdAsync(id);
    return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound();
});

products.MapPost("/", async (CreateProductDto dto, IProductService service) =>
{
    var result = await service.CreateAsync(dto.Name, dto.Price, dto.Description);
    return result.IsSuccess
        ? Results.Created($"/api/products/{result.Value.Id}", result.Value)
        : Results.BadRequest(result.Errors);
})
.RequireAuthorization(policy => policy.RequireRole("Admin"));

products.MapPut("/{id}", async (int id, UpdateProductDto dto, IProductService service) =>
{
    var result = await service.UpdateAsync(id, dto.Name, dto.Price, dto.Description);
    return result.IsSuccess ? Results.Ok() : Results.NotFound();
})
.RequireAuthorization();

products.MapDelete("/{id}", async (int id, IProductService service) =>
{
    var result = await service.DeleteAsync(id);
    return result.IsSuccess ? Results.NoContent() : Results.NotFound();
})
.RequireAuthorization(policy => policy.RequireRole("Admin"));

app.Run();

// DTOs
record CreateProductDto(string Name, decimal Price, string Description);
record UpdateProductDto(string Name, decimal Price, string Description);
```

---

## Pattern 11: DI Registration

**Extension Method Pattern:**

```csharp
using Microsoft.EntityFrameworkCore;

namespace YourProject.API.Extensions;

public static class ServiceCollectionExtensions
{
    public static void AddDatabaseContexts(this IServiceCollection services, IConfiguration configuration)
    {
        bool useInMemory = configuration.GetValue<bool>("UseInMemoryDatabase");

        if (useInMemory)
        {
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("AppDb"));
        }
        else
        {
            var connectionString = configuration.GetConnectionString("DefaultConnection");
            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlServer(connectionString));
        }
    }

    public static void AddRepositories(this IServiceCollection services)
    {
        // Generic repository registration
        services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
        services.AddScoped(typeof(IReadRepository<>), typeof(EfRepository<>));
    }

    public static void AddApplicationServices(this IServiceCollection services)
    {
        // Register all services
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IBasketService, BasketService>();

        // Register adapters/helpers
        services.AddScoped(typeof(IAppLogger<>), typeof(LoggerAdapter<>));
    }

    public static void AddJwtAuthentication(this IServiceCollection services, IConfiguration configuration)
    {
        var jwtSettings = configuration.GetSection("JwtSettings");
        var secretKey = jwtSettings.GetValue<string>("SecretKey");

        services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = jwtSettings.GetValue<string>("Issuer"),
                ValidAudience = jwtSettings.GetValue<string>("Audience"),
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey))
            };
        });
    }
}
```

**Program.cs with Extensions:**

```csharp
var builder = WebApplication.CreateBuilder(args);

// Use extension methods for clean registration
builder.Services.AddDatabaseContexts(builder.Configuration);
builder.Services.AddRepositories();
builder.Services.AddApplicationServices();
builder.Services.AddJwtAuthentication(builder.Configuration);

builder.Services.AddFastEndpoints();
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<Program>());
builder.Services.AddAutoMapper(typeof(Program).Assembly);

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseFastEndpoints();

app.Run();
```

---

## Pattern 12: Unit Testing with NSubstitute

**Test Class Structure:**

```csharp
using NSubstitute;
using Xunit;
using Ardalis.Result;

namespace UnitTests.ApplicationCore.Services;

public class ProductServiceTests
{
    private readonly IRepository<Product> _mockRepo;
    private readonly IAppLogger<ProductService> _mockLogger;
    private readonly ProductService _service;

    public ProductServiceTests()
    {
        _mockRepo = Substitute.For<IRepository<Product>>();
        _mockLogger = Substitute.For<IAppLogger<ProductService>>();
        _service = new ProductService(_mockRepo, _mockLogger);
    }

    [Fact]
    public async Task GetByIdAsync_ReturnsProduct_WhenExists()
    {
        // Arrange
        var product = new Product("Test Product", 10.99m);
        _mockRepo.FirstOrDefaultAsync(Arg.Any<Specification<Product>>())
            .Returns(product);

        // Act
        var result = await _service.GetByIdAsync(1);

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(product, result.Value);
    }

    [Fact]
    public async Task GetByIdAsync_ReturnsNotFound_WhenDoesNotExist()
    {
        // Arrange
        _mockRepo.FirstOrDefaultAsync(Arg.Any<Specification<Product>>())
            .Returns((Product)null);

        // Act
        var result = await _service.GetByIdAsync(999);

        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal(ResultStatus.NotFound, result.Status);
    }

    [Fact]
    public async Task CreateAsync_CreatesProduct_WithValidData()
    {
        // Arrange
        _mockRepo.CountAsync(Arg.Any<Specification<Product>>()).Returns(0);
        _mockRepo.AddAsync(Arg.Any<Product>()).Returns(x => x.Arg<Product>());

        // Act
        var result = await _service.CreateAsync("New Product", 19.99m, "Description");

        // Assert
        Assert.True(result.IsSuccess);
        await _mockRepo.Received(1).AddAsync(Arg.Any<Product>());
    }

    [Fact]
    public async Task CreateAsync_ReturnsError_WhenNameExists()
    {
        // Arrange
        _mockRepo.CountAsync(Arg.Any<Specification<Product>>()).Returns(1);

        // Act
        var result = await _service.CreateAsync("Existing Product", 19.99m, "Description");

        // Assert
        Assert.False(result.IsSuccess);
        Assert.Contains("already exists", result.Errors.First());
        await _mockRepo.DidNotReceive().AddAsync(Arg.Any<Product>());
    }

    [Fact]
    public async Task UpdateAsync_UpdatesProduct_WhenExists()
    {
        // Arrange
        var product = new Product("Old Name", 10m);
        _mockRepo.GetByIdAsync(1).Returns(product);

        // Act
        var result = await _service.UpdateAsync(1, "New Name", 20m, "New Description");

        // Assert
        Assert.True(result.IsSuccess);
        await _mockRepo.Received(1).UpdateAsync(product);
    }

    [Fact]
    public async Task DeleteAsync_DeletesProduct_WhenExists()
    {
        // Arrange
        var product = new Product("Test", 10m);
        _mockRepo.GetByIdAsync(1).Returns(product);

        // Act
        var result = await _service.DeleteAsync(1);

        // Assert
        Assert.True(result.IsSuccess);
        await _mockRepo.Received(1).DeleteAsync(product);
    }
}
```

**Testing with Sequential Returns:**

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrderFromBasket_TransfersItemsCorrectly()
    {
        // Arrange
        var mockBasketRepo = Substitute.For<IRepository<Basket>>();
        var mockProductRepo = Substitute.For<IRepository<Product>>();
        var mockOrderRepo = Substitute.For<IRepository<Order>>();
        var mockMediator = Substitute.For<IMediator>();

        var basket = new Basket("user123");
        basket.AddItem(1, 10.99m, 2);
        basket.AddItem(2, 5.99m, 1);

        var products = new List<Product>
        {
            new Product("Product 1", 10.99m) { Id = 1 },
            new Product("Product 2", 5.99m) { Id = 2 }
        };

        mockBasketRepo.FirstOrDefaultAsync(Arg.Any<Specification<Basket>>())
            .Returns(basket);
        mockProductRepo.ListAsync(Arg.Any<Specification<Product>>())
            .Returns(products);

        var service = new OrderService(mockOrderRepo, mockProductRepo, mockBasketRepo, mockMediator);

        // Act
        var result = await service.CreateOrderFromBasketAsync(
            1,
            new Address("123 Main", "City", "State", "Country", "12345"));

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(2, result.Value.OrderItems.Count);
        await mockOrderRepo.Received(1).AddAsync(Arg.Any<Order>());
        await mockMediator.Received(1).Publish(Arg.Any<OrderCreatedEvent>(), Arg.Any<CancellationToken>());
    }
}
```

---

## Pattern 13: Test Data Builders

**Product Builder:**

```csharp
namespace UnitTests.Builders;

public class ProductBuilder
{
    private string _name = "Test Product";
    private decimal _price = 19.99m;
    private string _description = "Test Description";
    private int _categoryId = 1;

    public ProductBuilder WithName(string name)
    {
        _name = name;
        return this;
    }

    public ProductBuilder WithPrice(decimal price)
    {
        _price = price;
        return this;
    }

    public ProductBuilder WithDescription(string description)
    {
        _description = description;
        return this;
    }

    public ProductBuilder WithCategory(int categoryId)
    {
        _categoryId = categoryId;
        return this;
    }

    public Product Build()
    {
        var product = new Product(_name, _price, _description);
        product.AssignToCategory(_categoryId);
        return product;
    }
}
```

**Order Builder:**

```csharp
public class OrderBuilder
{
    private string _buyerId = "test-buyer@example.com";
    private Address _address;
    private List<OrderItem> _items = new();

    public OrderBuilder()
    {
        _address = new AddressBuilder().Build();
    }

    public OrderBuilder WithBuyerId(string buyerId)
    {
        _buyerId = buyerId;
        return this;
    }

    public OrderBuilder WithAddress(Address address)
    {
        _address = address;
        return this;
    }

    public OrderBuilder WithItem(int productId, string productName, decimal price, int quantity)
    {
        _items.Add(new OrderItem(productId, productName, price, quantity));
        return this;
    }

    public OrderBuilder WithDefaultItems()
    {
        _items.Add(new OrderItem(1, "Product 1", 10.99m, 2));
        _items.Add(new OrderItem(2, "Product 2", 5.99m, 1));
        return this;
    }

    public Order Build()
    {
        if (!_items.Any())
        {
            WithDefaultItems();
        }

        return new Order(_buyerId, _address, _items);
    }
}
```

**Address Builder:**

```csharp
public class AddressBuilder
{
    private string _street = "123 Main St";
    private string _city = "TestCity";
    private string _state = "TS";
    private string _country = "TestCountry";
    private string _zipCode = "12345";

    public AddressBuilder WithStreet(string street)
    {
        _street = street;
        return this;
    }

    public AddressBuilder WithCity(string city)
    {
        _city = city;
        return this;
    }

    public AddressBuilder WithState(string state)
    {
        _state = state;
        return this;
    }

    public AddressBuilder WithCountry(string country)
    {
        _country = country;
        return this;
    }

    public AddressBuilder WithZipCode(string zipCode)
    {
        _zipCode = zipCode;
        return this;
    }

    public Address Build()
    {
        return new Address(_street, _city, _state, _country, _zipCode);
    }
}
```

**Using Builders in Tests:**

```csharp
[Fact]
public void Order_CalculatesTotal_Correctly()
{
    // Arrange - Fluent builder API
    var order = new OrderBuilder()
        .WithBuyerId("test@example.com")
        .WithItem(1, "Product 1", 10.00m, 2)
        .WithItem(2, "Product 2", 5.00m, 3)
        .Build();

    // Act
    var total = order.Total();

    // Assert
    Assert.Equal(35.00m, total); // (10 * 2) + (5 * 3)
}

[Fact]
public void Product_CanBeCreated_WithBuilder()
{
    // Arrange & Act
    var product = new ProductBuilder()
        .WithName("Gaming Laptop")
        .WithPrice(1299.99m)
        .WithDescription("High-performance gaming laptop")
        .WithCategory(3)
        .Build();

    // Assert
    Assert.Equal("Gaming Laptop", product.Name);
    Assert.Equal(1299.99m, product.Price);
}
```

---

## Pattern 14: Integration Testing

**Base Integration Test Class:**

```csharp
using Microsoft.EntityFrameworkCore;
using Xunit;

namespace IntegrationTests;

public abstract class BaseIntegrationTest : IDisposable
{
    protected readonly AppDbContext Context;
    protected readonly EfRepository<Product> ProductRepository;
    protected readonly EfRepository<Order> OrderRepository;

    protected BaseIntegrationTest()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString()) // Unique per test
            .Options;

        Context = new AppDbContext(options);
        ProductRepository = new EfRepository<Product>(Context);
        OrderRepository = new EfRepository<Order>(Context);
    }

    public void Dispose()
    {
        Context.Database.EnsureDeleted();
        Context.Dispose();
    }
}
```

**Repository Integration Test:**

```csharp
namespace IntegrationTests.Repositories;

public class ProductRepositoryTests : BaseIntegrationTest
{
    [Fact]
    public async Task AddAsync_PersistsProduct()
    {
        // Arrange
        var product = new Product("Test Product", 19.99m, "Test Description");

        // Act
        await ProductRepository.AddAsync(product);
        await Context.SaveChangesAsync();

        // Assert
        var retrieved = await ProductRepository.GetByIdAsync(product.Id);
        Assert.NotNull(retrieved);
        Assert.Equal("Test Product", retrieved.Name);
        Assert.Equal(19.99m, retrieved.Price);
    }

    [Fact]
    public async Task ListAsync_ReturnsFilteredProducts()
    {
        // Arrange
        var product1 = new Product("Product A", 10m);
        var product2 = new Product("Product B", 20m);
        var product3 = new Product("Product C", 30m);

        await ProductRepository.AddAsync(product1);
        await ProductRepository.AddAsync(product2);
        await ProductRepository.AddAsync(product3);
        await Context.SaveChangesAsync();

        // Act
        var spec = new ProductListPaginatedSpec(skip: 0, take: 2);
        var result = await ProductRepository.ListAsync(spec);

        // Assert
        Assert.Equal(2, result.Count());
    }

    [Fact]
    public async Task UpdateAsync_ModifiesProduct()
    {
        // Arrange
        var product = new Product("Original Name", 10m);
        await ProductRepository.AddAsync(product);
        await Context.SaveChangesAsync();

        // Act
        product.UpdateDetails("New Name", "New Description");
        await ProductRepository.UpdateAsync(product);
        await Context.SaveChangesAsync();

        // Assert
        var updated = await ProductRepository.GetByIdAsync(product.Id);
        Assert.Equal("New Name", updated.Name);
    }

    [Fact]
    public async Task Specification_LoadsRelatedEntities()
    {
        // Arrange
        var category = new Category("Electronics");
        Context.Categories.Add(category);
        await Context.SaveChangesAsync();

        var product = new Product("Laptop", 999m);
        product.AssignToCategory(category.Id);
        await ProductRepository.AddAsync(product);
        await Context.SaveChangesAsync();

        // Act
        var spec = new ProductWithCategorySpec(product.Id);
        var result = await ProductRepository.FirstOrDefaultAsync(spec);

        // Assert
        Assert.NotNull(result);
        Assert.NotNull(result.Category); // Eager loaded
        Assert.Equal("Electronics", result.Category.Name);
    }
}
```

---

# Section D: Best Practices

## Entity Design Guidelines

### 1. Use Private Setters
```csharp
✅ GOOD:
public string Name { get; private set; }

❌ BAD:
public string Name { get; set; } // Allows external mutation
```

### 2. Guard Clauses in Constructors
```csharp
✅ GOOD:
public Product(string name, decimal price)
{
    Guard.Against.NullOrEmpty(name, nameof(name));
    Guard.Against.NegativeOrZero(price, nameof(price));
    Name = name;
    Price = price;
}

❌ BAD:
public Product(string name, decimal price)
{
    Name = name;  // No validation
    Price = price;
}
```

### 3. Encapsulate Collections
```csharp
✅ GOOD:
private readonly List<OrderItem> _items = new();
public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

public void AddItem(OrderItem item) { _items.Add(item); }

❌ BAD:
public List<OrderItem> Items { get; set; } // Allows external mutation
```

### 4. Business Logic in Entities
```csharp
✅ GOOD:
public class Order : BaseEntity
{
    public decimal Total() => OrderItems.Sum(i => i.UnitPrice * i.Quantity);

    public bool CanBeCancelled() => OrderDate > DateTimeOffset.Now.AddDays(-7);
}

❌ BAD:
// Logic in service instead of entity
public class OrderService
{
    public decimal CalculateTotal(Order order)
    {
        return order.OrderItems.Sum(i => i.UnitPrice * i.Quantity);
    }
}
```

## When to Create Aggregates vs Entities

### Aggregate Roots (Implement IAggregateRoot)
- **Order** (manages OrderItems)
- **Basket** (manages BasketItems)
- **Product** (standalone, doesn't manage other entities)

### Child Entities (No IAggregateRoot)
- **OrderItem** (only accessible through Order)
- **BasketItem** (only accessible through Basket)

**Rule:** If an entity is only meaningful within the context of another entity, it's a child entity.

## Specification Design

### Good Specifications
```csharp
✅ Single Responsibility:
public class ProductsInCategorySpec : Specification<Product>
{
    public ProductsInCategorySpec(int categoryId)
    {
        Query.Where(p => p.CategoryId == categoryId);
    }
}

✅ Composable:
public class ProductsWithPriceRangeSpec : Specification<Product>
{
    public ProductsWithPriceRangeSpec(decimal? min, decimal? max)
    {
        Query.Where(p =>
            (!min.HasValue || p.Price >= min) &&
            (!max.HasValue || p.Price <= max));
    }
}

// Can be combined in service
var spec = new ProductsInCategorySpec(categoryId)
    .And(new ProductsWithPriceRangeSpec(10m, 100m));
```

### Avoid
```csharp
❌ God Specification (too many responsibilities):
public class ProductSearchSpec : Specification<Product>
{
    public ProductSearchSpec(
        int? categoryId,
        decimal? minPrice,
        decimal? maxPrice,
        string searchTerm,
        int? brandId,
        bool? inStock,
        string sortBy,
        int page,
        int pageSize)
    {
        // Too complex - split into smaller specs
    }
}
```

## Service Layer Responsibilities

### Services Should:
- ✅ Orchestrate multiple repositories
- ✅ Apply business rules spanning multiple aggregates
- ✅ Publish domain events
- ✅ Use Result pattern for operation outcomes

### Services Should NOT:
- ❌ Contain business logic that belongs in entities
- ❌ Directly use DbContext (use IRepository)
- ❌ Have presentation concerns (formatting, HTTP codes)

## Testing Strategy

### What to Test at Each Layer

**Unit Tests (ApplicationCore):**
- Entity business logic
- Service orchestration
- Specification query logic
- Domain event handlers

**Integration Tests (Infrastructure):**
- Repository operations
- EF Core configurations
- Database queries with specifications

**Functional Tests (API/Web):**
- HTTP endpoints
- Authentication/Authorization
- Request/Response contracts

### Test Pyramid
```
        /\
       /  \  Functional (Few)
      /____\
     /      \
    / Integr \  Integration (Some)
   /  ation  \
  /__________\
 /            \
/  Unit Tests  \  Unit (Many)
/________________\
```

## Performance Considerations

### N+1 Query Prevention
```csharp
✅ GOOD - Use Include:
var spec = new OrderWithItemsSpec(orderId);
var order = await _repository.FirstOrDefaultAsync(spec);
// Single query with JOIN

❌ BAD - Lazy loading causes N+1:
var order = await _repository.GetByIdAsync(orderId);
foreach (var item in order.Items) // Separate query per item!
{
    Console.WriteLine(item.Product.Name); // Another query!
}
```

### Pagination
```csharp
✅ GOOD - Skip/Take at database level:
var spec = new ProductListPaginatedSpec(page * pageSize, pageSize);
var products = await _repository.ListAsync(spec);

❌ BAD - Load all, paginate in memory:
var allProducts = await _repository.ListAsync(new AllProductsSpec());
var page = allProducts.Skip(page * pageSize).Take(pageSize);
```

### Select Only What You Need
```csharp
✅ GOOD - Use projections for DTOs:
var spec = new ProductListProjectionSpec(); // Select only Name, Price, Id
var dtos = await _repository.ListAsync(spec);

❌ BAD - Load full entities for display:
var products = await _repository.ListAsync(spec);
// Loads all properties even if only showing Name and Price
```

## Security Patterns

### Authentication in FastEndpoints
```csharp
public override void Configure()
{
    Post("api/orders");
    AuthSchemes(JwtBearerDefaults.AuthenticationScheme);
}
```

### Authorization by Role
```csharp
public override void Configure()
{
    Delete("api/products/{id}");
    Roles("Admin", "ProductManager");
}
```

### User Context in Services
```csharp
public class OrderService
{
    private readonly ICurrentUserService _currentUser;

    public async Task<Result<Order>> CreateOrderAsync(Address address)
    {
        var userId = _currentUser.UserId;

        // Use authenticated user's ID
        var order = new Order(userId, address, items);
        await _repository.AddAsync(order);

        return Result<Order>.Success(order);
    }
}
```

---

# Section E: Tech Stack Variations

## PostgreSQL instead of SQL Server

**Connection String:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=myapp;Username=postgres;Password=password"
  }
}
```

**Package:**
```bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

**Registration:**
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
```

**Migrations:**
```bash
dotnet ef migrations add InitialCreate --context AppDbContext
dotnet ef database update
```

## React Frontend

For React SPAs consuming your API:

**CORS Configuration:**
```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("ReactApp", policy =>
    {
        policy.WithOrigins("http://localhost:3000")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

app.UseCors("ReactApp");
```

**API Client (TypeScript):**
```typescript
export interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
}

export class ProductService {
  private baseUrl = 'https://localhost:5001/api';

  async getAll(): Promise<Product[]> {
    const response = await fetch(`${this.baseUrl}/products`, {
      headers: {
        'Authorization': `Bearer ${getToken()}`
      }
    });
    return await response.json();
  }

  async getById(id: number): Promise<Product> {
    const response = await fetch(`${this.baseUrl}/products/${id}`);
    return await response.json();
  }

  async create(product: Omit<Product, 'id'>): Promise<Product> {
    const response = await fetch(`${this.baseUrl}/products`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getToken()}`
      },
      body: JSON.stringify(product)
    });
    return await response.json();
  }
}
```

## Dapper instead of EF Core

For users preferring Dapper:

**Repository Implementation:**
```csharp
using Dapper;
using System.Data;

public class DapperProductRepository : IProductRepository
{
    private readonly IDbConnection _connection;

    public DapperProductRepository(IDbConnection connection)
    {
        _connection = connection;
    }

    public async Task<Product?> GetByIdAsync(int id)
    {
        const string sql = "SELECT * FROM Products WHERE Id = @Id";
        return await _connection.QuerySingleOrDefaultAsync<Product>(sql, new { Id = id });
    }

    public async Task<IEnumerable<Product>> ListAsync(int page, int pageSize)
    {
        const string sql = @"
            SELECT * FROM Products
            ORDER BY Name
            OFFSET @Offset ROWS
            FETCH NEXT @PageSize ROWS ONLY";

        return await _connection.QueryAsync<Product>(sql, new
        {
            Offset = page * pageSize,
            PageSize = pageSize
        });
    }

    public async Task<int> AddAsync(Product product)
    {
        const string sql = @"
            INSERT INTO Products (Name, Price, Description, CategoryId)
            VALUES (@Name, @Price, @Description, @CategoryId);
            SELECT CAST(SCOPE_IDENTITY() as int)";

        return await _connection.ExecuteScalarAsync<int>(sql, product);
    }

    public async Task UpdateAsync(Product product)
    {
        const string sql = @"
            UPDATE Products
            SET Name = @Name, Price = @Price, Description = @Description
            WHERE Id = @Id";

        await _connection.ExecuteAsync(sql, product);
    }

    public async Task DeleteAsync(int id)
    {
        const string sql = "DELETE FROM Products WHERE Id = @Id";
        await _connection.ExecuteAsync(sql, new { Id = id });
    }
}
```

**DI Registration:**
```csharp
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<IProductRepository, DapperProductRepository>();
```

## GraphQL API

Using HotChocolate:

**Package:**
```bash
dotnet add package HotChocolate.AspNetCore
```

**Query Type:**
```csharp
public class Query
{
    public async Task<Product?> GetProduct(
        [Service] IProductService service,
        int id)
    {
        var result = await service.GetByIdAsync(id);
        return result.IsSuccess ? result.Value : null;
    }

    public async Task<IEnumerable<Product>> GetProducts(
        [Service] IProductService service,
        int page = 0,
        int pageSize = 10)
    {
        var result = await service.ListAsync(page, pageSize);
        return result.Value;
    }
}
```

**Mutation Type:**
```csharp
public class Mutation
{
    public async Task<Product?> CreateProduct(
        [Service] IProductService service,
        string name,
        decimal price,
        string description)
    {
        var result = await service.CreateAsync(name, price, description);
        return result.IsSuccess ? result.Value : null;
    }
}
```

**Registration:**
```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>();

app.MapGraphQL();
```

---

# Section F: Quick Reference

## Common Commands

```bash
# Create new project
/dotnet-clean-arch:new

# Add feature
/dotnet-clean-arch:add-feature Product

# Migrate existing project
/dotnet-clean-arch:migrate

# Audit architecture
/dotnet-clean-arch:audit

# Browse patterns
/dotnet-clean-arch:patterns

# Create migration
dotnet ef migrations add MigrationName --project src/Infrastructure --startup-project src/API

# Update database
dotnet ef database update --project src/Infrastructure --startup-project src/API

# Run tests
dotnet test

# Run application
dotnet run --project src/API
```

## Dependency Rules
```
ApplicationCore → (nothing)
Infrastructure → ApplicationCore
API/Web → ApplicationCore, Infrastructure
Tests → (any)
```

## Key NuGet Packages
```
ApplicationCore:
- Ardalis.GuardClauses
- Ardalis.Specification
- Ardalis.Result
- MediatR

Infrastructure:
- Microsoft.EntityFrameworkCore.SqlServer
- Ardalis.Specification.EntityFrameworkCore
- Microsoft.AspNetCore.Identity.EntityFrameworkCore

API:
- FastEndpoints (or built-in Minimal APIs)
- AutoMapper.Extensions.Microsoft.DependencyInjection
- Microsoft.AspNetCore.Authentication.JwtBearer

Tests:
- xunit
- NSubstitute
- Microsoft.EntityFrameworkCore.InMemory
```

## Pattern Quick Links

1. **Repository Pattern** → Section C, Pattern 2
2. **Specification Pattern** → Section C, Pattern 3
3. **Entity Design** → Section C, Pattern 4
4. **Value Objects** → Section C, Pattern 5
5. **EF Configuration** → Section C, Pattern 6
6. **Service Layer** → Section C, Pattern 7
7. **Domain Events** → Section C, Pattern 8
8. **FastEndpoints** → Section C, Pattern 9
9. **Minimal APIs** → Section C, Pattern 10
10. **DI Registration** → Section C, Pattern 11
11. **Unit Testing** → Section C, Pattern 12
12. **Test Builders** → Section C, Pattern 13
13. **Integration Testing** → Section C, Pattern 14

---

**End of Skill Documentation**

This skill is based on Microsoft's eShopOnWeb reference application and represents production-ready patterns for building Clean Architecture monolithic applications in .NET.

For questions or issues, consult the pattern library or run `/dotnet-clean-arch:patterns` for interactive examples.
