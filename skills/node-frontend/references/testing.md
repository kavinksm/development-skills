# Node Frontend — Testing Reference (Vitest + RTL + Playwright + MSW)

## Setup

```bash
npm install -D vitest @testing-library/react @testing-library/user-event \
  @testing-library/jest-dom jsdom msw @playwright/test
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      exclude: ['src/test/**', '**/*.stories.*'],
    },
  },
})
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom'
import { cleanup } from '@testing-library/react'
import { afterEach, beforeAll, afterAll } from 'vitest'
import { server } from './mswServer'

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => { cleanup(); server.resetHandlers() })
afterAll(() => server.close())
```

---

## MSW — Mock Service Worker

```ts
// src/test/mswServer.ts
import { setupServer } from 'msw/node'
import { userHandlers } from './handlers/userHandlers'
import { productHandlers } from './handlers/productHandlers'

export const server = setupServer(...userHandlers, ...productHandlers)
```

```ts
// src/test/handlers/userHandlers.ts
import { http, HttpResponse } from 'msw'
import type { User } from '@/types/user.types'

const baseUrl = 'http://localhost:8080/api'

export const userHandlers = [
  http.get(`${baseUrl}/users`, () =>
    HttpResponse.json<{ items: User[] }>({
      items: [{ id: '1', email: 'alice@example.com', displayName: 'Alice', role: 'user', createdAt: '2024-01-01T00:00:00Z' }],
      total: 1, page: 1, limit: 20,
    }),
  ),

  http.get(`${baseUrl}/users/:id`, ({ params }) =>
    HttpResponse.json<User>({
      id: params.id as string,
      email: 'alice@example.com',
      displayName: 'Alice',
      role: 'user',
      createdAt: '2024-01-01T00:00:00Z',
    }),
  ),

  http.post(`${baseUrl}/users`, async ({ request }) => {
    const body = await request.json() as Record<string, unknown>
    return HttpResponse.json<User>(
      { id: 'new-id', ...body as Partial<User>, createdAt: new Date().toISOString() } as User,
      { status: 201 },
    )
  }),
]
```

---

## React Testing Library — Component Tests

```tsx
// src/components/UserCard/UserCard.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import { UserCard } from './index'

describe('UserCard', () => {
  const defaultProps = {
    userId: 'u1',
    displayName: 'Alice',
    avatarUrl: undefined,
    onSelect: vi.fn(),
  }

  it('renders the display name', () => {
    render(<UserCard {...defaultProps} />)
    expect(screen.getByText('Alice')).toBeInTheDocument()
  })

  it('calls onSelect with userId when clicked', async () => {
    const user = userEvent.setup()
    render(<UserCard {...defaultProps} />)

    await user.click(screen.getByRole('article'))
    expect(defaultProps.onSelect).toHaveBeenCalledWith('u1')
    expect(defaultProps.onSelect).toHaveBeenCalledTimes(1)
  })

  it('does not render avatar when avatarUrl is not provided', () => {
    render(<UserCard {...defaultProps} />)
    expect(screen.queryByRole('img')).not.toBeInTheDocument()
  })
})
```

---

## Testing Async / Query-backed Components

```tsx
// src/pages/UserDetail/UserDetail.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { MemoryRouter, Route, Routes } from 'react-router-dom'
import UserDetail from './index'
import { server } from '@/test/mswServer'
import { http, HttpResponse } from 'msw'

function renderWithProviders(ui: React.ReactElement, { route = '/users/1' } = {}) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })
  return render(
    <QueryClientProvider client={queryClient}>
      <MemoryRouter initialEntries={[route]}>
        <Routes>
          <Route path="/users/:userId" element={ui} />
        </Routes>
      </MemoryRouter>
    </QueryClientProvider>,
  )
}

describe('UserDetail', () => {
  it('renders user data after loading', async () => {
    renderWithProviders(<UserDetail />)

    expect(screen.getByRole('status')).toBeInTheDocument() // spinner

    await waitFor(() =>
      expect(screen.getByText('Alice')).toBeInTheDocument(),
    )
  })

  it('shows error message when API fails', async () => {
    server.use(
      http.get('*/users/1', () => HttpResponse.json({ message: 'Not found' }, { status: 404 })),
    )

    renderWithProviders(<UserDetail />)

    await waitFor(() =>
      expect(screen.getByRole('alert')).toHaveTextContent('Not found'),
    )
  })
})
```

