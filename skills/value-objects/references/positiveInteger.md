```typescript
import { err, ok, Result } from "neverthrow";

declare const uniqueBrand: unique symbol;

export type BrandedPositiveInteger = number & {
  readonly [uniqueBrand]: never;
};

declare const unique: unique symbol;

export class InvalidPositiveIntegerError extends Error {
  declare private readonly [unique]: never;

  private readonly invalidValue: number;

  constructor(value: number) {
    super(
      `The value ${value} is not a valid positive integer, it must be an integer greater than zero.`,
    );
    this.invalidValue = value;
  }

  getInvalidValue(): number {
    return this.invalidValue;
  }
}

export class PositiveInteger {
  private readonly value: number;

  private constructor(value: number) {
    this.value = value;
  }

  static random(): PositiveInteger {
    const randomValue =
      Math.floor(Math.random() * (Number.MAX_SAFE_INTEGER - 1)) + 1;
    return PositiveInteger.from(randomValue)._unsafeUnwrap({
      withStackTrace: true,
    });
  }

  static from(
    value: string,
  ): Result<PositiveInteger, InvalidPositiveIntegerError>;

  static from(
    value: number,
  ): Result<PositiveInteger, InvalidPositiveIntegerError>;

  static from(
    value: number | string,
  ): Result<PositiveInteger, InvalidPositiveIntegerError> {
    if (typeof value === "string") {
      const parsedValue = Number(value);
      if (Number.isNaN(parsedValue)) {
        return err(new InvalidPositiveIntegerError(NaN));
      }
      value = parsedValue;
    }

    const isInteger = Number.isInteger(value);

    if (!isInteger || value <= 0) {
      return err(new InvalidPositiveIntegerError(value));
    }

    return ok(new PositiveInteger(value));
  }

  isGreaterThan(other: PositiveInteger): boolean {
    return this.value > other.value;
  }

  isLessThan(other: PositiveInteger): boolean {
    return this.value < other.value;
  }

  max(other: PositiveInteger): PositiveInteger {
    return this.isGreaterThan(other) ? this : other;
  }

  min(other: PositiveInteger): PositiveInteger {
    return this.isLessThan(other) ? this : other;
  }

  add(other: PositiveInteger): PositiveInteger {
    return PositiveInteger.from(this.value + other.value)._unsafeUnwrap({
      withStackTrace: true,
    });
  }

  subtract(
    other: PositiveInteger,
  ): Result<PositiveInteger, InvalidPositiveIntegerError> {
    return PositiveInteger.from(this.value - other.value);
  }

  static one(): PositiveInteger {
    return PositiveInteger.from(1)._unsafeUnwrap({ withStackTrace: true });
  }

  toBranded(): BrandedPositiveInteger {
    return this.value as BrandedPositiveInteger;
  }
}
```