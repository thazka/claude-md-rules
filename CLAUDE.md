@AGENTS.md

# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Development Commands

### Package Manager

This project uses **npm**.

```bash
# Install dependencies
npm install

# Development server (http://localhost:3000)
npm run dev

# Production build
npm run build

# Start production server
npm run start

# Linting
npm run lint
```

## Architecture Overview

### Core Technology Stack

- **Framework:** Next.js 16 with App Router
- **React Version:** 19.2.x
- **Language:** TypeScript (strict mode)
- **UI Components:** HeroUI v3 (built on React Aria)
- **Styling:** Tailwind CSS v4
- **Server State:** TanStack Query v5 (caching, background refetching)
- **Client State:** Zustand v5 (lightweight, no Redux)
- **HTTP Client:** Axios v1 with interceptors
- **Validation:** Zod v4 schemas
- **Auth:** Auth.js v5 (JWT + httpOnly cookies)
- **Charts:** Highcharts v12 + highcharts-react-official (only chart library — no recharts/D3)

### End-to-End Architecture

```
Cloudflare CDN → Next.js Frontend (SSR/SSG)
  → HTTPS REST API → AWS ALB / Railway
  → NestJS Modular Monolith
  → PostgreSQL + Redis + S3
```

### API Integration Pattern

Auth and data APIs are served by the paired FastAPI backend in `C:\backend\apps\apps-backend`.

```
Components → Hooks (TanStack Query) → Services → API Client (Axios) → Backend
```

### Authentication Architecture

1. **Session Storage:** Auth.js v5 uses JWT session strategy with an encrypted httpOnly cookie.
2. **Refresh Token Boundary:** Backend refresh tokens are server-only. They may exist in the Auth.js JWT callback token, but must never be returned from `session()` or exposed through `useSession()` / `/api/auth/session`.
3. **Client Session Shape:** Browser-visible session data is limited to `accessToken`, `accessTokenExpiry`, `error`, and minimal `user` fields.
4. **API Client:** `SessionSyncer` copies only the short-lived access token into `token-store`; Axios injects it as `Authorization: Bearer <accessToken>`.
5. **Refresh Flow:** Access-token refresh happens in the Auth.js `jwt` callback with single-flight protection so parallel session checks do not rotate the same refresh token twice.
6. **Logout Flow:** Logout reads the refresh token server-side from the Auth.js JWT cookie, calls backend `/auth/logout` to revoke it, then calls Auth.js `signOut` to clear the local cookie.
7. **Auto-Logout on 401:** Response interceptor clears the local access token and redirects to `/auth/login`.
8. **Protected Routes:** Proxy at `src/proxy.ts` protects dashboard routes. In Next.js 16 this is `proxy.ts`, not `middleware.ts`.
9. **Backend Login Protection:** FastAPI applies Redis-backed login throttling by IP + normalized email before expensive password verification.
10. **Route Groups:** `auth/` contains public auth routes, `(dashboard)/` contains protected dashboard routes.

### Auth Deployment Guidance

**Development**
- `AUTH_URL` should point to the local frontend, usually `http://localhost:3000`.
- `NEXT_PUBLIC_API_URL` should point to the local backend API.
- Debug auth logs are allowed, but never print passwords, access tokens, refresh tokens, or raw auth response bodies.
- Redis may be local or mocked for backend tests.

**Staging**
- Use staging-only `AUTH_SECRET`, `AUTH_URL`, backend `APP_SECRET_KEY`, and Redis.
- Set backend `CORS_ORIGINS` to the exact staging frontend origin.
- Run smoke tests for login, token refresh after expiry, logout revoke, disabled-user rejection, rate-limit hit, and superadmin route guard.

**Production**
- Use strong stable secrets; never reuse dev or staging secrets.
- HTTPS is mandatory; `AUTH_URL` must be the final production origin.
- Backend `CORS_ORIGINS` must be explicit, not `*`.
- Keep `CORS_ALLOW_CREDENTIALS=false` while API auth is bearer-token based.
- Enable Redis-backed login rate limiting and monitor failed login, rate-limit hit, refresh failure, and logout revoke failure.
- If `ENABLE_LOGIN_VERIFICATION=true`, trust `X-Forwarded-For` only from configured reverse proxy IPs.

