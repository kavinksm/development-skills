---
name: node-frontend
description: >
  Generate, scaffold, and implement Node.js frontend application code covering the full
  development lifecycle. Covers React components (functional components, hooks, useState,
  useEffect, useCallback, useMemo, useRef, useContext, custom hooks, React.memo,
  forwardRef, lazy, Suspense, ErrorBoundary, compound components, render props,
  controlled/uncontrolled inputs), TypeScript (interfaces, types, generics, discriminated
  unions, utility types, satisfies, as const, type guards, enums, namespaces, declaration
  merging), state management (Zustand store, create, devtools middleware, immer middleware,
  persist middleware, TanStack Query useQuery, useMutation, useInfiniteQuery,
  QueryClient, QueryClientProvider, staleTime, gcTime, invalidateQueries, optimistic
  updates, Context API, useReducer), routing (React Router v6, createBrowserRouter,
  RouterProvider, Route, Outlet, Link, NavLink, useNavigate, useParams, useSearchParams,
  useLocation, loader, action, redirect, lazy route, protected route, nested routes),
  forms (React Hook Form useForm, register, handleSubmit, formState, errors, Controller,
  useFieldArray, useWatch, Zod schema, z.object, z.string, z.number, z.enum, z.array,
  z.union, refine, superRefine, zodResolver), API client (Axios instance, interceptors,
  request/response interceptors, retry logic, TanStack Query with Axios, fetch, AbortController,
  error handling, TypeScript typed responses, OpenAPI client generation), testing (Vitest,
  describe, it, expect, vi.fn, vi.mock, vi.spyOn, beforeEach, afterEach, React Testing
  Library, render, screen, fireEvent, userEvent, waitFor, within, renderHook, act,
  Playwright, test, expect, page.goto, locator, getByRole, getByText, getByLabel, fill,
  click, screenshot, MSW, rest, http, HttpResponse, setupServer, setupWorker), styling
  (Tailwind CSS, @apply, tailwind.config, cn utility, clsx, tailwind-merge, CSS Modules,
  component variants with cva, shadcn/ui, Radix UI primitives), build and deployment
  (Vite config, vite.config.ts, defineConfig, plugins, resolve alias, proxy, env vars,
  import.meta.env, VITE_ prefix, Docker multi-stage build, Nginx serving SPA, GitHub
  Actions workflow, preview deployment). Use when asked to scaffold, generate, create,
  write, implement, or fix Node.js frontend code for: React component, hook, page,
  layout, form, validation, API call, fetch, data fetching, state, store, route, router,
  navigation, guard, test, unit test, integration test, e2e test, Playwright, Vitest,
  RTL, Tailwind, CSS, styling, theme, Vite, build, Dockerfile, CI/CD pipeline, TypeScript,
  Next.js pattern, SPA, single page application, frontend, UI, React app.
allowed-tools: Read Grep Glob Write Edit Bash
argument-hint: "[area] [feature-name] -- e.g. components UserCard or forms LoginForm or testing UserProfile or api useProducts"
---

## Invocation and Argument Parsing

When invoked, parse `$ARGUMENTS` for:
- **First argument** (optional): area — one of:
  `components`, `component`, `react`, `hooks`, `hook`, `custom-hook`,
  `state`, `zustand`, `query`, `tanstack`, `context`, `reducer`,
  `routing`, `router`, `route`, `navigation`, `guard`,
  `forms`, `form`, `validation`, `zod`, `rhf`,
  `api`, `fetch`, `axios`, `client`, `http`,
  `testing`, `test`, `unit`, `integration`, `e2e`, `playwright`, `vitest`, `rtl`, `msw`,
  `styling`, `tailwind`, `css`, `theme`, `variants`,
  `build`, `vite`, `docker`, `ci`, `deploy`, `env`,
  `project`, `scaffold`, `structure`, `setup`, `tsconfig`
- **Second argument** (optional): feature or component name (e.g., `UserCard`, `LoginForm`, `useProducts`)