---

## Testing Custom Hooks

```tsx
// src/hooks/useDebounce.test.ts
import { renderHook, act } from '@testing-library/react'
import { vi, describe, it, expect, beforeEach, afterEach } from 'vitest'
import { useDebounce } from './useDebounce'

describe('useDebounce', () => {
  beforeEach(() => vi.useFakeTimers())
  afterEach(() => vi.useRealTimers())

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', { delay: 300 }))
    expect(result.current).toBe('hello')
  })

  it('updates value after delay', () => {
    const { result, rerender } = renderHook(
      ({ value }: { value: string }) => useDebounce(value, { delay: 300 }),
      { initialProps: { value: 'hello' } },
    )

    rerender({ value: 'world' })
    expect(result.current).toBe('hello')

    act(() => vi.advanceTimersByTime(300))
    expect(result.current).toBe('world')
  })
})
```

---

## Testing Forms

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { LoginForm } from './LoginForm'

describe('LoginForm', () => {
  it('shows validation errors on empty submit', async () => {
    const user = userEvent.setup()
    render(<LoginForm onSubmit={vi.fn()} />)

    await user.click(screen.getByRole('button', { name: /sign in/i }))

    expect(screen.getByText('Email is required')).toBeInTheDocument()
    expect(screen.getByText('Password must be at least 8 characters')).toBeInTheDocument()
  })

  it('calls onSubmit with form values', async () => {
    const user = userEvent.setup()
    const onSubmit = vi.fn().mockResolvedValue(undefined)
    render(<LoginForm onSubmit={onSubmit} />)

    await user.type(screen.getByLabelText('Email'), 'user@example.com')
    await user.type(screen.getByLabelText('Password'), 'password123')
    await user.click(screen.getByRole('button', { name: /sign in/i }))

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'password123',
      rememberMe: false,
    })
  })
})
```

---

## Playwright — E2E Tests

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  testMatch: '**/*.spec.ts',
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
})
```

```ts
// e2e/login.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Login flow', () => {
  test('redirects to dashboard on successful login', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('user@example.com')
    await page.getByLabel('Password').fill('password123')
    await page.getByRole('button', { name: /sign in/i }).click()

    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible()
  })

  test('shows error on invalid credentials', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('wrong@example.com')
    await page.getByLabel('Password').fill('wrongpass')
    await page.getByRole('button', { name: /sign in/i }).click()

    await expect(page.getByRole('alert')).toContainText('Invalid email or password')
    await expect(page).toHaveURL('/login')
  })
})
```

```ts
// e2e/users.spec.ts — Page Object Model
import { test, expect, type Page } from '@playwright/test'

class UserListPage {
  constructor(private page: Page) {}

  async goto() { await this.page.goto('/users') }
  async search(query: string) { await this.page.getByPlaceholder('Search users').fill(query) }
  getResultCount() { return this.page.getByTestId('result-count') }
  getUserCard(name: string) { return this.page.getByRole('article', { name }) }
}

test('user search filters results', async ({ page }) => {
  const userListPage = new UserListPage(page)
  await userListPage.goto()

  await userListPage.search('Alice')
  await expect(userListPage.getUserCard('Alice')).toBeVisible()
  await expect(userListPage.getResultCount()).toContainText('1')
})
```

---

## Vitest — Mocking Modules

```ts
import { vi, describe, it, expect, beforeEach } from 'vitest'

// Mock an entire module
vi.mock('@/api/users/users.api', () => ({
  getUser: vi.fn(),
  createUser: vi.fn(),
}))

import { getUser } from '@/api/users/users.api'

describe('useUser hook', () => {
  beforeEach(() => vi.mocked(getUser).mockResolvedValue(mockUser))

  it('returns user data', async () => {
    // ...
  })
})

// Spy on a method
const consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {})
// restore
consoleSpy.mockRestore()
```
