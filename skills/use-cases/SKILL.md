---
name: use-cases
description: "[Pragmatic DDD Architecture] How to structure **Use Cases** using DDD and Railway-Oriented Programming (neverthrow Result types). Tailored for TypeScript + drizzle-orm + node-postgres stack. **Use whenever creating or modifying any Use Case class — even simple ones like \"Exists\" or \"List\" operations — to ensure type-safe error unions, proper transactional boundaries, Value Object-only contracts, auth-first patterns, and Result-based error handling.** Includes references to working examples (Create, List, Exists patterns). Depends on 'repositories' skill."
---

# Use Cases

This skill documents how to structure **Use Cases** using **Domain-Driven Design (DDD)** and **Railway-Oriented Programming** via `neverthrow`. Use Cases orchestrate business logic against repositories while maintaining strict domain contracts: they work exclusively with **Value Objects**, never primitives.

**DEPENDENCY NOTICE**: This skill works in tandem with the `repositories` skill. For repository interface/implementation details, read the `repositories` skill.

## 1. Naming Convention

Use Case class names MUST follow the pattern: **`<Action><Entity>[For<Subject>]`**

- **Simple case**: `ListWorkspaces`, `CreateUser`, `ExistWorkspace`
- **Multi-actor case**: If different security/authorization requirements exist for the same action and entity, create separate Use Cases:
  - `CreateWorkspaceForUser` — user creates personal workspace
  - `CreateWorkspaceForAdmin` — admin creates workspace for tenant
- **Rationale**: Each Use Case has distinct authorization, validation, and error handling requirements. Separate classes ensure no accidental coupling of disparate business logic.

## 2. Core Principles

### Value Objects Only — Never Primitives

- **Params type**: Use Case params MUST accept only **Value Objects (VOs)** and domain types. Primitive types (`string`, `number`, `boolean`) MUST NOT be used. 
  - ✅ Example: `workspaceName: WorkspaceName` 
  - ❌ Wrong: `name: string`

- **Return type & Entities**: `execute()` MUST return domain Entities or arrays of Entities wrapped in `Result<T, ErrorUnion>`. Primitive types MUST NOT be returned.
  - ✅ Example: `Promise<Result<Workspace, CreateWorkspaceErrors>>`
  - ❌ Wrong: `Promise<Result<string, Error>>`

- **Snapshots for private data exclusion**: If an Entity contains truly private/internal data that MUST NOT be exposed (e.g., password hash, internal security state), a **Snapshot** type MAY be defined and returned instead. The Snapshot is a flat object containing exclusively Value Objects (extracted from the Entity) without the private fields.
  - **Snapshot definition**: MUST be defined in the same file as the Use Case.
  - **Use Case responsibility**: Exclude truly private data (cryptographic hashes, internal state). The presentation/infrastructure layer CAN serialize, filter, and transform fields as needed.
  - ✅ Example: `Promise<Result<UserSnapshot, CreateUserErrors>>` where `UserSnapshot` has `userId: UUID`, `email: Email` but NOT the password hash.
  - ❌ Wrong: Returning the full `User` Entity with password hash exposed.

```typescript
// Define snapshot in the same Use Case file
export type UserSnapshot = {
  userId: UUID;
  email: Email;
  name: UserName;
  // ❌ NOT included: passwordHash (private data)
};

export class CreateUser {
  // ... execute() returns Promise<Result<UserSnapshot, CreateUserErrors>>
}
```

### Session Management via Parameters

- `getSession` MUST be received as a parameter in the `execute()` method signature IF authentication is required for this Use Case operation. `getSession` MUST NOT be injected into the constructor.
  - **When optional**: Public Use Cases (e.g., `LoginUser`, `RegisterUser`) that allow unauthenticated access MUST NOT include `getSession` in params.
  - **When required**: Protected Use Cases (e.g., `CreateWorkspace`, `ListWorkspaces`) that require user context MUST include `getSession` in params.
  - **Rationale**: Passing `getSession` at invocation time ensures each call evaluates auth dynamically, preventing stale session state.
  - **Pattern**: `async execute({ ...params, getSession }: Params): Promise<Result<T, Errors>>`

- When `getSession` is included, session verification MUST be the first operation in `execute()`. All subsequent logic depends on session validity.

### Railway-Oriented Error Handling

- `execute()` MUST return `Promise<Result<T, ErrorUnion>>` via `neverthrow`.

- Exceptions MUST NOT be thrown manually for domain or business logic errors. All failures MUST be wrapped in `err(error)`.

- Result values MUST be destructured using `.isErr()` before proceeding. Every async operation returning `Result` MUST be checked for errors.

