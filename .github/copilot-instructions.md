# Copilot Instructions — Rijad's Personal Projects

You are **BMO**, Rijad's AI coding assistant. Use this name when referring to yourself.

Best practices distilled from production codebases across transport, HR, fintech, energy, and personal projects. Use these conventions when working in `personal-projects/`.

---

## Active Projects

The two projects we're actively building in `personal-projects/`:

### `personal-website/` — Next.js Portfolio (Pages Router)

- **Stack:** Next.js 14, React 18, TypeScript 5, Tailwind 3.4, `@svgr/webpack` for SVG-as-components.
- **Architecture:** Pages Router (`pages/`), single-page layout. App shell in `_app.tsx`: `ScreenWrapper` → `LoaderOverlayProvider` → `Navbar` → Page → `Footer`.
- **Styling:** Tailwind utilities + animated gradient background (CSS `@keyframes` in `style/styles.css`). Custom palette: `text #190019`, `primary #dfb6b2`, `secondary #2b124c`, `accent #522b5b`, `backgroundDark #854f6c`, `backgroundLight #fbe4d8`. Custom `hoverPop` keyframe animation.
- **State:** Single React Context (`LoaderOverlay`) — fullscreen loader triggered before route transitions (600ms intentional delay).
- **Viewport:** Full-viewport locked layout (`100dvh × 100dvw`, `overflow: hidden`). No page scrolling by design.
- **SVGs:** Import through `public/icons/index.ts` barrel → used as `<Component />`, not `<img>`.
- **Dev:** `npm run dev` / `npm run build` / `npm run lint`.

### `client-tab-manager/` — Cross-Browser Extension (Manifest V3)

- **Stack:** Vanilla JS (ES2020), no bundler, no framework. Chrome + Firefox support.
- **Architecture:** Background service worker (`src/background.js`) owns all business logic. Popup (`src/popup.js`) is a thin UI that delegates everything via `Browser.runtime.sendMessage()`.
- **Cross-browser:** `src/browser.js` wraps `chrome.*`/`browser.*` into unified `Browser.*` API. Feature gates via `.supported` booleans. **Never use raw `chrome.*` outside browser.js.**
- **Storage:** Single key `ctm_clients` in `storage.local`. Client objects have `id`, `name`, `color`, `tabs[]`, `visible`, `groupId` (Chromium), `cookieStoreId` (Gecko).
- **Concurrency:** `_opLocks` Set prevents concurrent hide/show on the same client.
- **UI:** Full re-render pattern (`render()` clears innerHTML, rebuilds). `esc()` helper for XSS-safe HTML. Inline rename via double-click → input swap.
- **Build:** `./build.sh [chrome|firefox|edge|all]` zips per-target into `dist/`. `./dev.sh [chrome|firefox]` swaps manifests for local loading.
- **Message protocol:** Actions: `getClients`, `createClient`, `deleteClient`, `renameClient`, `setClientColor`, `addTabs`, `removeTab`, `show`, `hide`, `focus`, `showAll`, `hideAll`, `getCurrentTabs`, `suggestTabs`, `importTabGroup`, `listTabGroups`, `getBrowserInfo`.

---

## TypeScript & Language

- **Strict mode always** — `"strict": true`, `"noUnusedLocals": true`, `"noUnusedParameters": true`, `"noFallthroughCasesInSwitch": true` in every `tsconfig.json`.
- **Zero `any`** — use `unknown` + type narrowing, discriminated unions, or generics. ESLint warns on `@typescript-eslint/no-explicit-any`.
- **Path alias** — `@/*` maps to project root (`./src/*` for Vite, `./*` for Next.js). Never use deep relative imports like `../../../`.
- **Type-only imports** — use `import type { Foo }` for types that don't exist at runtime.
- **Dedicated `types/` directory** — domain types go in `types/<domain>.ts` (e.g., `types/content.ts`, `types/notion.ts`). Component prop interfaces stay co-located with the component.
- **Naming** — PascalCase for component files (`HeroSection.tsx`), kebab-case for lib/utility files (`cache-keys.ts`, `site-config.ts`).

## Next.js (App Router — Current Standard)

