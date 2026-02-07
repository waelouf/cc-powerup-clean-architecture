---
description: Scan project for Clean Architecture violations and anti-patterns with actionable fixes
---

You are an expert .NET architect specializing in Clean Architecture. Use the `clean-architecture` skill to audit the current project for architecture violations.

## Your Task

Follow the workflow defined in the `clean-architecture` skill, Section B, Command `/clean-arch:audit`.

## Steps

1. **Scan Project Structure** using Glob:
   - `Glob: src/ApplicationCore/**/*.cs`
   - `Glob: src/Infrastructure/**/*.cs`
   - `Glob: src/API/**/*.cs` (or src/Web/**/*.cs)

2. **Check Dependency Violations** using Grep:
   - ApplicationCore should NOT reference Infrastructure or API:
     - `Grep in src/ApplicationCore/: "using.*Infrastructure"`
     - `Grep in src/ApplicationCore/: "using.*API"`
     - ⚠️ **VIOLATION** if found

   - ApplicationCore should NOT reference external packages (except allowed):
     - Allowed: Ardalis.*, MediatR, System.*
     - `Grep in src/ApplicationCore/*.csproj: "PackageReference"`

3. **Check Repository Pattern Usage**:
   - Controllers/Endpoints should NOT use DbContext directly:
     - `Grep in src/API/: "_context\.|DbContext"`
     - Should use IRepository instead

   - Check proper repository usage:
     - `Grep in src/API/: "IRepository<"`

4. **Check for Business Logic in Controllers**:
   - Look for business logic patterns in endpoints/controllers:
     - `Grep in src/API/: "if.*\.Price|for.*Items|while"`
     - `Grep in src/API/: "Calculate|Validate|Process"`
   - These should be in services, not controllers

5. **Check Aggregate Root Design**:
   - Find entities without IAggregateRoot used in repositories:
     - Extract types from `IRepository<T>` usage
     - Verify each type implements IAggregateRoot

   - Check for public setters (should be private):
     - `Grep in src/ApplicationCore/Entities/: "{ get; set; }"`
     - ⚠️ **VIOLATION** - should be `{ get; private set; }`

6. **Check Specification Usage**:
   - Direct LINQ in services instead of specifications:
     - `Grep in src/ApplicationCore/Services/: "\.Where\(|\.FirstOrDefault\("`
     - Should use specifications instead

7. **Check for Missing Tests**:
   - Services without tests:
     - Find all service files
     - Check if corresponding test file exists
   - Repositories without integration tests

8. **Generate Audit Report** (markdown format):
   ```markdown
   # Architecture Audit Report

   ## ✅ Passing Checks (X)
   - List passing checks

   ## ⚠️ Violations Found (Y)

   ### 1. [HIGH] Violation Title
   **Severity:** HIGH/MEDIUM/LOW
   **Location:** file.cs:line

   **Issue:**
   ```csharp
   // ❌ Bad code
   ```

   **Fix:**
   ```csharp
   // ✅ Good code
   ```

   ## 📊 Summary
   - Total files scanned: Z
   - Violations found: Y
   - Recommended priority: Fix HIGH severity first

   ## 🔧 Quick Fix Commands
   1. Command to fix issue 1
   2. Command to fix issue 2
   ```

9. **Provide Actionable Fixes**:
   - For each violation, show exact code changes needed
   - Prioritize by severity (HIGH → MEDIUM → LOW)
   - Offer to apply fixes automatically (with confirmation)

## Severity Levels

- **HIGH**: Breaks Clean Architecture principles
  - ApplicationCore depends on external layers
  - Direct DbContext usage in controllers
  - Business logic in presentation layer

- **MEDIUM**: Violates best practices
  - Public setters on entities
  - Missing specifications
  - No tests for services

- **LOW**: Code quality issues
  - Missing XML comments
  - Inconsistent naming
  - Missing validation

## Important

- Scan ALL files in the project
- Report specific file paths and line numbers
- Provide code examples for each violation
- Offer to fix violations interactively
- Re-run audit after fixes to verify

Execute this workflow, scanning the project and generating a comprehensive audit report with actionable fixes.
