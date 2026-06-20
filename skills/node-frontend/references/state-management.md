# Node Frontend — State Management Reference

## Zustand Store — Full Pattern

```ts
// src/stores/cartStore.ts
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

interface CartItem {
  id: string
  name: string
  price: number
  quantity: number
}

interface CartState {
  items: CartItem[]
  addItem: (item: Omit<CartItem, 'quantity'>) => void
  removeItem: (id: string) => void
  updateQuantity: (id: string, quantity: number) => void
  clearCart: () => void
  totalPrice: () => number
}

export const useCartStore = create<CartState>()(
  devtools(
    persist(
      immer((set, get) => ({
        items: [],

        addItem: (item) =>
          set((state) => {
            const existing = state.items.find((i) => i.id === item.id)
            if (existing) {
              existing.quantity += 1
            } else {
              state.items.push({ ...item, quantity: 1 })
            }
          }),

        removeItem: (id) =>
          set((state) => {
            state.items = state.items.filter((i) => i.id !== id)
          }),

        updateQuantity: (id, quantity) =>
          set((state) => {
            const item = state.items.find((i) => i.id === id)
            if (item) item.quantity = Math.max(0, quantity)
          }),

        clearCart: () => set({ items: [] }),

        totalPrice: () =>
          get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
      })),
      { name: 'cart-storage' },
    ),
    { name: 'CartStore' },
  ),
)
```

### Zustand Selector Pattern (prevents over-rendering)

```tsx
// Only re-renders when items.length changes
const itemCount = useCartStore((s) => s.items.length)

// Shallow compare for object/array slices
import { useShallow } from 'zustand/react/shallow'
const { addItem, removeItem } = useCartStore(
  useShallow((s) => ({ addItem: s.addItem, removeItem: s.removeItem })),
)
```

---

## TanStack Query — Data Fetching

### Setup

```tsx
// src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,   // 5 min
      gcTime: 1000 * 60 * 10,     // 10 min
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
})

ReactDOM.createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <App />
    <ReactQueryDevtools initialIsOpen={false} />
  </QueryClientProvider>,
)
```

### Query Keys Factory

```ts
// src/api/users/users.queries.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
}
```

### useQuery Hook

```tsx
import { useQuery } from '@tanstack/react-query'
import { getUser } from './users.api'
import { userKeys } from './users.queries'

export function useUser(userId: string) {
  return useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => getUser(userId),
    enabled: !!userId,
  })
}

// Usage
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isPending, isError, error } = useUser(userId)

  if (isPending) return <Spinner />
  if (isError) return <ErrorMessage message={error.message} />

  return <div>{user.displayName}</div>
}
```

### useMutation with Optimistic Updates

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { updateUser } from './users.api'
import { userKeys } from './users.queries'
import type { User } from '@/types/user.types'

export function useUpdateUser() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<User> }) =>
      updateUser(id, data),

    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: userKeys.detail(id) })
      const previous = queryClient.getQueryData<User>(userKeys.detail(id))

      queryClient.setQueryData<User>(userKeys.detail(id), (old) =>
        old ? { ...old, ...data } : old,
      )

      return { previous, id }
    },

    onError: (_err, _vars, ctx) => {
      if (ctx?.previous) {
        queryClient.setQueryData(userKeys.detail(ctx.id), ctx.previous)
      }
    },

    onSettled: (_data, _err, { id }) => {
      void queryClient.invalidateQueries({ queryKey: userKeys.detail(id) })
      void queryClient.invalidateQueries({ queryKey: userKeys.lists() })
    },
  })
}
```

### useInfiniteQuery

```tsx
import { useInfiniteQuery } from '@tanstack/react-query'
import { getProducts } from './products.api'
import { productKeys } from './products.queries'

export function useInfiniteProducts(filters: ProductFilters) {
  return useInfiniteQuery({
    queryKey: productKeys.list(filters),
    queryFn: ({ pageParam }) => getProducts({ ...filters, cursor: pageParam }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    select: (data) => ({
      pages: data.pages,
      items: data.pages.flatMap((p) => p.items),
    }),
  })
}

// Usage
function ProductList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteProducts({})

  return (
    <>
      {data?.items.map((p) => <ProductCard key={p.id} product={p} />)}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading…' : 'Load more'}
        </button>
      )}
    </>
  )
}
```

### Prefetching

```tsx
// In a route loader or on hover
await queryClient.prefetchQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => getUser(userId),
})
```

---

## Combining Zustand + TanStack Query

- **TanStack Query** — all server/async state (fetching, caching, mutations, pagination)
- **Zustand** — client-only UI state (cart, sidebar open, selected IDs, wizard step)
- **Context API** — low-frequency, tree-scoped values (theme, locale, auth session)
- **useState / useReducer** — component-local ephemeral state

```tsx
// DO: server state via TanStack Query
const { data: products } = useProducts()

// DO: UI state via Zustand
const selectedIds = useSelectionStore((s) => s.selectedIds)

// DON'T: duplicate server data in Zustand
// const products = useProductStore((s) => s.products)  ← avoid
```
