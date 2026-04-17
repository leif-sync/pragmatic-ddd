---
name: server-actions
description: "[Pragmatic DDD Architecture] Guide for creating Next.js Server Actions exactly tied to zod, neverthrow, and the domain architecture. Use when creating or editing any file in presentation/actions/. Covers \"use server\" placement, unknown parameters validated with `zod`, discriminated union response types, auth-first pattern, Value Object validation with TypeScript narrowing, use-case error mapping via `assertNever`, serviceContainer invocation, and revalidation strategy."
---

# Server Actions

## Naming Conventions

- Name: `<verb><Resource>Action` — e.g., `createShortUrlAction`, `deleteFolderAction`
- Always a **single export per file** (the action function itself).

---

## Anatomy of a Server Action

Every action MUST follow this strict structure with `unknown` inputs validated by Zod and domain validation via Value Objects:

```typescript
"use server"; // (1) Always first line

import { z } from "zod";
import { assertNever } from "@/shared/domain/utils/assertNever";
import { err, ok } from "neverthrow";
import { SessionError, type GetSession } from "@/auth/getSession";

export type MyActionErrorCode =
  | "NOT_AUTHENTICATED"
  | "INVALID_INPUT"
  | "INVALID_ID"
  | "NOT_FOUND"
  | "UNEXPECTED_ERROR";

// (2) Exported types. MUST return branded types, NEVER Value Objects or Entities explicitly.
// Some client components need validated data; since we use Result<T,E>, components cannot throw errors.
export type MyActionResponse =
  | { success: true; data: BrandedUUID }
  | { success: false; error: MyActionErrorCode };

// (3) Zod Schema for incoming unknown parameters
// Zod CAN validate native/primitive types (e.g., `z.string()`, `z.number()`, `z.boolean()`) and specific common formats (e.g., `z.uuid()`, `z.email()`, `z.url()`).
// You MUST NOT use Zod for specific business logic boundaries like checking if a number is greater than/less than a specific amount, or specific lengths. 
// Those specific business rules strictly belong in Value Objects.
// Zod validation returns a generic "INVALID_INPUT", while VO validation returns descriptive, domain-specific errors.
const formDataSchema = z.object({
  id: z.uuid(),
});

// (4) Map domain errors exhaustively to action error codes
function handleMyUseCaseErrors(error: MyUseCaseErrors): MyActionErrorCode {
  if (error instanceof NotFoundError) return "NOT_FOUND";
  if (error instanceof SessionError) return "NOT_AUTHENTICATED";
  if (error instanceof RepositoryError) return "UNEXPECTED_ERROR";
  
  return assertNever(error); // Prevents missing union members
}

// (5) The action function ALWAYS takes `unknown` to ensure type-safety at runtime
export async function myAction(rawData: unknown): Promise<MyActionResponse> {
  // zod parsing → vo validation → getSession construction → use case → return
}
```

---

## 1. Types

### Response Type

Always a **discriminated union** over `success`. The `error` field MUST always be typed using the exported error code union — never `string`. 
When returning data on `success: true`, you MUST return **branded primitive types** (like `BrandedUUID`), never rich class instances like Value Objects or Entities, as they cannot cross the Server ↔ Client boundary.

```typescript
// Simple: single error field returning a Branded Type
export type CreateFolderActionResult =
  | { success: true; folderId: BrandedUUID }
  | { success: false; error: CreateFolderActionErrorCode };

// Form actions: multiple error codes possible using an array
export type CreateShortUrlActionResponse =
  | { success: true }
  | { success: false; errors: CreateShortUrlActionErrorCodes[] };
```

### Error Code Type

Use a string literal union (not an enum). Export it. Always include `"INVALID_INPUT"` and `"UNEXPECTED_ERROR"`.

---

## 2. Auth and `unknown` Zod Parsing

Actions MUST NOT natively type the payload via TypeScript signatures because they are easily skipped at runtime. They MUST take `rawData: unknown` and use Zod.

Zod validations MUST remain primitive (e.g., `z.string()`). Detailed validation is deferred to Value Objects so specific errors can be returned for better client-side UX.

```typescript
export async function createThingAction(rawData: unknown): Promise<CreateThingResponse> {
  const paramsParse = formDataSchema.safeParse(rawData);
  if (!paramsParse.success) return { success: false, error: "INVALID_INPUT" };
  
  // Proceed to Value Object instantiation...
}
```

---

## 3. Session Construction & Inversion of Control

You MUST NOT check auth at the very top of the action. Instead, construct a `getSession: GetSession` function inside the action and pass it to the Use Case to be executed when the Use Case requires it.

```typescript
  const getSession: GetSession = async () => {
    const session = await auth.api.getSession({ headers: await headers() });

    if (!session) return err(new SessionError());

    const userIdResult = UUID.from(session.user.id);
    if (userIdResult.isErr()) return err(new SessionError());

    return ok({ userId: userIdResult.value });
  };
```

---

## 4. Value Object (VO) Validation

### Batch validation pattern (form actions with multiple fields)

Collect all domain errors explicitly so TS narrows all `neverthrow` Results. This is why Zod is only used for structural validation; Value Objects return more specific errors for better UX.

