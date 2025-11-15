# Aurora Next.js Admin – Newbie Guide

Welcome! This document was written for front-end developers who know Vue.js but are touching Next.js (App Router) + Material UI for the first time. The goal is to tell you *exactly* what to install, what to read first, how folders map to features, and how to ship a new dashboard page without feeling lost.

---

## 0. 5‑Day Suggested Onboarding Path

| Day | Goal | Concrete TODOs |
| --- | --- | --- |
| 1 | Run the project locally | Install Node 18, copy `.env.local`, run `npm run dev`, log in with default credentials (`demo@aurora.com` / `password123`). |
| 2 | Understand layouts | Read `src/app/layout.tsx`, `src/app/(main)/layout.tsx`, and `src/layouts/main-layout/MainLayout.tsx`. Trace provider usage. |
| 3 | Rebuild a widget | Pick one card in `components/sections/dashboards/e-commerce` (e.g., `MonthlyProfit`). Recreate it in isolation or tweak data to see how props flow. |
| 4 | Add a simple page | Create `src/app/(main)/dashboard/new-widget/page.tsx` that renders your custom widget. Confirm routing works and nav highlights correctly. |
| 5 | Connect data | Replace a mock data source with a real API call using `services/axios/axiosInstance.ts` and add TypeScript types under `src/types`. |

This cadence keeps learning focused and avoids jumping into complex auth or SSR topics before the basics feel comfortable.

---

## 1. Prerequisites & Environment Setup

1. **Node.js 18+**  
   - `nvm install 18 && nvm use 18` or download from nodejs.org.  
   - Verify with `node -v` and `npm -v`.

2. **Clone & install**  
   ```bash
   git clone <repo-url>
   cd nextjs-wed-admin-main
   npm install
   ```

3. **Environment variables**  
   Create `.env.local` with at least:
   ```env
   NEXTAUTH_URL=http://localhost:3000
   NEXT_PUBLIC_API_URL=http://localhost:8000/api
   NEXTAUTH_SECRET=<generate via `openssl rand -hex 32`>
   ```
   Add OAuth creds when testing Auth0/Google/Azure providers later.

4. **Dev tooling**  
   - VS Code + extensions: ESLint, Prettier, Material UI Snippets, i18n Ally.
   - Browser: Chrome with React DevTools and Redux DevTools (handy for context debugging).

---

## 2. Core Commands & Scripts

| Purpose | Command | Notes |
| --- | --- | --- |
| Start dev server (Turbopack) | `npm run dev` | Runs on `http://localhost:3000`. Auto refresh + fast refresh. |
| Type + lint check | `npm run lint` | Fails on any warnings due to `--max-warnings 0`. |
| Prettier & import sorting | `npm run pretty` | Uses `@trivago/prettier-plugin-sort-imports`. |
| Production build | `npm run build` | Sets `NODE_OPTIONS=--max-old-space-size=4096`. |
| Serve build | `npm start` | Use when validating production output. |

> Tip: Keep `npm run dev` and `npm run lint -- --watch` in separate terminals. Fixing lint/TS errors early saves time.

---

## 3. Vue → Next.js Mental Model

Understanding how familiar Vue patterns translate into Next.js App Router + React hooks is the fastest way to feel at home. Below is a deeper comparison with concrete examples.

### 3.1 Component anatomy

| Vue (single-file) | Next.js / React |
| --- | --- |
| `<template>` for markup, `<script>` for logic, `<style>` for scoped CSS. | One `.tsx` file containing JSX + hooks + optional CSS-in-JS/SX. |
| `export default { props, setup() { ... } }` | `const ComponentName = (props) => { ... return <JSX />; }; export default ComponentName;` |
| Scoped styles via `<style scoped>` | Use MUI `sx`, styled components, or CSS modules. |

