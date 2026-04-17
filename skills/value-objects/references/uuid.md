```typescript 
import { err, ok, Result } from "neverthrow";
import { v7 as uuidV7, validate } from "uuid";

declare const brandedUUIDSymbol: unique symbol;
export type BrandedUUID = string & { readonly [brandedUUIDSymbol]: never };

declare const unique: unique symbol;

export class InvalidUUIDError extends Error {
  declare private readonly [unique]: never;

  private readonly invalidValue: string;

  constructor(invalidValue: string) {
    super(`The UUID value "${invalidValue}" is not valid.`);
    this.invalidValue = invalidValue;
  }

  getInvalidUUID(): string {
    return this.invalidValue;
  }
}

// this should be immutable
export class UUID {
  private readonly value: string;

  private constructor(value: string) {
    this.value = value.trim();
  }

  /**
   * Generates a new UUID instance with a random UUIDv7 value.
   */
  static random(): UUID {
    return new UUID(uuidV7());
  }

  static checkValidity(value: string): boolean {
    return validate(value);
  }

  static from(value: string): Result<UUID, InvalidUUIDError> {
    const valueTrimmed = value.trim();
    if (!UUID.checkValidity(valueTrimmed)) {
      return err(new InvalidUUIDError(valueTrimmed));
    }
    return ok(new UUID(valueTrimmed));
  }

  toBranded(): BrandedUUID {
    return this.value as BrandedUUID;
  }
}
```