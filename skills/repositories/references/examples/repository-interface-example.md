```typescript
// Example: ShortUrlRepository (Domain Interface)
// Pattern: Abstract Class, Value Objects strictly enforced, Explicit Error Unions
// RFC 2119: This example shows MUST/MUST NOT requirements for Repository interfaces

// MUST: Input parameters use Value Objects
export type ShortUrlRepositoryCreateParams = {
  shortUrlId: UUID;
  shortUrlName: ShortUrlName;
  path: URLPath;
  targetUrl: AppUrl;
  domainRecordId?: UUID;
  folderId: UUID; // MUST: Include scope parent ID
  createdAt: Date;
  updatedAt: Date;
};

export type ShortUrlRepositoryUpdateParams = {
  shortUrlId: UUID;
  folderId: UUID; // MUST: Include scope parent ID
  shortUrlName: ShortUrlName;
  path: URLPath;
  targetUrl: AppUrl;
  domainRecordId?: UUID;
  updatedAt: Date;
};

export type ShortUrlRepositoryDeleteParams = {
  shortUrlId: UUID;
  folderId: UUID; // MUST: Include scope parent ID to prevent overriding
};

// MUST: Explicit error unions
// NOTE: This mutation hits structural constraints (Unique URL Path, Foreign Keys to Folder/DomainRecord)
// Therefore, it returns domain-specific database errors alongside RepositoryError.
export type ShortUrlRepositoryCreateErrors =
  | RepositoryError
  | PathAlreadyInUseError
  | FolderNotFoundError
  | DomainRecordNotFoundError;

export type ShortUrlRepositoryUpdateErrors =
  | RepositoryError
  | PathAlreadyInUseError
  | DomainRecordNotFoundError; // Assuming folder can't be changed, it just acts as scope

// MUST: Be defined as an `abstract class` (not an `interface`) to support DI tokens
export abstract class ShortUrlRepository {
  // MUST: Support optional injected transactions for mutations
  abstract create(
    params: ShortUrlRepositoryCreateParams & { tx?: Transaction }
  ): Promise<Result<void, ShortUrlRepositoryCreateErrors>>;

  // MUST: Also support optional injected transactions for updates
  abstract update(
    params: ShortUrlRepositoryUpdateParams & { tx?: Transaction }
  ): Promise<Result<void, ShortUrlRepositoryUpdateErrors>>;

  // MUST: Also support optional injected transactions for deletes
  abstract delete(
    params: ShortUrlRepositoryDeleteParams & { tx?: Transaction }
  ): Promise<Result<void, RepositoryError>>;

  // MUST: Include pagination for LIST operations and isolate by parent scope (folderId)
  abstract list(params: {
    folderId: UUID;
    itemsPerPage: PositiveInteger;
    page: PositiveInteger;
  }): Promise<Result<ShortUrl[], RepositoryError>>;
  
  // MUST: Find operations return the Entity or null if not found.
  // Scope Isolation constraint MUST be applied.
  abstract find(params: {
    folderId: UUID;
    shortUrlId: UUID;
  }): Promise<Result<ShortUrl | null, RepositoryError>>;

  // MUST: Provide Scope Isolation. Check by constraint AND scope.
  // Standard read operations only return RepositoryError on failure.
  abstract exists(params: {
    folderId: UUID;
    urlPath: URLPath;
    domainRecordId?: UUID;
  }): Promise<Result<boolean, RepositoryError>>;

  abstract exists(params: {
    folderId: UUID; // Scope Isolation constraint
    shortUrlId: UUID;
  }): Promise<Result<boolean, RepositoryError>>;

  abstract count(params: {
    folderId: UUID; // Scope Isolation constraint
  }): Promise<Result<NonNegativeInteger, RepositoryError>>;
}
```