### Folder Structure (Feature-Sliced)

```
src/
├── app/
│   ├── (auth)/                  # public auth routes (login, register)
│   ├── (dashboard)/             # protected routes with shared layout
│   │   └── [feature]/
│   │       ├── page.tsx         # route entry (Server Component)
│   │       └── layout.tsx
│   ├── layout.tsx               # root layout
│   └── page.tsx
├── components/
│   ├── base/                    # reusable components (used in 2+ features)
│   │   ├── index.ts             # barrel: re-exports all sub-folders
│   │   ├── layout/              # DashboardShell, AppNavbar, AppSidebar
│   │   ├── filters/             # all filter controls
│   │   ├── charts/              # all Highcharts components
│   │   ├── data-display/        # PageTitle, EntityRankingCard
│   │   └── icons/               # custom SVG icons
│   └── [feature]/               # feature-specific components
│       ├── index.ts
│       ├── FeatureContainer.tsx
│       ├── FeatureList.tsx
│       └── FeatureForm.tsx
├── config/
│   ├── constants.ts             # all app-wide constants (NEVER inline constants in components)
│   └── site.config.ts           # app identity: name, description, URL, author, pageMetadata
├── data/                        # all mock/dummy data files (JSON or TS)
│   └── [feature]/               # grouped by feature domain
├── hooks/
│   ├── queries/                 # TanStack Query GET hooks
│   └── mutations/               # TanStack Query POST/PUT/DELETE hooks
├── lib/
│   ├── utils/                   # pure utility functions
│   └── validations.ts           # all Zod schemas
├── providers/
│   ├── query-provider.tsx       # TanStack Query provider
│   └── index.tsx                # root providers composition
├── services/
│   ├── client.ts                # Axios instance + interceptors
│   ├── endpoints.ts             # centralized endpoint constants
│   ├── modules/                 # service modules per domain
│   │   ├── index.ts
│   │   ├── auth.service.ts
│   │   └── [feature].service.ts
│   └── adapters/                # data transformation adapters
│       └── [feature].adapter.ts
├── stores/
│   ├── index.ts
│   └── [feature].store.ts      # Zustand stores
└── types/
    ├── index.ts                 # re-export all types
    ├── auth.types.ts
    ├── chart.types.ts           # chart data types (TimeSeriesPoint, PieSlice, etc.)
    ├── common.types.ts
    └── [feature].types.ts
```

### Path Aliases

TypeScript path mappings configured in `tsconfig.json`:

- `@/*` → `./src/*`

Always use `@/` instead of relative imports.

### Environment Variables

```env
# Public (exposed to client)
NEXT_PUBLIC_API_URL=          # backend REST API base URL

# Server-only (never expose to client)
AUTH_SECRET=                  # Auth.js secret (openssl rand -base64 32)
AUTH_URL=                     # canonical URL for Auth.js
```

Never prefix server secrets with `NEXT_PUBLIC_`.

## Before Building a New Component

Follow this decision flow every time you need a new component:

```
Need a new component?
        │
        ▼
1. Does something in base/ already do this (or close to it)?
   → YES: Use it. Extend via props. Do NOT build a duplicate.
   → NO: continue ↓

2. Will this be used in 2+ features, OR is it a generic UI pattern
   (list card, chart, filter, layout shell)?
   → YES: Build in base/[sub-folder]/, keep it data-agnostic
   → NO: continue ↓

3. Build it in components/[feature]/ as a feature-specific component.
   Revisit if a second feature needs the same pattern — extract to base/ then.
```

**Quick lookup:**
- Need a chart? → Check `base/charts/`. Specific chart type exists → use it.
- Need a tabbed ranked list? → `base/data-display/EntityRankingCard` exists → use it.
- Need a filter? → Check `base/filters/`. `MultiSelectFilter` is the generic primitive.
- Need a layout wrapper? → Check `base/layout/`.
- Need a component only for one feature? → Build in `components/[feature]/` directly.

