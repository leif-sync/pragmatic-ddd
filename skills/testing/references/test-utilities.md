# Core Test Utilities

When bootstrapping a new project using our exact architecture, create these utility files within the Testing setup to integrate `vitest`, `testcontainers`, `node-postgres`, and `neverthrow`.

## 1. `vite.config.ts` Multi-Project Setup

We separate our tests natively in the Vite configuration.

### `vite.config.ts`
```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    coverage: { provider: "v8" },
    workspace: [
      {
        extends: true,
        test: {
          name: "unit",
          environment: "node",
          include: ["src/**/*.test.ts", "!src/**/*.integration.test.ts"],
        },
      },
      {
        extends: true,
        test: {
          name: "integration",
          environment: "node",
          include: ["src/**/*.integration.test.ts"],
          globalSetup: ["./testIntegrationSetup.ts"],
          fileParallelism: false, // Prevents deadlocks sequentially pushing transactions
        },
      },
      {
        extends: true,
        test: {
          name: "ui",
          environment: "happy-dom",
          include: ["src/**/*.test.tsx"],
        },
      },
    ],
  },
});
```

## 2. Docker DB Start: `testIntegrationSetup.ts`

Setups the ephemeral database instance before any integration test runs using `testcontainers`.

### `testIntegrationSetup.ts`
```typescript
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { exec } from "node:child_process";
import { promisify } from "node:util";

const execAsync = promisify(exec);

export default async function () {
  console.log("Starting PostgreSQL Testcontainer...");
  const container = await new PostgreSqlContainer("postgres:16-alpine").start();

  const connectionUri = container.getConnectionUri();
  console.log("Testcontainer started.");
  process.env.DATABASE_URL = connectionUri; // Injected instantly for everything matching

  console.log("Running Drizzle Migrations...");
  try {
    await execAsync("npx drizzle-kit migrate", {
      env: { ...process.env, DATABASE_URL: connectionUri },
    });
    console.log("Migrations complete.");
  } catch (error) {
    console.error("Migration failed:", error);
    throw error;
  }

  return async () => {
    console.log("Stopping Testcontainer...");
    await container.stop();
  };
}
```

## 3. Transactional Testing (`txTest`)

Wraps tests in a real Postgres transaction (`drizzle-orm`) that is guaranteed to rollback, keeping the database perfectly isolated for each test. It intercepts Vitest's `test` runner rather than running inside it.

### `src/shared/test/integrationUtils.ts`

```typescript
import { TransactionRollbackError } from "drizzle-orm";
import { drizzle, NodePgDatabase } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import { inject, test } from "vitest";

const testContainerPostgresUrl = inject("testContainerPostgresUrl");

const pool = new Pool({
  connectionString: testContainerPostgresUrl,
  max: 1,
});

const connection = drizzle(pool);

export function txTest(
  testName: string,
  testFn: (tx: NodePgDatabase) => Promise<void>,
) {
  test(testName, async () => {
    await connection
      .transaction(async (tx) => {
        await testFn(tx);
        tx.rollback();
      })
      .catch((error) => {
        if (error instanceof TransactionRollbackError) return;
        throw error;
      });
  });
}
```

## 4. `assertNever` assertions

### `src/shared/test/utils.ts`
```typescript
import { isErr, isOk } from "neverthrow";
import assert from "node:assert";

export function assertOk<T, E>(
  result: import("neverthrow").Result<T, E>,
  message?: string | Error,
): asserts result is import("neverthrow").Ok<T, E> {
  if (isErr(result)) {
    throw new Error(`Expected Ok, but got Err: ${result.error}`);
  }
  assert(isOk(result), message);
}

export function assertErr<T, E>(
  result: import("neverthrow").Result<T, E>,
  message?: string | Error,
): asserts result is import("neverthrow").Err<T, E> {
  if (isOk(result)) {
    throw new Error(`Expected Err, but got Ok: ${result.value}`);
  }
  assert(isErr(result), message);
}
```