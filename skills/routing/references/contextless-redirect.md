# Contextless Route Redirection (The `redirect-without-[id]` Pattern)

When building applications with a multi-tenant or workspace-based architecture, some URLs logically represent an "entry point" but theoretically lack the necessary ID or routing segment to display relevant data.

For example, `/dashboard` is a valid entry point, but if the app operates strictly under a specific `/dashboard/workspaces/[workspaceId]/short-urls/[folderId]`, rendering `/dashboard` without a workspace and folder context is an anti-pattern. You **MUST NOT** render a generic UI if a seamless default context is identifiable.

Instead, the `app/dashboard/page.tsx` file acts **exclusively as a programmatic redirect gateway** by rendering a component specifically designed to resolve the missing context.

### Example: Cascading Context Resolution

This pattern resolves the missing context step-by-step. 

**CRITICAL RULE: The Most General View Principle**
You MUST redirect the user to the "most general" available view that makes sense for the resolved context. 
- If the project has a general workspace overview page, redirect to `/dashboard/[workspaceId]`.
- If the minimum general view is a list of folders, redirect to `/dashboard/[workspaceId]/folders`.
- Do not force a deep hierarchy if a higher-level generic overview exists and provides value.

The following example demonstrates a deep cascade (Workspace -> Folder) because this specific project requires a folder to show links, but in many projects, simply finding the `workspaceId` and stopping there is the correct approach.

`src/app/dashboard/redirect-without-workspaceId-folderId.tsx`
```tsx
import { redirect } from "next/navigation";
import { serviceContainer } from "@/shared/infrastructure/bootstrap";
import { PositiveInteger } from "@/shared/domain/value-objects/positiveInteger";
import { auth } from "@/auth/auth";
import { headers } from "next/headers";
import { UUID } from "@/shared/domain/value-objects/uuid";
import { GetSession, SessionError } from "@/auth/getSession";
import { err, ok } from "neverthrow";
import { determineLocale } from "@/shared/infrastructure/i18n/utils";
import { ErrorDisplay } from "@/shared/presentation/components/ErrorDisplay";
// Example domain error:
import { UserNotFoundError } from "@/users/domain/errors/userNotFoundError"; 

export async function RedirectWithoutWorkspaceIdFolderId() {
  const getSession: GetSession = async () => {
    const session = await auth.api.getSession({ headers: await headers() });
    if (!session) return err(new SessionError());

    const userIdResult = UUID.from(session.user.id);
    if (userIdResult.isErr()) return err(new SessionError());

    return ok({ userId: userIdResult.value });
  };

  // 1. Try to find the user's first workspace
  const listWorkspacesResult = await serviceContainer.workspaces.listWorkspaces.execute({
    getSession,
    itemsPerPage: PositiveInteger.one(),
    page: PositiveInteger.one(),
  });

  if (listWorkspacesResult.isErr()) {
    const error = listWorkspacesResult.error;
    if (error instanceof UserNotFoundError) {
      const locale = await determineLocale();
      // Redirect to public login with the correct locale
      redirect(`/${locale}/login`);
    }
    
    // MUST NOT use throws. Instead, fall back to an Error display securely.
    // Exhaustive mapping with assertNever could also be placed here.
    return <ErrorDisplay title="Error" description="Failed to load workspaces." />;
  }

  const workspaces = listWorkspacesResult.value;
  
  // 2. If no workspaces exist, send them to the creation flow
  if (workspaces.length === 0) {
    redirect("/dashboard/workspaces/new");
  }

  const workspace = workspaces[0];
  const workspaceId = workspace.getId();

  // 3. Try to find the first folder inside that workspace
  const listFoldersResult = await serviceContainer.folders.listFolders.execute({
    getSession,
    workspaceId,
    itemsPerPage: PositiveInteger.one(),
    page: PositiveInteger.one(),
  });

  if (listFoldersResult.isErr()) {
    // MUST NOT use throws. Return a UI component or map Exhaustively via assertNever
    return <ErrorDisplay title="Error" description="Failed to load folders." />;
  }

  const folders = listFoldersResult.value;
  
  // 4. If no folders exist, redirect to the workspace overview (or folder creation)
  if (folders.length === 0) {
    // MUST use `.toBranded()` for routing bounds
    redirect(`/dashboard/workspaces/${workspaceId.toBranded()}`);
  }

  const folder = folders[0];
  const folderId = folder.getId();

  // 5. Successful identification: Immediately transition to the most general valid contextual URL.
  // In this project, a folder is required to view links, so we go deep.
  // In other projects, `redirect(\`/dashboard/${workspaceId.toBranded()}\`)` is often enough.
  // MUST use `.toBranded()` to serialize the parameters securely to the URL string.
  redirect(`/dashboard/workspaces/${workspaceId.toBranded()}/short-urls/${folderId.toBranded()}`);
}
```

`src/app/dashboard/page.tsx`
```tsx
import { RedirectWithoutWorkspaceIdFolderId } from "./redirect-without-workspaceId-folderId";

export default async function DashboardPage() {
  return <RedirectWithoutWorkspaceIdFolderId />;
}
```