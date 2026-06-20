# Node Frontend — Styling Reference (Tailwind CSS + CSS Modules + cva)

## Tailwind CSS Setup

```bash
npm install tailwindcss @tailwindcss/forms @tailwindcss/typography
npx tailwindcss init -p
```

```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss'
import forms from '@tailwindcss/forms'
import typography from '@tailwindcss/typography'

export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
      },
      borderRadius: {
        DEFAULT: '0.5rem',
      },
    },
  },
  plugins: [forms, typography],
} satisfies Config
```

```css
/* src/styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --radius: 0.5rem;
  }
  body {
    @apply bg-white text-gray-900 antialiased;
  }
}
```

---

## cn Utility (clsx + tailwind-merge)

```bash
npm install clsx tailwind-merge
```

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs))
}
```

```tsx
// Usage — conflicting classes are resolved correctly
<div className={cn(
  'rounded px-4 py-2 text-sm',
  isError && 'border-red-500 text-red-700',
  isLarge && 'px-6 py-3 text-base',   // overrides px-4/py-2
  className,                           // caller overrides last
)} />
```

---

## Component Variants with cva

```bash
npm install class-variance-authority
```

```tsx
// src/components/Button/index.tsx
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'
import { type ButtonHTMLAttributes } from 'react'

const buttonVariants = cva(
  // base classes applied to all variants
  'inline-flex items-center justify-center rounded font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-brand-500 text-white hover:bg-brand-600 focus-visible:ring-brand-500',
        secondary: 'border border-gray-300 bg-white text-gray-700 hover:bg-gray-50',
        destructive: 'bg-red-600 text-white hover:bg-red-700 focus-visible:ring-red-500',
        ghost: 'text-gray-700 hover:bg-gray-100',
        link: 'text-brand-500 underline-offset-4 hover:underline',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  },
)

interface ButtonProps
  extends ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  loading?: boolean
}

export const Button = ({
  className,
  variant,
  size,
  loading,
  disabled,
  children,
  ...props
}: ButtonProps) => (
  <button
    className={cn(buttonVariants({ variant, size }), className)}
    disabled={disabled || loading}
    aria-busy={loading}
    {...props}
  >
    {loading && <span className="mr-2 h-4 w-4 animate-spin rounded-full border-2 border-current border-t-transparent" />}
    {children}
  </button>
)
```

---

## shadcn/ui Integration

```bash
npx shadcn@latest init          # configure components.json
npx shadcn@latest add button dialog input select table
```

```tsx
// shadcn components live in src/components/ui/
// Customize by editing the generated files directly — they are owned by your project
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog'
```

---

## CSS Modules

```tsx
// src/components/Badge/Badge.module.css
.badge {
  display: inline-flex;
  align-items: center;
  padding: 0.125rem 0.5rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 500;
}

.success { background-color: #dcfce7; color: #166534; }
.warning { background-color: #fef9c3; color: #854d0e; }
.danger  { background-color: #fee2e2; color: #991b1b; }
```

```tsx
// src/components/Badge/index.tsx
import styles from './Badge.module.css'
import { cn } from '@/lib/utils'

type Variant = 'success' | 'warning' | 'danger'

interface BadgeProps {
  variant?: Variant
  children: React.ReactNode
  className?: string
}

export const Badge = ({ variant = 'success', children, className }: BadgeProps) => (
  <span className={cn(styles.badge, styles[variant], className)}>
    {children}
  </span>
)
```

---

## Dark Mode

```tsx
// src/stores/themeStore.ts
export const useThemeStore = create<{
  theme: 'light' | 'dark'
  toggleTheme: () => void
}>()(
  persist(
    (set) => ({
      theme: 'light',
      toggleTheme: () =>
        set((s) => ({ theme: s.theme === 'light' ? 'dark' : 'light' })),
    }),
    { name: 'theme' },
  ),
)

// src/App.tsx — apply class to root
function App() {
  const theme = useThemeStore((s) => s.theme)
  return (
    <div className={theme}>
      <RouterProvider router={router} />
    </div>
  )
}

// Component usage
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
```

---

## Responsive Design Patterns

```tsx
// Mobile-first, Tailwind breakpoints: sm(640) md(768) lg(1024) xl(1280) 2xl(1536)

// Grid: 1 col on mobile, 2 on tablet, 3 on desktop
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">

// Hide on mobile, show on desktop
<nav className="hidden lg:flex">

// Stack on mobile, side-by-side on desktop
<div className="flex flex-col gap-4 md:flex-row">

// Clamp font size
<h1 className="text-2xl font-bold sm:text-3xl lg:text-4xl">
```

---

## Animation Patterns

```tsx
// Tailwind built-in animations
<div className="animate-spin" />      // loading spinner
<div className="animate-pulse" />     // skeleton loader
<div className="animate-bounce" />

// Skeleton loader component
export const Skeleton = ({ className }: { className?: string }) => (
  <div className={cn('animate-pulse rounded bg-gray-200 dark:bg-gray-700', className)} />
)

// Usage
<Skeleton className="h-4 w-3/4" />
<Skeleton className="h-10 w-full" />
```