**Example** (Vue):
```vue
<template>
  <div class="card">{{ title }}</div>
</template>
<script setup lang="ts">
defineProps<{ title: string }>();
</script>
<style scoped>
.card { padding: 16px; }
</style>
```

**Equivalent** (React):
```tsx
type Props = { title: string };
const Card = ({ title }: Props) => (
  <Box sx={{ p: 2, borderRadius: 2 }}>{title}</Box>
);
export default Card;
```

### 3.2 Reactivity & state

| Vue concept | React equivalent | Example |
| --- | --- | --- |
| `ref` / `reactive` | `useState` or `useReducer` | `const [count, setCount] = useState(0);` |
| `computed` | `useMemo` | `const total = useMemo(() => price * qty, [price, qty]);` |
| `watch` | `useEffect` | `useEffect(() => { fetchData(id); }, [id]);` |
| Lifecycle hooks (`onMounted`, `onBeforeUnmount`) | `useEffect` with cleanup | `useEffect(() => { ...; return () => cleanup(); }, []);` |

### 3.3 Routing & layouts

| Vue Router | Next.js App Router |
| --- | --- |
| Define routes via `createRouter` and `routes` array. | File-system routing: every folder containing `page.tsx` becomes a route automatically. |
| Nested routes use `<router-view>` inside parent components. | Nested layouts via `layout.tsx`. Route groups `(foo)` let you nest without affecting the URL. |
| Route guards (`beforeEach`) | Use Next.js Middleware (`src/middleware.ts`) or server-side checks. |

**Example**: Vue nested route
```js
{
  path: '/dashboard',
  component: DashboardLayout,
  children: [{ path: 'analytics', component: AnalyticsPage }],
}
```
Next.js equivalent:
```
src/app/(main)/layout.tsx     -> DashboardLayout
src/app/(main)/dashboard/analytics/page.tsx -> Analytics page
```

### 3.4 Props, slots, children

| Vue | React |
| --- | --- |
| `<slot>` or named slots for content injection. | `children` prop. |
| Scoped slots via `<template #header="{ data }">`. | Render props `{(props) => <Component {...props} />}`. |
| Provide/inject for dependency sharing. | React Context via `createContext` + Provider. |

Example layout:
```tsx
const Layout = ({ children }: PropsWithChildren) => (
  <MainLayout>{children}</MainLayout>
);
```

### 3.5 Server vs client components

- Vue components always run on the client unless SSR framework (Nuxt) is used.
- Next.js App Router defaults to **server components** (rendered on the server, no bundle cost). You can fetch data directly in them:  
  ```tsx
  export default async function Page() {
    const data = await getData();
    return <div>{data.name}</div>;
  }
  ```
- To use hooks or browser APIs, mark the file with `'use client';` at the top. This is analogous to opting into client-side only behavior.

### 3.6 Global store / dependency injection

| Vue | React |
| --- | --- |
| Vuex/Pinia stores (modules, getters, actions). | React Context + reducers/hooks (see `src/providers`). |
| Provide/Inject for child communication. | Context `Provider` / `useContext`. |
| Vue DevTools for inspecting stores. | React DevTools or custom logging within providers. |

`SettingsProvider` is a concrete example: it wraps the app, exposes `config` state, and stores preferences in localStorage (`src/providers/SettingsProvider.tsx`).

### 3.7 Directives vs JSX

| Vue directive | React/JSX pattern |
| --- | --- |
| `v-if="condition"` | `{condition && <Component />}` |
| `v-else-if / v-else` | Ternaries: `{condition ? <A /> : <B />}` |
| `v-for="item in list"` | `{list.map((item) => <Component key={item.id} />)}` |
| `v-bind:prop="value"` | `<Component prop={value} />` |
| `v-model` | Controlled input: `<TextField value={value} onChange={(e) => setValue(e.target.value)} />` |

### 3.8 Composition API vs hooks

