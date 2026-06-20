# Node Frontend — React Components & Hooks Reference

## Functional Component — Full Pattern

```tsx
import { type ReactNode, useCallback, useId } from 'react'

interface UserCardProps {
  userId: string
  displayName: string
  avatarUrl?: string
  onSelect?: (id: string) => void
  children?: ReactNode
}

export const UserCard = ({
  userId,
  displayName,
  avatarUrl,
  onSelect,
  children,
}: UserCardProps) => {
  const labelId = useId()

  const handleClick = useCallback(() => {
    onSelect?.(userId)
  }, [userId, onSelect])

  return (
    <article aria-labelledby={labelId} onClick={handleClick}>
      {avatarUrl && <img src={avatarUrl} alt="" aria-hidden="true" />}
      <h3 id={labelId}>{displayName}</h3>
      {children}
    </article>
  )
}
```

---

## Custom Hook Pattern

```tsx
import { useCallback, useEffect, useRef, useState } from 'react'

interface UseDebounceOptions {
  delay?: number
}

export function useDebounce<T>(value: T, { delay = 300 }: UseDebounceOptions = {}): T {
  const [debounced, setDebounced] = useState<T>(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debounced
}
```

```tsx
// useToggle
export function useToggle(initial = false): [boolean, () => void, (v: boolean) => void] {
  const [state, setState] = useState(initial)
  const toggle = useCallback(() => setState((v) => !v), [])
  return [state, toggle, setState]
}
```

```tsx
// useLocalStorage
export function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key)
      return item ? (JSON.parse(item) as T) : initialValue
    } catch {
      return initialValue
    }
  })

  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      setStoredValue((prev) => {
        const next = typeof value === 'function' ? (value as (p: T) => T)(prev) : value
        window.localStorage.setItem(key, JSON.stringify(next))
        return next
      })
    },
    [key],
  )

  return [storedValue, setValue] as const
}
```

---

## Context API Pattern

```tsx
import { createContext, useContext, useMemo, type ReactNode } from 'react'

interface ThemeContextValue {
  theme: 'light' | 'dark'
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextValue | null>(null)

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, toggleTheme] = useToggle(false)   // reuse custom hook

  const value = useMemo<ThemeContextValue>(
    () => ({ theme: theme ? 'dark' : 'light', toggleTheme }),
    [theme, toggleTheme],
  )

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}

export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext)
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider')
  return ctx
}
```

---

## forwardRef Component

```tsx
import { forwardRef, type InputHTMLAttributes } from 'react'
import { cn } from '@/lib/utils'

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label: string
  error?: string
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, className, ...props }, ref) => {
    const id = useId()
    return (
      <div className="flex flex-col gap-1">
        <label htmlFor={id} className="text-sm font-medium">
          {label}
        </label>
        <input
          ref={ref}
          id={id}
          aria-invalid={!!error}
          aria-describedby={error ? `${id}-error` : undefined}
          className={cn(
            'rounded border px-3 py-2 text-sm outline-none',
            error ? 'border-red-500' : 'border-gray-300',
            className,
          )}
          {...props}
        />
        {error && (
          <p id={`${id}-error`} role="alert" className="text-xs text-red-600">
            {error}
          </p>
        )}
      </div>
    )
  },
)
Input.displayName = 'Input'
```

---

## Compound Component Pattern

```tsx
import {
  createContext,
  useContext,
  type ReactNode,
  type ButtonHTMLAttributes,
} from 'react'

interface DropdownContextValue {
  isOpen: boolean
  toggle: () => void
}

const DropdownContext = createContext<DropdownContextValue | null>(null)

function useDropdownContext() {
  const ctx = useContext(DropdownContext)
  if (!ctx) throw new Error('Must be used inside <Dropdown>')
  return ctx
}

function Dropdown({ children }: { children: ReactNode }) {
  const [isOpen, toggle] = useToggle()
  return (
    <DropdownContext.Provider value={{ isOpen, toggle }}>
      <div className="relative inline-block">{children}</div>
    </DropdownContext.Provider>
  )
}

function Trigger({ children, ...props }: ButtonHTMLAttributes<HTMLButtonElement>) {
  const { toggle } = useDropdownContext()
  return (
    <button type="button" onClick={toggle} {...props}>
      {children}
    </button>
  )
}

function Menu({ children }: { children: ReactNode }) {
  const { isOpen } = useDropdownContext()
  if (!isOpen) return null
  return (
    <ul role="menu" className="absolute z-10 mt-1 rounded border bg-white shadow-md">
      {children}
    </ul>
  )
}

// Attach sub-components
Dropdown.Trigger = Trigger
Dropdown.Menu = Menu
export { Dropdown }
```

---

## React.memo with Custom Comparison

```tsx
import { memo } from 'react'

interface ListItemProps {
  id: string
  label: string
  selected: boolean
  onToggle: (id: string) => void
}

export const ListItem = memo(
  ({ id, label, selected, onToggle }: ListItemProps) => (
    <li>
      <button
        type="button"
        aria-pressed={selected}
        onClick={() => onToggle(id)}
        className={selected ? 'font-bold' : ''}
      >
        {label}
      </button>
    </li>
  ),
  (prev, next) =>
    prev.id === next.id &&
    prev.label === next.label &&
    prev.selected === next.selected,
  // onToggle excluded: caller wraps in useCallback
)
```

---

## Lazy Loading with Suspense

```tsx
import { lazy, Suspense } from 'react'
import { Spinner } from '@/components/Spinner'

const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner label="Loading chart…" />}>
      <HeavyChart />
    </Suspense>
  )
}
```

---

## Error Boundary (Class component — required by React)

```tsx
import { Component, type ErrorInfo, type ReactNode } from 'react'

interface Props {
  fallback: ReactNode
  children: ReactNode
}

interface State {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error('Uncaught error:', error, info)
  }

  render() {
    if (this.state.hasError) return this.props.fallback
    return this.props.children
  }
}
```

---

## useReducer for Complex Local State

```tsx
type Action =
  | { type: 'SET_LOADING' }
  | { type: 'SET_DATA'; payload: string[] }
  | { type: 'SET_ERROR'; payload: string }

interface State {
  status: 'idle' | 'loading' | 'success' | 'error'
  data: string[]
  error: string | null
}

const initialState: State = { status: 'idle', data: [], error: null }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'SET_LOADING':
      return { ...state, status: 'loading', error: null }
    case 'SET_DATA':
      return { status: 'success', data: action.payload, error: null }
    case 'SET_ERROR':
      return { ...state, status: 'error', error: action.payload }
  }
}

export function useAsyncData() {
  const [state, dispatch] = useReducer(reducer, initialState)
  // ...fetch logic dispatching actions
  return state
}
```
