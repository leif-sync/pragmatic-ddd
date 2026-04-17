---
name: architecture
description: "[Pragmatic DDD Architecture] Guide for the overall project architecture, Domain-Driven Design (DDD) principles, Modular Monolith (Bounded Contexts) organization, and file structure. Use when deciding where to place a new file, creating a new module, or understanding the application's layers and dependency flow."
---

# Project Architecture and File Organization

This file is the **SINGLE SOURCE OF TRUTH** for folder structures and file naming conventions.

## 1. High-Level Architecture

This project MUST be structured as a **Modular Monolith** built with Next.js 16+ (App Router). It strictly adheres to **Domain-Driven Design (DDD)** and **Clean Architecture** principles. 

Instead of organizing files exclusively by technical responsibilities (e.g., all controllers together, all models together), you MUST divide the root of `src/` into **Bounded Contexts** (feature modules) such as `auth`, `folders`, `short-urls`, `users`, and `workspaces`.

## 2. Bounded Context Structure (The 4 Layers)

Every module inside `src/` (except `app/` and `shared/`) MUST follow a strict 4-layer architecture. Dependencies MUST **always point inwards** toward the Domain.

```text
src/<bounded-context>/
├── domain/            # 1. CORE: Pure typescript. No frameworks. No DB imports.
│   ├── entities/      # Rich domain models with getters and .toSnapshot() rules.
│   ├── value-objects/ # Immutable primitives (e.g., folderName.ts) with validation.
│   ├── errors/        # Domain errors (e.g., folderAlreadyExistsError.ts).
│   └── interfaces/    # Contracts (Abstract Classes) for Repositories.
│
├── use-cases/         # 2. APPLICATION: Orchestration layer.
│   ├── createFolder.ts      # Specific business flows. Uses repositories and domain models.
│   └── createFolder.test.ts # Co-located unit tests.
│
├── infrastructure/    # 3. INFRASTRUCTURE: External concerns. Database, Email, etc.
│   ├── postgresFolderRepository.ts                  # Implementation of domain interfaces.
│   └── postgresFolderRepository.integration.test.ts # Co-located integration tests.
│
└── presentation/      # 4. PRESENTATION: Next.js and UI concerns.
    ├── actions/       # Server Actions (e.g., createFolderAction.ts).
    ├── components/    # React Components (e.g., FolderList.tsx).
    └── hooks/         # React Hooks.
```

## 3. Naming Conventions and Locations

You MUST adhere to the following file naming and location conventions across all Bounded Contexts:

| Concept | Location | Naming Convention | Example |
|---|---|---|---|
| **Bounded Context** | `src/<context>/` | `kebab-case` | `src/short-urls/` |
| **[Value Objects](../value-objects/SKILL.md)** | `domain/value-objects/` | `camelCase.ts` | `folderName.ts` |
| **[Entities](../entities/SKILL.md)** | `domain/entities/` | `camelCase.ts` | `folder.ts` |
| **[Domain Errors](../errors/SKILL.md)** | `domain/errors/` | `camelCase.ts` (suffix `Error`) | `folderNotFoundError.ts` |
| **[Interfaces](../repositories/SKILL.md)** | `domain/interfaces/` | `camelCase.ts` (suffix `Repository`) | `folderRepository.ts` |
| **[Use Cases](../use-cases/SKILL.md)** | `use-cases/` | `camelCase.ts` (verb first) | `createFolder.ts` |
| **[Infrastructure](../repositories/SKILL.md)** | `infrastructure/` | `camelCase.ts` (tech prefix) | `postgresFolderRepository.ts` |
| **[Server Actions](../server-actions/SKILL.md)** | `presentation/actions/` | `camelCase.ts` (suffix `Action`) | `createFolderAction.ts` |
| **[Components](../components/SKILL.md)** | `presentation/components/` | `PascalCase.tsx` | `FolderList.tsx` |

## 4. The Dependency Rule & Inversion of Control

- **Domain is isolated**: The `domain/` folder MUST NOT import from `use-cases/`, `infrastructure/`, or `presentation/`.
  - *Pragmatic Exception*: You MAY import focused libraries like `zod` (for complex validations like emails/URLs), `uuid` (for UUIDv7 generation), and `neverthrow` (for `Result` types) inside the Domain. We delegate these tasks because manually parsing emails or generating UUIDv7s is too complex, error-prone, and provides no business value to reinvent.
- **Dependency Inversion**: Use Cases MUST depend on Repository **Interfaces** (from `domain/interfaces/`), not on concrete implementations (like Postgres).
- **No Use Case Chaining**: A Use Case MUST NOT instantiate, inject, or call another Use Case.
- **Service Container**: Instantiation is centralized in `src/shared/infrastructure/bootstrap.ts` (and exported as `serviceContainer`). 
  - UI [Components](../components/SKILL.md) and [Server Actions](../server-actions/SKILL.md) MUST NOT instantiate Repositories or Use Cases directly.
  - They MUST import the pre-configured use case instance from `serviceContainer`.
  - Example: `serviceContainer.folders.createFolder.execute(params)`

## 5. The `src/app/` Layer ([Routing](../routing/SKILL.md))

The `src/app/` directory MUST be treated strictly as a **Routing and Entry-Point Layer**. It MUST contain minimal logic. 
- It MUST wire HTTP requests to your `presentation/components/`.
- **`app/[locale]/`**: Public-facing pages supporting the custom [i18n](../i18n/SKILL.md) implementation (e.g., Landing, Login).
- **`app/dashboard/`**: Authenticated application views. MUST NOT use the `[locale]` segment natively in the URL but still supports i18n through `LocaleProvider`.
- You MUST NOT place domain logic or direct database queries inside `page.tsx` or `layout.tsx`. You MUST call `serviceContainer` use cases or [Server Actions](../server-actions/SKILL.md) instead.

