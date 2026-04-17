```typescript
// Example: CreateShortUrl Use Case
// Pattern: CREATE — Multi-step orchestration, retry pattern, and hierarchy verification
// RFC 2119: This example shows MUST/MUST NOT requirements for Create use cases
// Returns: A Domain Entity or typed result (CreateShortUrlSuccess) — NEVER primitives
//
// NOTE ON ERRORS:
// Repository methods that ADD new information to the database must handle structural constraints
// and return domain-specific database errors alongside standard RepositoryErrors:
// - Unique Constraints: e.g., PathAlreadyInUseError
// - Foreign Key Constraints: e.g., FolderNotFoundError, DomainRecordNotFoundError
// Standard creations without these constraints typically only return RepositoryError.

export type CreateShortUrlRetryConfig = {
  maxRetries: PositiveInteger;
  delayMilliseconds: NonNegativeInteger;
};

declare const unique: unique symbol;

export class CouldNotGenerateUniquePathError extends Error {
  declare private readonly [unique]: never;

  constructor(maxRetries: PositiveInteger) {
    super(
      `Could not generate a unique URL path after ${maxRetries.getValue()} attempts`,
    );
  }
}

// MUST: All params use Value Objects, MUST NOT use primitives
export type CreateShortUrlParams = {
  getSession: GetSession;                 // ✅ MUST be parameter, NOT constructor
  workspaceId: UUID;                      // ✅ VO (not string)
  folderId: UUID;                         // ✅ VO (not string)
  shortUrlName: ShortUrlName;             // ✅ VO (not string)
  targetUrl: AppUrl;                      // ✅ VO (not string)
  shortUrlPath?: URLPath;                 // ✅ Optional VO (not string)
  domainRecordId?: UUID;                  // ✅ Optional VO (not string)
};

export type CreateShortUrlSuccess = {
  shortUrlId: UUID;
};

// MUST: Error union is exhaustive
export type CreateShortUrlErrors =
  | SessionError
  | RepositoryError
  | WorkspaceNotFoundError
  | FolderNotFoundError
  | DomainRecordNotFoundError
  | PathAlreadyInUseError
  | CouldNotGenerateUniquePathError;

export class CreateShortUrl {
  private readonly workspaceRepository: WorkspaceRepository;
  private readonly folderRepository: FolderRepository;
  private readonly shortUrlRepository: ShortUrlRepository;
  private readonly domainRecordRepository: DomainRecordRepository;
  private readonly retryConfig: CreateShortUrlRetryConfig;

  // MUST: Dependencies injected via constructor
  constructor(p: {
    workspaceRepository: WorkspaceRepository;
    folderRepository: FolderRepository;
    shortUrlRepository: ShortUrlRepository;
    domainRecordRepository: DomainRecordRepository;
    retryConfig: CreateShortUrlRetryConfig;
  }) {
    this.workspaceRepository = p.workspaceRepository;
    this.folderRepository = p.folderRepository;
    this.shortUrlRepository = p.shortUrlRepository;
    this.domainRecordRepository = p.domainRecordRepository;
    this.retryConfig = p.retryConfig;
  }

  // MUST: Single public method named execute()
  async execute({
    getSession,
    workspaceId,
    folderId,
    domainRecordId,
    targetUrl,
    shortUrlPath: candidateShortUrl,
    shortUrlName,
  }: CreateShortUrlParams): Promise<
    Result<CreateShortUrlSuccess, CreateShortUrlErrors>
  > {
    // MUST: Session verification first (if getSession is included in params)
    const sessionResult = await getSession();
    if (sessionResult.isErr()) return err(sessionResult.error);

    const { userId } = sessionResult.value;

    // MUST: Verify scope hierarchy BEFORE delegating to repository or taking action
    // Hierarchy: User -> Workspace -> Folder (and optionally DomainRecord)
    const dataConsistencyResult = await this.ensureIsDataConsistent({
      userId,
      workspaceId,
      folderId,
      domainRecordId,
    });

    if (dataConsistencyResult.isErr()) return err(dataConsistencyResult.error);

    // Retry logic for unique path generation
    const retryResult = await withRetry(this.retryConfig, async ({ retry }) => {
      const shortUrlId = UUID.random();

      if (candidateShortUrl) {
        const shortUrl: ShortUrlRepositoryCreateParams = {
          shortUrlId,
          folderId,
          domainRecordId,
          path: candidateShortUrl,
          shortUrlName,
          targetUrl,
          createdAt: new Date(),
          updatedAt: new Date(),
        };

        const createResult = await this.shortUrlRepository.create(shortUrl);

        if (createResult.isOk()) return ok({ shortUrlId });
        return err(createResult.error);
      }

      const shortUrl: ShortUrlRepositoryCreateParams = {
        shortUrlId,
        folderId,
        domainRecordId,
        path: URLPath.random(),
        shortUrlName,
        targetUrl,
        createdAt: new Date(),
        updatedAt: new Date(),
      };

      const createResult = await this.shortUrlRepository.create(shortUrl);

      // Railway-Oriented Programming: Return the error, retrying only if conflict
      if (createResult.isErr()) {
        const { error } = createResult;
        if (error instanceof PathAlreadyInUseError) retry();
        return err(error);
      }

      return ok({ shortUrlId });
    });

    if (retryResult.isErr()) {
      const { error } = retryResult;
      const isMaxRetriesExceeded = error instanceof MaxRetriesReachedError;
      if (!isMaxRetriesExceeded) return err(error);
      return err(new CouldNotGenerateUniquePathError(this.retryConfig.maxRetries));
    }

    return ok(retryResult.value);
  }

  // MUST: Private method for scope consistency verification
  // Each repository verifies only its immediate parent scope
  private async ensureIsDataConsistent({
    userId,
    folderId,
    workspaceId,
    domainRecordId,
  }: {
    userId: UUID;
    workspaceId: UUID;
    folderId: UUID;
    domainRecordId?: UUID;
  }): Promise<
    Result<
      void,
      | WorkspaceNotFoundError
      | FolderNotFoundError
      | RepositoryError
      | DomainRecordNotFoundError
    >
  > {
    if (domainRecordId) {
      // verifies domain record is in workspace
      const domainRecordExistsResult = await this.domainRecordRepository.exists(
        { domainRecordId, workspaceId },
      );

      if (domainRecordExistsResult.isErr()) return err(domainRecordExistsResult.error);
      if (!domainRecordExistsResult.value) return err(new DomainRecordNotFoundError({ domainRecordId }));
    }

    // Verify in parallel: workspace (parent of folder) and folder (parent of short URLs)
    const [workspaceExistsResult, folderExistsResult] = await Promise.all([
      this.workspaceRepository.exists({ workspaceId, userId }), // Scope: user in workspace
      this.folderRepository.exists({ folderId, workspaceId }),  // Scope: folder in workspace
    ]);

    if (workspaceExistsResult.isErr()) return err(workspaceExistsResult.error);
    if (!workspaceExistsResult.value) return err(new WorkspaceNotFoundError({ workspaceId }));

    if (folderExistsResult.isErr()) return err(folderExistsResult.error);
    if (!folderExistsResult.value) return err(new FolderNotFoundError({ folderId }));

    return ok();
  }
}
```