### Dependency Injection via Constructor

- Repositories and `WithTransaction` MUST be injected in the constructor, not instantiated within `execute()`.

- Dependencies MUST be stored as `private readonly` fields and remain immutable.

### No Dependency Between Use Cases

- Use Cases MUST NEVER depend on other Use Cases. They orchestrate repositories, not each other.
  - ✅ Example: `execute()` calls `this.userRepo.create()` and `this.workspaceRepo.create()`.
  - ❌ Wrong: `CreateWorkspace` injecting and calling `CreateFolder.execute()`. 
- **Rationale**: Chaining use cases muddles transactional boundaries, creates circular dependencies, and couples error handling contexts together unnecessarily. 

### Single Public Method

- Each Use Case class MUST expose exactly one public method: `async execute(Params): Promise<Result<T, Errors>>`.

- Complex logic MAY be split into `private` methods within the same class for readability and maintainability.

## 3. Common Patterns

Most Use Cases follow one of four patterns. **Working examples** are in `references/examples/`:

### Pattern 1: CREATE — Multi-step Orchestration
**File**: `references/examples/create-example.md`

**When to use:** Creating entities that require coordination across multiple repositories or validation tables.

**Characteristics**:
- Receives multiple Value Objects as params.
- Orchestrates multiple repository operations **within a transaction boundary** when REQUIRED for data consistency.
- Returns a domain **Entity**.
- **Important Note on Errors:** Repository methods that ADD new information to the database must handle structural constraints and return domain-specific database errors alongside the standard `RepositoryError`:
  - **Unique Constraints:** e.g., creating a URL with a path that must be unique maps to `PathAlreadyInUseError`.
  - **Foreign Key Constraints:** e.g., attaching to missing parents maps to `<Dependency>NotFoundError`.
  Standard creations without these constraints typically only return `RepositoryError`.
- **Transactions**: Use `withTransaction(async (tx) => { ... })` ONLY when:
  - Multiple repositories are modified in the same operation (e.g., create workspace + initial folder)
  - Partial failures would leave data in an inconsistent state
  - Atomicity of multiple operations is a business requirement
  - **NOT recommended** for single-table creations (see: unnecessary overhead)

**Key steps**:
1. Session verification (if `getSession` param included) — MUST be first.
2. Create domain entities using constructors.
3. If multi-step: wrap in transaction, check `.isErr()` on each result.
4. Return created entity or error.

### Pattern 2: LIST — Query with Pagination & Validation
**File**: `references/examples/list-example.md`

**When to use:** Retrieving paginated collections with business rule validation.

**Characteristics**:
- List operations MUST include pagination to prevent unbounded query results.
- Pagination MAY be offset-based (`page`, `itemsPerPage`) or cursor-based (`cursor`, `limit`).
- Receives pagination params as **Value Objects** (`PositiveInteger`, `Cursor`, etc.).
- Validates business constraints (pagination limits) BEFORE repository call.
- When listing within a hierarchical scope, MUST verify parent scope access first (e.g., verify user owns workspace before listing folders).
- Returns array of Entities: `Entity[]`.
- **Transactions**: MUST NOT be used. Reads are inherently consistent.

**Key steps**:
1. Validate pagination params (e.g., `itemsPerPage <= maxItemsPerPage` or cursor validity).
2. Session verification (if `getSession` param included).
3. **Optional**: Verify scope hierarchy via `ensureIsDataConsistent()` (if resource is in a hierarchy).
4. Delegate to repository's `list()` method.

### Pattern 3: EXISTS — Authorization Check
**File**: `references/examples/exists-example.md`

**When to use:** Checking if a resource exists and belongs to the current user.

**Characteristics**:
- Minimal logic: auth + repository query.
- Returns `boolean` wrapped in `Result`.
- **Transactions**: MUST NOT be used. Single read operation.

**Key steps**:
1. Session verification (if `getSession` param included).
2. Delegate to repository's `exists()` method.

### Pattern 4: COUNT — Aggregation with Scope Verification
**File**: `references/examples/count-example.md`

**When to use:** Retrieving aggregate counts for resources within a hierarchical scope.

**Characteristics**:
- Returns a **Value Object** representing count (`NonNegativeInteger`, not primitive `number`).
- MUST verify scope hierarchy before delegation (e.g., user access to workspace, workspace access to folder).
- Validates parent scopes exist before counting child resources.
- **Transactions**: MUST NOT be used. Read-only aggregation operation.