If either is missing, infer from the user's description. Ask only if both are completely absent:
> "Which area? And what should the component/feature be named?"

---

## Generation Procedure

Follow these steps for every request.

### Step 1 — Identify the area and load the reference file

Map the requested area to its reference file:

| Area / topic | Reference file |
|---|---|
| React components, hooks, TypeScript component patterns, Context | `references/components.md` |
| Zustand, TanStack Query, Context API, useReducer, global state | `references/state-management.md` |
| React Router v6, routes, loaders, guards, nested routing | `references/routing.md` |
| React Hook Form, Zod, form validation, field arrays | `references/forms.md` |
| Axios, fetch, API client, interceptors, typed responses | `references/api-client.md` |
| Vitest, React Testing Library, Playwright, MSW, test patterns | `references/testing.md` |
| Tailwind CSS, CSS Modules, cva, shadcn/ui, component variants | `references/styling.md` |
| Vite config, env vars, Dockerfile, GitHub Actions, deploy | `references/build-deploy.md` |
| Project scaffold, tsconfig, package.json, directory layout | `references/project-structure.md` |

**Critical**: Use `Glob` to find the absolute path first, then `Read` the result:

```
Glob pattern: **/skills/node-frontend/references/<filename>
```

If Glob returns no results:
> "Could not locate the skill reference files. Verify the repo is added via `--add-dir /path/to/development-claude-skills`."

For full-stack features, load multiple reference files (e.g., components + forms + api-client + testing).

### Step 2 — Understand the project context

Before generating code, use `Grep` and `Glob` to inspect the user's project:
- Check `package.json` for React version, TypeScript, existing dependencies (Tailwind, Zustand, TanStack Query, etc.)
- Check `tsconfig.json` for path aliases, strict mode, target
- Check `vite.config.ts` for plugins and path aliases
- Check `src/` directory structure to match naming and folder conventions
- Check for an existing `tailwind.config.*` to understand content paths and theme extensions

Adapt all generated code to match the existing project's conventions.

### Step 3 — Generate the code

Apply patterns from the reference file. Rules:
- Always use TypeScript — never plain JS unless the project has no TypeScript
- Prefer named exports over default exports for components (except page-level route components)
- Use functional components only — no class components
- Co-locate component files: `ComponentName/index.tsx`, `ComponentName/ComponentName.test.tsx`
- Prefer `const` arrow functions: `const MyComponent = () => {}`
- Use explicit return types on hooks and utility functions
- Destructure props with inline types: `({ name, onClick }: Props) =>`
- TanStack Query for all server state; Zustand for client-only global state
- Never use `any` — use `unknown` and type-guard where needed

### Step 4 — Generate the test

Always include a test file unless the user explicitly says not to:
- Unit/component tests with Vitest + React Testing Library
- Mock API calls with MSW (`setupServer` / `http.get`)
- E2E flows with Playwright when the area is a full page or user journey
- Load `references/testing.md` for test patterns

### Step 5 — Summarise

After writing all files, print:
- Files created and their paths
- Key dependencies to add to `package.json` (if not already present), with install command
- Any env vars to add to `.env.example`
- Next steps (e.g., "run `npm test` to verify", "register the route in `router.tsx`")

---

## Package and Naming Conventions

```
src/
├── main.tsx                    # React entry point
├── App.tsx                     # Root component / RouterProvider
├── router.tsx                  # createBrowserRouter definition
├── components/                 # Reusable UI components
│   └── <ComponentName>/
│       ├── index.tsx
│       ├── <ComponentName>.test.tsx
│       └── <ComponentName>.stories.tsx   # optional Storybook
├── pages/                      # Route-level page components
│   └── <PageName>/
│       ├── index.tsx
│       └── <PageName>.test.tsx
├── hooks/                      # Custom hooks
│   ├── use<Feature>.ts
│   └── use<Feature>.test.ts
├── stores/                     # Zustand stores
│   └── <feature>Store.ts
├── api/                        # API client and query hooks
│   ├── client.ts               # Axios instance
│   └── <feature>/
│       ├── <feature>.api.ts    # API functions
│       └── <feature>.queries.ts # TanStack Query hooks
├── types/                      # Shared TypeScript types and interfaces
│   └── <domain>.types.ts
├── utils/                      # Pure utility functions
│   └── <util>.ts
├── lib/                        # Third-party wrappers (cn, date, etc.)
│   └── utils.ts
└── styles/                     # Global CSS / Tailwind base
    └── globals.css
```

