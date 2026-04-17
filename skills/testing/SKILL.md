---
name: testing
description: "[Pragmatic DDD Architecture] Guide for creating tests. Use when creating unit tests, integration tests, or understanding test conventions. Covers our tightly coupled stack: Vitest (unit, integration, ui projects), file naming, transactional database integration tests (txTest) with testcontainers/node-postgres/drizzle, mock patterns (createMock*RepoWithAssertions), and neverthrow Result assertions."
---

# Testing

**DEPENDENCY NOTICE**: If you are bootstrapping a new project, read [Test Utilities](references/test-utilities.md) immediately.

## 2. Unit Testing (Use Cases)

Unit tests focus on validating business logic. All external dependencies (Repositories) MUST be strictly mocked.

### Assertions on `neverthrow` Results
Because we use Railway-Oriented Programming:
- Use type narrowing directly: `assert(result.isOk())` from `node:assert`.
- If successful, make assertions on `.value`.
- If failing, make assertions on `.error` types: `expect(result.error).toBeInstanceOf(SomeDomainError);`.

### Setup and Mocking using `vitest` Mock Objects
You MUST always create factory functions tailored for the test file to produce strictly typed Mock Repositories that assert boundaries natively. Pass `expect.hasAssertions()` at the beginning of each test where state is validated heavily inside mocks.

```typescript
import { expect, test, describe, vi } from "vitest";
import { CreateFolder, CreateFolderParams } from "./createFolder";
import { FolderRepository } from "../domain/interfaces/folderRepository";
import { ok, err } from "neverthrow";
import assert from "node:assert";

// Example Mock Factory Pattern
function createMockFolderRepo(p?: Partial<FolderRepository>): FolderRepository {
  return {
    create: vi.fn(async () => ok(undefined)),
    exists: vi.fn(async () => ok(false)),
    ...p,
  };
}

describe(CreateFolder.name, () => {
  test("successfully creates a folder", async () => {
    // 1. Arrange
    const mockRepo = createMockFolderRepo({
      create: vi.fn(async (params) => {
        expect(params.folder.folderName.getValue()).toBe("Test Folder");
        return ok(undefined);
      })
    });
    
    const useCase = new CreateFolder({ folderRepo: mockRepo, ... });
    
    // 2. Act
    const result = await useCase.execute({ ... });
    
    // 3. Assert Narrowing
    assert(result.isOk()); // Narrows result into Ok
    expect(result.value.folderId).toBeDefined();
  });
});
```

## 3. Integration Testing (Repositories)

Integration tests validate that the Domain Repositories correctly translate to DB queries via `drizzle-orm` and handle unique constraints (returning explicitly mapped generic DB errors to specific Domain Errors via `neverthrow`). 

**CRITICAL**: Every single integration test MUST run inside the `txTest` wrapper (documented in [Test Utilities](references/test-utilities.md)). `txTest` **REEMPLAZA** la funcion nativa `test` de Vitest. Provee el entorno de base de datos a través de Testcontainers con `NodePgDatabase`, ejecuta la prueba y luego captura y anula la cascada del `TransactionRollbackError`.

```typescript
import { describe, expect } from "vitest";
import { txTest } from "@/shared/test/integrationUtils";
import { PostgresFolderRepository } from "./postgresFolderRepository";

// ⚠️ No importar el 'test' estandar de vitest cuando se usa txTest
// Usa type Tx = NodePgDatabase para helpers de creacion si existen

describe(PostgresFolderRepository.name, () => {
  txTest("successfully creates a folder and returns ok", async (tx) => {
    // txTest wraps the vitest test directly.
    // Uses the isolated database connection 'tx'
            
    const repo = new PostgresFolderRepository({ db: tx });
      
    // Pass the transaction down to actual methods
    const result = await repo.create({ tx, folder, workspaceId });
      
    expect(result.isOk()).toBe(true);
      
    const exists = await repo.exists({ folderId: folder.getId() });
    assert(exists.isOk()); // with node:assert
    expect(exists.value).toBe(true);
  }); // txTest automatically rolls back via tx.rollback() internally
});
```

