```typescript
// Example: CountShortUrls Use Case
// Pattern: COUNT — Query aggregation with scope hierarchy verification
// RFC 2119: This example shows MUST/MUST NOT requirements for Count use cases
// Returns: A Value Object representing the count (NonNegativeInteger) — NEVER primitive number

// MUST: All params use Value Objects, MUST NOT use primitives
export type CountShortUrlsParams = {
  workspaceId: UUID;      // ✅ VO - part of scope hierarchy
  folderId: UUID;         // ✅ VO - part of scope hierarchy
  getSession: GetSession; // ✅ MUST be parameter, NOT constructor
};

// MUST: Error union is exhaustive
export type CountShortUrlsErrors =
  | RepositoryError
  | SessionError
  | WorkspaceNotFoundError
  | FolderNotFoundError;

export class CountShortUrls {
  private readonly shortUrlRepository: ShortUrlRepository;
  private readonly workspaceRepository: WorkspaceRepository;
  private readonly folderRepository: FolderRepository;

  // MUST: Dependencies injected via constructor
  constructor(params: {
    shortUrlRepository: ShortUrlRepository;
    workspaceRepository: WorkspaceRepository;
    folderRepository: FolderRepository;
  }) {
    this.shortUrlRepository = params.shortUrlRepository;
    this.workspaceRepository = params.workspaceRepository;
    this.folderRepository = params.folderRepository;
  }

  // MUST: Single public method named execute()
  // Returns: Result<NonNegativeInteger, Errors> — NOT primitive number
  async execute({
    getSession,
    workspaceId,
    folderId,
  }: CountShortUrlsParams): Promise<
    Result<NonNegativeInteger, CountShortUrlsErrors>
  > {
    // MUST: Session verification first (if getSession is included in params)
    // In this example, getSession is required for CountShortUrls (protected operation)
    const sessionResult = await getSession();
    if (sessionResult.isErr()) return err(sessionResult.error);

    const { userId } = sessionResult.value;

    // MUST: Verify scope hierarchy BEFORE delegating to repository
    // Count operations that work with resources in a hierarchy MUST verify parent scopes
    // Example: User → Workspace → Folder → ShortUrl
    // Each level verifies its immediate parent scope:
    // - workspaceRepository.exists({ workspaceId, userId }) → WHERE workspaceId = ? AND userId = ?
    // - folderRepository.exists({ folderId, workspaceId }) → WHERE folderId = ? AND workspaceId = ?
    const consistencyResult = await this.ensureIsDataConsistent({
      userId,
      workspaceId,
      folderId,
    });

    if (consistencyResult.isErr()) return err(consistencyResult.error);

    // MUST NOT: Use transaction for read-only operations
    // Single Repository.count() call needs no transaction:
    // - No writes = no atomicity concerns
    // - Scope hierarchy already verified above
    // - Aggregation queries are inherently isolated
    return await this.shortUrlRepository.count({
      folderId,
    });
  }

  // MUST: Private method for scope consistency verification
  // Each repository verifies only its immediate parent scope
  private async ensureIsDataConsistent({
    userId,
    folderId,
    workspaceId,
  }: {
    userId: UUID;
    workspaceId: UUID;
    folderId: UUID;
  }): Promise<
    Result<void, WorkspaceNotFoundError | FolderNotFoundError | RepositoryError>
  > {
    // Verify in parallel: workspace (parent of folder) and folder (parent of short URLs we're counting)
    const [workspaceExistsResult, folderExistsResult] = await Promise.all([
      this.workspaceRepository.exists({ workspaceId, userId }),     // ← Scope: user in workspace
      this.folderRepository.exists({ folderId, workspaceId }),      // ← Scope: folder in workspace
    ]);

    if (workspaceExistsResult.isErr()) return err(workspaceExistsResult.error);
    if (!workspaceExistsResult.value) {
      return err(new WorkspaceNotFoundError({ workspaceId }));
    }

    if (folderExistsResult.isErr()) return err(folderExistsResult.error);
    if (!folderExistsResult.value) {
      return err(new FolderNotFoundError({ folderId }));
    }

    return ok();
  }
}
```