| Vue Composition API | React Hooks |
| --- | --- |
| `setup()` returns reactive state, computed, watchers. | Hook body (component function) returns JSX; hooks called at top level. |
| Custom composables (`useX`) | Custom hooks (files in `src/hooks`). |
| Provide typed injection via returned object. | Return typed values from hooks; rely on TS generics when needed. |

Example hook translation:
```ts
// Vue composable
export function useCountdown() { ... return { timeLeft }; }

// React hook (already in project)
import { useCountdown } from 'hooks/useCountdown';
const { days, hours } = useCountdown(targetDate);
```

### 3.9 File-based conventions

- Vue often groups by feature using `.vue` files.  
- Next.js also groups by feature, but pages and layouts must follow the `page.tsx` / `layout.tsx` naming convention. Components can live anywhere (we store them under `components/`).
- Styles:
  - Vue: `<style scoped>` or `*.scss`.
  - Next.js: uses CSS Modules, global CSS, or (preferred here) Material UI `sx`/`styled`.

### 3.10 SSR/CSR differences

- Nuxt handles SSR/CSR toggling; Next.js App Router does it automatically per component.
- Data fetching:
  - Vue (Nuxt): `asyncData`, `fetch`.
  - Next.js: server components, `fetch` with caching, or Route Handlers in `app/api`.
- Client-only components should guard against SSR (e.g., check `typeof window !== 'undefined'`) just like you would with Nuxt `client-only`.

By keeping these mappings in mind, you can translate any Vue intuition directly into the React/Next.js ecosystem used in this project.

---

## 4. Project Structure (Opinionated Walkthrough)

```
src/
  app/                  # Routes + layouts + loading/error boundaries
    (main)/             # Dashboard experience
      layout.tsx        # Wraps pages with MainLayout shell
      page.tsx          # Default dashboard home -> e-commerce view
    (ecommerce)/        # Customer storefront flows
    (auth)/             # Auth screens
    api/                # Route handlers (Next.js serverless APIs)
  components/
    sections/           # Full-feature widgets (dashboards, apps, etc.)
      dashboards/e-commerce/MonthlyProfit.tsx
    common/             # Low-level reusable UI blocks
  layouts/              # Shells: main, auth, landing, etc.
  providers/            # Context providers (SettingsProvider, BreakpointsProvider...)
  data/                 # Mock data (charts, tables)
  services/             # axios, SWR fetchers, Firebase helpers
  routes/               # URL + API path helpers
  reducers/             # Reducers backing context providers
  theme/                # Material UI theme config
  types/                # Shared TypeScript types
```

### Must-read files (in order)
1. `src/app/layout.tsx` – server root layout + providers.
2. `src/app/App.tsx` – client wrapper enabling settings panel + scroll behavior.
3. `src/app/(main)/layout.tsx` – uses `MainLayout`.
4. `src/layouts/main-layout/MainLayout.tsx` – nav logic, drawers, footers.
5. `src/components/sections/dashboards/e-commerce/index.tsx` – shows how to compose widgets with context + data.

### 4.1 `src/app` – routing surfaces
- **Root**: `layout.tsx`, `App.tsx`, global providers, fonts, `globals.css`.
- **Route groups**: parentheses `(main)`, `(ecommerce)`, `(auth)` keep URL clean but allow separate layouts.
- **Special files**: `loading.tsx`, `error.tsx`, `not-found.tsx` provide fallback UI per route tree.
- **API routes**: `app/api/**` implement serverless handlers (e.g., NextAuth). They run on the server edge/runtime.

### 4.2 `src/components`
- **`base/`**: Primitive building blocks (buttons, cards) that wrap MUI components with standard props/styles.
- **`common/`**: Shared UI fragments (avatars, charts, wrappers).
- **`sections/`**: Feature-level compositions. Each dashboard/app/landing page widget lives here. Follow the folder naming pattern `sections/<domain>/<feature>`.
- **`settings-panel/`**: UI for runtime customization of nav, colors, etc.