## 4. Additional Unit Test Patterns

See [`references/unit-tests.md`](./references/unit-tests.md) for full patterns and annotated examples.

Key conventions:

- `describe` uses `` `${ClassName.name} use case` ``
- Define shared VOs and entities at the **top of `describe`** — keep setup explicit, avoid abstracting into helper factories
- Mock repos with `createMock*RepoWithAssertions()` from `@/shared/test/utils`
- Only configure the methods the test needs — unconfigured methods throw `"This method should not be called"` if invoked
- For use-cases needing many repos, use `createNonCallableMockRepos()` for the ones not relevant to the test
- Error tests: both `toBeInstanceOf` + `toEqual` double assertion
- Error test names: `` `should return ${ErrorClass.name} error ...` ``
- Repo error test names: `` `should return ${RepositoryError.name} error when ${RepoClass.name} fails` ``
- Instantiate the use-case explicitly in each test — do NOT abstract into a helper

## 5. Additional Integration Test Patterns

See [`references/integration-tests.md`](./references/integration-tests.md) for full patterns and annotated examples.

Key conventions:

- `describe` uses `ClassName.name` directly (no suffix)
- Use `txTest` from `@/shared/test/integrationUtils` — never import `test` from vitest
- Each `txTest` receives a `tx` that auto-rollbacks after the test
- Define `type Tx = NodePgDatabase` alias at file top
- All helper functions go **outside** `describe`
- Use `toEqual` for deep object comparison on retrieved entities
- Use `toHaveLength(n)` to confirm exact count on list results
- Test pagination by creating N items and verifying page sizes

### Helper Structure

Each integration test file defines two kinds of helpers. All helpers use **named params** (`{ tx, ... }`) — never positional arguments.

#### 1 — `setupParent` (one level above the subject)

Creates the parent entity with all its own FK dependencies internally. Returns only the parent ID. Tests never call `insertX` raw directly — everything goes through a setup helper.

```typescript
// UserRepository has no parent dependencies, so there's no setupParent.

// FolderRepository's parent is workspace — setupWorkspace creates user + workspace internally.
async function setupWorkspace({ tx }: { tx: Tx }): Promise<{ workspaceId: UUID }> {
  const userId = UUID.random();
  const workspaceId = UUID.random();
  await tx.insert(users).values({ id: userId.getValue(), ... });
  await tx.insert(workspaces).values({ id: workspaceId.getValue(), ownerId: userId.getValue(), ... });
  return { workspaceId };  // userId never leaves — callers don't need it
}
```

#### 2 — `setupSubject` (the entity under test)

Creates the full FK chain down to the subject entity. The parent ID is an **optional param** — when omitted, a new parent chain is created automatically. This single function covers two roles:

- **Noise** — call without parent ID: `await setupWorkspace({ tx })` — creates an isolated chain that won't pollute the subject's queries
- **Sibling** — call with an existing parent ID to create multiple subjects under the same parent

```typescript
async function setupWorkspace(p: {
  tx: Tx;
  userId?: UUID;       // optional — if absent, a fresh user is created automatically
}): Promise<{ workspace: Workspace; userId: UUID }> {
  const userId = p.userId ?? (await setupUser({ tx: p.tx })).userId;
  const workspace = buildWorkspace({ ownerId: userId });
  await insertWorkspace({ tx: p.tx, workspace });
  return { workspace, userId };
}

// Noise (isolated chain):
await setupWorkspace({ tx });

// Siblings under the same user (for testing pagination, list isolation, etc.):
const { userId, workspace: workspace1 } = await setupWorkspace({ tx });
const { workspace: workspace2 } = await setupWorkspace({ tx, userId });
const { workspace: workspace3 } = await setupWorkspace({ tx, userId });
```

