---
name: i18n
description: "[Pragmatic DDD Architecture] Guide for internationalization. Use when adding or editing translations in any component, page, or layout — also when wrapping a subtree in LocaleProvider, reading the active locale in a new component, building or modifying a locale switcher, or touching any file that calls determineLocale() or useLocale(). Covers the custom library-free implementation, Translations co-location, Server vs Client locale access patterns, and setLocaleAction."
---

# i18n (Internationalization)

This project uses a **custom, library-free i18n implementation**. There is no `next-intl`, `i18next`, or similar dependency.

**DEPENDENCY NOTICE**: If you are bootstrapping a new project using this architecture, read [i18n Utilities Reference](references/i18n-utilities.md) immediately to copy the exact implementation for `LocaleProvider`, `determineLocale`, and `setLocaleAction` so your components compile correctly.

## 1. Core Primitives

```typescript
export const LOCALES = ["en", "es"] as const;
export const DEFAULT_LOCALE = "en";
export type LOCALE = (typeof LOCALES)[number];
export type Translations<T extends object> = Record<LOCALE, T>;
```

## 2. Translation Co-location Rule

Translations MUST NOT be stored in separate JSON/YAML files. They MUST be defined **in the same file** (or a sibling) as the component or page that uses them, as a typed `const`.

### Flat structure (few keys)
```typescript
const translations: Translations<{
  title: string;
  description: string;
}> = {
  en: { title: "Create", description: "Fill details." },
  es: { title: "Crear", description: "Complete detalles." },
};
```

### Extracting to a sibling file (large components)
When the `translations` object grows large enough to distract from the component logic, you MUST move it to a sibling file named **`<component>.i18n.ts`**, co-located next to the component.

## 3. Accessing the Locale

> **CRITICAL**: You MUST NOT read the locale from the `[locale]` URL segment (page `params`). You MUST always use `determineLocale()` in Server Components or `useLocale()` in Client Components. The source of truth is the cookie/header injected by middleware.

### In Server Components (RSC)
You MUST use `determineLocale()`. It is memoized per-request via React `cache`.

```typescript
import { determineLocale } from "@/shared/infrastructure/i18n/utils";

export default async function MyPage() {
  const locale = await determineLocale();
  const t = translations[locale];

  return <h1>{t.title}</h1>;
}
```

### In Client Components
You MUST use the `useLocale()` hook. The Client Component MUST be rendered inside a parent `<LocaleProvider locale={locale}>`.

```typescript
"use client";
import { useLocale } from "@/shared/presentation/components/LocaleProvider";

export function MyClientComponent() {
  const locale = useLocale();
  const t = translations[locale];

  return <p>{t.description}</p>;
}
```

## 4. Changing Locale (Server Action)

Locale switching MUST be handled globally through cookies via a Server Action.

```typescript
"use client";
import { setLocaleAction } from "@/shared/infrastructure/i18n/actions";
import { useRouter } from "next/navigation";

export function Switcher() {
    const router = useRouter();
    
    const handleSwitch = async (newLocale: string) => {
        // MUST call the server action to update the cookie
        await setLocaleAction(newLocale);
        // MUST trigger a router.refresh() to update all server components
        router.refresh(); 
    }
    
    // ...
}
```