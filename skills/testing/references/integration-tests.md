# Integration Test Patterns

## Annotated Example: Workspace Repository

The `postgresWorkspaceRepository.integration.test.ts` is the canonical reference for the full helper pattern. It has two parent levels (User → Workspace), making it the clearest illustration of how `setupParent` and `setupSubject` interact.

```typescript
import { assert, describe, expect, vi } from "vitest";
import { PostgresWorkspaceRepository } from "./postgresWorkspaceRepository";
import { Workspace } from "../domain/entities/workspace";
import { UUID } from "@/shared/domain/value-objects/uuid";
import { WorkspaceName } from "../domain/value-objects/workspaceName";
import { workspaces, users } from "@/shared/infrastructure/drizzle-postgres/schema";
import { Email } from "@/shared/domain/value-objects/email";
import { UserDisplayName } from "@/users/domain/value-objects/userDisplayName";
import { txTest } from "@/shared/test/integrationUtils";
import { PositiveInteger } from "@/shared/domain/value-objects/positiveInteger";
import { NodePgDatabase } from "drizzle-orm/node-postgres";
import { RepositoryError } from "@/shared/domain/errors/repositoryError";

// Always define this alias
type Tx = NodePgDatabase;

// ─── Internal helpers ────────────────────────────────────────────────────────
// buildX and insertX are the raw building blocks. They are never called
// directly from tests — only from setup helpers. Named params prevent
// silent arg-order mistakes.

function buildWorkspace({ ownerId }: { ownerId: UUID }): Workspace {
  return new Workspace({
    workspaceId: UUID.random(),
    ownerId,
    workspaceName: WorkspaceName.random(),
    subscriptionPlan: "free",
    subscriptionStatus: "active",
    billingProvider: "stripe",
    externalId: "ext-id",
    createdAt: new Date(),
    updatedAt: new Date(),
  });
}

async function insertWorkspace({ tx, workspace }: { tx: Tx; workspace: Workspace }): Promise<void> {
  await tx.insert(workspaces).values({
    id: workspace.getId().getValue(),
    name: workspace.getName().getValue(),
    ownerId: workspace.getOwnerId().getValue(),
    plan: workspace.getSubscriptionPlan(),
    subscriptionStatus: workspace.getSubscriptionStatus(),
    billingProvider: workspace.getBillingProvider(),
    externalId: workspace.getExternalId(),
    createdAt: workspace.getCreatedAt(),
    updatedAt: workspace.getUpdatedAt(),
  });
}

// ─── setupUser — parent setup ────────────────────────────────────────────────
// Creates a User with all random fields. User is a dependency of Workspace,
// not the subject of this test file. Returns only what callers need.
// The raw insert logic lives here so tests never touch schema directly.

async function setupUser({ tx }: { tx: Tx }): Promise<{ userId: UUID }> {
  const userId = UUID.random();
  await tx.insert(users).values({
    id: userId.getValue(),
    email: Email.random().getValue(),       // .random() for fields irrelevant to tests
    name: UserDisplayName.random().getValue(),
    createdAt: new Date(),
    updatedAt: new Date(),
  });
  return { userId };
}

// ─── setupWorkspace — subject setup ─────────────────────────────────────────
// Creates the subject entity (Workspace). The userId param is optional:
//
//   • Omitted → creates a fresh user too. Use this for noise (isolated chain)
//     or for any test that only needs one workspace.
//
//   • Provided → reuses an existing user. Use this to create sibling workspaces
//     under the same owner (e.g., pagination tests, isolation tests).
//
// This single function replaces both "setup helper" and "noise helper" from
// older patterns. The optional param is what makes it versatile.

async function setupWorkspace(p: {
  tx: Tx;
  userId?: UUID;
}): Promise<{ workspace: Workspace; userId: UUID }> {
  const userId = p.userId ?? (await setupUser({ tx: p.tx })).userId;
  const workspace = buildWorkspace({ ownerId: userId });
  await insertWorkspace({ tx: p.tx, workspace });
  return { workspace, userId };
}

// ─── Tests ───────────────────────────────────────────────────────────────────

describe(PostgresWorkspaceRepository.name, () => {
  txTest("should list workspaces with pagination", async (tx) => {
    const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

    await setupWorkspace({ tx }); // noise — belongs to a different user

    // Three siblings under the same user
    const { userId, workspace: workspace1 } = await setupWorkspace({ tx });
    const { workspace: workspace2 } = await setupWorkspace({ tx, userId });
    const { workspace: workspace3 } = await setupWorkspace({ tx, userId });

    const two = PositiveInteger.from(2)._unsafeUnwrap({ withStackTrace: true });

    const resultPage1 = await workspaceRepository.list({
      userId,
      page: PositiveInteger.one(),
      itemsPerPage: two,
    });

    const resultPage2 = await workspaceRepository.list({
      userId,
      page: two,
      itemsPerPage: two,
    });

    // Repos sort by name — mirror that here so toEqual works
    const workspacesSortedByName = [workspace1, workspace2, workspace3].sort(
      (a, b) => a.getName().getValue().localeCompare(b.getName().getValue()),
    );

    assert(resultPage1.isOk());
    expect(resultPage1.value).toHaveLength(2);
    expect(resultPage1.value).toEqual(workspacesSortedByName.slice(0, 2));

    assert(resultPage2.isOk());
    expect(resultPage2.value).toHaveLength(1);
    expect(resultPage2.value).toEqual(workspacesSortedByName.slice(2));
  });

  txTest("should return empty list for user with no workspaces", async (tx) => {
    const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

    await setupWorkspace({ tx }); // noise

    // Descriptive name: communicates that this user is the point of the test
    const { userId: userWithoutWorkspace } = await setupUser({ tx });

    const result = await workspaceRepository.list({
      userId: userWithoutWorkspace,
      page: PositiveInteger.one(),
      itemsPerPage: PositiveInteger.from(10)._unsafeUnwrap({ withStackTrace: true }),
    });

    assert(result.isOk());
    expect(result.value).toHaveLength(0);
  });

  txTest("should return empty list if user does not exist", async (tx) => {
    const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

    await setupWorkspace({ tx }); // noise

    // Descriptive name: makes the intent obvious without reading the assertion
    const inexistentUserId = UUID.random();

    const result = await workspaceRepository.list({
      userId: inexistentUserId,
      page: PositiveInteger.one(),
      itemsPerPage: PositiveInteger.from(10)._unsafeUnwrap({ withStackTrace: true }),
    });

    assert(result.isOk());
    expect(result.value).toHaveLength(0);
  });

  // exists — happy path
  txTest("should check if workspace exists for a user", async (tx) => {
    const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

    await setupWorkspace({ tx }); // noise
    const { userId, workspace } = await setupWorkspace({ tx });

    const resultExists = await workspaceRepository.exists({
      workspaceId: workspace.getId(),
      userId,
    });

    assert(resultExists.isOk());
    expect(resultExists.value).toBe(true);
  });

  // exists false — case 1: the ID simply doesn't exist anywhere
  txTest("should return false when workspace ID does not exist in the system", async (tx) => {
    const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

    const { userId } = await setupUser({ tx });
    const nonExistentWorkspaceId = UUID.random(); // never inserted

    const result = await workspaceRepository.exists({
      workspaceId: nonExistentWorkspaceId,
      userId,
    });

    assert(result.isOk());
    expect(result.value).toBe(false);
  });

  // exists false — case 2: the workspace exists but belongs to a different user
  // Splitting this from case 1 matters because they can expose different bugs in the query.
  txTest("should return false when workspace exists but does not belong to the user", async (tx) => {
    const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

    const { workspace: someoneElseWorkspace } = await setupWorkspace({ tx });
    const { userId: userIdWithoutWorkspace } = await setupUser({ tx });

    const resultExists = await workspaceRepository.exists({
      workspaceId: someoneElseWorkspace.getId(),
      userId: userIdWithoutWorkspace,
    });

    assert(resultExists.isOk());
    expect(resultExists.value).toBe(false);
  });

  // RepositoryError — one test per public method
  txTest(
    `should return ${RepositoryError.name} when database error occurs on list()`,
    async (tx) => {
      const workspaceRepository = new PostgresWorkspaceRepository({ db: tx });

      await setupWorkspace({ tx }); // noise
      const { userId } = await setupUser({ tx });

      const error = new Error("Database connection lost");
      const dbSpy = vi.spyOn(tx, "select").mockImplementationOnce(() => {
        throw error;
      });

      const result = await workspaceRepository.list({
        userId,
        page: PositiveInteger.one(),
        itemsPerPage: PositiveInteger.from(10)._unsafeUnwrap({ withStackTrace: true }),
      });

      assert(result.isErr());
      // Double assertion: instanceof narrows the type, toEqual checks the message
      expect(result.error).toBeInstanceOf(RepositoryError);
      expect(result.error).toEqual(new RepositoryError(error));

      dbSpy.mockRestore();
    },
  );
});
```

