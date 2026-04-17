# Client Component (No i18n)

Client Components CANNOT receive Value Objects as class instances in their props. They MUST receive `Branded Types` (e.g. `BrandedFolderId`), which are serialized as primitives (e.g. `string` or `number`).

```tsx
"use client";

import { useState, useTransition } from "react";
import { useRouter } from "next/navigation";
import { type BrandedFolderId } from "@/folders/domain/value-objects/folderId";
import { deleteFolderAction } from "@/folders/presentation/actions/deleteFolderAction";
import { Button } from "@/shared/presentation/components/ui/button";

interface DeleteFolderButtonProps {
  // MUST receive a Branded Type natively, never a Value Object class instance
  folderId: BrandedFolderId; 
}

export function DeleteFolderButton({ folderId }: DeleteFolderButtonProps) {
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
      {isPending ? "Deleting..." : "Delete folder"}
    </Button>
  );
}
```