## `base/` Sub-Folder Structure

All reusable components live in `src/components/base/` organized into five sub-folders.

| Sub-folder | Purpose | Add here when... |
|---|---|---|
| `layout/` | App shell chrome | Component arranges page regions; used once per render; reads from UIStore/TopicStore |
| `filters/` | Filter controls | Component controls a filterable dimension with `value`/`onChange`; or composes filters |
| `charts/` | Highcharts visualizations | Component wraps a Highcharts chart, calls `applyAppsChartTheme()`, accepts plain data arrays |
| `data-display/` | Presentational components | HeroUI + Tailwind only, used in 2+ features, no direct API hooks |
| `icons/` | Custom SVG icons | Custom-drawn icon not available in `lucide-react` |

**New sub-folder threshold:** Only create one when 3+ components share a concern that genuinely doesn't fit the existing five.

**Barrel pattern:** Every sub-folder has `index.ts`. Root `base/index.ts` re-exports all:

```typescript
export * from './layout'
export * from './filters'
export * from './charts'
export * from './data-display'
export * from './icons'
```

Always import from `@/components/base` — never from sub-folder paths directly.

## Chart Components (`base/charts/`)

**Only Highcharts is used.** Do not add recharts, D3, or any other chart library.

### Available charts

| Component | Highcharts type | Dynamic module? | Use for |
|---|---|---|---|
| `LineChart` | `line` | No | Multi-source time-series with mode toggle |
| `WordCloudChart` | `wordcloud` | Yes | Keyword/hashtag clouds |
| `DonutChart` | `pie` + `innerSize` | No | Ring/donut with optional center label |
| `PieChart` | `pie` | No | Full pie with data labels |
| `BarChart` | `bar` (horizontal) | No | Horizontal ranked bars |
| `ColumnChart` | `column` (vertical) | No | Vertical bars, optional stacking |
| `BarChartLine` | `column` + `line` mixed | No | Stacked bars with a line overlay |
| `AreaChart` | `area` / `areaspline` | No | Area fill, optional stacking/smooth |
| `StreamgraphChart` | `streamgraph` | Yes | Flowing stacked area (stream effect) |
| `ChartContainer` | (wrapper) | No | Loading/error/title wrapper for any chart |
| `AnalysisChart` | `line` | No | **@deprecated** — use `LineChart` instead |

### Rules for every chart component

1. Add `'use client'` at the top.
2. Call `applyAppsChartTheme()` unconditionally at component body start (it is idempotent).
3. Declare `const chartRef = useRef<HighchartsReact.RefObject>(null)`.
4. Build Highcharts options in `useMemo` with all reactive values in the dep array.
5. Do NOT add chart-type-specific `plotOptions` to `chart-theme.ts` — override inside `useMemo`.
6. Do NOT hard-code colors — use `CHART_COLOR_SEQUENCE` or per-item `color` prop.
7. Loading skeleton: `<div className="rounded-lg animate-pulse" style={{ height, background: 'var(--nav-overlay)' }} />`

### Dynamic module loading pattern (WordCloudChart, StreamgraphChart)

For any Highcharts module that mutates `SeriesRegistry` at import time:

```typescript
const [ready, setReady] = useState(false)

useEffect(() => {
  let cancelled = false
  import('highcharts/modules/[module-name]').then((mod) => {
    if (cancelled) return
    const init = (mod.default ?? mod) as unknown as (hc: typeof Highcharts) => void
    if (typeof init === 'function') init(Highcharts)
    setReady(true)
  })
  return () => { cancelled = true }
}, [])

if (isLoading || !ready) return <div className="rounded-lg animate-pulse" style={{ height, background: 'var(--nav-overlay)' }} />
```

### Chart data types (`src/types/chart.types.ts`)