```typescript
const workspaceIdResult = UUID.from(formData.workspaceId);
const nameResult = ShortUrlName.from(formData.name);

const errors: CreateShortUrlActionErrorCodes[] = [];

if (workspaceIdResult.isErr()) errors.push("INVALID_WORKSPACE_ID");
if (nameResult.isErr()) errors.push("INVALID_NAME");

// Guard ensures narrow types
if (
  errors.length > 0 || 
  workspaceIdResult.isErr() || 
  nameResult.isErr()
) {
  return { success: false, errors };
}

// All are now Ok -> `.value` avoids _unsafeUnwrap!
const result = await serviceContainer.foo.create.execute({
  workspaceId: workspaceIdResult.value,
  name: nameResult.value,
  getSession
});
```

---

## 5. Use-Case Error Mapping with `assertNever`

When a use case fails, strictly map its specific errors through a dedicated function relying on `assertNever`. 
`assertNever` will throw a TypeScript error if you update the Use Case to emit a new Error class but forget to update the Action.

```typescript
function handleCreateShortUrlErrors(error: CreateShortUrlErrors): CreateShortUrlActionErrorCodes {
  if (error instanceof UserNotFoundError) return "USER_NOT_FOUND";
  if (error instanceof WorkspaceNotFoundError) return "WORKSPACE_NOT_FOUND";
  if (error instanceof PathAlreadyInUseError) return "PATH_ALREADY_IN_USE";
  if (error instanceof SessionError) return "NOT_AUTHENTICATED";
  if (error instanceof RepositoryError) return "UNEXPECTED_ERROR"; // Must explicitly map!
  
  // Enforces TS compile-time error if new use-case errors are unhandled
  return assertNever(error);
}
```

### Infrastructure errors → always UNEXPECTED_ERROR

Errors from the infrastructure layer (database failures, `RepositoryError`, network issues) MUST always be collapsed to `"UNEXPECTED_ERROR"`. NEVER expose internals.

---

## 6. Use Case Execution

Always use `serviceContainer` — never instantiate use cases manually:

```typescript
import { serviceContainer } from "@/shared/infrastructure/bootstrap";

const result = await serviceContainer.folders.createFolder.execute({ ... });
```

---

## Complete Example

```typescript
"use server";

import { z } from "zod";
import { auth } from "@/auth/auth";
import { headers } from "next/headers";
import { err, ok } from "neverthrow";
import { serviceContainer } from "@/shared/infrastructure/bootstrap";
import { BrandedUUID, UUID } from "@/shared/domain/value-objects/uuid";
import { ShortUrlName } from "@/short-urls/domain/value-objects/shortUrlName";
import type { CreateShortUrlErrors } from "@/short-urls/use-cases/createShortUrl";
import { PathAlreadyInUseError } from "@/short-urls/domain/errors/pathAlreadyInUseError";
import { UserNotFoundError } from "@/users/domain/errors/userNotFoundError";
import { RepositoryError } from "@/shared/domain/errors/repositoryError";
import { assertNever } from "@/shared/domain/utils/assertNever";
import { GetSession, SessionError } from "@/auth/getSession";

export type CreateShortUrlActionErrorCode =
  | "NOT_AUTHENTICATED"
  | "INVALID_INPUT"
  | "INVALID_NAME"
  | "USER_NOT_FOUND"
  | "PATH_ALREADY_IN_USE"
  | "UNEXPECTED_ERROR";

export type CreateShortUrlActionResponse =
  | { success: true; id: BrandedUUID } // MUST output primitives or branded primitive types. 
  | { success: false; error: CreateShortUrlActionErrorCode };

function handleCreateShortUrlActionErrors(error: CreateShortUrlErrors): CreateShortUrlActionErrorCode {
  if (error instanceof UserNotFoundError) return "USER_NOT_FOUND";
  if (error instanceof PathAlreadyInUseError) return "PATH_ALREADY_IN_USE";
  if (error instanceof SessionError) return "NOT_AUTHENTICATED";
  if (error instanceof RepositoryError) return "UNEXPECTED_ERROR";
  return assertNever(error);
}

const formDataSchema = z.object({
  name: z.string(),
});

export async function createShortUrlAction(rawData: unknown): Promise<CreateShortUrlActionResponse> {
  const parseResult = formDataSchema.safeParse(rawData);
  if (!parseResult.success) {
    return { success: false, error: "INVALID_INPUT" };
  }

  const nameResult = ShortUrlName.from(parseResult.data.name);
  if (nameResult.isErr()) {
    return { success: false, error: "INVALID_NAME" };
  }

  const getSession: GetSession = async () => {
    const session = await auth.api.getSession({ headers: await headers() });
    
    if (!session) return err(new SessionError());

    const userIdResult = UUID.from(session.user.id);
    
    if (userIdResult.isErr()) return err(new SessionError());
    
    return ok({ userId: userIdResult.value });
  }

  const result = await serviceContainer.shortUrls.create.execute({
    name: nameResult.value,
    getSession
  });

  if (result.isErr()) {
    return { success: false, error: handleCreateShortUrlActionErrors(result.error) };
  }

  // Value Objects MUST cross to the client using `.toBranded()`
  return { success: true, id: result.value.getId().toBranded() };
}
```

## Advanced Utility (New Projects): `todo.ts`
If you are rapidly building out the architecture and have Server Actions or Use Cases mocked up that are not fully built, import the `todo.ts` utility (added in `core-utilities.md`) directly into your server action to safely halt compilation with `todo.panic("missing mapping")` or `todo.unimplemented("WIP")` instead of placing random throws. All throws should eventually be cleared out into explicit `neverthrow` results.