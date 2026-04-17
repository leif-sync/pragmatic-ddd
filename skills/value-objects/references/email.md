```typescript
import { err, ok, Result } from "neverthrow";
import z from "zod";

declare const brandedEmailSymbol: unique symbol;
export type BrandedEmail = string & { readonly [brandedEmailSymbol]: never };

declare const unique: unique symbol;

export class InvalidEmailError extends Error {
  declare private readonly [unique]: never;

  private readonly invalidEmail: string;

  constructor(email: string) {
    super(`The email "${email}" is invalid.`);
    this.invalidEmail = email;
  }

  getInvalidEmail(): string {
    return this.invalidEmail;
  }
}

export class Email {
  private readonly value: string;

  private constructor(value: string) {
    this.value = value.trim();
  }

  static from(value: string): Result<Email, InvalidEmailError> {
    const valueTrimmed = value.trim();
    const isValidEmail = Email.checkValidity(valueTrimmed);
    if (!isValidEmail) return err(new InvalidEmailError(valueTrimmed));
    return ok(new Email(valueTrimmed));
  }

  static checkValidity(value: string): boolean {
    return z.email().safeParse(value).success;
  }

  toBranded(): BrandedEmail {
    return this.value as BrandedEmail;
  }

  static random(): Email {
    const randomEmail = `user.${Math.floor(Math.random() * 10000)}@example.com`;
    return new Email(randomEmail);
  }
}
```