### 4.3 `src/layouts`
- `main-layout/`: The authenticated dashboard shell (AppBar, side nav variations, top nav variations, drawers, footer).
- `auth-layout/`: Frames login/signup flows.
- `ecommerce-layout/`, `landing-layout/`, etc., for other verticals.
- Layouts are client components; they rely on `useSettingsContext`, `useBreakpoints`, and other hooks.

### 4.4 `src/providers`
- Context providers for settings, breakpoints, theme, session, feature domains (chat, ecommerce, CRM, email, etc.).
- Each provider typically exports both the provider component and a `useXContext` hook for easy consumption.
- Providers are composed in `src/app/layout.tsx` so any route component can access them.

### 4.5 `src/services`
- `axios/axiosInstance.ts`: Configured HTTP client with auth interceptors.
- `swr/`: Shared fetchers/types for SWR hooks.
- `firebase/`: Firebase auth/provider configs used in NextAuth credentials provider.
- Keep network logic out of components—import hooks/services from here.

### 4.6 `src/data`, `src/types`, `src/reducers`, `src/helpers`
- **`data/`**: Mock JSON/TS objects for dashboards (until backend exists). Replace gradually with real APIs.
- **`types/`**: TypeScript interfaces for each domain (accounts, dashboards, ecommerce, etc.). Always define new models here.
- **`reducers/`**: `SettingsReducer`, `EmailReducer`, etc., backing the context providers.
- **`helpers/`**: Utility functions specialized per domain (e.g., chart data transformers).

### 4.7 `src/theme`, `src/lib`, `src/config.ts`
- **`theme/`**: Material UI theme creator, component overrides, typography, mixins, palette, shadows. Update to change global look and feel.
- **`lib/`**: Cross-cutting utilities (NextAuth options, constants, icon helpers, suspense wrapper).
- **`config.ts`**: App-wide configuration types and defaults (navigation menu types, drawer width, locales, default credentials).

Use this breakdown as a sitemap—when asked to change something, this should tell you which folder to inspect first.

---

## 5. Routing & Layouts in Practice

### 5.1 Creating a route
1. Decide the URL: e.g., `/dashboard/analytics`.
2. Create folder `src/app/(main)/dashboard/analytics`.
3. Add `page.tsx`:
   ```tsx
   import Analytics from 'components/sections/dashboards/analytics';

   const Page = () => <Analytics />;
   export default Page;
   ```
4. (Optional) Add `loading.tsx` or `error.tsx` in the same folder for Suspense boundaries.

### 5.2 Working with layouts
- Root layout (server) injects providers and fonts (`src/app/layout.tsx`).
- Segment layout `(main)/layout.tsx` wraps every dashboard route with `<MainLayout>`.
- To customize just one subsection, add another `layout.tsx` deeper in the tree (just like nested `<router-view>` in Vue).

### 5.3 Client vs server components
- **Server (default)**: render-only components; can fetch data directly inside the component (async function). No hooks allowed.
- **Client**: add `'use client'` at the top. Needed when you use hooks (`useState`, `useEffect`), contexts, or browser APIs.
- Many shared providers and widgets are client components (e.g., `MainLayout`, `SettingsPanel`, `MonthlyProfit`) because they depend on hooks.

---

## 6. Global Providers & Config

### SettingsProvider (`src/providers/SettingsProvider.tsx`)
- Stores UI config (`navigationMenuType`, `sidenavType`, `locale`, `navColor`, etc.) defined in `src/config.ts`.
- Persists user choices in `localStorage` so nav preferences survive reloads.
- Access via:
  ```tsx
  const { config, setConfig, toggleNavbarCollapse } = useSettingsContext();
  ```

### BreakpointsProvider (`src/providers/BreakpointsProvider.tsx`)
- Wraps Material UI `useMediaQuery` for consistent responsive utilities.
- Usage:
  ```tsx
  const { up, down, currentBreakpoint } = useBreakpoints();
  const isDesk = up('lg');
  ```