The underlying `buildX` and `insertX` helpers exist but are considered **internal** — they're only called from within setup helpers, not directly from tests.

### Named Params

All helpers MUST use named params. This makes call sites self-documenting and avoids positional arg order mistakes.

```typescript
// ✓
function buildFolder({ name }: { name?: FolderName } = {}): Folder { ... }
async function insertFolder({ tx, workspaceId, folder }: { tx: Tx; workspaceId: UUID; folder: Folder }): Promise<void> { ... }
await insertFolder({ tx, workspaceId, folder });

// ✗
async function insertFolder(tx: Tx, workspaceId: UUID, folder: Folder): Promise<void> { ... }
await insertFolder(tx, workspaceId, folder);  // easy to swap args silently
```

### Descriptive Variable Names

Variable names MUST communicate the **role** in the test, not just the type. The name is documentation.

```typescript
// ✓ — intent is clear at a glance
const nonExistentWorkspaceId = UUID.random();
const { userId: userWithoutWorkspace } = await setupUser({ tx });
const { workspace: someoneElseWorkspace } = await setupWorkspace({ tx });

// ✗ — forces reader to infer meaning from usage
const workspaceId = UUID.random();
const userId2 = UUID.random();
```

### Testing `exists` false — always two tests

When `exists` receives two IDs (e.g. `workspaceId + userId`, or `folderId + workspaceId`), split the "false" case into two separate tests. They catch different bugs:

```
// ✓ Two distinct tests:
"should return false when workspace exists but does not belong to the user"  // exists globally, wrong owner
"should return false when workspace ID does not exist in the system"          // never existed

// ✗ One combined test that only proves one of the two:
"should return false when workspace not found"
```

## Assertions with neverthrow Results

You MUST always use `assert()` for type narrowing before accessing `.value` or `.error`:

```typescript
// Ok path
assert(result.isOk());
expect(result.value).toEqual(expectedEntity);

// Err path
assert(result.isErr());
expect(result.error).toBeInstanceOf(SpecificError);
expect(result.error).toEqual(new SpecificError({ ... }));
```

## Value Objects in Tests

- Create random: `UUID.random()`, `WorkspaceName.random()`, `DomainName.random()`
- Create from raw value (unwrap for tests):
  ```typescript
  const itemsPerPage = PositiveInteger.from(10)._unsafeUnwrap({
    withStackTrace: true,
  });
  ```
- Static zero/one factories: `PositiveInteger.one()`, `NonNegativeInteger.zero()`

## Imports

You MUST always use `@/*` path aliases. Use explicit `type` imports where applicable.

```typescript
// Unit tests
import { expect, test, describe, assert, vi } from "vitest";
import { err, ok } from "neverthrow";
import { createMockWorkspaceRepoWithAssertions } from "@/shared/test/utils";

// Integration tests — do NOT import 'test' from vitest
import { assert, describe, expect } from "vitest";
import { txTest } from "@/shared/test/integrationUtils";
```

## Test Coverage Checklist

### Unit tests for use-cases

- [ ] Happy path (all repos succeed)
- [ ] Each domain error case (not found, already exists, validation)
- [ ] Each repository failure (`RepositoryError`)
- [ ] Edge cases (pagination limits, empty results)

### Integration tests for repositories

- [ ] Basic create / find / list operations
- [ ] Pagination (multiple pages, verify sizes and that noise data is excluded)
- [ ] Empty list for entity with no children
- [ ] Empty list / null when the parent does not exist
- [ ] `exists` returns `true` for the happy path
- [ ] `exists` returns `false` when the ID does not exist at all in the system
- [ ] `exists` returns `false` when the entity exists but belongs to a different parent (if applicable)
- [ ] Constraint violations where relevant (duplicate name, FK missing)
- [ ] `RepositoryError` for every public method