**Key steps**:
1. Session verification (if `getSession` param included).
2. Verify scope hierarchy via `ensureIsDataConsistent()` (each repository verifies immediate parent scope).
3. If hierarchy valid, delegate to repository's `count()` method.

**Example hierarchy**: User → Workspace → Folder → ShortUrl
- Verify: Does user own workspace? → Does workspace own folder? → Count items in folder.
- Each repository call passes only its immediate parent scope (`{ workspaceId, userId }`, then `{ folderId, workspaceId }`).

### Pattern 5: UPDATE — Modifying Resources
**File**: `references/examples/update-example.md`

**When to use:** Modifying existing entity fields without changing ownership logic.

**Characteristics**:
- MUST include all parent level IDs the same as CREATE/DELETE.
- Accepts both required IDs and optional Value Objects representing fields to update.
- **Important Note on Errors:** Repository methods that MODIFY structural constraints must return domain-specific database errors alongside standard `RepositoryError`:
  - **Unique Constraints:** e.g., updating a path to one that is already occupied maps to `PathAlreadyInUseError`.
  - **Foreign Key Constraints:** e.g., moving a short URL to a folder that was deleted maps to `FolderNotFoundError`.
  Standard updates without constraint changes usually just return `RepositoryError`.
- **Transactions**: MUST NOT be used for single-resource updates.

**Key steps**:
1. Session verification (if `getSession` param included).
2. Verify scope hierarchy via `ensureIsDataConsistent()` (passing all parent IDs).
3. Delegate to repository's `update()` method.

### Pattern 6: DELETE — Hard vs Soft Deletion
**File**: `references/examples/delete-example.md`

**When to use:** Removing resources or marking them as unavailable.

**Characteristics**:
- **Hard Delete** (Physical removal): Use for low-value data or leaf nodes (e.g., `ShortUrl`).
  - If deleting an intermediate node (e.g., `Folder`), you MUST consider foreign key constraints. You must handle child dependencies (either via DB-level `ON DELETE CASCADE` or via a transaction orchestrating multiple repository deletions).
  - Returns `Result<void, Errors>`.
  - **Transactions**: MUST NOT be used for single-table deletes. MUST be used if manually orchestrating child deletions.
- **Soft Delete** (Logical removal): Use for high-value data, legal compliance, or recoverable resources.
  - MUST NOT use the `Delete` prefix to avoid ambiguity. Use explicit domain verbs like `Archive<Entity>`, `Deactivate<Entity>`, or `Trash<Entity>`.
  - This is technically an `UPDATE` operation in the repository.

**Key steps (Hard Delete)**:
1. Session verification (if `getSession` param included).
2. Verify scope hierarchy via `ensureIsDataConsistent()` (passing all parent IDs).
3. Delegate to repository's `delete()` method.

## 4. Transaction Requirements

Transactions MUST be used only when necessary. Misuse causes performance degradation and unnecessary complexity.

**Use transactions WHEN:**
- ✅ Multiple repositories are written in one Use Case
- ✅ Data consistency across tables is a business requirement
- ✅ Partial failure would violate domain invariants

**MUST NOT use transactions WHEN:**
- ❌ Single read operation (no writes)
- ❌ Single write to one table (no cross-table dependencies)
- ❌ No business rule requires atomic writes

**Example scenarios:**
- `ListWorkspaces`: No transaction (single read)
- `CreateUser` (username + profile in one table): No transaction (single write)
- `CreateWorkspace` (workspace + default folder): **Yes transaction** (two tables, atomicity required)

## 5. Type Definitions Pattern

Each Use Case MUST define two types (*Params* and *Errors*). Optionally, if the Entity contains private data that MUST NOT be exposed, define a **Snapshot type**:

```typescript
// Example 1: Protected Use Case (requires authentication)
export type ListWorkspacesParams = {
  itemsPerPage: PositiveInteger;  // ✅ Value Object
  page: PositiveInteger;          // ✅ Value Object
  getSession: GetSession;         // ✅ Included if auth is required
};

// Example 2: Public Use Case (no authentication required)
export type LoginUserParams = {
  email: Email;                   // ✅ Value Object
  password: Password;             // ✅ Value Object
  // ❌ NO getSession: unauthenticated operation
};

// Errors: MUST be explicit and exhaustive Union
export type YourUseCaseErrors =
  | RepositoryError
  | SessionError
  | YourDomainSpecificError;

// Snapshot (optional): Defined in the same file, excludes private data
export type YourEntitySnapshot = {
  id: UUID;                       // ✅ Value Object
  name: SomeName;                 // ✅ Value Object
  // ❌ NOT included: internalPasswordHash, privateState
};
```