### LocalizationProvider / ThemeProvider
- `ThemeProvider` uses `createTheme` (`src/theme/theme.ts`) to build light/dark palettes and handle RTL.
- `LocalizationProvider` wires up i18next (`src/locales/i18n.ts`).

### Settings Panel
- `src/app/App.tsx` shows/hides the settings panel outside the `showcase` routes.
- Use it to preview nav types, color schemes, and RTL without changing code.

---

## 7. Data Fetching, Auth, APIs

1. **Axios instance** (`src/services/axios/axiosInstance.ts`)
   - Base URL from `NEXT_PUBLIC_API_URL`.
   - Automatically attaches `Authorization: Bearer <token>` using session data from NextAuth.
   - Normalizes responses by returning `response.data.data` when available.

2. **SWR fetchers** (`src/services/swr`)
   - Use for client-side fetching with caching and revalidation.
   - Example pattern:
     ```tsx
     import useSWR from 'swr';
     import axiosInstance from 'services/axios/axiosInstance';

     const useOrders = () => {
       return useSWR('/orders', (url) => axiosInstance.get(url));
     };
     ```

3. **NextAuth config** (`src/lib/next-auth/nextAuthOptions.ts`)
   - Supports Credentials (JWT/Firebase), Auth0, Google, Azure AD.
   - Customize callbacks to add fields to the session token (e.g., roles, tenant IDs).

4. **Middleware** (`src/middleware.ts`)
   - `withAuth` is prepared for protecting routes, but `config.matcher` is empty. Fill it in once the product requires auth gating.

5. **API routes** (`src/app/api/**`)
   - Use for lightweight backend endpoints (e.g., proxying to external APIs, form submissions).
   - Example: NextAuth handler at `src/app/api/auth/[...nextauth]/route.ts`.

---

## 8. Building the Dashboard Home (Step-by-Step Example)

Let’s dissect `src/app/(main)/page.tsx` (the `/dashboard` homepage) to understand how everything connects.

1. **page.tsx** simply renders the E-commerce dashboard section.
   ```tsx
   import ECommerce from 'components/sections/dashboards/e-commerce';
   export default function Page() {
     return <ECommerce />;
   }
   ```

2. **ECommerce section** (`components/sections/dashboards/e-commerce/index.tsx`)
   - `'use client';` because it uses `useBreakpoints`.
   - Organizes cards inside Material UI `Grid` containers with responsive sizes.
   - Imports data from `data/e-commerce` to supply mock stats.

3. **Widget example – MonthlyProfit**
   - Lives in `components/sections/dashboards/e-commerce/monthly-profit/MonthlyProfit.tsx`.
   - Typically uses chart libraries (ECharts) + `useTheme` for consistent colors.
   - Receives `useBreakpoints` info to adjust layout.

4. **How to modify it**
   - Replace `data/e-commerce/...` with an API hook:
     ```tsx
     const { data, isLoading } = useSWR('/dashboard/monthly-profit', fetcher);
     ```
   - Add skeleton loaders while `isLoading`.
   - Update TypeScript types in `src/types/dashboard.ts`.

5. **Adding a new card**
   - Create `components/sections/dashboards/e-commerce/new-card/NewCard.tsx`.
   - Export via `components/sections/dashboards/e-commerce/index.tsx`.
   - Slot it into the grid:
     ```tsx
     <Grid size={{ xs: 12, md: 6 }}>
       <NewCard />
     </Grid>
     ```

---

## 9. Forms, Tables, and State Patterns

- **Forms**: use `react-hook-form` + `@hookform/resolvers/yup`. Check `components/sections/pages/account/profile/ProfileForm.tsx` for patterns.
- **Tables/data grids**: prefer `@mui/x-data-grid` for complex tables or `Table` components for simple ones. Configure columns/types in `src/types`.
- **Drag-and-drop**: `@dnd-kit` is installed (used in Kanban). Reuse existing providers instead of writing custom DnD logic from scratch.
- **Charts**: ECharts is the default; create configuration objects in helper files to keep JSX clean.

