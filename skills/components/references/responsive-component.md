# Logic-Based Responsive Separation (Mobile & Desktop)

If a component behaves and renders differently on a Mobile screen vs a Desktop screen (e.g. `<Mobile> vs <Desktop>`), you SHALL separate this logic into distinct rendering sub-components IN THE SAME FILE if the underlying functionality is the same. Note this is specifically for **logic or structural differences** driven by viewport size, **not simple CSS media query styling**.

```tsx
"use client";

import { useState } from "react"
import { useIsDesktop } from "@/shared/presentation/hooks/useIsDesktop";
import { Dialog, DialogContent, DialogTrigger } from "@/shared/presentation/components/ui/dialog"
import { Drawer, DrawerContent, DialogTrigger as DrawerTrigger } from "@/shared/presentation/components/ui/drawer"

interface UserProfileModalProps {
  userId: BrandedUserId;
  triggerText: string; // No branded type because a trigget text not have a specific value object, it's just a string
}

// 1. The wrapper component orchestrating the separation logic
export function UserProfileModal({ userId, triggerText }: UserProfileModalProps) {
  // Common state shared across both mobile and desktop
  const [open, setOpen] = useState(false)
  // Hook checking viewport logic
  const isDesktop = useIsDesktop() 

  // 2. Structural/UX difference: Desktop users receive a Dialog (Center)
  if (isDesktop) {
    return (
      <Dialog open={open} onOpenChange={setOpen}>
        <DialogTrigger asChild>
          <button>{triggerText}</button>
        </DialogTrigger>
        <DialogContent className="sm:max-w-[425px]">
          <UserProfileDesktop userId={userId} setOpen={setOpen} />
        </DialogContent>
      </Dialog>
    )
  }

  // 3. Structural/UX difference: Mobile users receive a Drawer (Bottom swipe)
  return (
    <Drawer open={open} onOpenChange={setOpen}>
      <DrawerTrigger asChild>
        <button>{triggerText}</button>
      </DrawerTrigger>
      <DrawerContent>
        <UserProfileMobile userId={userId} setOpen={setOpen} />
      </DrawerContent>
    </Drawer>
  )
}

// Sub-component tailored to Desktop interactions or layout expectations.
function UserProfileDesktop({ userId, setOpen }: { userId: string, setOpen: (open: boolean) => void }) {
  return (
    <form className="grid items-start gap-4">
      {/* Desktop-specific layout, grid flows, interactions */}
      <UserProfileForm userId={userId} onSaved={() => setOpen(false)} />
    </form>
  )
}

// Sub-component tailored to Mobile interactions or layout expectations.
function UserProfileMobile({ userId, setOpen }: { userId: string, setOpen: (open: boolean) => void }) {
  return (
    <form className="px-4 pb-6 pt-0 flex flex-col gap-2">
      {/* Mobile-specific layout, stacked flows, larger touch targets */}
      <UserProfileForm userId={userId} onSaved={() => setOpen(false)} />
    </form>
  )
}

// A completely shared component for the interior common functionality (like the exact form fields)
function UserProfileForm({ userId, onSaved }: { userId: string, onSaved: () => void }) {
  return (
    <div>
      <input type="text" placeholder="Username" />
      <button onClick={onSaved}>Save</button>
    </div>
  )
}
```