**Naming rules:**
- Components: `PascalCase` (`UserCard`, `ProductList`)
- Hooks: `camelCase` prefixed with `use` (`useUser`, `useProductList`)
- Stores: `camelCase` suffixed with `Store` (`userStore`, `cartStore`)
- API files: `kebab-case` (`user.api.ts`, `product.queries.ts`)
- Types: `PascalCase` interfaces/types, suffix with `Props` for component props
- Tests: `<subject>.test.tsx` (unit) or `<feature>.spec.ts` (Playwright e2e)

---

## TypeScript Strictness

Always generate code compatible with:
```json
{
  "strict": true,
  "noUncheckedIndexedAccess": true,
  "exactOptionalPropertyTypes": true
}
```

If the project's `tsconfig.json` is less strict, match its settings but note where stricter types would help.

---

## Reference File Routing

| User says | Load these reference files |
|---|---|
| "component", "React component", "functional component", "hook", "useState", "useEffect", "custom hook", "context", "provider", "memo", "forwardRef", "compound", "render prop" | `references/components.md` |
| "state", "store", "Zustand", "TanStack Query", "useQuery", "useMutation", "cache", "invalidate", "optimistic", "global state", "context state", "useReducer" | `references/state-management.md` |
| "route", "router", "navigation", "React Router", "Link", "NavLink", "useNavigate", "loader", "action", "protected route", "nested route", "outlet", "guard" | `references/routing.md` |
| "form", "React Hook Form", "useForm", "register", "validation", "Zod", "schema", "zodResolver", "field array", "useFieldArray", "controlled input", "error message" | `references/forms.md` |
| "API", "fetch", "Axios", "HTTP", "request", "response", "interceptor", "typed response", "client", "endpoint", "REST", "abort" | `references/api-client.md` |
| "test", "Vitest", "React Testing Library", "render", "screen", "userEvent", "fireEvent", "waitFor", "mock", "MSW", "Playwright", "e2e", "locator", "getByRole" | `references/testing.md` |
| "Tailwind", "CSS", "styling", "className", "clsx", "cn", "cva", "variants", "shadcn", "Radix", "theme", "dark mode", "CSS Module" | `references/styling.md` |
| "Vite", "build", "config", "env", "VITE_", "import.meta.env", "Dockerfile", "Nginx", "GitHub Actions", "CI", "deploy", "proxy" | `references/build-deploy.md` |
| "scaffold", "new project", "setup", "tsconfig", "package.json", "directory", "structure", "init", "create" | `references/project-structure.md` |
| "full feature", "full page", "end to end", "complete", "everything" | all reference files |

When in doubt, load the file — it is cheap and prevents hallucinated patterns.

---

## Error Handling

- If the project uses JavaScript (no `tsconfig.json`), generate `.jsx` files and remove type annotations, but note that TypeScript is strongly recommended.
- If the project uses Next.js, note that page routing, `'use client'` / `'use server'` directives, and Server Components differ from the Vite/React Router patterns in these references; adapt accordingly and flag the differences.
- If a reference file cannot be located by Glob, report the attempted pattern and ask the user to verify the `--add-dir` path.
- Never invent hook or library APIs not present in the reference templates. Add `// TODO: verify` if uncertain.
- If the user asks for Vue or Svelte, note that reference files are React-based and translate patterns to the requested framework idioms.
