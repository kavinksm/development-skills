# Node Frontend — Project Structure Reference

## Standard Directory Layout

```
<project-root>/
├── public/                             # Static assets served as-is
│   └── favicon.ico
├── src/
│   ├── main.tsx                        # React entry — ReactDOM.createRoot
│   ├── App.tsx                         # Root component / QueryClientProvider
│   ├── router.tsx                      # createBrowserRouter definition
│   ├── vite-env.d.ts                   # ImportMetaEnv type augmentation
│   │
│   ├── components/                     # Reusable, domain-agnostic UI components
│   │   ├── Button/
│   │   │   ├── index.tsx
│   │   │   └── Button.test.tsx
│   │   ├── Input/
│   │   ├── Modal/
│   │   └── ui/                         # shadcn/ui generated components
│   │
│   ├── pages/                          # Route-level components (one per route)
│   │   ├── RootLayout/
│   │   ├── Dashboard/
│   │   └── Users/
│   │       ├── UserList/
│   │       │   ├── index.tsx
│   │       │   └── UserList.test.tsx
│   │       └── UserDetail/
│   │
│   ├── hooks/                          # Shared custom hooks
│   │   ├── useDebounce.ts
│   │   └── useDebounce.test.ts
│   │
│   ├── stores/                         # Zustand stores (client state only)
│   │   ├── authStore.ts
│   │   └── cartStore.ts
│   │
│   ├── api/                            # API client + TanStack Query hooks
│   │   ├── client.ts                   # Axios instance
│   │   ├── queryClient.ts              # Shared QueryClient instance
│   │   └── users/
│   │       ├── users.api.ts            # Axios API functions
│   │       ├── users.queries.ts        # useQuery / useMutation hooks + key factory
│   │       └── users.loaders.ts        # React Router data loaders
│   │
│   ├── types/                          # Shared TypeScript types
│   │   ├── user.types.ts
│   │   └── common.types.ts
│   │
│   ├── utils/                          # Pure utility functions (no React)
│   │   └── format.ts
│   │
│   ├── lib/                            # Thin wrappers around third-party libs
│   │   └── utils.ts                    # cn() helper
│   │
│   ├── styles/
│   │   └── globals.css                 # Tailwind directives + base styles
│   │
│   └── test/                           # Test infrastructure only
│       ├── setup.ts                    # Vitest global setup
│       ├── mswServer.ts                # MSW node server
│       └── handlers/                   # MSW request handlers
│           └── userHandlers.ts
│
├── e2e/                                # Playwright e2e tests
│   ├── login.spec.ts
│   └── users.spec.ts
│
├── index.html                          # Vite entry HTML
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── vitest.config.ts                    # or merged into vite.config.ts
├── playwright.config.ts
├── eslint.config.js
├── .prettierrc
├── .env.example                        # Committed env var template
├── .env.local                          # Local values — git-ignored
├── Dockerfile
└── package.json
```

---

## Entry Point

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { RouterProvider } from 'react-router-dom'
import { queryClient } from './api/queryClient'
import { router } from './router'
import './styles/globals.css'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </StrictMode>,
)
```

---

## Shared QueryClient

```ts
// src/api/queryClient.ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,
      gcTime: 1000 * 60 * 10,
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
})
```

---

## package.json — Key Dependencies

```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.0.0",
    "@tanstack/react-query": "^5.0.0",
    "zustand": "^5.0.0",
    "react-hook-form": "^7.0.0",
    "@hookform/resolvers": "^3.0.0",
    "zod": "^3.0.0",
    "axios": "^1.0.0",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.0.0",
    "class-variance-authority": "^0.7.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "vite": "^6.0.0",
    "typescript": "^5.0.0",
    "tailwindcss": "^3.0.0",
    "vitest": "^2.0.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/user-event": "^14.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "msw": "^2.0.0",
    "@playwright/test": "^1.0.0",
    "eslint": "^9.0.0",
    "typescript-eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "prettier-plugin-tailwindcss": "^0.6.0"
  }
}
```

---

## Vite + Vitest Together (single config)

```ts
// vite.config.ts — merge vitest config here to avoid two files
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    exclude: ['e2e/**', 'node_modules/**'],
  },
})
```

---

## Scaffolding a New Project from Scratch

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install

# Core dependencies
npm install react-router-dom @tanstack/react-query zustand \
  react-hook-form @hookform/resolvers zod axios \
  clsx tailwind-merge class-variance-authority

# Dev tools
npm install -D tailwindcss @tailwindcss/forms @tailwindcss/typography \
  vite-tsconfig-paths \
  vitest @testing-library/react @testing-library/user-event \
  @testing-library/jest-dom msw \
  @playwright/test \
  eslint typescript-eslint eslint-plugin-react-hooks \
  prettier prettier-plugin-tailwindcss

npx tailwindcss init -p
npx playwright install chromium
```
