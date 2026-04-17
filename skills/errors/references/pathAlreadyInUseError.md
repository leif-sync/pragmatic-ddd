```typescript
import { UUID } from "@/shared/domain/value-objects/uuid";
import { URLPath } from "../value-objects/urlPath";

declare const unique: unique symbol;

export class PathAlreadyInUseError extends Error {
  declare private readonly [unique]: never;

  private readonly urlPath: URLPath;
  private readonly domainRecordId?: UUID;

  constructor(params: { urlPath: URLPath; domainRecordId?: UUID }) {
    // MUST use explicit extraction methods on the Value Objects
    const message = params.domainRecordId
      ? `The URL path "${params.urlPath.getEncoded()}" is already in use for domain record ID "${params.domainRecordId.toBranded()}".`
      : `The URL path "${params.urlPath.getEncoded()}" is already in use.`;

    super(message);
    this.urlPath = params.urlPath;
    this.domainRecordId = params.domainRecordId;
  }

  getUrlPath(): URLPath {
    return this.urlPath;
  }

  getDomainRecordId(): UUID | undefined {
    return this.domainRecordId;
  }
}
```