---

## 10. Styling & Theming Deep Dive

1. **Theme tokens**: `src/theme/theme.ts` defines palette, typography, component overrides. Extend them instead of writing raw CSS.
2. **`sx` prop best practices**:
   ```tsx
   <Box
     sx={{
       p: 3,
       bgcolor: (theme) => theme.palette.background.paper,
       borderRadius: 3,
     }}
   />
   ```
3. **Spacing & sizing**: use multiples of 8 (Material UI scale) unless the design requires custom values.
4. **RTL**: toggle from the settings panel to ensure components look correct in RTL mode. Avoid absolute positioning when possible.
5. **Global styles**: `src/app/globals.css` contains minimal resets. Add new globals only when they truly must apply app-wide.

---

## 11. Internationalization Checklist

1. Wrap user-facing strings with `useTranslation`:
   ```tsx
   const { t } = useTranslation();
   <Typography>{t('dashboard.revenue.title')}</Typography>
   ```
2. Add keys under each language: `src/locales/langs/en.json`, `fr.json`, etc.
3. Register new namespaces if needed, but prefer the default `translation` namespace for shared labels.
4. Update `SupportedLocales` in `src/config.ts` if you add a new locale.
5. Verify translations with the settings panel locale switcher.

---

## 12. Debugging & Troubleshooting

- **Hydration mismatch**: Usually caused by using browser-only APIs in server components. Move logic into a `'use client'` component or guard with `useEffect`.
- **Absolute import errors**: Ensure VS Code uses the workspace `tsconfig.json`. Sometimes restarting TS server (`Cmd+Shift+P → TypeScript: Restart TS server`) helps.
- **Auth issues**: Check `NEXTAUTH_URL` and `NEXTAUTH_SECRET`. Use `/api/auth/signin` to test each provider manually.
- **Slow build**: Clear `.next` folder (`rm -rf .next`) and rerun `npm run dev`. Make sure `NODE_OPTIONS` is set when running `npm run build`.

---

## 13. Ready-to-use Checklists

### Before opening a PR
1. ✅ `npm run lint`
2. ✅ `npm run build` (optional but recommended)
3. ✅ Manually tested in Chrome at the relevant route
4. ✅ Checked dark mode + RTL if UI touches layout
5. ✅ Added translations + types + docs for new features

### When creating a new page
- [ ] Folder under correct route group
- [ ] `page.tsx` is exported as default and uses PascalCase component names
- [ ] Added to navigation (if needed) through the menu config (search in `layouts/main-layout/sidenav`)
- [ ] Strings wrapped with `t('...')`
- [ ] Data fetching handled via server component or SWR
- [ ] Loading/error states implemented if async

---

## 14. Useful References

- Root layout: `src/app/layout.tsx`
- Client wrapper + settings panel: `src/app/App.tsx`
- Dashboard layout: `src/app/(main)/layout.tsx`
- Main layout shell: `src/layouts/main-layout/MainLayout.tsx`
- E-commerce dashboard widgets: `src/components/sections/dashboards/e-commerce`
- Settings context: `src/providers/SettingsProvider.tsx`
- Breakpoints context: `src/providers/BreakpointsProvider.tsx`
- Routes helper: `src/routes/paths.ts`
- Axios instance: `src/services/axios/axiosInstance.ts`
- Theme factory: `src/theme/theme.ts`

---

## 15. Final Notes

- Always lean on existing patterns. Search in `components/sections` before building a new widget; chances are there is already a similar implementation.
- Keep TypeScript strictness in mind. Define interfaces in `src/types` and prefer explicit props.
- Update this guide whenever you discover a better workflow or run into an onboarding gotcha. Your improvements will help the next teammate ramp up even faster.