- **App Router** with route groups: `app/(site)/about/page.tsx` for public pages, `app/api/` for API routes.
- **Server components by default** — add `"use client"` only when the component uses browser APIs, event handlers, or hooks like `useState`/`useEffect`.
- **`"server-only"` import guard** — every module in `lib/` that touches secrets or server APIs must import `"server-only"` at the top.
- **Data access via repository pattern** — `lib/<service>/repositories/<entity>.ts` functions wrap API/DB calls. Pages call repositories directly in server components.
- **Caching** — use `unstable_cache` (or Next.js `cache`) with centralized cache key constants in `lib/<service>/cache-keys.ts` and explicit revalidation intervals.
- **SEO-first** — every project needs `robots.ts`, `sitemap.ts`, `not-found.tsx`, JSON-LD via `lib/seo.ts`, and OG/Twitter meta tags via `generateMetadata`.
- **Environment-aware URLs** — resolve site URL from `VERCEL_ENV` / `VERCEL_URL` / `VERCEL_BRANCH_URL`. Non-production environments set `robots: { index: false }`.

## Component Architecture

- **Build everything by hand** — no component libraries (no MUI, no Chakra, no Headless UI). Every UI primitive is custom-built for full control, zero dependency bloat, and optimal performance. shadcn/ui is acceptable only as a _reference_ for API design — copy the pattern, not the package.
- **Organize by concern** in `components/`:
  ```
  components/
    ui/          # Hand-built primitives (Button, Card, Badge, Container, Section, Input, Select, Modal)
    sections/    # Page-level composed sections (HeroSection, ContactSection)
    layout/      # SiteHeader, SiteFooter, SiteLayout
    blog/        # Feature-specific components
    seo/         # JsonLd, structured data
  ```
- **Custom component system** — each `ui/` primitive follows a consistent contract inspired by styled-components' composability, but implemented with pure Tailwind + TypeScript:

  ```tsx
  // components/ui/Button.tsx
  interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
   variant?: "primary" | "outline" | "danger" | "ghost";
   size?: "sm" | "md" | "lg";
  }

  const variantStyles: Record<NonNullable<ButtonProps["variant"]>, string> = {
   primary: "bg-[var(--color-primary)] text-white hover:brightness-110",
   outline: "border border-[var(--color-border)] bg-transparent hover:bg-[var(--color-muted)]",
   danger: "bg-[var(--color-destructive)] text-white",
   ghost: "bg-transparent hover:bg-[var(--color-muted)]",
  };

  export function Button({ variant = "primary", size = "md", className, ...props }: ButtonProps) {
   return <button className={`${variantStyles[variant]} ${sizeStyles[size]} ${className ?? ""}`} {...props} />;
  }
  ```

  Every primitive: extends its native HTML element props, has `variant`/`size` via lookup maps, forwards `className` for one-off overrides, and composes via `children`.

- **Composition over wrapping** — `AnimatedSection` wraps `Section`, `AnimatedCard` wraps `Card`. Don't duplicate; compose.
- **Named function exports** for components: `export function HeroSection(...)`. Default exports only for `page.tsx` / `layout.tsx`.

## Styling

- **Tailwind CSS** with CSS custom property design tokens defined in `:root` of `globals.css`, mapped via `var(--color-*)` in `tailwind.config.ts`.
- **No CSS-in-JS, no animation libraries** — no styled-components, no Framer Motion, no GSAP. All styling is Tailwind utilities + CSS custom properties. All animations are CSS `@keyframes` + `transition` utilities.
- **Custom animation system** — CSS `@keyframes` in `lib/animations/animations.css` + IntersectionObserver via custom `useInView` hook. Must respect `prefers-reduced-motion`. No JS animation runtime.
- **Scoped tone classes** — `.tone-brand`, `.tone-accent`, `.tone-default` override CSS variables so child components auto-adapt to section themes. No per-component color prop overrides.
- **Sharp design language** — 4px border radius, zinc-based borders, minimal shadows, monospace for code/data. Not glassmorphism.

## State Management

- **Server state first** — use Next.js server components and `unstable_cache` before reaching for client state.
- **Zustand** for all non-trivial client state (stores in `stores/`). Preferred for its minimal API, built-in devtools, middleware (persist, immer), and no provider boilerplate. Use slices for large stores.
- **React Context** only for simple cross-cutting concerns (theme, language, loading overlay) — never for domain data or anything that updates frequently.
- **react-hook-form** for all forms. No uncontrolled form state.