- **`TimeSeriesPoint`** — `{ date: string; [seriesKey: string]: number | string }` — for Line, Area, Streamgraph, BarChartLine
- **`ChartSeries`** — `{ key: string; name: string; color?: string }` — series config
- **`PieSlice`** — `{ name: string; y: number; color?: string }` — for DonutChart, PieChart
- **`BarEntry`** — `{ category: string; value: number; color?: string }` — for BarChart
- **`StackedBarPoint`** — `{ date: string; [stackKey: string]: number | string }` — for ColumnChart, BarChartLine
- **`StackedBarSeries`** — `{ key: string; name: string; color?: string }` — stacked bar series config
- **`OverlayLineSeries`** — `{ key: string; name: string; color?: string }` — line overlay config for BarChartLine

### Feature-specific charts (do NOT move to `base/charts/`)

If a chart bakes in feature-specific series config or UI chrome (e.g., `DashboardTrendChart` with its hard-coded series array and tab controls), it stays in the feature folder.

## EntityRankingCard (Generic Tabbed Ranked List)

`src/components/base/data-display/EntityRankingCard.tsx`

A generic render-props card: tabs + ranked list + footer. Used by `TopEntitiesCard` and `OriginatorCard`.

### Props

```typescript
interface EntityRankingCardProps<TTab extends string, TRow> {
  tabs: RankingCardTab<TTab>[]    // tab pills
  activeTab: TTab
  onTabChange: (tab: TTab) => void

  rows: TRow[] | undefined        // undefined = not yet loaded
  isLoading: boolean
  skeletonCount?: number          // default 5

  renderRow: (row: TRow) => React.ReactNode  // row layout is caller's responsibility
  getRowKey: (row: TRow) => string | number

  onDownload?: () => void         // renders Download button if provided
  viewAllHref?: string            // renders View All link if provided
  viewAllLabel?: string           // default 'View All'
  className?: string
}
```

### Usage pattern

The card owns chrome (tabs, skeleton, footer). The **row renderer** owns the row layout. Keep row sub-components (`EntityRow`, `OriginatorRow`) as private unexported functions in the feature file.

```typescript
// src/components/dashboard/TopEntitiesCard.tsx
export function TopEntitiesCard() {
  const [activeTab, setActiveTab] = useState<EntityCategory>('person')
  const { data: entities, isLoading } = useTopEntities(activeTab)

  return (
    <EntityRankingCard
      tabs={TABS}
      activeTab={activeTab}
      onTabChange={setActiveTab}
      rows={entities}
      isLoading={isLoading}
      renderRow={(entity) => <EntityRow entity={entity} />}
      getRowKey={(entity) => entity.rank}
      onDownload={() => {}}
      viewAllHref="/dashboard/entities"
    />
  )
}

// Private — not exported
function EntityRow({ entity }: { entity: TopEntity }) { ... }
```

## Best Practices for Adding New Features

Before implementing, identify what you need:

- [ ] Requires API calls? → Create service module + endpoint + hooks
- [ ] Needs shared client state? → Add Zustand store
- [ ] Needs server state caching? → Use TanStack Query
- [ ] Involves forms? → Create Zod schema in `src/lib/validations.ts`
- [ ] Transforms external data? → Create adapter in `src/services/adapters/`
- [ ] New UI component? → Check `base/` first, then HeroUI v3 via MCP
- [ ] Requires new behavior? → Add tests for every touched layer before the feature is considered done

### Testing Definition of Done

Every new feature or behavior change must include the relevant tests:

- Logic/helper/schema/adapter changes require unit tests.
- TanStack Query hooks or service modules require integration tests with MSW.
- Forms, modals, filters, and client components require React Testing Library tests.
- New routes or user flows require at least one Playwright smoke test.
- Auth, permission, or protected-route changes require E2E or integration coverage.
- Adapters with non-trivial transformation require raw API fixture coverage from `src/data/[feature]/`.
- Zod schemas must cover valid and invalid payloads.

Use `npm run verify` before handoff. For E2E, use mock API responses by default; keep real backend/staging smoke tests small and explicit.

Testing file placement:

