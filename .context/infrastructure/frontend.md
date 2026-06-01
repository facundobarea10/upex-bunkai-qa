# Infrastructure — Frontend

> **Discovery date**: 2026-05-31  
> **Sources**: `next.config.ts`, `tsconfig.json`, `package.json`, `app/layout.tsx`, `components/`, `DESIGN.md`

---

## Runtime

| Property | Value |
|---|---|
| **Framework** | Next.js 15 (App Router) |
| **UI library** | React 19 |
| **Language** | TypeScript 5.8 (strict mode) |
| **Bundler** | Next.js built-in (Turbopack dev / Webpack prod) |
| **Package manager** | Bun |
| **Component library** | shadcn/ui (Radix UI primitives) |
| **Styling** | Tailwind CSS 3.4 + CSS custom properties (design tokens) |

---

## Build Configuration (`next.config.ts`)

```typescript
// next.config.ts (key settings)
reactStrictMode: true
experimental: {
  typedRoutes: true    // type-safe <Link href="..."> — catches bad URLs at compile time
}
// No standalone mode, no rewrites, no custom headers
// remotePatterns: [] — no external image domains configured
```

---

## TypeScript Configuration (`tsconfig.json`)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "incremental": true,
    "jsx": "preserve",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "types": ["bun-types", "node"]
  },
  "paths": {
    "@/*":           ["./*"],
    "@app/*":        ["./app/*"],
    "@components/*": ["./components/*"],
    "@lib/*":        ["./lib/*"]
  }
}
```

**Key flags**:
- `typedRoutes: true` (Next.js config) — compile-time route safety
- `strict: true` — no implicit any, strict null checks, etc.
- `allowImportingTsExtensions: true` — Bun-native TS import support
- `moduleResolution: bundler` — optimized for Next.js/Bun bundler resolution

---

## App Router Structure

```
app/
├── layout.tsx              — Root layout: fonts (Inter, JetBrains Mono, Noto Serif JP), dark HTML class, Sonner toast
├── page.tsx                — Home: redirect('/login')
├── globals.css             — Tailwind base + CSS design tokens
│
├── (app)/                  — Protected route group (requires auth via middleware)
│   ├── layout.tsx          — App shell: AuthProvider context
│   ├── onboarding/         — Workspace creation form
│   ├── projects/[projectSlug]/
│   │   ├── page.tsx        — Project view: module tree + ATC table
│   │   └── atcs/[atcId]/
│   │       ├── page.tsx    — ATC editor page (server component)
│   │       └── actions.ts  — saveAtcAction (Next.js Server Action)
│   └── workspaces/[id]/members/
│       ├── page.tsx        — Members + invites management
│       └── members-client.tsx — Client mutations
│
├── (auth)/                 — Public route group
│   └── login/
│       ├── page.tsx        — Login page (magic-link form + brand story)
│       └── magic-link-form.tsx — Email OTP client component
│
├── api/                    — Backend API (see infrastructure/backend.md)
│   ├── v1/                 — REST API v1
│   ├── auth/callback/      — OTP exchange handler
│   ├── openapi/            — OpenAPI spec endpoint
│   └── docs/               — Scalar interactive docs
│
├── auth/callback/          — Magic-link OTP exchange (browser redirect target)
├── invites/accept/         — Invite redemption landing
├── qa/                     — Public QA testing guide (no auth)
└── design-tokens/          — Design system reference (no auth)
```

---

## Component Architecture

### Rendering Pattern
- **Server Components (RSC)**: page.tsx files load data server-side (Supabase direct queries, no client fetch)
- **Client Components**: interactive components (`"use client"`) — AtcEditor, MembersClient, CommandPalette, WorkspaceSwitcher, MagicLinkForm
- **Server Actions**: `saveAtcAction` in `app/(app)/projects/[projectSlug]/atcs/[atcId]/actions.ts`

### Component Library (`components/`)

```
components/
├── atcs/
│   ├── AtcTable.tsx        — TanStack Table, paginated + filterable ATC list
│   ├── AtcEditor.tsx       — Monaco Editor wrapper for ATC steps/assertions
│   ├── StepEditor.tsx      — inline step row editor (content, input_data, expected)
│   └── AnchoringPanel.tsx  — story/AC multi-select picker
│
├── layout/
│   ├── Topbar.tsx          — workspace breadcrumb + command palette + action buttons
│   ├── Sidebar.tsx         — hierarchical module tree (recursive)
│   ├── CommandPalette.tsx  — Cmd+K overlay (cmdk library)
│   ├── WorkspaceSwitcher.tsx — workspace dropdown
│   └── Wordmark.tsx        — Bunkai logo (kanji + Latin)
│
├── providers/
│   └── auth-context.tsx    — AuthProvider (React context: session + user)
│
└── ui/                     — shadcn/ui primitives
    button, input, label, card, tabs, badge
    dialog, dropdown-menu, tooltip