**Return type choices:**
- Without private data: `Promise<Result<Entity, YourUseCaseErrors>>`
- With private data to exclude: `Promise<Result<EntitySnapshot, YourUseCaseErrors>>`

## 6. Authorization (Use Case Permission Verification + Repository Scope Isolation)

**Critical principle**: Authorization has **two distinct layers**:
1. **Scope isolation** (Repository): WHERE clauses prevent cross-user/cross-context data access.
2. **Permission verification** (Use Case): Business logic validates that the authenticated user has permission to perform the action.

### Scope Isolation vs. Permission Verification

**Scope Isolation (Repository Responsibility):**
- Applies WHERE clauses to enforce context boundaries.
- Example: `WHERE userId = #{currentUserId}` prevents User A from accessing User B's data.
- Handled via repository method parameters: `exists({ workspaceId, userId })` vs. `exists({ workspaceId })` for different access levels.

**Permission Verification (Use Case Responsibility):**
- Validates the user has the **specific role or permission** to perform the action.
- Example: User exists in workspace X, but lacks "Delete" permission → Use Case MUST reject.
- MUST run **after** scope hierarchy verification.

### Authorization Flow Example

```typescript
// Use Case: DeleteShortUrl
async execute({ shortUrlId, userId, workspaceId, folderId }: Params): Promise<Result<void, Errors>> {
  // 1. Verify user has "Delete" permission in this workspace
  const userRole = await this.permissionRepository.hasPermission({ workspaceId, userId, action: 'delete' });
  if (userRole.isErr()) {
    return err(userRole.error); // userRole.error is RepositoryError
  }
  if (!userRole.value) {
    return err(new UnauthorizedError({ userId, action: 'delete', resource: 'shortUrl' })); // user lacks delete permission
  }

  // 2. Verify hierarchy (Extracted to private method for readability)
  const consistencyResult = await this.ensureIsDataConsistent({ folderId, workspaceId });
  if (consistencyResult.isErr()) {
    return err(consistencyResult.error);
  }

  // 3. Perform the delete operation, which also enforces scope isolation via WHERE clauses
  return this.shortUrlRepository.delete({ folderId, shortUrlId });
}

private async ensureIsDataConsistent({ folderId, workspaceId }: { folderId: UUID; workspaceId: UUID }): Promise<Result<void, Errors>> {
  /* this.workspaceRepository.exists({ workspaceId, userId }); is not needed here because permission check already verifies user has access to workspace, but if we had a different Use Case that didn't require permission check, we would need to verify workspace access first before checking folder and short URL existence.*/

  const folderExistsResult = await this.folderRepository.exists({ folderId, workspaceId });
  if (folderExistsResult.isErr()) {
    return err(folderExistsResult.error); // folderExistsResult.error is RepositoryError
  }
  if (!folderExistsResult.value) {
    return err(new FolderNotFoundError({ folderId })); // folder doesn't exist in this workspace
  }

  return ok();
}
```

**Key insight**: Each repository call passes ONLY its immediate parent scope:
- `workspaceRepository.exists({ workspaceId, userId })` → WHERE `id = workspaceId AND userId = ?`
- `folderRepository.exists({ folderId, workspaceId })` → WHERE `id = folderId AND workspaceId = ?`
- `shortUrlRepository.exists({ shortUrlId, folderId })` → WHERE `id = shortUrlId AND folderId = ?`

The Use Case orchestrates the hierarchy verification; each repository enforces its single-level scope isolation.

### Scope Hierarchy Verification

In hierarchical domains (e.g., User → Workspace → Folder → ShortUrl), the Use Case MUST verify each level. **To achieve this, the Use Case `Params` MUST explicitly include the IDs for ALL parent levels in the hierarchy** (e.g., `workspaceId`, `folderId`), regardless of how deep the target entity is. The Use Case MUST NOT rely on the database to infer or JOIN parent scopes; it must receive them as inputs to verify the unbroken chain.

```
User (from session)
  └─ Workspace (does user have access?)
      └─ Folder (does workspace own this folder?)
          └─ ShortUrl (does folder own this short URL?)
```

**Example from `countShortUrls.ts`:**