Happy coding! 🚀

---

## Appendix A – Tech Stack Cheat Sheet

- **Framework**: Next.js 15 App Router (React 19). Server components by default, Turbopack dev bundler, SWC compiler.
- **Language**: TypeScript (strict mode).
- **UI**: Material UI v7 (core, lab, x-data-grid, x-date-pickers) + custom theme (`theme/`).
- **State**: React Context (providers), reducers in `src/reducers`, local hooks.
- **Forms**: `react-hook-form` + `yup`.
- **Charts**: ECharts via `echarts-for-react`.
- **Drag & drop**: `@dnd-kit`.
- **Internationalization**: `i18next` + `react-i18next`.
- **Routing**: Next.js file-system routing + `routes/paths.ts` helper.
- **Auth**: NextAuth with credentials, Firebase, Auth0, Google, Azure AD.
- **Networking**: Axios instance with interceptors, SWR for caching/revalidation.
- **Notifications**: `notistack`.
- **Media**: `swiper`, `lottie-react`, `@uiw/react-color`, `wavesurfer`.

---

## Appendix B – Folder-by-Folder Deep Dive

| Path | What lives there | Tips |
| --- | --- | --- |
| `src/app` | App Router routes, layouts, loading/error boundaries, API routes. | Stick to the same structure as existing modules; every `page.tsx` must default export a component. |
| `src/components` | UI building blocks. `base/` and `common/` for low-level; `sections/` for feature-level assemblies. | Before creating a new component, search for similar UI to reuse styles. |
| `src/layouts` | Shells for different experiences (main dashboard, auth, landing, etc.). | Layouts are typically client components because they use hooks. |
| `src/providers` | Global contexts (settings, breakpoints, data domains). | Providers are composed in `src/app/layout.tsx`. |
| `src/data` | Mock data for cards, charts, tables. | Useful for rapid UI prototyping; abstract out to services when backend endpoints exist. |
| `src/lib` | Utilities that don’t fit elsewhere (constants, NextAuth config, icon helpers, suspense wrapper). | Keep pure helpers here to avoid circular dependencies. |
| `src/helpers` | Domain-specific helper functions (e.g., chart configurations). | Use when logic is larger than a one-liner. |
| `src/services` | API clients, SWR hooks, Firebase integration. | Always import `services/axios/axiosInstance` for HTTP calls. |
| `src/theme` | Material UI theme, component overrides, palette definitions. | Update here for global look & feel changes. |
| `src/types` | TypeScript interfaces/enums for each domain. | When adding data models, create or extend the relevant file. |

---

## Appendix C – Navigation & Menu Config

- Side navigation items are defined inside `layouts/main-layout/sidenav`. Each nav variant (default, slim, stacked) has its own component.
- Helpers:
  - `layouts/main-layout/NavProvider.tsx` exposes `navItems` and handles filtering.
  - `routes/paths.ts` centralizes URLs; import `paths` wherever you need to construct links.
- To add a menu entry:
  1. Create/confirm the route in `src/app`.
  2. Update the nav config (search for `navItems` or look in `layouts/main-layout/sidenav/navConfig.ts` if present).
  3. Provide icon (see `src/components/icons`).
  4. Add translation keys for labels.

---

## Appendix D – Common Tasks Walkthroughs

### D.1 Add a new dashboard page and nav item
1. `mkdir -p src/app/(main)/dashboard/performance`
2. Create `page.tsx`:
   ```tsx
   import PerformanceDashboard from 'components/sections/dashboards/performance';
   const Page = () => <PerformanceDashboard />;
   export default Page;
   ```
3. (If the section doesn’t exist) scaffold `components/sections/dashboards/performance/index.tsx`.
4. Update navigation config to include the new route using `paths.dashboardPerformance` (define it in `routes/paths.ts`).
5. Add translations under `src/locales/langs/*`.

