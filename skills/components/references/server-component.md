# Server Component (No i18n)

Server Components without localization still follow the same rules regarding Value Objects. Instead of importing `determineLocale`, they simply render the component.

```tsx
import { auth } from "@/auth/auth";
import { headers } from "next/headers";
import { GetSession, SessionError } from "@/auth/getSession";
import { FolderId } from "@/folders/domain/value-objects/folderId";
import { UUID } from "@/shared/domain/value-objects/uuid";
import { err, ok } from "neverthrow";
import { serviceContainer } from "@/shared/infrastructure/bootstrap";

interface FolderDetailsProps {
  // Server Components MAY receive Value Objects
  folderId: FolderId;
}

export async function FolderDetails({ folderId }: FolderDetailsProps) {
  // Inline getSession closure for the use case
  const getSession: GetSession = async () => {
    const session = await auth.api.getSession({ headers: await headers() });
    const resultUserId = UUID.from(session?.user?.id ?? "");

    if (resultUserId.isErr()) {
      return err(new SessionError());
    }
    return ok({ userId: resultUserId.value });
  };

  const result = await serviceContainer.folders.getFolder.execute({
    folderId,
    getSession, 
  });
  
  if (result.isErr()) return <div>Error loading folder details</div>;

  const folder = result.value;

  return (
    <div>
      {/* Entity properties are exposed through Voice Object branding */}
      <h2>Folder: {folder.getName().toBranded()}</h2>
      <p>Created exactly on: {folder.getCreatedAt().toLocaleDateString()}</p>
    </div>
  );
}
```