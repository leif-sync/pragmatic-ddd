```typescript
// Example: DeleteShortUrl Use Case
// Pattern: DELETE — Hard Delete of a leaf node with hierarchy verification
// RFC 2119: This example shows MUST/MUST NOT requirements for Delete use cases
// Returns: Result<void> — NEVER primitives
//
// NOTE ON DELETIONS: 
// 1. Hard Deletes: This example performs a physical database deletion (Hard Delete). 
//    It is ideal for low-value or leaf-node objects (like ShortUrl) that don't have dependent 
//    children. If deleting an intermediate node (like Folder), you must handle child dependencies 
//    to avoid foreign key constraint errors (either via DB-level CASCADE or manual transactional 
//    orchestration).
// 2. Soft Deletes: For high-value objects (users, workspaces, billing data) requiring legal 
//    compliance or recoverability, DO NOT use the "Delete" action verb. Instead, use an explicit 
//    update use case like "ArchiveWorkspace" or "DeactivateUser" to avoid ambiguity, as soft 
//    deletes are technically just UPDATE operations.

// MUST: All params use Value Objects, MUST NOT use primitives
// MUST: Include ALL parent IDs in the hierarchy regardless of depth
export type DeleteShortUrlParams = {
  shortUrlId: UUID;       // ✅ VO (not string)
  folderId: UUID;         // ✅ VO - parent level
  workspaceId: UUID;      // ✅ VO - grandparent level
  getSession: GetSession; // ✅ MUST be parameter, NOT constructor
};

// MUST: Error union is exhaustive
export type DeleteShortUrlErrors = 
  | RepositoryError 
  | SessionError 
  | WorkspaceNotFoundError 
  | FolderNotFoundError;

export class DeleteShortUrl {
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
  // RETURNS: Result<void, Errors> for pure delete operations
  async execute({
    getSession,
    shortUrlId,
    folderId,
    workspaceId,
  }: DeleteShortUrlParams): Promise<Result<void, DeleteShortUrlErrors>> {
    // MUST: Session verification first
    const resultSession = await getSession();
    if (resultSession.isErr()) return err(resultSession.error);

    const { userId } = resultSession.value;

    // MUST: Verify scope hierarchy BEFORE delegating to repository
    // Extracts isolation validation to a private method
    const dataConsistencyResult = await this.ensureIsDataConsistent({
      userId,
      workspaceId,
      folderId,
    });

    if (dataConsistencyResult.isErr()) return err(dataConsistencyResult.error);

    // MUST NOT: Use transaction for simple single-table delete operation
    // The repository MUST apply scope isolation (e.g. WHERE shortUrlId = ? AND folderId = ?)
    return await this.shortUrlRepository.delete({
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