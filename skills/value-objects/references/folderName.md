```typescript
import { Result, ok, err } from "neverthrow";

declare const brandedFolderNameSymbol: unique symbol;
export type BrandedFolderName = string & { readonly [brandedFolderNameSymbol]: never };

declare const unique: unique symbol;

export class InvalidFolderNameError extends Error {
  declare private readonly [unique]: never;

  private readonly invalidName: string;

  constructor(invalidName: string) {
    super(
      `Invalid folder name: "${invalidName}". Folder name must be between ${FolderName.MIN_LENGTH} and ${FolderName.MAX_LENGTH} characters long.`,
    );
    this.invalidName = invalidName;
  }

  getInvalidName(): string {
    return this.invalidName;
  }
}

export class FolderName {
  static readonly MAX_LENGTH = 30;
  static readonly MIN_LENGTH = 1;

  private readonly value: string;

  private constructor(value: string) {
    this.value = value.trim();
  }

  static random(): FolderName {
    const value = `folder-name-${Math.floor(Math.random() * 10000)}`;
    return new FolderName(value);
  }

  static checkValidity(value: string): boolean {
    const trimmedValue = value.trim();
    return (
      trimmedValue.length >= FolderName.MIN_LENGTH &&
      trimmedValue.length <= FolderName.MAX_LENGTH
    );
  }

  static from(value: string): Result<FolderName, InvalidFolderNameError> {
    const trimmedValue = value.trim();

    if (!this.checkValidity(trimmedValue)) {
      return err(new InvalidFolderNameError(trimmedValue));
    }
    return ok(new FolderName(trimmedValue));
  }

  toBranded(): BrandedFolderName {
    return this.value as BrandedFolderName;
  }
}
```