```

---

## Design System

### Typography
| Font | Source | Usage |
|---|---|---|
| Inter | Google Fonts | Body text, UI prose |
| JetBrains Mono | Google Fonts | IDs (`ATC-001`, `RUN-451`) — all monospace contexts |
| Noto Serif JP | Google Fonts | Brand wordmark kanji (分解) |

Loaded in `app/layout.tsx` via `next/font/google`. Dark mode forced: `<html lang="en" class="dark">`.

### Color Tokens (CSS custom properties, `globals.css`)
```css
/* Surfaces */
--bg-0: #0a0b0d;   /* page background — deepest */
--bg-1: #101216;   /* sidebar / chrome */
--bg-2: #14171c;   /* default card / panel */
--bg-3: #1a1e25;   /* elevated / focused input */
--bg-4: #232830;   /* hover state */
--bg-5: #2d333c;   /* active / pressed */

/* Text */
--fg-0: #f1f3f5;   /* titles, primary actions */
--fg-1: #d4d8de;   /* default body text */
--fg-2: #9aa1ab;   /* secondary */
--fg-3: #6b727c;   /* muted */
--fg-4: #4a5057;   /* disabled / placeholder */

/* Accent */
--accent: #d9543f;  /* vermillion — primary actions, focus rings */

/* Status (ATC) */
--pass:    #2fb673;  /* green */
--fail:    #e5484d;  /* red */
--blocked: #e8a838;  /* amber */
--skipped: #8a91a0;  /* gray */
--running: #4f8cf7;  /* blue, pulsing */

/* Layer chips */
--layer-ui:   #8b6df0;  /* purple */
--layer-api:  #4f8cf7;  /* blue */
--layer-unit: #2fb673;  /* green */
```

### Design Principles (inform test assertions)
1. Information density — no extra whitespace; 4px base grid
2. Dark mode canonical — light mode is Phase 2
3. Monospace for IDs, sans-serif for prose
4. Status color always paired with text + icon (no color-only signals)
5. Sharp radii (max 10px), low elevation shadows

---

## Key Dependencies

| Package | Version | Purpose |
|---|---|---|
| `next` | ^15 | Framework |
| `react` / `react-dom` | ^19 | UI |
| `@supabase/ssr` | ^0.10.3 | Server-side Supabase client |
| `@supabase/supabase-js` | ^2.106 | Supabase JS client |
| `tailwindcss` | ^3.4 | Styling |
| `@radix-ui/*` | various | Accessible primitives (dialog, dropdown, tabs, tooltip) |
| `@monaco-editor/react` | ^4.7 | Code/text editor for ATC steps |
| `@tanstack/react-table` | ^8.21 | ATC list table |
| `cmdk` | ^1.1 | Command palette (Cmd+K) |
| `sonner` | ^2.0 | Toast notifications |
| `lucide-react` | ^1.16 | Icon library |
| `zod` | ^4.4 | Schema validation |
| `@asteasolutions/zod-to-openapi` | ^8.5 | OpenAPI spec from Zod schemas |
| `class-variance-authority` | ^0.7 | shadcn variant utility |
| `clsx` + `tailwind-merge` | latest | Class merging utilities |

---

## Test ID Strategy

No `data-testid` conventions found in Phase 1 source. Test selectors will rely on:
- Semantic HTML elements (buttons, inputs, links)
- ARIA roles and labels
- Text content (visible labels)
- Class-based selectors for custom design token classes

**Discovery Gap**: No documented test ID strategy — KATA test components will need to establish locator conventions.

---

## Discovery Gaps

- [ ] **Test ID conventions**: No `data-testid` or `data-test-*` attributes found in components. Playwright locators will use semantic selectors.
- [ ] **State management**: No global state library (Redux, Zustand, Jotai) detected. React context via `auth-context.tsx` is the only shared state. Component state is local.
- [ ] **Error boundary**: No `error.tsx` files found in App Router — no custom error UI for route-level errors.
- [ ] **Loading states**: No `loading.tsx` files found — Next.js default suspense behavior.
- [ ] **Internationalization**: No i18n library detected. English-only in Phase 1.
