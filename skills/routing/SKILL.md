---
name: routing
description: "[Pragmatic DDD Architecture] Guide for Next.js 16 Proxy (formerly middleware), app router segments, layout composition, i18n URL localization, cookie management, and redirect strategies between public and protected routes. Use when updating proxy.ts, configuring public vs private environments, modifying the [locale] vs /dashboard routing structure, and appending headers."
---

# Routing & Proxy (Next.js 16)

## 1. High-Level Concept

This project MUST rely on the **Next.js 16 Proxy** (the successor to `middleware.ts`) to intercept requests at the edge. The Proxy's primary job is to enforce Authentication boundaries, perform Localization (i18n) redirects, and append security cookies/headers before a Page or Layout is ever rendered.

- **Edge Execution**: The Proxy runs at the edge. It MUST NOT load Heavy Node modules or the Drizzle ORM directly.
- **Header Injection**: You MUST append the `x-current-path` or `x-locale` to the requested headers so Server Components can read them later.
- **Cookies**: You MUST use `NextResponse.cookies` strictly for high-level state like session identifiers and localization preferences.

## 2. Proxy Structure (Localization and Redirection)

You MUST export a `proxy` function instead of the legacy `middleware` function. 

The most common responsibility is mapping the user's `locales` to the correct URL segment or redirecting them out of `dashboard` if they lack a session.

```typescript
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import { LOCALE, LOCALE_COOKIE_NAME } from "./shared/infrastructure/i18n/config";
import { isValidLocale, determineLocaleFromAcceptLangOrDefault } from "./shared/infrastructure/i18n/utils";

// Exclude static assets and api routes in config
export const config = {
  matcher: [`/((?!api|trpc|_next|_next/image|favicon.ico|_vercel|.*\\..*).*)`],
};

const pathWithoutLocale = ["dashboard"];

// 1. Helper to append locale headers
function createLocaleHeaders({ request, locale }: { request: NextRequest; locale: LOCALE; }) {
  const newHeaders = new Headers(request.headers);
  newHeaders.set(LOCALE_COOKIE_NAME, locale);
  return newHeaders;
}

// 2. Helper to forward request
function createNextResponse({ request, locale }: { request: NextRequest; locale: LOCALE; }) {
  const requestHeaders = createLocaleHeaders({ request, locale });
  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.cookies.set(LOCALE_COOKIE_NAME, locale, { httpOnly: true, path: "/" });
  return response;
}

// 3. Helper to redirect request
function createLocaleRedirect({ newPath, request, locale }: { newPath: string; request: NextRequest; locale: string; }) {
  const redirectUrl = new URL(newPath, request.url);
  const response = NextResponse.redirect(redirectUrl);
  response.cookies.set(LOCALE_COOKIE_NAME, locale, { httpOnly: true, path: "/" });
  return response;
}

export function proxy(request: NextRequest) {
  const { pathname, search } = request.nextUrl;
  const localeCookie = request.cookies.get(LOCALE_COOKIE_NAME)?.value;
  const segments = pathname.split("/");
  const firstSegment = segments[1];

  const preferredLocale = isValidLocale(localeCookie)
    ? localeCookie
    : determineLocaleFromAcceptLangOrDefault(request.headers.get("accept-language"));

  // Dashboards don't use i18n segments in URL
  if (pathWithoutLocale.includes(firstSegment)) {
    // Add authentication checks here
    return createNextResponse({ request, locale: preferredLocale });
  }

  // Segment matches exactly
  if (isValidLocale(firstSegment)) {
    if (firstSegment === preferredLocale) {
      return createNextResponse({ request, locale: preferredLocale });
    }
    // Redirect to preferred locale if wrong
    const newPath = pathname.replace(`/${firstSegment}`, `/${preferredLocale}`);
    return createLocaleRedirect({ newPath, request, locale: preferredLocale });
  }

  // Missing segment entirely
  const newPath = `/${preferredLocale}${pathname}${search}`;
  return createLocaleRedirect({ newPath, request, locale: preferredLocale });
}
```

## 3. Route Layouts (Public vs Private)

If the project is multilingual (i18n), you MUST cleanly separate pages into routes that contain the `[locale]` segment in the URL (public) and those that do not (private/authenticated). A proxy handles the boundary between them.

1. **`app/[locale]/`** (Public Routes):
   - You MUST place public paths like the root landing page, `/login`, `/register`, or `/pricing` here.
   - The UI automatically wraps inside the localized context derived from the URL parameters instead of headers.
   - You MUST NOT fetch the session aggressively from use-cases unless your explicit goal is to redirect an already-logged-in user to an authenticated route.

2. **`app/dashboard/`** (Private / Authenticated Routes):
   - You MUST NOT include `[locale]` in the URL segment for private, authenticated functionality.
   - It MUST rely entirely on the `x-locale` cookie injected by the Proxy to determine language translation for components.
   - Auth validation MUST be the primary responsibility of `serviceContainer` use-cases called in this layout, not the Proxy, to preserve Domain rules.

## 4. Resource Routing (IDs vs Slugs)

When constructing dynamic URL structures, particularly in a REST API-like hierarchy, you MUST use stable entity Identifiers (IDs), NOT slugs or entity names. Names can change over time via user edits, which causes slugs to break bookmarks and historical references.

- **Correct**: `/dashboard/[workspaceId]/folders/[folderId]`
- **Incorrect**: `/dashboard/[workspaceSlug]/folders/[folderSlug]`

## 5. Contextless Route Redirection

Any route that cannot render intelligently without context MUST NOT attempt to load a generic UI. It MUST act exclusively as a redirection layer.

For example, in a multi-tenant or workspace-based application, navigating to `/dashboard` directly lacks the necessary `{ workspaceId }` routing parameter to display relevant details. Therefore, `/dashboard/page.tsx` MUST NOT exist as a rendered page. Instead, it MUST execute a Use Case to identify the user's primary or last-used entity and immediately `redirect()` to the proper context path, such as `/dashboard/[workspaceId]`.

## 6. Protected Paths and Server Components

You MUST NOT throw `Forbidden` or natively redirect inside a Server Component layout unless exhausting `assertNever` domain errors. While the proxy CAN intercept unauthorized navigation statically, complex auth rules (e.g., "Does this user own Workspace X?") MUST be resolved defensively in the Use Case executed by a Server Component.

## 7. References

For practical examples of routing and proxy patterns:
- [Server Component Domain Redirect](references/server-component-domain-redirect.md) - How to properly handle complex domain-based auth and redirection inside a Next.js Layout/Page without depending on the Proxy.
- [Contextless Redirect](references/contextless-redirect.md) - How to handle root authenticated paths (e.g., `/dashboard`) that lack context parameters and MUST redirect to specific resource URLs.