```typescript
// Example: UpdateShortUrl Use Case
// Pattern: UPDATE — Modify an existing resource with scope hierarchy verification
// RFC 2119: This example shows MUST/MUST NOT requirements for Update use cases
// Returns: Result<void> or Result<Entity> — NEVER primitives
//
// NOTE ON ERRORS: 
// Repository methods that MODIFY structural constraints must return domain-specific 
// database errors alongside the standard RepositoryError:
// - Unique Constraints: e.g., PathAlreadyInUseError (updating to an existing path)
// - Foreign Key Constraints: e.g., FolderNotFoundError (moving URL to a deleted folder)
// Standard updates without these constraints usually only return RepositoryError.

// MUST: All params use Value Objects, MUST NOT use primitives
// MUST: Include ALL parent IDs in the hierarchy regardless of depth
export type UpdateShortUrlParams = {
  shortUrlId: UUID;              // ✅ VO (not string)
  folderId: UUID;                // ✅ VO - parent level
  workspaceId: UUID;             // ✅ VO - grandparent level
  targetUrl?: AppUrl;            // ✅ Optional VO to update
  shortUrlName?: ShortUrlName;   // ✅ Optional VO to update
  getSession: GetSession;        // ✅ MUST be parameter, NOT constructor
};

// MUST: Error union is exhaustive
export type UpdateShortUrlErrors = 
  | RepositoryError 
  | SessionError 
  | WorkspaceNotFoundError 
  | FolderNotFoundError;

export class UpdateShortUrl {
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
  async execute({
    getSession,
    shortUrlId,
    folderId,
    workspaceId,
    targetUrl,
    shortUrlName,
  }: UpdateShortUrlParams): Promise<Result<void, UpdateShortUrlErrors>> {
    // MUST: Session verification first
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

    // MUST NOT: Use transaction for single-table updates
    // The repository MUST apply scope isolation (e.g. WHERE shortUrlId = ? AND folderId = ?)
    return await this.shortUrlRepository.update({
      shortUrlId,
      folderId,
      targetUrl,
      shortUrlName,
      updatedAt: new Date(),
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