---

## Key Patterns

### setupSubject's optional parent param

The `userId?` optional param is the central design decision. It means one function covers three scenarios:

| Call | Scenario |
| --- | --- |
| `await setupWorkspace({ tx })` | Noise, or single test that only needs one workspace |
| `const { userId, workspace: w1 } = await setupWorkspace({ tx })` | First sibling — captures the userId for reuse |
| `await setupWorkspace({ tx, userId })` | Second/third sibling under the same user |

### Descriptive variable names

The name of a variable documents its role in the test. Prefer names that make the assertion story readable without requiring the reader to trace data flow.

```typescript
// ✓
const nonExistentWorkspaceId = UUID.random();
const { userId: userWithoutWorkspace } = await setupUser({ tx });
const { workspace: someoneElseWorkspace } = await setupWorkspace({ tx });

// ✗
const workspaceId = UUID.random();
const userId2 = UUID.random();
```

### `exists` false — always two tests

When `exists` takes two IDs, always have two distinct false tests:

1. **Entity doesn't exist at all** — `UUID.random()` passed as the entity ID
2. **Entity exists but belongs to a different parent** — real entity ID, different parent

They can expose different bugs in the query (missing WHERE clause vs wrong JOIN condition).

### RepositoryError for every public method

Every method (`create`, `list`, `exists`, `find`, `count`) needs a `RepositoryError` test using `vi.spyOn`:

