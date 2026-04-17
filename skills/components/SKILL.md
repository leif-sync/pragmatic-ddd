---
name: components
description: "[Pragmatic DDD Architecture] Guide for Server and Client components in Next.js App Router. Use when creating any .tsx file under presentation/components/, pages, or layouts — also when deciding whether to add \"use client\" to an existing component, passing data from a Server Component to a Client Component, composing Server content inside a Client slot, handling the VO serialization boundary, creating Compound Components, separating logic for Mobile/Desktop screens, or styling with `cva` and `cn`. Covers: Server vs Client decision, async Server Component patterns, creating getSession callbacks for Use Cases, Client Component restrictions, toBranded() boundary pattern, children slot composition, and props interface rules. Depends on 'use-cases' and 'server-actions'."
---

# Components (Server & Client)

> **IMPORTANT:** This skill defines architecture, rules, structure, component capabilities, constraints, and Value Object usage. It **DOES NOT** explain or restrict visual design or styling logic (e.g. CSS layout choices). You MAY use component libraries like `shadcn/ui` or similar inside your components.

## Server vs Client — Decision Rule

If the component needs interactivity (hooks, events, browser APIs) → `"use client"`.  
If not → **Server Component** (no directive needed).

The key project-specific distinction is what each type can **access**:

| Capability | Server Component | Client Component |
| --- | --- | --- |
| `serviceContainer` (use cases) | ✅ | ❌ |
| Value Object instantiation (`new`, `.from()`) | ✅ | ✅ |
| Receive Class Instances as parameters | ✅ | ❌ (MUST use `.toBranded()`) |
| `auth.api.getSession` + `headers()` | ✅ | ❌ |
| `await determineLocale()` | ✅ | ❌ — MUST use `useLocale()` |
| `useLocale()`, `useRouter()`, other hooks | ❌ | ✅ |
| Server-only modules (drizzle, `load-env`) | ✅ | ❌ |
| `async` function | ✅ | ❌ |

> Default to Server Components. Push the `"use client"` boundary as low in the tree as possible.

---

## File & Naming Conventions

- Name: `PascalCase` matching the exported function — `LinksTable.tsx` → `LinksTable`
- `"use client"` MUST be the **absolute first line** of the file when present — before any imports.
- One primary component per file. Small co-located sub-components used only in that file are acceptable.

---

## 1. Value Objects in Components

### Server Components
Server Components CAN explicitly receive, instantiate, and use Value Objects natively in their parameters and body. Since they never cross the network boundary as JSON payloads, they aren't subject to serialization limits.

### Client Components
Client Components CANNOT receive Value Objects as class instances via parameters from a Server Component. Next.js RSC serialization fails on classes. Client Components MUST receive primitive **Branded Types** via props.
However, a Client Component MAY instantiate or manipulate its own Value Objects internally (e.g., during form validation before submitting to a Server Action).

---

## 2. Shared Library Components (`shadcn` style)

### Common Components (`cva` & `cn`)
When building common, reusable UI components (e.g., Buttons, Badges, Inputs) that appear in multiple contexts and possess differing styles but identical underlying functionality, you MUST manage these styling variants using the `cva` (class-variance-authority) library and the `cn` utility. This directly mirrors the `shadcn/ui` architectural style.

### Simple Flat Composition
When designing complex reusable UI components (e.g., Table, DropdownMenu, Select), keep them as **independent, flat components** (e.g., `Table`, `TableRow`, `TableBody`, `TableCell`). You MUST NOT use the dot-notation Compound Component pattern (`Table.Row`) for now to avoid unnecessary complexity. Each part should simply be imported and used as a distinct component.

---

## 3. Logic-Based Responsive Separation

If a component requires completely different React structural elements or rendering logic for Mobile screens versus Desktop screens (e.g., a Mobile user gets a swipe-up `<Drawer>`, and a Desktop user gets a centered `<Dialog>`), you SHALL separate this into multiple distinct local components within the same file.

For example, you MUST structure your file with:
1. `function ContentList()` (The wrapper that measures `useIsMobile()`)
2. `function ContentListMobile()`
3. `function ContentListDesktop()`

This separation MUST be based on structural or lifecycle logic (`{isMobile ? <Mobile> : <Desktop>}`), NOT merely on visual styling differences which belong in CSS (Tailwind classes).

---

## 4. The Server → Client Serialization Boundary

Next.js serializes props when passing them from a Server Component to a Client Component (RSC payload). **Class instances are not serializable** — this includes all Value Object classes. This is a **transport constraint**, not a general restriction: a Client Component can instantiate or use VOs internally just fine.

### The `toBranded()` pattern

VOs expose a `.toBranded()` method that returns a **branded primitive** (a plain `number` or `string` tagged with a unique TypeScript symbol). It solves two problems at once:

1. **Serializability** — a branded primitive crosses the RSC boundary without issues.
2. **Validation guarantee** — because `toBranded()` can only be called on an already-constructed VO, the receiving Client Component knows the value has already been validated. There is no need to call `.from()` again or handle `Result` errors — the branded type itself is the proof that the value is valid.

```typescript
// In the VO definition
declare const uniqueBrand: unique symbol;
export type BrandedFolderId = string & { readonly [uniqueBrand]: never };

export class FolderId {
  // ...
  toBranded(): BrandedFolderId {
    return this.value as BrandedFolderId;
    // Caller receives a plain string at runtime,
    // but TypeScript knows it came from a valid FolderId.
  }
}
```

**Server Component** — calls `.toBranded()` before passing the prop:

