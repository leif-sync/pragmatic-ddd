```typescript
declare const unique: unique symbol;

export class RepositoryError extends Error {
  declare private readonly [unique]: never;

  constructor(params: Error) {
    const { message, ...rest } = params;
    super(message, rest);
  }
}
```