- Unit, component, and integration tests live in `src/__tests__/[feature]/`.
- Playwright tests live in `tests/e2e/`.
- Shared test setup and mocks live in `src/test/`.
- Test fixtures live in `src/data/[feature]/`.

### 1. Type-First Development

Always define types before implementation.

**ALL exported types/interfaces MUST go to `src/types/`:**

```typescript
// ✅ CORRECT
// src/types/report.types.ts
export interface Report {
  id: string
  title: string
  createdAt: Date
}

export interface ReportState {
  reports: Report[]
  selectedId: string | null
}

// src/types/index.ts
export * from './report.types'
```

```typescript
// ❌ WRONG — exported from wrong location
// src/stores/reportStore.ts
export interface Report { ... }  // must be in src/types/

// src/services/adapters/report.adapter.ts
export interface ReportDTO { ... }  // must be in src/types/
```

**Organization by domain:**

```
src/types/
├── index.ts           # re-export all
├── auth.types.ts
├── chart.types.ts     # chart data shapes
├── common.types.ts    # ApiError, PaginatedResponse, RankingCardTab, etc.
└── [feature].types.ts
```

### 2. Service Module Pattern

```typescript
// src/services/modules/report.service.ts
import api from '@/services/client'
import { API_ENDPOINTS } from '@/services/endpoints'
import type { Report, CreateReportRequest } from '@/types'

export const reportService = {
  async getAll(): Promise<Report[]> {
    const { data } = await api.get(API_ENDPOINTS.REPORTS.GET_ALL)
    return data
  },

  async create(payload: CreateReportRequest): Promise<Report> {
    const { data } = await api.post(API_ENDPOINTS.REPORTS.CREATE, payload)
    return data
  },

  async delete(id: string): Promise<void> {
    await api.delete(API_ENDPOINTS.REPORTS.DELETE(id))
  },
}
```

**Rules:**
- Always use the imported `api` instance, never create new axios instances
- Always type request parameters and return values
- Add endpoints to `src/services/endpoints.ts` first
- Export from `src/services/modules/index.ts`

### 3. TanStack Query Hook Pattern

**Queries (GET):**

```typescript
// src/hooks/queries/useGetReports.ts
import { useQuery } from '@tanstack/react-query'
import { reportService } from '@/services/modules'
import type { Report } from '@/types'

export const useGetReports = () => {
  return useQuery<Report[]>({
    queryKey: ['reports'],
    queryFn: reportService.getAll,
  })
}
```

**Mutations (POST/PUT/DELETE):**

```typescript
// src/hooks/mutations/useCreateReport.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { reportService } from '@/services/modules'

export const useCreateReport = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: reportService.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['reports'] })
    },
  })
}
```

**Rules:**
- Use descriptive query keys that match the resource name
- Always invalidate related queries on mutation success
- Handle loading and error states in the component

### 4. Zustand Store Pattern

Only create stores for client-side state that:
- Is shared across multiple components
- Should persist across navigation
- Is not server data (server data → TanStack Query)

```typescript
// src/stores/ui.store.ts
import { create } from 'zustand'
import type { UIState } from '@/types'

export const useUIStore = create<UIState>()((set) => ({
  sidebarOpen: false,
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
}))
```

**With persistence:**

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import type { UserPreferencesState } from '@/types'

export const useUserPreferencesStore = create<UserPreferencesState>()(
  persist(
    (set) => ({
      theme: 'light',
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'user-preferences',
      partialize: (state) => ({ theme: state.theme }),
    }
  )
)
```

### 5. Form Validation with Zod

**Add schemas to `src/lib/validations.ts`:**

```typescript
import { z } from 'zod'

export const createReportSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100),
  description: z.string().optional(),
  startDate: z.date(),
  endDate: z.date(),
})

export type CreateReportData = z.infer<typeof createReportSchema>
```

### 6. Data Transformation Strategy

**Simple (1-2 lines) — inline in service:**

```typescript
async getReport(id: string): Promise<Report> {
  const { data } = await api.get(endpoint)
  return { ...data, createdAt: new Date(data.created_at) }
}
```

**Complex — dedicated adapter:**

```typescript
// src/services/adapters/report.adapter.ts
import type { ReportDTO, Report } from '@/types'