```typescript
const error = new Error("Database connection lost");
const dbSpy = vi.spyOn(tx, "insert").mockImplementationOnce(() => { throw error; });

const result = await repo.create(...);

assert(result.isErr());
expect(result.error).toBeInstanceOf(RepositoryError);
expect(result.error).toEqual(new RepositoryError(error));

dbSpy.mockRestore();
```

Use `"insert"` for `create`, `"select"` for `list` / `exists` / `find` / `count`.

### FK dependency chain per repo

Each repo's setup helpers depend on those from the level above:

| Repo | setupParent | setupSubject |
| --- | --- | --- |
| `UserRepository` | — (no parent) | `buildUser()` + `insertUser({ tx, user })` directly |
| `WorkspaceRepository` | `setupUser({ tx })` | `setupWorkspace({ tx, userId? })` |
| `FolderRepository` | `setupWorkspace({ tx })` | `setupFolder({ tx, workspaceId? })` |
| `DomainRecordRepository` | `setupWorkspace({ tx })` | `setupDomainRecord({ tx, workspaceId? })` |
| `ShortUrlRepository` | `setupFolder({ tx })` | `setupShortUrl({ tx, folderId? })` |
| `ShortUrlClickRepository` | `setupShortUrl({ tx })` | — (no subject entity; insert via `repo.increment`) |
