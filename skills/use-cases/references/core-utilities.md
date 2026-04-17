# Core Use Case Utilities

When bootstrapping a new project using our exact Next.js/Drizzle standard, you must create these abstractions in the `src/shared/` directory to make `use-cases` and `repositories` compile exactly as expected.

## 1. The Drizzle postgres transaction wrapper (WithTransaction)

The domain must orchestrate multi-step rollbacks perfectly typed for `drizzle-orm/pg-core` and `node-postgres`. 

### `src/shared/domain/types/withTransaction.ts`
```typescript
import { NodePgQueryResultHKT } from "drizzle-orm/node-postgres";
import { PgTransaction } from "drizzle-orm/pg-core";
import { ExtractTablesWithRelations } from "drizzle-orm";

// Exact Transaction type matching node-postgres and empty schema relation extraction
export type Transaction = PgTransaction<
  NodePgQueryResultHKT,
  Record<string, never>,
  ExtractTablesWithRelations<Record<string, never>>
>;

export type WithTransaction = <T>(
  cb: (tx: Transaction) => Promise<T>,
) => Promise<T>;
```

## 2. Collision Resistor: withRetry

Tasks creating UUIDs or random paths often fail with Postgres unique constraints. `withRetry` acts as a fail-safe using `neverthrow`.

### `src/shared/domain/utils/withRetry.ts`
```typescript
import { Result, ok, err } from "neverthrow";
import { NonNegativeInteger } from "../value-objects/nonNegativeInteger";
import { PositiveInteger } from "../value-objects/positiveInteger";

declare const unique1: unique symbol;

export class MaxRetriesReachedError extends Error {
  declare private readonly [unique1]: never;

  constructor(props: { message?: string } = {}) {
    super(props.message ?? "Maximum number of retries reached");
  }
}

declare const unique2: unique symbol;

class RetrySignal {
  declare private readonly [unique2]: never;
}

export async function withRetry<T, E>(
  options: {
    maxRetries: PositiveInteger;
    delayMilliseconds?: NonNegativeInteger;
  },
  task: (helpers: { retry: () => never }) => Promise<Result<T, E> | void>,
): Promise<
  Result<
    unknown extends T | never ? void : T,
    unknown extends E | never ? never : E | MaxRetriesReachedError
  >
> {
  const maxAttempts = options.maxRetries.getValue();
  const delayMs = options.delayMilliseconds?.getValue() ?? 0;
  const exhaustionError = new MaxRetriesReachedError();

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const isLastAttempt = attempt === maxAttempts;
    const retry = () => {
      throw isLastAttempt ? exhaustionError : new RetrySignal();
    };

    try {
      const result = await task({ retry });

      if (result === void 0) {
        return ok(void 0 as unknown extends T | never ? void : T);
      }

      return result as Result<
        unknown extends T | never ? void : T,
        unknown extends E | never ? never : E | MaxRetriesReachedError
      >;
    } catch (error) {
      if (error instanceof RetrySignal) {
        if (delayMs > 0) {
          await new Promise((resolve) => setTimeout(resolve, delayMs));
        }
        continue;
      }

      if (error instanceof MaxRetriesReachedError) {
        return err(
          error as unknown extends E | never
            ? never
            : E | MaxRetriesReachedError,
        );
      }

      throw error;
    }
  }

  return err(
    exhaustionError as unknown extends E | never
      ? never
      : E | MaxRetriesReachedError,
  );
}
```

## 3. Exhaustiveness Checking: assertNever

Used to ensure `switch` statements or conditional blocks exhaustively handle every typing state. Extremely useful for mapped errors in `Server Actions`.

### `src/shared/domain/utils/assertNever.ts`
```typescript
export function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${x}`);
}
```

## 4. Development Markers: todo

Helper object for handling NotImplemented errors gracefully or panic aborts without standardizing unstructured `throw`.

### `src/shared/domain/utils/todo.ts`
```typescript
import { styleText } from "node:util";

declare const unique: unique symbol;

class TODO extends Error {
  declare private readonly [unique]: never;

  constructor(message: string) {
    super(message);
  }
}
const getCallerInfo = () => {
  const stack = new Error().stack;
  if (!stack) return "unknown location";

  // [0]: Error
  // [1]: at getCallerInfo (path/to/file:line:column)
  const todoMethodIndex = 2;
  const todoCallerIndex = todoMethodIndex + 1;
  const lines = stack.split("\n");
  const callerLine = lines[todoCallerIndex] || lines[todoMethodIndex];

  return callerLine.trim().replace("at ", "");
};

export const todo = {
  panic(msg: string): never {
    throw new TODO(styleText("red", `🔥 [PANIC]: ${msg}`));
  },

  unimplemented(msg: string): never {
    throw new TODO(styleText("red", `🛠️ [UNIMPLEMENTED]: ${msg}`));
  },

  warn(msg: string): void {
    const loc = getCallerInfo();
    console.warn(
      styleText("yellow", `⚠️ [TODO WARN] at ${loc}\nMessage: ${msg}`),
    );
  },

  fixme(msg: string): void {
    const loc = getCallerInfo();
    console.info(styleText("yellow", `🧹 [FIXME] at ${loc}\nMessage: ${msg}`));
  },
};
```