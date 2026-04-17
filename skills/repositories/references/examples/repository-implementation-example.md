```typescript
// Example: PostgresShortUrlRepository (Database Implementation)
// Pattern: Drizzle ORM queries, Transaction Fallback, Postgres Error Mapping
// RFC 2119: This example shows MUST/MUST NOT requirements for Postgres repository implementations
// Returns: Result logic mapped into explicit error types mapped from node-postgres error codes

import { NodePgDatabase } from "drizzle-orm/node-postgres";
import { DrizzleQueryError } from "drizzle-orm";
import { DatabaseError } from "pg";
import { PG_ERROR_CODES } from "@/shared/infrastructure/drizzle-postgres/pgErrorCodes";
// MUST: Import explicitly named constraint constants from schema instead of using magic strings
import { 
  FK_SHORT_URLS_FOLDER, 
  FK_SHORT_URLS_DOMAIN_RECORD, 
  UNIQUE_SHORT_URLS_DOMAIN_PATH 
} from "@/shared/infrastructure/drizzle-postgres/schema";

export class PostgresShortUrlRepository implements ShortUrlRepository {
  private readonly db: NodePgDatabase;

  // MUST: Dependencies injected via constructor
  constructor(params: { db: NodePgDatabase }) {
    this.db = params.db;
  }

  // MUST: Execute using provided optional `tx` fallback to `this.db`
  async create({
    tx,
    shortUrlId,
    shortUrlName,
    path,
    targetUrl,
    domainRecordId,
    folderId,
    createdAt,
    updatedAt,
  }: ShortUrlRepositoryCreateParams & { tx?: Transaction }): Promise<
    Result<void, ShortUrlRepositoryCreateErrors>
  > {
    const executor = tx ?? this.db;

    try {
      await executor.insert(shortUrls).values({
        id: shortUrlId.getValue(),
        name: shortUrlName.getValue(),
        path: path.getValue(),
        targetUrl: targetUrl.getValue(),
        domainRecordId: domainRecordId?.getValue() ?? null,
        folderId: folderId.getValue(),
        createdAt,
        updatedAt,
      });

      return ok(undefined);
    } catch (error) {
      // MUST: Map infrastructural errors (PG node codes) to Business Domain errors
      // Postgres error codes: PG_ERROR_CODES.UNIQUE_VIOLATION, PG_ERROR_CODES.FOREIGN_KEY_VIOLATION
      // MUST: Catch errors using structural instanceof guards instead of checking arbitrary 'any' fields.
      if (error instanceof DrizzleQueryError && error.cause instanceof DatabaseError) {
        
        if (error.cause.code === PG_ERROR_CODES.UNIQUE_VIOLATION) {
          // MUST: Use constraint constants instead of raw magic strings
          if (error.cause.constraint === UNIQUE_SHORT_URLS_DOMAIN_PATH) {
            return err(
              new PathAlreadyInUseError({
                urlPath: path,
                domainRecordId,
              }),
            );
          }
        }

        if (error.cause.code === PG_ERROR_CODES.FOREIGN_KEY_VIOLATION) {
          // MUST: Use constraint constants instead of raw magic strings
          if (error.cause.constraint === FK_SHORT_URLS_FOLDER) {
            return err(new FolderNotFoundError({ folderId }));
          }
          if (
            error.cause.constraint === FK_SHORT_URLS_DOMAIN_RECORD && 
            domainRecordId
          ) {
            return err(new DomainRecordNotFoundError({ domainRecordId }));
          }
        }
      }

      return err(new RepositoryError(error as Error));
    }
  }

  // MUST: Apply isolated scope (e.g. searching only by the explicit folderId too)
  async exists({ folderId, shortUrlId }: { folderId: UUID; shortUrlId: UUID }): Promise<Result<boolean, RepositoryError>> {
    try {
      const result = await this.db.query.shortUrls.findFirst({
        where: (table, { and, eq }) => and(
          eq(table.id, shortUrlId.getValue()), 
          eq(table.folderId, folderId.getValue()) // Scope isolation applied at DB level
        ),
        columns: { id: true },
      });

      return ok(result !== undefined);
    } catch (error) {
      return err(new RepositoryError({ message: "Failed finding shorturl.", cause: error }));
    }
  }

  // MUST: Rehydrate into Entities. MUST handle corrupted DB data.
  async find({ folderId, shortUrlId }: { folderId: UUID; shortUrlId: UUID }): Promise<Result<ShortUrl | null, RepositoryError>> {
    try {
      const rawShortUrl = await this.db.query.shortUrls.findFirst({
        where: (table, { and, eq }) =>
          and(
            eq(table.id, shortUrlId.getValue()),
            eq(table.folderId, folderId.getValue())
          ),
      });

      if (!rawShortUrl) return ok(null);

      // Rehydrate Value Objects strictly
      const idResult = UUID.from(rawShortUrl.id);
      const nameResult = ShortUrlName.from(rawShortUrl.name);
      const pathResult = URLPath.from(rawShortUrl.path);
      const targetUrlResult = AppUrl.from(rawShortUrl.targetUrl);
      const dbFolderIdResult = UUID.from(rawShortUrl.folderId);
      const domainRecordIdResult = rawShortUrl.domainRecordId 
        ? UUID.from(rawShortUrl.domainRecordId) 
        : ok(undefined);

      // Validate data corruption. If any rule changed or data was corrupted, return a RepositoryError 
      // wrapped around a DataConsistencyError or similar instead of throwing.
      if (
        idResult.isErr() ||
        nameResult.isErr() ||
        pathResult.isErr() ||
        targetUrlResult.isErr() ||
        dbFolderIdResult.isErr() ||
        domainRecordIdResult.isErr()
      ) {
        return err(
          new RepositoryError({
            message: `CRITICAL: Corrupt data found in database for ShortUrl ID ${rawShortUrl.id}`,
            // Ideally attach all validation errors to troubleshoot
            cause: {
              id: idResult.error,
              name: nameResult.error,
              path: pathResult.error,
              targetUrl: targetUrlResult.error,
              folderId: dbFolderIdResult.error,
              domainRecordId: domainRecordIdResult.error,
            },
          })
        );
      }

      // Restore into Entity
      const entity = new ShortUrl({
        id: idResult.value,
        folderId: dbFolderIdResult.value,
        domainRecordId: domainRecordIdResult.value,
        name: nameResult.value,
        targetUrl: targetUrlResult.value,
        path: pathResult.value,
        createdAt: rawShortUrl.createdAt,
        updatedAt: rawShortUrl.updatedAt,
      });

      return ok(entity);
    } catch (error) {
      return err(new RepositoryError({ message: "Failed executing find query", cause: error }));
    }
  }

  // MUST: List paginates and applies scope. Rehydrates into Entities via map.
  async list({
    folderId,
    itemsPerPage,
    page,
  }: {
    folderId: UUID;
    itemsPerPage: PositiveInteger;
    page: PositiveInteger;
  }): Promise<Result<ShortUrl[], RepositoryError>> {
    try {
      const limit = itemsPerPage.getValue();
      const offset = (page.getValue() - 1) * limit;

      const rawShortUrls = await this.db.query.shortUrls.findMany({
        where: (table, { eq }) => eq(table.folderId, folderId.getValue()),
        limit,
        offset,
        orderBy: (table, { desc }) => desc(table.createdAt),
      });

      const entities: ShortUrl[] = [];

      for (const raw of rawShortUrls) {
        // Rehydrate Value Objects strictly
        const idResult = UUID.from(raw.id);
        const nameResult = ShortUrlName.from(raw.name);
        const pathResult = URLPath.from(raw.path);
        const targetUrlResult = AppUrl.from(raw.targetUrl);
        const dbFolderIdResult = UUID.from(raw.folderId);
        const domainRecordIdResult = raw.domainRecordId
          ? UUID.from(raw.domainRecordId)
          : ok(undefined);

        // If corrupted, halt or log. In strict DDD, one corrupted row halts the list 
        // to prevent partial, inconsistent data from propagating.
        if (
          idResult.isErr() ||
          nameResult.isErr() ||
          pathResult.isErr() ||
          targetUrlResult.isErr() ||
          dbFolderIdResult.isErr() ||
          domainRecordIdResult.isErr()
        ) {
          return err(
            new RepositoryError({
              message: `CRITICAL: Corrupt data in list result for ShortUrl ID ${raw.id}`,
            })
          );
        }

        entities.push(
          new ShortUrl({
            id: idResult.value,
            folderId: dbFolderIdResult.value,
            domainRecordId: domainRecordIdResult.value,
            name: nameResult.value,
            targetUrl: targetUrlResult.value,
            path: pathResult.value,
            createdAt: raw.createdAt,
            updatedAt: raw.updatedAt,
          })
        );
      }

      return ok(entities);
    } catch (error) {
      return err(new RepositoryError({ message: "Failed executing list query", cause: error }));
    }
  }

  // MUST: Support update via explicit scoped parameters and catch constraint errors
  async update({
    tx,
    shortUrlId,
    folderId,
    shortUrlName,
    path,
    targetUrl,
    domainRecordId,
    updatedAt,
  }: ShortUrlRepositoryUpdateParams & { tx?: Transaction }): Promise<
    Result<void, ShortUrlRepositoryUpdateErrors>
  > {
    const executor = tx ?? this.db;

    try {
      await executor
        .update(shortUrls)
        .set({
          name: shortUrlName.getValue(),
          path: path.getValue(),
          targetUrl: targetUrl.getValue(),
          domainRecordId: domainRecordId?.getValue() ?? null,
          updatedAt,
        })
        // MUST: Explicitly constrain by BOTH ID and parent scope (folderId)
        .where(
          and(
            eq(shortUrls.id, shortUrlId.getValue()),
            eq(shortUrls.folderId, folderId.getValue())
          )
        );

      return ok(undefined);
    } catch (error) {
      if (error instanceof DrizzleQueryError && error.cause instanceof DatabaseError) {
        if (error.cause.code === PG_ERROR_CODES.UNIQUE_VIOLATION) {
          if (error.cause.constraint === UNIQUE_SHORT_URLS_DOMAIN_PATH) {
            return err(
              new PathAlreadyInUseError({
                urlPath: path,
                domainRecordId,
              }),
            );
          }
        }

        if (error.cause.code === PG_ERROR_CODES.FOREIGN_KEY_VIOLATION) {
          if (
            error.cause.constraint === FK_SHORT_URLS_DOMAIN_RECORD && 
            domainRecordId
          ) {
            return err(new DomainRecordNotFoundError({ domainRecordId }));
          }
        }
      }

      return err(new RepositoryError(error as Error));
    }
  }

  // MUST: Include scope parent for hard deletion to prevent cross-tenant override
  async delete({
    tx,
    shortUrlId,
    folderId,
  }: ShortUrlRepositoryDeleteParams & { tx?: Transaction }): Promise<Result<void, RepositoryError>> {
    const executor = tx ?? this.db;

    try {
      await executor
        .delete(shortUrls)
        .where(
          and(
            eq(shortUrls.id, shortUrlId.getValue()),
            eq(shortUrls.folderId, folderId.getValue())
          )
        );

      return ok(undefined);
    } catch (error) {
      return err(new RepositoryError(error as Error));
    }
  }
}
```