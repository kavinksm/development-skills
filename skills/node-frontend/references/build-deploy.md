# Node Frontend — Build & Deploy Reference

## Vite Configuration

```ts
// vite.config.ts
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')

  return {
    plugins: [
      react(),
      tsconfigPaths(),   // honours tsconfig path aliases (@/...)
    ],

    resolve: {
      alias: {
        '@': '/src',
      },
    },

    server: {
      port: 5173,
      proxy: {
        '/api': {
          target: env.VITE_API_BASE_URL ?? 'http://localhost:8080',
          changeOrigin: true,
          rewrite: (path) => path.replace(/^\/api/, ''),
        },
      },
    },

    build: {
      outDir: 'dist',
      sourcemap: true,
      rollupOptions: {
        output: {
          manualChunks: {
            vendor: ['react', 'react-dom'],
            router: ['react-router-dom'],
            query: ['@tanstack/react-query'],
          },
        },
      },
    },

    preview: {
      port: 4173,
    },
  }
})
```

---

## TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },
    "skipLibCheck": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

---

## Environment Variables

```bash
# .env.example — commit this file
VITE_API_BASE_URL=http://localhost:8080/api
VITE_APP_TITLE=My App
VITE_SENTRY_DSN=

# .env.local — DO NOT COMMIT (git-ignored)
VITE_API_BASE_URL=http://localhost:8080/api
```

```ts
// All VITE_ vars are type-safe with vite/client
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
  readonly VITE_SENTRY_DSN?: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

---

## Dockerfile — Multi-Stage Build

```dockerfile
# Build stage
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_BASE_URL
ARG VITE_APP_TITLE
RUN npm run build

# Production stage — Nginx to serve SPA
FROM nginx:1.27-alpine AS runner
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback — all paths serve index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # No cache for index.html
    location = /index.html {
        add_header Cache-Control "no-store";
    }

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
}
```

---

## Docker Compose (dev + API together)

```yaml
# docker-compose.yml
services:
  frontend:
    build:
      context: .
      target: builder          # use builder stage for dev
    ports: ['5173:5173']
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - VITE_API_BASE_URL=http://localhost:8080/api
    command: npm run dev -- --host

  api:
    image: my-api:latest
    ports: ['8080:8080']
```

---

## GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run type-check
      - run: npm run lint
      - run: npm test -- --coverage

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  build:
    runs-on: ubuntu-latest
    needs: [test]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          build-args: |
            VITE_API_BASE_URL=${{ vars.VITE_API_BASE_URL }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "type-check": "tsc --noEmit",
    "lint": "eslint src --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint src --ext ts,tsx --fix",
    "format": "prettier --write src",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

---

## ESLint + Prettier

```js
// eslint.config.js (flat config)
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'

export default tseslint.config(
  { ignores: ['dist', 'coverage'] },
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    plugins: { 'react-hooks': reactHooks, 'react-refresh': reactRefresh },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    },
    languageOptions: {
      parserOptions: { project: true, tsconfigRootDir: import.meta.dirname },
    },
  },
)
```

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 100,
  "trailingComma": "all",
  "plugins": ["prettier-plugin-tailwindcss"]
}
```
