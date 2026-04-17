# Client Component with i18n

This example shows how a Client Component utilizes `useLocale()` alongside `BrandedTypes` for its inputs.

```tsx
"use client";

import { useState, useTransition } from "react";
import { useRouter } from "next/navigation";
import { useLocale } from "@/shared/presentation/components/LocaleProvider";
import { Translations } from "@/shared/infrastructure/i18n/config";
import { type BrandedFolderId } from "@/folders/domain/value-objects/folderId";
import { deleteFolderAction } from "@/folders/presentation/actions/deleteFolderAction";
import { Button } from "@/shared/presentation/components/ui/button";

// Translations co-located as plain objects
const translations: Translations<{ deleteLabel: string; loading: string }> = {
  en: { deleteLabel: "Delete Folder", loading: "Deleting..." },
  es: { deleteLabel: "Eliminar Carpeta", loading: "Eliminando..." },
};

interface DeleteFolderButtonProps {
  // MUST receive a Branded Type natively, never a Value Object class instance
  folderId: BrandedFolderId;
}

export function DeleteFolderButton({ folderId }: DeleteFolderButtonProps) {
  // MUST use `useLocale()` instead of `await determineLocale()`
  const locale = useLocale();
  const t = translations[locale];
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  function handleDelete() {
    startTransition(async () => {
      // The branded type safely reaches the Server Action
      const result = await deleteFolderAction({ folderId });
      
      if (result.success) {
        // MUST refresh the router to update Server Components
        router.refresh(); 
      }
    });
  }

  return (
    <Button 
      variant="destructive" 
      onClick={handleDelete} 
      disabled={isPending}
    >
      {isPending ? t.loading : t.deleteLabel}
    </Button>
  );
}
```