export function adaptReport(dto: ReportDTO): Report {
  return {
    id: dto.id,
    title: dto.title,
    createdAt: new Date(dto.created_at),
    author: `${dto.first_name} ${dto.last_name}`.trim(),
    tags: dto.tags?.map((t) => t.toLowerCase()) ?? [],
  }
}
```

**Use adapter when:**
- Transformation is 5+ lines
- Reused across multiple endpoints
- Needs unit testing separately
- Converting fundamentally different structures

**Skip adapter when:**
- Backend returns exact format needed
- Only 1-2 field renames

### 7. Component Organization

```typescript
// Server Component (default)
// src/app/(dashboard)/reports/page.tsx
import { ReportContainer } from '@/components/report'

export default function ReportsPage() {
  return <ReportContainer />
}

// Client Component (only when needed)
// src/components/report/ReportForm.tsx
'use client'

import { useCreateReport } from '@/hooks/mutations/useCreateReport'

export function ReportForm() {
  const { mutate, isPending } = useCreateReport()
  // ...
}
```

### 8. Error Handling Strategy

**Services** — let errors propagate:

```typescript
async getData(): Promise<Data> {
  const { data } = await api.get(endpoint) // let axios throw
  return data
}
```

**Hooks** — handle and surface:

```typescript
onError: (error) => {
  const message = error.response?.data?.message ?? 'An error occurred'
  console.error('[useCreateReport]', error)
  // show toast or set error state
}
```

**Components** — render error states:

```typescript
const { data, isLoading, error } = useGetReports()

if (isLoading) return <Spinner />
if (error) return <ErrorState message={error.message} />
```

## Naming Conventions

### Files
- React components: PascalCase `.tsx` — `BarChartLine.tsx`, `EntityRankingCard.tsx`
- Non-component modules: kebab-case `.ts` — `chart-theme.ts`
- Type files: `[domain].types.ts` — `chart.types.ts`, `dashboard.types.ts`
- Barrel files: always `index.ts`

### Components
- Base (generic, reusable): `[Noun][Noun]` — `EntityRankingCard`, `ChartContainer`, `MultiSelectFilter`
- Feature-specific: `[Feature][Noun]` — `TopEntitiesCard`, `DashboardTrendChart`
- Chart components: `[ChartType]Chart` — `DonutChart`, `AreaChart`, `StreamgraphChart`
  - Exception for compound types: `BarChartLine` (no trailing "Chart")

### Props interfaces
- Always `[ComponentName]Props` — `DonutChartProps`, `EntityRankingCardProps`
- Always exported from `src/types/` — never from the component file itself

### Types
- Union strings: PascalCase — `ChartMode`, `EntityCategory`
- Data shapes: singular noun — `PieSlice`, `BarEntry`, `ChartSeries`
- Config shapes: noun + purpose — `RankingCardTab`, `StackedBarSeries`
- No `I` prefix on interfaces

## Common Patterns

### Adding a New Protected Route

1. Add folder under `src/app/(dashboard)/your-route/`
2. Create `page.tsx` as Server Component
3. Middleware automatically protects the route

### Adding a New API Service

1. Define types in `src/types/[feature].types.ts`, export from `src/types/index.ts`
2. Add endpoints in `src/services/endpoints.ts`
3. Create service in `src/services/modules/[feature].service.ts`
4. Export from `src/services/modules/index.ts`
5. Create hooks in `src/hooks/queries/` or `src/hooks/mutations/`

### Adding a New Zustand Store

1. Define state interface in `src/types/[feature].types.ts`
2. Create store in `src/stores/[feature].store.ts`
3. Export from `src/stores/index.ts`

## Constants & Mock Data

### Site Config (`src/config/site.config.ts`)

App identity and per-page metadata live in `src/config/site.config.ts`. Two exports:

- **`siteConfig`** — app name, fullName, description, URL, author, keywords, theme defaults
- **`pageMetadata`** — `{ title, description }` per page, consumed by each `page.tsx` via `export const metadata`

```typescript
// src/app/(dashboard)/dashboard/page.tsx
import { pageMetadata } from '@/config/site.config'
export const metadata: Metadata = pageMetadata.dashboard
```

```typescript
// src/app/layout.tsx
import { siteConfig } from '@/config/site.config'
export const metadata: Metadata = {
  title: { default: siteConfig.name, template: `%s | ${siteConfig.name}` },
  description: siteConfig.description,
  // ...
}
```

When adding a new page, add a corresponding key to `pageMetadata` in `site.config.ts` first.

### Constants (`src/config/constants.ts`)

All app-wide constants **must** go in `src/config/constants.ts`. Never define constants inline in components, hooks, or services.

```typescript
// ✅ CORRECT
// src/config/constants.ts
export const PAGINATION_DEFAULTS = {
  PAGE_SIZE: 20,
  MAX_PAGES: 100,
} as const

