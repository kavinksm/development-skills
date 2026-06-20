# Node Frontend — API Client Reference

## Axios Instance Setup

```ts
// src/api/client.ts
import axios, {
  type AxiosError,
  type AxiosResponse,
  type InternalAxiosRequestConfig,
} from 'axios'
import { useAuthStore } from '@/stores/authStore'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15_000,
  headers: { 'Content-Type': 'application/json' },
})

// Attach auth token to every request
apiClient.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  const token = useAuthStore.getState().token
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Global response error handling
apiClient.interceptors.response.use(
  (response: AxiosResponse) => response,
  async (error: AxiosError<ApiError>) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout()
      window.location.href = '/login'
    }
    return Promise.reject(toAppError(error))
  },
)

function toAppError(error: AxiosError<ApiError>): Error {
  const message =
    error.response?.data?.message ??
    error.message ??
    'An unexpected error occurred'
  return Object.assign(new Error(message), { status: error.response?.status })
}
```

---

## Typed API Functions

```ts
// src/api/users/users.api.ts
import { apiClient } from '../client'
import type { User, CreateUserDto, UpdateUserDto, PaginatedResponse } from '@/types/user.types'

export async function getUsers(params?: {
  page?: number
  limit?: number
  search?: string
}): Promise<PaginatedResponse<User>> {
  const { data } = await apiClient.get<PaginatedResponse<User>>('/users', { params })
  return data
}

export async function getUser(id: string): Promise<User> {
  const { data } = await apiClient.get<User>(`/users/${id}`)
  return data
}

export async function createUser(dto: CreateUserDto): Promise<User> {
  const { data } = await apiClient.post<User>('/users', dto)
  return data
}

export async function updateUser(id: string, dto: UpdateUserDto): Promise<User> {
  const { data } = await apiClient.patch<User>(`/users/${id}`, dto)
  return data
}

export async function deleteUser(id: string): Promise<void> {
  await apiClient.delete(`/users/${id}`)
}
```

---

## TypeScript Types for API

```ts
// src/types/user.types.ts
export interface User {
  id: string
  email: string
  displayName: string
  role: 'admin' | 'user' | 'viewer'
  createdAt: string
}

export type CreateUserDto = Pick<User, 'email' | 'displayName' | 'role'>
export type UpdateUserDto = Partial<Omit<User, 'id' | 'createdAt'>>

export interface PaginatedResponse<T> {
  items: T[]
  total: number
  page: number
  limit: number
  nextCursor?: string
}

export interface ApiError {
  message: string
  code?: string
  details?: Record<string, string[]>
}
```

---

## File Upload

```ts
// src/api/uploads/uploads.api.ts
export async function uploadFile(
  file: File,
  onProgress?: (percent: number) => void,
): Promise<{ url: string; key: string }> {
  const formData = new FormData()
  formData.append('file', file)

  const { data } = await apiClient.post<{ url: string; key: string }>(
    '/uploads',
    formData,
    {
      headers: { 'Content-Type': 'multipart/form-data' },
      onUploadProgress: (event) => {
        if (event.total) {
          onProgress?.(Math.round((event.loaded * 100) / event.total))
        }
      },
    },
  )
  return data
}
```

---

## Abort / Cancellation

```ts
// src/api/search/search.api.ts
export async function searchProducts(
  query: string,
  signal?: AbortSignal,
): Promise<Product[]> {
  const { data } = await apiClient.get<Product[]>('/products/search', {
    params: { q: query },
    signal,
  })
  return data
}
```

```tsx
// Usage in component — cancel on cleanup
useEffect(() => {
  const controller = new AbortController()
  searchProducts(debouncedQuery, controller.signal)
    .then(setResults)
    .catch((err) => { if (err.name !== 'CanceledError') setError(err) })
  return () => controller.abort()
}, [debouncedQuery])
```

---

## Native fetch Alternative

```ts
// src/api/lib/fetcher.ts
export async function fetcher<T>(
  input: RequestInfo,
  init?: RequestInit,
): Promise<T> {
  const token = useAuthStore.getState().token
  const response = await fetch(input, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...init?.headers,
    },
  })

  if (!response.ok) {
    const body = await response.json().catch(() => ({}))
    throw Object.assign(new Error(body.message ?? response.statusText), {
      status: response.status,
    })
  }

  if (response.status === 204) return undefined as T
  return response.json() as Promise<T>
}
```

---

## Retry with Exponential Back-off

```ts
// src/api/lib/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
): Promise<T> {
  let attempt = 0
  while (true) {
    try {
      return await fn()
    } catch (err) {
      attempt++
      const status = (err as { status?: number }).status
      const isRetryable = !status || status >= 500
      if (attempt >= maxAttempts || !isRetryable) throw err
      await new Promise((r) => setTimeout(r, 2 ** attempt * 200))
    }
  }
}
```

---

## Environment Variables

```ts
// All API env vars must be prefixed VITE_ and accessed via import.meta.env
const BASE_URL: string = import.meta.env.VITE_API_BASE_URL
const TIMEOUT: number = Number(import.meta.env.VITE_API_TIMEOUT ?? '15000')

// .env.example
// VITE_API_BASE_URL=http://localhost:8080/api
// VITE_API_TIMEOUT=15000
```

---

## OpenAPI / Generated Client (optional)

```bash
# Generate typed client from OpenAPI spec
npx openapi-typescript https://api.example.com/openapi.json -o src/types/api.types.ts
npx openapi-fetch --input https://api.example.com/openapi.json --output src/api/generated
```

```ts
// Usage with openapi-fetch
import createClient from 'openapi-fetch'
import type { paths } from '@/types/api.types'

export const apiClient = createClient<paths>({
  baseUrl: import.meta.env.VITE_API_BASE_URL,
})

const { data, error } = await apiClient.GET('/users/{id}', {
  params: { path: { id: '123' } },
})
```