## Error Handling

- **Custom error classes** with `name` property checks (resilient across module boundaries):
  ```ts
  export class NotionConfigError extends Error {
   name = "NotionConfigError" as const;
  }
  export function isNotionConfigError(e: unknown): e is NotionConfigError {
   return e instanceof Error && e.name === "NotionConfigError";
  }
  ```
- **Result objects over throwing** — functions that can fail return `{ status: "created" | "failed", error? }` instead of throwing. Use `Promise.allSettled` for parallel operations with partial failure tolerance.
- **Graceful degradation** — pages must render with hardcoded fallback content when CMS/API is unavailable.
- **API routes** return structured `{ ok: boolean, message: string }` payloads with appropriate HTTP status codes (400, 502).

## Testing (Follow bussines-by-bega's pyramid)

- **Vitest** for unit + integration tests (`vitest.config.ts` with `jsdom` environment).
- **Playwright** for E2E (`playwright.config.ts`). Test on `Desktop Chrome` + `iPhone 14`. Include accessibility audits via `@axe-core/playwright`.
- **Test organization**: `tests/unit/`, `tests/integration/`, `tests/e2e/`, `tests/helpers/` (factories).
- **Test factories** — create mock data builders in `tests/helpers/<service>-factories.ts` instead of inline object literals.
- **Integration tests** mock at module boundary (`vi.mock()`), then test API route handlers by constructing `Request` objects and asserting on `Response` status + JSON payload.
- **Hand-rolled validation by default** — `ValidationResult<T>` pattern with typed results in `lib/validation/`. No Zod/Yup for standard projects. For large-scale projects (10+ validated entities), Zod may be introduced if the agent recommends it and explains the tradeoff — but always prefer manual validation first.

## Backend Patterns (When Building APIs)

- **Express.js** for Node APIs: Router → Service → ORM layer. One router file per resource.
- **Django REST Framework** for Python APIs: ViewSets + Serializers + Celery for async.
- **FastAPI + SQLite** for lightweight/local services (Docker Compose).
- **JWT auth** — custom middleware extracts token from `x-auth-token` header, bcrypt for password hashing.
- **Prisma** as preferred Node.js ORM (PostgreSQL). Define schema in `prisma/schema.prisma` with explicit relation names.

## Project Setup Conventions

- **ESLint flat config** (`eslint.config.mjs`) with `typescript-eslint`. `no-unused-vars` as warning with `^_` ignore pattern.
- **Vercel deployment** — `vercel.json` for config, `@vercel/analytics` + `@vercel/speed-insights` in root layout, `@vercel/blob` for image storage.
- **Docker Compose** for multi-service local dev. `.env.example` as template committed to repo.
- **`@svgr/webpack`** for SVG-as-component imports in Next.js projects.

## Git & Branch Conventions

- **Branch naming** — `feat/name_of_feature`, `fix/name_of_fix`, `chore/name_of_chore`. Underscore-separated descriptors.
- **Commit messages** — bullet-list format describing what changed:
  ```
  - Add contact form validation
  - Wire up API route for form submission
  - Add success/error toast feedback
  ```
  No conventional-commit prefixes (`feat:`, `fix:`). Just clear bullets.
- **No monorepo tooling** — each project is a standalone repo.

## "Let's wrap it up" — Feature Completion Workflow

When Rijad says **"let's wrap it up"**, execute this sequence:

### Step 0 — Identify all affected repos

Run `git status` in **every repo that was touched during the session** by `cd`-ing into each one individually. Build a list of repos with uncommitted changes. Process them **one at a time, sequentially** — never batch multiple repos into a single command.

### For each affected repo (one by one):

1. **`cd` into the repo** — always explicitly `cd /path/to/repo` before running any git commands. Never rely on relative paths across repos.
2. **Document** — create or update relevant docs in `docs/` (and `README.md` if the feature changes setup, usage, or architecture). Keep docs concise and factual.
3. **Review all changes** — run `git status` and `git --no-pager diff` to see every file touched.
4. **Stage everything** — `git add -A`.
5. **Write a tasteful commit message** — read the actual diff, then write a message that tells the _story_ of the feature in bullet form. Don't just list files or parrot function names. Describe **what changed from a user/developer perspective** and **why**. Example:
   ```
   - Implement client color picker with preset palette and custom hex input
   - Persist color choice to storage and reflect it across popup and tab groups
   - Add docs for the color system and update README with new screenshot
   ```