```tsx
// Server Component
import { FolderItem } from "./FolderItem"; // client component

export async function FolderList() {
  const folders = /* from use case */;

  return folders.map((folder) => (
    <FolderItem
      key={folder.getId().toBranded()}
      folderId={folder.getId().toBranded()} // ✅ branded primitive
      name={folder.getName().toBranded()}    // ✅ plain string
    />
  ));
}
```

**Client Component** — receives and uses the branded primitive:

```tsx
"use client";

import { type BrandedFolderId } from "@/folders/domain/value-objects/folderId";

interface FolderItemProps {
  folderId: BrandedFolderId; // branded primitive — valid by construction, no need to re-validate
  name: string;
}

export function FolderItem({ folderId, name }: FolderItemProps) {
  // folderId is a string at runtime, branded for type safety.
  // Pass it directly to a server action — no .from() or error handling needed.
  // The server action receives it as a string and reconstructs the VO internally.
}
```

### What can cross the boundary

| Type | Serializable | How |
| --- | --- | --- |
| `string`, `number`, `boolean`, `null` | ✅ | Directly |
| Plain objects `{ key: value }` | ✅ | Directly (no class methods) |
| Arrays of the above | ✅ | Directly |
| `Date` | ✅ | Passed as-is (Next.js handles it) |
| VO class instance | ❌ | Use `.toBranded()` |
| Functions / callbacks | ❌ | Not serializable — use Server Actions instead |
| `undefined` | ⚠️ | Omit the prop or use `null` |

---

## 4. Composition Patterns

### Server renders Client (standard)

A Server Component can render a Client Component and pass props. Props must be serializable (see table above):

```tsx
// ServerPage.tsx (server)
import { ClientWidget } from "./ClientWidget";

export default async function ServerPage() {
  const data = await fetchData();
  return <ClientWidget items={data.map((d) => ({ id: d.getId().toBranded(), name: d.getName().toBranded() }))} />;
}
```

### Client renders Server via `children` (slot pattern)

A Client Component **cannot import** Server Components. But a Server Component can **pass** Server Components as `children` to a Client Component:

```tsx
// Layout.tsx (server)
import { Sidebar } from "./Sidebar"; // client component
import { UserProfile } from "./UserProfile"; // server component

export default async function Layout({ children }: { children: ReactNode }) {
  return (
    <Sidebar>
      <UserProfile /> {/* server component passed as children — valid */}
      {children}
    </Sidebar>
  );
}
```

```tsx
// Sidebar.tsx (client)
"use client";

export function Sidebar({ children }: { children: ReactNode }) {
  const [open, setOpen] = useState(true);
  return <aside>{children}</aside>; // renders server components correctly
}
```

> **Rule**: Client Components must never `import` Server-only modules. Pass server-rendered content via `children` or other slot props instead.

### Pushing the client boundary down

Keep the `"use client"` boundary as **low** in the tree as possible. Extract only the interactive part into a Client Component and keep the rest as a Server Component:

```tsx
// ✅ Only the button is a client component
export async function FolderCard({ folder }: Props) {
  return (
    <div>
      <h2>{folder.getName().toBranded()}</h2>
      <DeleteFolderButton folderId={folder.getId().toBranded()} /> {/* client */}
    </div>
  );
}

// ❌ The entire card becomes client just because of one button
"use client";
export function FolderCard({ folder }: Props) { ... }
```

---

## 5. Props Interface Rules

- Always define an explicit `interface` or `type` for component props — never inline complex shapes or use `any`.
- Export the props type if other components or tests need it.
- VO **class instances** MUST NOT appear in Client Component **props received from a Server Component** — use the branded primitive type. A Client Component may, however, instantiate or work with VOs internally.

```tsx
// ✅ Correct — serializable props for data coming from a Server Component
interface LinksTableProps {
  links: {
    id: BrandedUUID;
    name: BrandedLinkName;
    shortLink: BrandedShortLink;
    destination: BrandedDestination;
    clicks: BrandedNonNegativeInteger;
  }[];
}

// ❌ Wrong — VO class in props passed from a Server Component (not serializable)
interface LinksTableProps {
  links: ShortUrl[]; // ShortUrl is a class — cannot cross the RSC boundary
}
```

---

## Checklist

### Server Components
- [ ] No `"use client"` directive
- [ ] Function is `async`
- [ ] Locale via `await determineLocale()`
- [ ] Data fetched via `serviceContainer` use cases
- [ ] VO values passed to Client Components via `.toBranded()`, never as class instances

### Client Components
- [ ] `"use client"` is the **first line** of the file — before any imports
- [ ] Function is **not** `async`
- [ ] Locale via `useLocale()` — never `determineLocale()`
- [ ] Mutations call a Server Action then `router.refresh()` — never `window.location.reload()`
- [ ] No imports of server-only modules (drizzle, `load-env`, `auth.ts`, etc.)
- [ ] Props received from Server Components use branded primitives or plain values — never VO class instances
- [ ] Explicit props `interface` defined and named

### Boundary
- [ ] All props crossing Server → Client are serializable (strings, numbers, booleans, plain objects, arrays, Date)
- [ ] IDs passed across the boundary use `.toBranded()` — branded type signals the value is already validated
- [ ] `"use client"` boundary pushed as low in the tree as possible
- [ ] Server content passed into Client Components via `children` slot, not by importing Server Components

## 7. References

For practical examples of components adhering to these rules, see the `.agents/skills/components/references/` directory:
- [server-component-i18n.md](references/server-component-i18n.md)
- [server-component.md](references/server-component.md)
- [client-component-i18n.md](references/client-component-i18n.md)
- [client-component.md](references/client-component.md)
- [responsive-component.md](references/responsive-component.md)