## 6. Shared and Specialized Modules

### The `src/shared/` Module
The `shared` context MUST contain cross-cutting concerns that apply to the entire application:
- `shared/domain/`: Base classes, generic utilities (`Result` wrappers), and shared branded errors (like `RepositoryError`).
- `shared/infrastructure/`: 
  - `bootstrap.ts` and `serviceContainer.ts` (Dependency Injection).
  - `load-env.ts` (Zod environment validation).
  - `drizzle-postgres/` (Database [schema](../drizzle-schema/SKILL.md) and setup).
  - `i18n/` (Custom [translation](../i18n/SKILL.md) logic).
- `shared/presentation/`: Reusable [Components](../components/SKILL.md) (e.g., UI library `shadcn`, `LocaleProvider`).
- `shared/test/`: Reusable [testing](../testing/SKILL.md) utilities like `txTest` and database seeder helpers.

### The `src/auth/` Module ([Auth](../auth/SKILL.md))
The `auth` context MUST act as a specialized Bounded Context that integrates the **Better Auth** library. Unlike standard 4-layer modules, it MUST act as an infrastructural bridge:
- MUST contain `auth.ts` (Server Config) and `auth-client.ts` (Client Config).
- MUST provide `getSession.ts` for injecting session state into Server Actions and Use Cases.
- MUST house specific auth UI in `presentation/components/` and React Email templates in `templates/`.

## 7. Environment Variables (Zod Validation)

You MUST NOT access `process.env` directly in application code. All environment variables MUST be defined, validated, and exported from a single file: `src/shared/infrastructure/load-env.ts`.

1. **Zod Validation**: We MUST use a `retrieveEnvVar` helper that takes the environment variable name and a `z.ZodType` schema. This MUST throw an immediate, descriptive error on startup if an environment variable is missing or malformed.
2. **Type Coercion**: For numbers or booleans, you MUST use Zod's coercion (e.g., `z.coerce.number().int().positive()`).
3. **Value Object Integration**: Variables MAY be mapped directly into a [Value Object](../value-objects/SKILL.md) on load (e.g., `AppUrl.from(value)._unsafeUnwrap()`) so the rest of the application gets a valid, typed primitive.
4. **Public Variables Prefix**: Any exported constant in `load-env.ts` that will be exposed to the client/browser MUST have a `PUBLIC_` prefix in its TypeScript variable name (e.g., `export const PUBLIC_APP_NAME`). This visually distinguishes client-safe variables from server-only secrets within the codebase. (The actual environment variable name must still follow Next.js conventions, e.g., `NEXT_PUBLIC_APP_NAME`).
5. **App Usage**: Anywhere in the codebase that needs an environment variable, you MUST import the exported constant directly from `load-env.ts`.

### Example from `load-env.ts`:
```typescript
import { z } from "zod";
import { AppUrl } from "@/shared/domain/value-objects/appUrl";

// Base helper
function retrieveEnvVar<T>(params: { name: string; schema: z.ZodType<T> }): T {
  const value = process.env[params.name];
  const parsed = params.schema.safeParse(value);
  if (parsed.success) return parsed.data;
  throw new TypeError(`Environment variable ${params.name} is invalid...`);
}

// 1. Standard string validation
export const PUBLIC_APP_NAME = retrieveEnvVar({
  name: "NEXT_PUBLIC_APP_NAME",
  schema: z.string().min(3),
});

// 2. Coercion (e.g., port to number)
export const POSTGRES_PORT = retrieveEnvVar({
  name: "POSTGRES_PORT",
  schema: z.coerce.number().int().positive(),
});

// 3. Wrapping in a Value Object
export const PUBLIC_APP_BASE_URL = AppUrl.from(
  retrieveEnvVar({
    name: "NEXT_PUBLIC_APP_BASE_URL",
    schema: z.url(),
  }),
)._unsafeUnwrap({ withStackTrace: true });
```

## 8. [Testing Strategy](../testing/SKILL.md) & Co-location

Tests MUST live right next to the code they verify. 
- `*.test.ts`: Fast, isolated Unit Tests logic (Domain Models, Use Cases). MUST be handled by Vitest's unit worker.
- `*.integration.test.ts`: Slower tests requiring infrastructure (Postgres/Testcontainers). MUST be placed inside `infrastructure/` directories and run via the `txTest` helper to rollback changes. 
- `*.ui.test.ts`: For testing components in `presentation/`.

## 9. Control Flow Summary (A Complete Request)

The control flow MUST be as follows:
1. **User interacts** with a Client [Component](../components/SKILL.md) (`presentation/components/PascalCase.tsx`).
2. Component MUST call a **[Server Action](../server-actions/SKILL.md)** (`presentation/actions/camelCaseAction.ts`).
3. Server Action MUST validate raw inputs (via `Zod`), authenticate the user, and call a **[Use Case](../use-cases/SKILL.md)** via `serviceContainer`.
4. The **[Use Case](../use-cases/SKILL.md)** (`use-cases/camelCase.ts`) MUST orchestrate logic: 
   - MUST use `static from()` on **[Value Objects](../value-objects/SKILL.md)** to create valid domain primitives.
   - MUST call **[Repository Interfaces](../repositories/SKILL.md)** (`domain/interfaces/`) to fetch/save data.
   - MUST return a `Result<Data, DomainError>` (`neverthrow`).
5. Server Action MUST receive the `Result`. If error, it MUST use `assertNever()` to exhaustively map it to a UI-friendly error response string. If success, it MUST return data mapped via `.toBranded()`. Layer boundaries MUST be maintained.