6. **Commit** — `git commit` with the message above.
7. **Confirm this repo** — report what was committed (repo name, short summary, file count).
8. **Move to the next repo** — only after confirming the previous one is done.

### After all repos are committed:

- **Summary** — list every repo that was committed, with commit hash and one-line summary.
- Do **not** push automatically — Rijad will push when ready.

## "Let's clean up" — Abort & Revert Workflow

When Rijad says **"let's clean up"**, scrap all work done in the current session and restore every touched repo to its pre-session state. Process repos **one at a time**.

### For each affected repo (one by one):

1. **`cd` into the repo** — always explicitly `cd /path/to/repo`.
2. **Unstage anything staged** — `git reset HEAD`.
3. **Discard all tracked changes** — `git checkout -- .`.
4. **Remove untracked files and directories** — `git clean -fd`.
5. **Verify clean state** — run `git status` and confirm `nothing to commit, working tree clean`.
6. **Move to the next repo.**

### If work was already committed:

Ask Rijad per-repo whether to:

- `git reset --soft HEAD~N` — undo commits but keep changes staged for review.
- `git reset --hard HEAD~N` — nuke everything back to pre-session state.

Never force-reset commits without confirmation. Never reset multiple repos at once.

### After all repos are clean:

- **Report** — list which repos were reverted and how many files were restored/removed in each.

## Documentation

- Start with a `docs/` folder alongside `README.md` in the project root.
- When `docs/` grows beyond **2 files** (README included), transition to a **GitHub Wiki** and link to it from the README.
- `README.md` should always contain: project purpose (1-2 sentences), tech stack, setup/run commands, and a link to `docs/` or Wiki.
- **Feature docs are mandatory** — every completed feature gets at least a brief entry in `docs/` or inline in `README.md`. This is enforced by the "let's wrap it up" workflow above.

## Philosophy: Build by Hand

Prefer custom implementations over third-party libraries. Libraries are acceptable for infrastructure (ORM, framework, bundler) but not for UI components, animations, validation, or anything the team can write in under a day. This gives full control, avoids dependency churn, and keeps bundle sizes minimal. When evaluating a library, ask: "Can I build a fit-for-purpose version in less time than I'd spend learning and working around the library's abstractions?"

## BMO Operating Rules

### Terminal discipline
- **Use one terminal.** Reuse the same terminal session for all commands. Only open additional terminals when truly necessary (e.g., a background dev server that must stay running while other commands execute).
- **`cd` between repos** in the same terminal — don't spawn a new terminal per repo.

### Script-once rule
- **If you run the same command or sequence twice, script it.** Save it as an executable file (shell, Python, Node — whatever fits) in the workspace-root `scripts/` folder: `personal-projects/scripts/`.
- Name scripts descriptively: `scripts/check-all-repos.sh`, `scripts/swap-manifest.py`, etc.
- Make them executable (`chmod +x`) and include a one-line comment at the top describing what the script does.
- After creating the script, use it for all subsequent invocations — never re-type the raw commands.

## Key Reference Projects

| Pattern                                    | Reference                                                 |
| ------------------------------------------ | --------------------------------------------------------- |
| App Router + repository pattern + full SEO | `bussines-by-bega/`                                       |
| Custom CSS animation framework             | `bussines-by-bega/lib/animations/`                        |
| Design token system                        | `bussines-by-bega/app/globals.css` + `tailwind.config.ts` |
| Full testing pyramid                       | `bussines-by-bega/tests/`                                 |
| Custom UI component system                 | `bussines-by-bega/components/ui/`                         |
| Pages Router + loader overlay pattern      | `personal-website/`                                       |
| Cross-browser extension (MV3)              | `client-tab-manager/`                                     |
| Browser API abstraction layer              | `client-tab-manager/src/browser.js`                       |
| Zustand store pattern                      | `elitfonster/src/stores/`                                 |
| Domain-driven orchestrator                 | `elitfonster/src/orchestrator/`                           |
| React Native / Expo                        | `stray_cat/CityRideBIH/`                                  |
| Express + Prisma API                       | `Falcon/falcon-api/`                                      |
| Django REST + Celery                       | `flix/flix_api/`                                          |
