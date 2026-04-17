```typescript
import { UUID } from "@/shared/domain/value-objects/uuid";

declare const unique: unique symbol;

export class WorkspaceNotFoundError extends Error {
  declare private readonly [unique]: never;

  private readonly workspaceId: UUID;

  constructor(params: { workspaceId: UUID }) {
    // MUST use .toBranded() or explicitly extraction methods for the message
    super(`Workspace with ID "${params.workspaceId.toBranded()}" not found.`);
    this.workspaceId = params.workspaceId;
  }

  getWorkspaceId(): UUID {
    return this.workspaceId;
  }
}
```