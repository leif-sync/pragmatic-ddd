# Domain-Driven Next.js Redirects

The Next.js 16 Proxy is excellent for broad strokes like checking if a user has a session cookie or redirecting languages. However, the Proxy **MUST NOT** handle deeper Auth validation (like "Does this user own the workspace `wksp_123`?") because it cannot execute complex Domain Use Cases or Drizzle ORM queries effectively at the Edge.

Instead, you MUST resolve these complex authorization rules defensively in the Use Case executed by a **Server Component** (typically a Next.js `layout.tsx` or `page.tsx`).

### Example: Protecting a Workspace Layout

If the user tries to access a workspace they do not own or one that does not exist, the component intercepts the domain error and redirects accordingly.

```tsx
import { ReactNode } from "react";
import { redirect } from "next/navigation";
import { headers } from "next/headers";
import { auth } from "@/auth/auth";
import { serviceContainer } from "@/shared/infrastructure/bootstrap";
import { err, ok } from "neverthrow";
import { GetSession, SessionError } from "@/auth/getSession";
import { assertNever } from "@/shared/domain/utils/assertNever";
import { WorkspaceId } from "@/workspaces/domain/value-objects/workspaceId";
import { UnauthorizedWorkspaceAccess } from "@/workspaces/domain/errors/unauthorizedWorkspaceAccess";
import { WorkspaceNotFoundError } from "@/workspaces/domain/errors/workspaceNotFoundError";
import { RepositoryError } from "@/shared/domain/errors/repositoryError";
import { ErrorDisplay } from "@/shared/presentation/components/ErrorDisplay";

interface WorkspaceLayoutProps {
  params: Promise<{ workspaceId: string }>;
  children: ReactNode;
}

export default async function WorkspaceLayout({
  params,
  children,
}: WorkspaceLayoutProps) {
  const { workspaceId } = await params;
  const parsedWorkspaceId = WorkspaceId.from(workspaceId);

  // 1. Invalid ID formats redirect to a safe default path immediately
  if (parsedWorkspaceId.isErr()) {
    redirect("/dashboard");
  }

  // 2. Fetch the session inline
  const getSession: GetSession = async () => {
    const session = await auth.api.getSession({ headers: await headers() });
    if (!session?.user?.id) {
      return err(new SessionError());
    }
    return ok({ userId: session.user.id });
  };

  // 3. Delegate the complex auth/DB checks to the Domain Use Case
  const result = await serviceContainer.workspaces.checkWorkspaceAccess.execute({
    workspaceId: parsedWorkspaceId.value,
    getSession,
  });

  if (result.isErr()) {
    const error = result.error;

    // 4. Exhaustively map Domain Errors to the correct router actions
    if (
      error instanceof SessionError ||
      error instanceof UnauthorizedWorkspaceAccess ||
      error instanceof WorkspaceNotFoundError
    ) {
      redirect("/dashboard"); // Silent fallback or specific error page
    }

    if (error instanceof RepositoryError) {
      // MUST NOT use throws. Return a UI Error fallback
      return <ErrorDisplay title="Internal Error" description="Something went wrong." />;
    }

    // Forces TS to ensure all instances were checked
    assertNever(error);
  }

  // 5. User passed all domain checks, render the workspace layout
  return (
    <div className="workspace-container">
      {children}
    </div>
  );
}
```