# auth-browser

Next.js SDK for [Bastion](https://github.com/shadovxw/bastion) — the `shadovxw.me` identity platform.

Reads the `auth_session` cookie set by Bastion and exposes the authenticated user in both server and client components. No crypto dependencies — JWT signature verification is done by the Go backend. This package only decodes claims.

## Installation

```bash
npm install @shadovxw/auth-browser
```

## Setup — middleware.ts (copy once, never touch)

```ts
import { authMiddleware } from '@shadovxw/auth-browser/server'

export default authMiddleware({
  authServer: 'https://api.auth.shadovxw.me',
  clientId: 'app1',
  publicPaths: ['/about', '/public'],
})

export const config = {
  matcher: ['/((?!_next|favicon).*)'],
}
```

Unauthenticated requests are redirected to Bastion's `/authorize` endpoint, which checks the SSO session and either issues a token immediately or shows the login page.

## Setup — /api/auth/me route (copy once, never touch)

```ts
// app/api/auth/me/route.ts
import { getUser } from '@shadovxw/auth-browser/server'

export async function GET() {
  const user = await getUser()
  return Response.json(user ?? null, { status: user ? 200 : 401 })
}
```

Required by the `useUser()` client hook.

## Server-side usage

```ts
import { getUser, hasPermission, hasRole } from '@shadovxw/auth-browser/server'

// Server Component
export default async function Page() {
  const user = await getUser()
  return <h1>Hello {user?.displayName}</h1>
}

// Route Handler
export async function GET() {
  const user = await getUser()
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 })
  if (!hasPermission(user, 'posts:write')) {
    return Response.json({ error: 'Forbidden' }, { status: 403 })
  }
  return Response.json({ user })
}
```

## Client-side usage

```tsx
'use client'
import { useUser, useHasPermission, useHasRole } from '@shadovxw/auth-browser'

export function Avatar() {
  const user = useUser()
  if (!user) return null
  return <img src={user.avatar ?? ''} alt={user.displayName} />
}

export function DeleteButton() {
  const canDelete = useHasPermission('posts:delete')
  if (!canDelete) return null
  return <button>Delete</button>
}
```

## User type

```ts
interface User {
  id:          string
  email:       string
  displayName: string
  avatar:      string | null
  provider:    'google' | 'github'
  roles:       string[]
  permissions: string[]
  app:         string
  exp:         number
}
```

## How it works

- **`getUser()`** reads the `auth_session` cookie via Next.js `cookies()`, base64-decodes the JWT payload, checks expiry, and returns a typed `User` or `null`. No network calls, no crypto.
- **`authMiddleware()`** runs in Next.js Edge middleware. Reads the same cookie, checks expiry, redirects to Bastion if missing or expired.
- **`useUser()`** calls `GET /api/auth/me` once on mount (module-level cache — one fetch per page load regardless of how many components call the hook).
- The Go backend (via [auth-sdk](https://github.com/shadovxw/auth-sdk)) is the actual trust boundary — it verifies the RS256 signature on every API request.

## Exports

| Import path | Exports |
|---|---|
| `@shadovxw/auth-browser` | `useUser`, `useHasPermission`, `useHasRole` |
| `@shadovxw/auth-browser/server` | `getUser`, `authMiddleware`, `hasPermission`, `hasRole` |

## Part of Bastion

- [shadovxw/bastion](https://github.com/shadovxw/bastion) — auth server + admin UI
- [shadovxw/auth-sdk](https://github.com/shadovxw/auth-sdk) — Go backend SDK
