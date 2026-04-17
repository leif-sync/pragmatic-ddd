# Core i18n Utilities

When bootstrapping a new project using our custom, library-free i18n Next.js stack, you MUST pull in these abstraction files to define `LocaleProvider`, `Translations`, and dynamic resolution through Middleware.

## 1. Global Configurations

### `src/shared/infrastructure/i18n/config.ts`
```typescript
export const LOCALE_COOKIE_NAME = "x-locale";
export const LOCALES = ["en", "es"] as const;
export const DEFAULT_LOCALE = "en";
export type LOCALE = (typeof LOCALES)[number];
export type Translations<T extends object> = Record<LOCALE, T>;
```

## 2. Setting Locales via Next.js Server Actions

### `src/shared/infrastructure/i18n/actions.ts`
```typescript
"use server";

import { cookies } from "next/headers";
import { LOCALE_COOKIE_NAME } from "@/shared/infrastructure/i18n/config";
import { isValidLocale } from "./utils";
// Ensure your middleware exports cookie config options
import { localeCookieConfig } from "@/proxy"; 

export type SetLocaleResponse =
  | { success: true }
  | { success: false; error: "INVALID_LOCALE" };

const localeSchema = z.string().enum(LOCALES);

export async function setLocaleAction(rawData: unknown): Promise<SetLocaleResponse> {
  const parsed = localeSchema.safeParse(rawData);
  if (!parsed.success) {
    return { success: false, error: "INVALID_LOCALE" };
  }

  const locale = parsed.data;

  const cookieHandler = await cookies();
  cookieHandler.set(LOCALE_COOKIE_NAME, locale, localeCookieConfig);

  return { success: true };
}
```

## 3. Resolving the Locale on the Server 

Relies on `negotiator` and `@formatjs/intl-localematcher` to scrape `accept-language` dynamically falling back to the strictly read injected `cookies`.

### `src/shared/infrastructure/i18n/utils.ts`
```typescript
import Negotiator from "negotiator";
import { DEFAULT_LOCALE, LOCALE, LOCALE_COOKIE_NAME, LOCALES } from "./config";
import { match } from "@formatjs/intl-localematcher";
import { cookies, headers } from "next/headers";
import { cache } from "react";

export function determineLocaleFromAcceptLangOrDefault(
  acceptLanguageHeader: string | undefined | null,
): LOCALE {
  if (!acceptLanguageHeader) return DEFAULT_LOCALE;

  const languages = new Negotiator({
    headers: { "accept-language": acceptLanguageHeader },
  }).languages();

  const matched = match(languages, LOCALES, DEFAULT_LOCALE);
  const isValid = isValidLocale(matched);
  return isValid ? matched : DEFAULT_LOCALE;
}

export function isValidLocale(
  candidate: string | undefined,
): candidate is LOCALE {
  return candidate !== undefined && LOCALES.includes(candidate as LOCALE);
}

/**
 * Determines the appropriate locale for the current request.
 * Priority:
 * 1. Locale cookie value
 * 2. Accept-Language header
 * 3. Default application locale (Cached per-request via React cache)
 */
export const determineLocale = cache(async (): Promise<LOCALE> => {
  const currentHeaders = await headers();
  const currentCookies = await cookies();

  const localeCookie = currentCookies.get(LOCALE_COOKIE_NAME)?.value;
  const localeHeaderAcceptLang = determineLocaleFromAcceptLangOrDefault(
    currentHeaders.get("accept-language"),
  );

  const candidate = localeCookie ?? localeHeaderAcceptLang;
  return isValidLocale(candidate) ? candidate : DEFAULT_LOCALE;
});
```

## 4. Distributing Locales to Client Components

Because Client Components cannot securely or synchronously look up server-level Cookies safely during standard RSC initialization without boundary crossing, we provide a strictly-typed custom context.

### `src/shared/presentation/components/LocaleProvider.tsx`
```typescript
"use client";

import { createContext, useContext, ReactNode } from "react";
import type { LOCALE } from "@/shared/infrastructure/i18n/config";

const LocaleContext = createContext<LOCALE | undefined>(undefined);

export function LocaleProvider({
  children,
  locale,
}: {
  children: ReactNode;
  locale: LOCALE;
}) {
  return (
    <LocaleContext.Provider value={locale}>{children}</LocaleContext.Provider>
  );
}

export function useLocale() {
  const context = useContext(LocaleContext);
  if (context === undefined) {
    throw new Error("useLocale must be used within a LocaleProvider");
  }
  return context;
}
```