### D.2 Connect a widget to a real API
1. Define types under `src/types/<domain>.ts`.
2. Create an SWR hook:
   ```tsx
   import useSWR from 'swr';
   import axiosInstance from 'services/axios/axiosInstance';

   export const useTopCustomers = () =>
     useSWR('/dashboard/top-customers', (url) => axiosInstance.get(url));
   ```
3. In your widget:
   ```tsx
   const { data, isLoading, error } = useTopCustomers();
   if (isLoading) return <Skeleton variant="rounded" height={200} />;
   if (error) return <Alert severity="error">Failed to load</Alert>;
   ```
4. Wire it into the grid layout.

### D.3 Build a form with validation
1. Define schema:
   ```tsx
   import * as yup from 'yup';
   const schema = yup.object({
     name: yup.string().required(),
     email: yup.string().email().required(),
   });
   ```
2. Use React Hook Form:
   ```tsx
   const methods = useForm({ resolver: yupResolver(schema) });
   ```
3. Compose with MUI inputs and show errors via `helperText={errors.name?.message}`.
4. Submit handler should call an API via `axiosInstance.post(...)`.

### D.4 Implementing translations
1. Determine key path (e.g., `dashboard.storageUsage.title`).
2. In component:
   ```tsx
   const { t } = useTranslation();
   <Typography>{t('dashboard.storageUsage.title')}</Typography>
   ```
3. Update each JSON under `src/locales/langs`.
4. Verify by changing locale in settings panel.

---

## Appendix E – Auth Flows & Session Usage

- **Default credentials**: `demo@aurora.com` / `password123`.
- **Session data**:
  - `session.user` includes `id`, `email`, `name`, `image`.
  - Custom fields (like `designation`, `authToken`) can be attached in NextAuth callbacks.
- **Protecting routes**: Once `src/middleware.ts` `config.matcher` is populated (e.g., `['/dashboard/:path*']`), unauthenticated visits redirect to `paths.defaultJwtLogin`.
- **Logout**: Use NextAuth `signOut` function. The session provider is available globally, so `const { data: session } = useSession();`.
- **Custom login flows**: For JWT-based login, edit `useAuthApi` (under `services/api-hooks`) to hit backend auth endpoints.

---

## Appendix F – Testing & Quality Strategy

- **Static analysis**: `npm run lint` (ESLint + TypeScript). Resolve warnings immediately.
- **Formatter**: `npm run pretty` before pushing.
- **Unit/component tests**: Not yet configured; if you add tests, use Jest/React Testing Library and locate files under `__tests__` or `*.test.tsx`.
- **Visual testing**: Manual via Storybook (not included) or Chromatic-style snapshots; for now rely on manual QA.
- **Performance checks**: Use Chrome Lighthouse on key dashboard pages.
- **Bundle analysis**: `npm run analyze` is not defined, but you can enable `@next/bundle-analyzer` by wrapping `next.config.ts`.

---

## Appendix G – Deployment Overview

1. **Build**: `npm run build` (uses Turbopack). Ensure environment variables for production are set (`NEXT_PUBLIC_API_URL`, OAuth secrets, etc.).
2. **Start**: `npm start` or deploy on Vercel (recommended). Next.js App Router works flawlessly with Vercel’s serverless infra.
3. **Environment parity**: keep `.env.local` for dev, `.env.production` for deployment (or Vercel project env vars).
4. **Monitoring**: integrate with Vercel Analytics or your chosen APM tool for error tracking (Sentry, Datadog). Not wired yet—add as needed.
5. **CI checks**: Add `npm run lint` and `npm run build` into your CI pipeline (GitHub Actions, GitLab CI, etc.).

---

Keep this appendix handy for quick answers while ramping up. Update it whenever the tech stack, workflows, or deployment process evolves. This living document is your single source of truth for all things Aurora Next.js Admin.🏁
