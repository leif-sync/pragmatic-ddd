---
name: errors
description: "[Pragmatic DDD Architecture] Guide for creating strictly typed Domain validation and infrastructure errors. Use when handling or creating new errors to ensure they conform to the Railway-oriented programming model (neverthrow Result), TypeScript error branding, and constructor parameter constraints. Covers co-location rules vs. shared domain error usage."
---

# DDD Error Management

## 1. Core Principles (RFC 2119)

Errors in this architecture are never "thrown" as standard exceptions during domain validation. They are instead strictly typed and returned as failures inside `neverthrow`'s `Result<T, E>`.
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

- **No Enums**: You MUST NOT use `enum` or string union types to represent different error cases or mappings natively within the Use Cases/Domain (Actions can, but Domain MUST NOT). Each domain error MUST be a distinct class extending `Error`.
- **TypeScript Error Branding**: Every custom error class MUST include `declare private readonly [unique]: never` using a `unique symbol`. This strictly prevents duck-typing issues and allows 100% safe `instanceof` checks when exhaustively resolving the error via `assertNever` in the Server Action layer.
- **Payload Objects**: Except for wrapping native errors (e.g., Node.js or Database `Error` instances), constructors MUST ALWAYS accept a single `params: { ... }` object argument for dependencies/properties, even if it only has one property.
- **Strict Encapsulation**: Internal properties capturing the invalid value or context MUST be typed as Value Objects (or Branded primitives). They MUST be `private readonly` with explicit `.get...()` getters. You MUST NOT expose properties directly via `public` access modifiers. 
- **Message Formatting**: When building the `super(...)` message string inside the constructor, you MUST extract the primitive from the Value Object using its `.toBranded()` method (or its semantic equivalent like `.getEncoded()`).

## 2. Location & Granularity 

- **Local Errors (Co-location)**: If an error is strictly limited to validating a single Value Object (e.g., `InvalidEmailError`), define the error in the **exact same file** as the Value Object. If an error is only emitted by a specific Use Case parsing logic, define it with the Use Case.
- **Shared Domain Errors**: If an error spans multiple entities, use cases, or represents a core domain rule (e.g., `FolderAlreadyExistsError`, `UserNotFoundError`), define it in the shared domain errors location as dictated by the architecture skill.

## 3. Creating Domain Errors (Business Rules)

This format strictly applies for any rules related to validation, business constraints, or entity invariants. Notice the object parameter payload `params: { ... }`.

```typescript
import { FolderName } from "../value-objects/folderName";

declare const unique: unique symbol;

export class FolderAlreadyExistsError extends Error {
  // BRANDING: This ensures TypeScript treats this type strictly.
  declare private readonly [unique]: never;
  
  // MUST store the actual Value Object privately
  private readonly folderName: FolderName;

  constructor(params: { folderName: FolderName }) {
    // MUST extract the primitive value via `.toBranded()` for message formatting
    super(`Folder with name "${params.folderName.toBranded()}" already exists.`);

    this.folderName = params.folderName;
  }

  // MUST return the Value Object
  getFolderName(): FolderName {
    return this.folderName;
  }
}
```

## 4. Creating Infrastructure Errors (Database/External)

When catching external exceptions (e.g., Drizzle/Postgres errors), the objective is to wrap the native `Error` inside our branded Domain Architecture. This is the only time the constructor argument should be an `Error` instance.

```typescript
declare const unique: unique symbol;

export class RepositoryError extends Error {
  declare private readonly [unique]: never;

  constructor(error: Error) {
    const { message, ...rest } = error;
    super(message, rest);
  }
}
```

## 5. Usage in the Application Flow

In your business logic—like Repositories or Use Cases—these classes are always used inside pure functions returning `Result.err()` instead of `throw`.

```typescript
// ✅ RECOMMENDED - Railway flow
if (folders.length > 0) {
  return err(new FolderAlreadyExistsError({ folderName }));
}

// ❌ MUST NOT - Never throw domain validation exceptions directly
if (folders.length > 0) {
  throw new FolderAlreadyExists({ folderName });
}
```

## 6. References

For examples based on existing models, review real project implementations stored in the `.agents/skills/errors/references/` folder:
- [workspaceNotFoundError.md](references/workspaceNotFoundError.md) - Standard domain error taking a single Value Object parameter.
- [pathAlreadyInUseError.md](references/pathAlreadyInUseError.md) - Domain error showcasing optional parameters and conditional messages.