```typescript
// Verification extracted to a private method for readability
private async ensureIsDataConsistent({
  userId,
  folderId,
  workspaceId,
}: {
  userId: UUID;
  workspaceId: UUID;
  folderId: UUID;
}): Promise<Result<void, Errors>> {
  // Verify the complete hierarchy
  const [workspaceExistsResult, folderExistsResult] = await Promise.all([
    this.workspaceRepository.exists({ workspaceId, userId }),        // ← Scope: user in workspace
    this.folderRepository.exists({ folderId, workspaceId }),         // ← Scope: folder in workspace
  ]);

  // If hierarchy is broken, the user cannot access the short URLs
  if (workspaceExistsResult.isErr()) {
    return err(workspaceExistsResult.error); // workspaceExistsResult.error is RepositoryError
  }
  if (!workspaceExistsResult.value) {
    return err(new WorkspaceNotFoundError({ workspaceId })); // user doesn't have access to this workspace
  }

  if (folderExistsResult.isErr()) {
    return err(folderExistsResult.error); // folderExistsResult.error is RepositoryError
  }
  if (!folderExistsResult.value) {
    return err(new FolderNotFoundError({ folderId })); // folder doesn't exist in this workspace
  }

  return ok();
}

// Then inside the core execute() method:
// const consistencyResult = await this.ensureIsDataConsistent({ userId, folderId, workspaceId });
// if (consistencyResult.isErr()) return err(consistencyResult.error);
// return this.shortUrlRepository.count({ folderId });
```

### Repository Support for Different Scopes

Repository methods MAY expose overloaded signatures for different authorization levels:

```typescript
abstract class WorkspaceRepository {
  // Admin scope: no user restriction
  abstract exists(params: { workspaceId: UUID }): Promise<Result<boolean, RepositoryError>>;

  // User scope: must provide both user and workspace
  abstract exists(params: { 
    workspaceId: UUID; 
    userId: UUID; 
  }): Promise<Result<boolean, RepositoryError>>;
}
```

**Use Case determines which scope to use:**

- `DeleteWorkspace` (admin): Calls `workspaceRepository.exists({ workspaceId })`
- `LeaveWorkspace` (user): Calls `workspaceRepository.exists({ workspaceId, userId })`

### MUST and MUST NOT Requirements

- **MUST**: Verify scope hierarchy in Use Case (to ensure data consistency across levels).
- **MUST**: Verify user role/permissions in Use Case (business logic enforcement).
- **MUST**: Apply scope-based WHERE clauses in Repository (to isolate users/contexts).
- **MUST NOT**: Bypass scope verification in the Use Case.
- **MAY**: Overload Repository methods only when different Use Cases genuinely require different scopes (admin vs. user patterns).

## 7. Mandatory Scaffolding for New Projects

If setting up these patterns in a **new project** using our standard stack (`drizzle-orm`, `zod`, `neverthrow`):

1. Ensure `WithTransaction` with `PgTransaction` type is available in `src/shared/domain/types/`.
2. Ensure `withRetry` function exists in `src/shared/` (for resilience patterns).
3. Ensure `GetSession` type and related auth utilities are available in `src/auth/`.

If these do not exist, read `references/core-utilities.md` to scaffold the exact implementations.

## 8. RFC 2119 Language

This skill uses RFC 2119 keywords for precise requirement specification:

- **MUST** / **MUST NOT**: Mandatory requirements. Violations break correctness or security.
- **SHOULD** / **SHOULD NOT**: Strong recommendations. Only deviate with documented justification.
- **MAY**: Optional. Permitted if needed, but not required.
- **Example**: "Use Case names MUST follow `<Action><Entity>` pattern." is non-negotiable.

## 9. Anti-Patterns to Avoid

| ❌ Anti-Pattern | ✅ Correct Pattern |
|---|---|
| `getSession` in constructor | `getSession` as execute parameter (if auth required) |
| Session check after business logic | Session check as FIRST operation (if included) |
| Mixing primitives with VOs in params | All params are Value Objects |
| Throwing domain errors manually | Wrapping in `err(error)` via neverthrow |
| Transacting single-table writes | Only multi-step/multi-table writes need transactions |
| Multiple public methods per class | Single `execute()` method only |
| Direct dependency instantiation | Constructor dependency injection |
| Returning Entities with private data exposed | Return Snapshot type (defined in same file) without private fields |
| Use Case decides all presentation details | Use Case excludes private data only; presentation layer transforms/filters as needed |
| Getting session for unauthenticated Use Cases | Only include `getSession` if authentication is required for that operation |
| Skipping role/permission checks in Use Case | MUST verify user has permission to perform the action (e.g., role-based or capability-based checks) |
| Same Use Case for different access levels | Separate Use Cases: `DeleteWorkspace` (admin) vs `LeaveWorkspace` (user) with different scopes |
| Repository without scope-based WHERE clauses | MUST enforce scope isolation via WHERE: `WHERE userId = ?` or `WHERE workspaceId = ?` |