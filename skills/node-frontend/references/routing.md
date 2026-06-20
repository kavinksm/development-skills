# Node Frontend — Routing Reference (React Router v6)

## Router Setup — createBrowserRouter

```tsx
// src/router.tsx
import { createBrowserRouter, Navigate } from 'react-router-dom'
import { lazy, Suspense } from 'react'
import { RootLayout } from '@/pages/RootLayout'
import { AuthLayout } from '@/pages/AuthLayout'
import { ProtectedRoute } from '@/components/ProtectedRoute'
import { Spinner } from '@/components/Spinner'
import { queryClient } from '@/api/queryClient'

const Dashboard = lazy(() => import('@/pages/Dashboard'))
const UserList = lazy(() => import('@/pages/UserList'))
const UserDetail = lazy(() => import('@/pages/UserDetail'))
const Login = lazy(() => import('@/pages/Login'))
const NotFound = lazy(() => import('@/pages/NotFound'))

function withSuspense(element: React.ReactNode) {
  return <Suspense fallback={<Spinner fullPage />}>{element}</Suspense>
}

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <Navigate to="/dashboard" replace /> },
      {
        element: <ProtectedRoute />,
        children: [
          {
            path: 'dashboard',
            element: withSuspense(<Dashboard />),
          },
          {
            path: 'users',
            children: [
              {
                index: true,
                element: withSuspense(<UserList />),
                loader: userListLoader(queryClient),
              },
              {
                path: ':userId',
                element: withSuspense(<UserDetail />),
                loader: userDetailLoader(queryClient),
              },
            ],
          },
        ],
      },
      {
        element: <AuthLayout />,
        children: [{ path: 'login', element: withSuspense(<Login />) }],
      },
      { path: '*', element: withSuspense(<NotFound />) },
    ],
  },
])
```

```tsx
// src/main.tsx
import { RouterProvider } from 'react-router-dom'
import { router } from './router'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <RouterProvider router={router} />
  </QueryClientProvider>,
)
```

---

## Layout Components

```tsx
// src/pages/RootLayout/index.tsx
import { Outlet, ScrollRestoration } from 'react-router-dom'
import { Toaster } from '@/components/Toaster'

export function RootLayout() {
  return (
    <>
      <ScrollRestoration />
      <Outlet />
      <Toaster />
    </>
  )
}
```

```tsx
// src/pages/AuthLayout/index.tsx — wraps unauthenticated pages
import { Outlet, Navigate } from 'react-router-dom'
import { useAuthStore } from '@/stores/authStore'

export function AuthLayout() {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated)
  if (isAuthenticated) return <Navigate to="/dashboard" replace />
  return (
    <main className="flex min-h-screen items-center justify-center">
      <Outlet />
    </main>
  )
}
```

---

## Protected Route

```tsx
// src/components/ProtectedRoute/index.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom'
import { useAuthStore } from '@/stores/authStore'

export function ProtectedRoute() {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated)
  const location = useLocation()

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />
  }

  return <Outlet />
}
```

```tsx
// Redirect back after login
function Login() {
  const navigate = useNavigate()
  const location = useLocation()
  const from = (location.state as { from?: Location })?.from?.pathname ?? '/dashboard'

  const handleLogin = async (credentials: Credentials) => {
    await loginMutation.mutateAsync(credentials)
    navigate(from, { replace: true })
  }
}
```

---

## Data Loader (TanStack Query + React Router)

```ts
// src/api/users/users.loaders.ts
import type { QueryClient } from '@tanstack/react-query'
import type { LoaderFunctionArgs } from 'react-router-dom'
import { getUser } from './users.api'
import { userKeys } from './users.queries'

export function userDetailLoader(queryClient: QueryClient) {
  return async ({ params }: LoaderFunctionArgs) => {
    const userId = params.userId!
    return queryClient.ensureQueryData({
      queryKey: userKeys.detail(userId),
      queryFn: () => getUser(userId),
    })
  }
}
```

```tsx
// In the page — data is already cached by loader
import { useUser } from '@/api/users/users.queries'
import { useParams } from 'react-router-dom'

export default function UserDetail() {
  const { userId } = useParams<{ userId: string }>()
  const { data: user } = useUser(userId!)  // cache hit — no loading state needed

  return <div>{user?.displayName}</div>
}
```

---

## Navigation Hooks

```tsx
// useNavigate — programmatic navigation
const navigate = useNavigate()
navigate('/users/123')
navigate(-1)                          // go back
navigate('/login', { replace: true }) // replace history entry
navigate('/checkout', { state: { from: 'cart' } })

// useParams — typed route params
const { userId } = useParams<{ userId: string }>()

// useSearchParams — query string
const [searchParams, setSearchParams] = useSearchParams()
const query = searchParams.get('q') ?? ''
const page = Number(searchParams.get('page') ?? '1')

setSearchParams({ q: newQuery, page: '1' }) // replaces all params
setSearchParams((prev) => { prev.set('page', '2'); return prev }) // merge

// useLocation — current location object
const location = useLocation()
location.pathname  // '/users/123'
location.search    // '?page=2'
location.state     // navigation state

// useMatch — check active route
const match = useMatch('/users/:userId')
```

---

## Link Components

```tsx
import { Link, NavLink } from 'react-router-dom'

// Basic link
<Link to="/users/123">View Profile</Link>
<Link to={{ pathname: '/users', search: '?role=admin' }}>Admins</Link>

// NavLink — receives isActive for styling
<NavLink
  to="/dashboard"
  className={({ isActive }) =>
    cn('px-3 py-2 rounded', isActive && 'bg-blue-100 font-semibold')
  }
>
  Dashboard
</NavLink>
```

---

## Error Element

```tsx
// src/pages/ErrorPage/index.tsx
import { isRouteErrorResponse, useRouteError, Link } from 'react-router-dom'

export function ErrorPage() {
  const error = useRouteError()

  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>{error.status} {error.statusText}</h1>
        <p>{error.data}</p>
        <Link to="/">Go home</Link>
      </div>
    )
  }

  return <div>An unexpected error occurred.</div>
}
```
