# Unit Test Patterns

## Complete Example: Use-Case Test

```typescript
import { expect, test, describe, assert } from "vitest";
import { ListWorkspaces } from "./listWorkspaces";
import { UUID } from "@/shared/domain/value-objects/uuid";
import { UserRepository } from "@/users/domain/interfaces/userRepository";
import { err, ok } from "neverthrow";
import { WorkspaceRepository } from "../domain/interfaces/workspaceRepository";
import { Workspace } from "../domain/entities/workspace";
import { WorkspaceName } from "../domain/value-objects/workspaceName";
import { UserNotFoundError } from "@/users/domain/errors/userNotFoundError";
import { RepositoryError } from "@/shared/domain/errors/repositoryError";
import {
  createMockUserRepoWithAssertions,
  createMockWorkspaceRepoWithAssertions,
} from "@/shared/test/utils";
import { PositiveInteger } from "@/shared/domain/value-objects/positiveInteger";
import { ExceedsMaxItemsPerPageError } from "@/shared/domain/errors/exceedsMaxItemsPerPageError";

describe(`${ListWorkspaces.name} Use Case`, () => {
  // Shared VOs and data — explicit, at the top of describe
  const userId = UUID.random();
  const workspaceName = WorkspaceName.random();
  const workspaceId = UUID.random();
  const workspace = new Workspace({ workspaceId, workspaceName });

  const databaseError = new RepositoryError(
    new Error("Database connection failed"),
  );

  const itemsPerPage = PositiveInteger.from(10)._unsafeUnwrap({
    withStackTrace: true,
  });
  const page = PositiveInteger.one();
  const maxItemsPerPage = PositiveInteger.from(50)._unsafeUnwrap({
    withStackTrace: true,
  });

  // --- Happy path test ---
  test("should list workspaces for a valid user", async () => {
    // Only configure the mock methods the use-case will call
    const userRepository = createMockUserRepoWithAssertions({
      exists: {
        toBeCalledWith: [{ userId }],
        result: ok(true),
      },
    });

    const workspaceRepository = createMockWorkspaceRepoWithAssertions({
      list: {
        toBeCalledWith: [{ userId, itemsPerPage, page }],
        result: ok([workspace]),
      },
    });

    // Instantiate use-case explicitly in each test
    const useCase = new ListWorkspaces({
      userRepository,
      workspaceRepository,
      maxItemsPerPage,
    });

    const result = await useCase.execute({ userId, itemsPerPage, page });

    // assert() narrows the Result type
    assert(result.isOk());
    expect(result.value).toContainEqual(workspace);
  });

  // --- Domain error test: naming convention uses ErrorClass.name ---
  test(`should return ${ExceedsMaxItemsPerPageError.name} error when itemsPerPage exceeds the maximum allowed`, async () => {
    // Repos not called — pass empty mock (unconfigured methods throw if invoked)
    const userRepository = createMockUserRepoWithAssertions();
    const workspaceRepository = createMockWorkspaceRepoWithAssertions();

    const useCase = new ListWorkspaces({
      userRepository,
      workspaceRepository,
      maxItemsPerPage,
    });

    const invalidItemsPerPage = maxItemsPerPage.add(itemsPerPage);

    const result = await useCase.execute({
      userId,
      itemsPerPage: invalidItemsPerPage,
      page,
    });

    assert(result.isErr());
    // Double assertion: instanceof + toEqual
    expect(result.error).toBeInstanceOf(ExceedsMaxItemsPerPageError);
    expect(result.error).toEqual(
      new ExceedsMaxItemsPerPageError({
        invalidValue: invalidItemsPerPage,
        maxAllowed: maxItemsPerPage,
      }),
    );
  });

  // --- Not-found error test ---
  test(`should return ${UserNotFoundError.name} error for non-existing user`, async () => {
    const userRepository = createMockUserRepoWithAssertions({
      exists: {
        toBeCalledWith: [{ userId }],
        result: ok(false),
      },
    });
    const workspaceRepository = createMockWorkspaceRepoWithAssertions();

    const useCase = new ListWorkspaces({
      userRepository,
      workspaceRepository,
      maxItemsPerPage,
    });

    const result = await useCase.execute({ userId, itemsPerPage, page });

    assert(result.isErr());
    expect(result.error).toBeInstanceOf(UserNotFoundError);
    expect(result.error).toEqual(new UserNotFoundError({ userId }));
  });

  // --- Repository error test: naming mentions the repo class ---
  test(`should return ${RepositoryError.name} error when ${UserRepository.name} fails`, async () => {
    const userRepository = createMockUserRepoWithAssertions({
      exists: {
        toBeCalledWith: [{ userId }],
        result: err(databaseError),
      },
    });
    const workspaceRepository = createMockWorkspaceRepoWithAssertions();

    const useCase = new ListWorkspaces({
      maxItemsPerPage,
      userRepository,
      workspaceRepository,
    });

    const result = await useCase.execute({ userId, itemsPerPage, page });

    assert(result.isErr());
    expect(result.error).toBeInstanceOf(RepositoryError);
    expect(result.error).toEqual(databaseError);
  });

  test(`should return ${RepositoryError.name} error when ${WorkspaceRepository.name} fails`, async () => {
    const userRepository = createMockUserRepoWithAssertions({
      exists: {
        toBeCalledWith: [{ userId }],
        result: ok(true),
      },
    });
    const workspaceRepository = createMockWorkspaceRepoWithAssertions({
      list: {
        toBeCalledWith: [{ userId, itemsPerPage, page }],
        result: err(databaseError),
      },
    });

    const useCase = new ListWorkspaces({
      userRepository,
      maxItemsPerPage,
      workspaceRepository,
    });

    const result = await useCase.execute({ userId, itemsPerPage, page });

    assert(result.isErr());
    expect(result.error).toBeInstanceOf(RepositoryError);
    expect(result.error).toEqual(databaseError);
  });
});
```

## Mock Pattern: `createMock*RepoWithAssertions`

Available mock factories in `@/shared/test/utils`:

- `createMockUserRepoWithAssertions(p?)`
- `createMockWorkspaceRepoWithAssertions(p?)`
- `createMockFolderRepoWithAssertions(p?)`
- `createMockShortUrlRepoWithAssertions(p?)`
- `createMockDomainRecordRepoWithAssertions(p?)`
- `createNonCallableMockRepos()` — returns all five repos as unconfigured mocks

Each accepts a `MethodsMock<T>` parameter where you configure only the methods you expect to be called:

```typescript
const userRepository = createMockUserRepoWithAssertions({
  exists: {
    toBeCalledWith: [{ userId }], // expected params
    result: ok(true), // return value
  },
});
```

If a method is NOT configured and gets called at runtime, it throws `"This method should not be called"`, making unexpected calls visible.

## Context Pattern for Complex Use-Cases

When a use-case has many parameters and expected entities, use a `Context` type + `createContext()`:

```typescript
type Context = {
  userId: UUID;
  workspaceId: UUID;
  folderId: UUID;
  targetUrl: AppUrl;
  // ...
};

function createContext(): Context {
  const userId = UUID.random();
  // ... build all VOs and entities
  return { userId /* ... */ };
}
```

This is suitable when the same test file has many tests sharing complex setup. For simpler use-cases, prefer inline VOs at the top of `describe`.
