```typescript
// Example: ExistsShortUrl Use Case
// Pattern: EXISTS — Simple authorization & presence verification with hierarchy
// RFC 2119: This example shows MUST/MUST NOT requirements for Exists use cases
// Returns: Result<boolean> — NEVER primitive boolean

// MUST: All params use Value Objects, MUST NOT use primitives
export type ExistsShortUrlParams = {
  shortUrlId: UUID;       // ✅ VO (not string)
  folderId: UUID;         // ✅ VO (not string)
  workspaceId: UUID;      // ✅ VO (not string)
  getSession: GetSession; // ✅ MUST be parameter, NOT constructor
};

// MUST: Error union is exhaustive
export type ExistsShortUrlErrors = 
  | RepositoryError 
  | SessionError 
  | WorkspaceNotFoundError 
  | FolderNotFoundError;

export class ExistsShortUrl {
  private readonly shortUrlRepository: ShortUrlRepository;
  private readonly folderRepository: FolderRepository;
  private readonly workspaceRepository: WorkspaceRepository;

  // MUST: Dependencies injected via constructor
  constructor(params: {
    shortUrlRepository: ShortUrlRepository;
    folderRepository: FolderRepository;
    workspaceRepository: WorkspaceRepository;
  }) {
    this.shortUrlRepository = params.shortUrlRepository;
    this.folderRepository = params.folderRepository;
    this.workspaceRepository = params.workspaceRepository;
  }

  // MUST: Single public method named execute()
  // RETURNS: Result<boolean> — NOT primitive boolean
  async execute({
    getSession,
    shortUrlId,
    folderId,
    workspaceId,
  }: ExistsShortUrlParams): Promise<Result<boolean, ExistsShortUrlErrors>> {
    // MUST: Session verification first (if getSession is included in params)
    const resultSession = await getSession();
    if (resultSession.isErr()) return err(resultSession.error);

    const { userId } = resultSession.value;

    // MUST: Verify scope hierarchy BEFORE delegating to repository
    const dataConsistencyResult = await this.ensureIsDataConsistent({
      userId,
      workspaceId,
      folderId,
    });

    if (dataConsistencyResult.isErr()) return err(dataConsistencyResult.error);

    // MUST NOT: Use transaction for simple read operations
    return await this.shortUrlRepository.exists({
      shortUrlId,
      folderId,
    });
  }

  // MUST: Private method for scope consistency verification
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
    const [workspaceExistsResult, folderExistsResult] = await Promise.all([
      this.workspaceRepository.exists({ workspaceId, userId }),
      this.folderRepository.exists({ folderId, workspaceId }),
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