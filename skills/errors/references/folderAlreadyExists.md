```typescript
import { FolderName } from "../value-objects/folderName";

declare const unique: unique symbol;

export class FolderAlreadyExists extends Error {
  declare private readonly [unique]: never;
  private readonly folderName: FolderName;

  constructor(params: { folderName: FolderName }) {
    super(`Folder with name "${params.folderName.toBranded()}" already exists`);

    this.folderName = params.folderName;
  }

  getFolderName(): FolderName {
    return this.folderName;
  }
}
```