export const DATE_FORMAT = 'DD MMM YYYY' as const
```

```typescript
// ❌ WRONG — constant defined inline in a component
export function ArticleFeed() {
  const PAGE_SIZE = 20  // must be in src/config/constants.ts
}
```

### Mock / Dummy Data (`src/data/`)

All mock and dummy data **must** live under `src/data/[feature]/`. Never inline mock data inside components, hooks, or services.

```
src/data/
├── social-network/
│   └── sna.json
└── [feature]/
    └── mock-[resource].ts   # or .json
```

```typescript
// ✅ CORRECT
// src/data/search/mock-articles.ts
export const MOCK_ARTICLES = [
  { id: '1', title: 'Article One', ... },
]

// in component:
import { MOCK_ARTICLES } from '@/data/search/mock-articles'
```

```typescript
// ❌ WRONG — dummy data defined inline
export function ArticleFeed() {
  const articles = [{ id: '1', title: 'Article One' }]  // must be in src/data/
}
```

## Anti-Patterns to Avoid

**DON'T:**
- ❌ Use `any` type
- ❌ Import axios directly — use `@/services/client`
- ❌ Put API URLs in components — use `API_ENDPOINTS`
- ❌ Store server data in Zustand — use TanStack Query
- ❌ Use relative imports — use `@/` path alias
- ❌ Use arbitrary Tailwind classes like `min-w-[80px]`; prefer design token classes, semantic utilities, or reusable component styles
- ❌ Export types from stores, adapters, or components
- ❌ Add `'use client'` to layouts or pages unless truly needed
- ❌ Skip loading and error state handling
- ❌ Create unnecessary adapters for trivial transformations
- ❌ Use HeroUI v2 patterns or docs
- ❌ Add new chart libraries — only Highcharts
- ❌ Import from `@/components/base/[sub-folder]` directly — always import from `@/components/base`
- ❌ Add `plotOptions` for new chart types to `chart-theme.ts` — put overrides in the component's `useMemo`
- ❌ Import Highcharts modules at the top level in chart components — use the dynamic import pattern
- ❌ Define constants inline in components or outside `src/config/constants.ts`
- ❌ Hardcode app name, description, or page titles in `layout.tsx` or `page.tsx` — use `site.config.ts`
- ❌ Put mock/dummy data inline in components or outside `src/data/`

**DO:**
- ✅ Define types before implementation
- ✅ Follow the layered architecture (Component → Hook → Service → API)
- ✅ Fetch HeroUI component docs via MCP before implementing
- ✅ Check `base/` before building any new component
- ✅ Use Server Components by default
- ✅ Invalidate TanStack Query cache after mutations
- ✅ Use Zod for all user input validation
- ✅ Keep `src/types/` as single source of truth for all exported types
- ✅ Use named exports everywhere (no default exports for components)
- ✅ Put all constants in `src/config/constants.ts`
- ✅ Put app identity and page metadata in `src/config/site.config.ts`
- ✅ Put all mock/dummy data in `src/data/[feature]/`
