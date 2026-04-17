```typescript
// Example: ListShortUrls Use Case
// Pattern: LIST — Query with pagination, scope hierarchy verification, and validation
// RFC 2119: This example shows MUST/MUST NOT requirements for List use cases
// Returns: Array of domain Entities (ShortUrl[]) — NEVER primitives

// MUST: All params use Value Objects, MUST NOT use primitives
// MUST: List operations MUST include pagination (offset-based or cursor-based)
export type ListShortUrlsParams = {
  workspaceId: UUID;              // ✅ VO - part of scope hierarchy
  folderId: UUID;                 // ✅ VO - part of scope hierarchy
  itemsPerPage: PositiveInteger;  // ✅ VO for pagination (offset-based example)
  page: PositiveInteger;          // ✅ VO for pagination (offset-based example)
  getSession: GetSession;         // ✅ MUST be parameter, NOT constructor
};

// MUST: Error union is exhaustive
export type ListShortUrlsErrors =
  | RepositoryError
  | ExceedsMaxItemsPerPageError
  | SessionError
  | WorkspaceNotFoundError
  | FolderNotFoundError;

export class ListShortUrls {
  private readonly shortUrlRepository: ShortUrlRepository;
  private readonly workspaceRepository: WorkspaceRepository;
  private readonly folderRepository: FolderRepository;
  private readonly maxItemsPerPage: PositiveInteger;

  // MUST: Dependencies injected via constructor
  constructor(params: {
    shortUrlRepository: ShortUrlRepository;
    workspaceRepository: WorkspaceRepository;
    folderRepository: FolderRepository;
    maxItemsPerPage: PositiveInteger;
  }) {
    this.shortUrlRepository = params.shortUrlRepository;
    this.workspaceRepository = params.workspaceRepository;
    this.folderRepository = params.folderRepository;
    this.maxItemsPerPage = params.maxItemsPerPage;
  }

  // MUST: Single public method named execute()
  async execute({
    folderId,
    itemsPerPage,
    page,
    workspaceId,
    getSession,
  }: ListShortUrlsParams): Promise<Result<ShortUrl[], ListShortUrlsErrors>> {
    // MUST: Validate pagination constraints FIRST
    // Pagination MUST be present and MUST have business rule constraints (e.g., max items per page)
    const isInvalidItemsPerPage = itemsPerPage.isGreaterThan(
      this.maxItemsPerPage
    );

    if (isInvalidItemsPerPage) {
      return err(
        new ExceedsMaxItemsPerPageError({
          invalidValue: itemsPerPage,
          maxAllowed: this.maxItemsPerPage,
        })
      );
    }

    // MUST: Session verification (if getSession is included in params)
    // In this example, getSession is required for ListShortUrls (protected operation)
    const sessionResult = await getSession();
    if (sessionResult.isErr()) return err(sessionResult.error);
    
    const { userId } = sessionResult.value;

    // MUST: Verify scope hierarchy BEFORE delegating to repository
    // Example: User → Workspace → Folder → ShortUrl
    // Each level verifies its immediate parent scope:
    // - workspaceRepository.exists({ workspaceId, userId }) → WHERE workspaceId = ? AND userId = ?
    // - folderRepository.exists({ folderId, workspaceId }) → WHERE folderId = ? AND workspaceId = ?
    const dataConsistencyResult = await this.ensureIsDataConsistent({
      userId,
      workspaceId,
      folderId,
    });

    if (dataConsistencyResult.isErr()) return err(dataConsistencyResult.error);

    // MUST NOT: Use transaction for read-only operations
    // Single Repository.list() call needs no transaction:
    // - No writes = no atomicity concerns
    // - Scope hierarchy already verified above
    return await this.shortUrlRepository.list({
      folderId,
      itemsPerPage,
      page,
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
    // Verify in parallel: workspace (parent of folder) and folder (parent of short URLs)
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