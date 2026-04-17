# Server Component with i18n

Server Components can explicitly receive and use Value Objects in their parameters and body since they never cross the network boundary as serialized JSON.

```tsx
import { auth } from "@/auth/auth";
import { headers } from "next/headers";
import { GetSession, SessionError } from "@/auth/getSession";
import { WorkspaceId } from "@/workspaces/domain/value-objects/workspaceId";
import { UUID } from "@/shared/domain/value-objects/uuid";
import { err, ok } from "neverthrow";
import { determineLocale } from "@/shared/infrastructure/i18n/utils";
import { serviceContainer } from "@/shared/infrastructure/bootstrap";
import { Translations } from "@/shared/infrastructure/i18n/config";

// Translations co-located with the component
const translations: Translations<{ title: string; empty: string }> = {
  en: { title: "My Folders", empty: "No folders found." },
  es: { title: "Mis Carpetas", empty: "No se encontraron carpetas." },
};

interface FolderListProps {
  // Server Components MAY receive Value Objects as parameters
  workspaceId: WorkspaceId;
}

export async function FolderList({ workspaceId }: FolderListProps) {
  // MUST use determineLocale in Server Components
  const locale = await determineLocale();
  const t = translations[locale];

  // Inline getSession closure for the use case
  const getSession: GetSession = async () => {
    const session = await auth.api.getSession({ headers: await headers() });
    const resultUserId = UUID.from(session?.user?.id ?? "");

    if (resultUserId.isErr()) {
      return err(new SessionError());
    }
    return ok({ userId: resultUserId.value });
  };

  const result = await serviceContainer.folders.listFolders.execute({
    workspaceId,
    getSession, 
  });
  
  if (result.isErr()) return <div>Error loading folders</div>;

  const folders = result.value;

  if (folders.length === 0) return <p>{t.empty}</p>;

  return (
    <div>
      <h2>{t.title}</h2>
      <ul>
        {folders.map((folder) => (
          // MUST use `.toBranded()` to serialize Entity properties to the DOM
          <li key={folder.getId().toBranded()}>
            {folder.getName().toBranded()}
          </li>
        ))}
      </ul>
    </div>
  );
}
```