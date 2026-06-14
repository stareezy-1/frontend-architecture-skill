# Frontend Architecture Skill

[![skills.sh](https://skills.sh/b/stareezy-1/frontend-architecture-skill)](https://skills.sh/stareezy-1/frontend-architecture-skill)

A collection of portable, **framework-agnostic** skills for any React or React Native frontend — packaged as [Agent Skills](https://www.anthropic.com/news/skills) (`SKILL.md`) that Claude Code, Cursor, OpenCode, Codex, Windsurf, and other agents can read and apply.

This repo ships six skills:

- **`frontend-architecture`** — organizes apps into **feature modules** with **page/screen directories**, a strict **server-state vs UI-state split**, **barrel-only** cross-module imports, **co-located styles**, and clear **component-promotion** rules.
- **`frontend-seo`** — a complete **SEO system**: centralized site identity, canonical URLs, per-route metadata (Open Graph + Twitter cards), generated `sitemap.xml` / `robots.txt` / RSS feed, and typed **JSON-LD structured data** (Person, WebSite, BlogPosting, CreativeWork, BreadcrumbList, FAQPage).
- **`frontend-lighthouse`** — a **Lighthouse CI performance gate**: Core Web Vitals budgets (LCP, INP via the TBT lab proxy, CLS) and category score floors enforced as a blocking PR check, with a `lighthouserc.cjs`, an `lhci` script, and a GitHub Actions workflow.
- **`frontend-data-contracts`** — type safety at the **network edge**: one typed API client as the single fetch boundary, parse-don't-validate, one `{ data } / { error }` envelope, one normalized `ApiError`, branded IDs, and per-field error mapping.
- **`frontend-optimistic-mutations`** — the **write path**: the optimistic lifecycle (cancel → snapshot → patch → rollback → invalidate), idempotency keys for safe retries, and lock-step detail/list cache coherence.
- **`frontend-observability`** — the **field side**: a typed event taxonomy, a best-effort non-blocking provider fan-out, real-user Core Web Vitals (the complement to Lighthouse's lab gate), error reporting, and consent gating.

All share the same design goals:

- ✅ **State-management agnostic** — Zustand, Redux Toolkit, MobX, Jotai, Valtio, or Context.
- ✅ **Styling agnostic** — Tailwind, CSS Modules, Tamagui, React Native StyleSheet, styled-components.
- ✅ **Framework agnostic** — Next.js App Router, React + Vite, Remix, Astro, and Expo / React Native.
- ✅ **Naming conventions** — interfaces prefixed with `I` (e.g. `IFeatureUiState`).

> The skills describe **structure and rules**, not a component library or a visual design. Pair them with a design/component skill for the look-and-feel.

---

## Install

With the [`skills` CLI](https://skills.sh) (works across Claude Code, Cursor, Codex, and more):

```bash
# install just the architecture skill
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-architecture"

# install just the SEO skill
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-seo"

# install just the Lighthouse performance-gate skill
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-lighthouse"

# install just the data-layer skills
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-data-contracts"
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-optimistic-mutations"

# install just the observability skill
npx skills add stareezy-1/frontend-architecture-skill --skill "frontend-observability"

# or install every skill in the repo
npx skills add stareezy-1/frontend-architecture-skill
```

`--skill` (short: `-s`) selects a single skill from the repo; omit it to add all skills. Add `-g` to install globally into `~/` instead of the current project.

Or copy it manually into your agent's skills directory:

```bash
# Claude Code (project scope)
mkdir -p .claude/skills && cp -r skills/frontend-architecture .claude/skills/
cp -r skills/frontend-seo .claude/skills/
cp -r skills/frontend-lighthouse .claude/skills/
cp -r skills/frontend-data-contracts .claude/skills/
cp -r skills/frontend-optimistic-mutations .claude/skills/
cp -r skills/frontend-observability .claude/skills/

# Cursor / others that read .ai/skills
mkdir -p .ai/skills && cp -r skills/frontend-architecture .ai/skills/
cp -r skills/frontend-seo .ai/skills/
cp -r skills/frontend-lighthouse .ai/skills/
cp -r skills/frontend-data-contracts .ai/skills/
cp -r skills/frontend-optimistic-mutations .ai/skills/
cp -r skills/frontend-observability .ai/skills/
```

> Note: `gh skill install` is a preview feature of the GitHub CLI and is not yet available in stable `gh` releases. Use the `npx skills` command above (or manual copy) until it ships.

---

## The `frontend-architecture` skill at a glance

The whole model is five rules applied mechanically. The diagrams below show how they fit together.

### 1. Layered structure — routing is thin, modules hold the app

A route file does almost nothing: it imports a page component from a module barrel and mounts it. All real work lives inside feature modules, which lean on a shared layer.

```mermaid
graph TD
    subgraph Routing["Routing layer (thin)"]
        R["app/ · routes/ · navigation/<br/>mounts pages, owns layout/auth boundaries"]
    end

    subgraph Modules["src/modules/ — feature modules"]
        direction LR
        Auth["auth/"]
        Invoice["invoice/"]
        Dash["dashboard/"]
    end

    subgraph Shared["src/shared/ — cross-module building blocks"]
        direction LR
        Comp["components/"]
        ApiClient["api-client/<br/>(only network entry)"]
        Utils["utils/ · hooks/"]
    end

    R -->|imports page via barrel| Auth
    R -->|imports page via barrel| Invoice
    R -->|imports page via barrel| Dash
    Auth --> Shared
    Invoice --> Shared
    Dash --> Shared
```

### 2. Inside a module — the barrel is the only public door

Everything inside a module is private except what its `index.ts` re-exports. Other modules and the router may import **only** from `@/modules/{feature}` — never a deep internal path.

```mermaid
graph TD
    Outside["Other modules / routing layer"] -->|"@/modules/invoice"| Barrel

    subgraph ModuleInvoice["modules/invoice/"]
        Barrel["index.ts<br/>PUBLIC BARREL (the contract)"]
        Pages["pages/&#123;page&#125;/<br/>page.tsx + page.styles.ts + components/ + hooks/"]
        Components["components/<br/>reused by 2+ pages in this module"]
        Hooks["hooks/<br/>query/mutation + key factory"]
        Stores["stores/<br/>UI state only"]
        Services["services/<br/>data access"]
        Types["types/"]

        Barrel --- Pages
        Barrel --- Components
        Barrel --- Hooks
        Barrel --- Stores
        Barrel --- Types
        Pages --> Components
        Pages --> Hooks
        Pages --> Stores
        Hooks --> Services
    end
```

### 3. State is split by origin — and never crosses over

Server data lives in the query/cache layer; UI data lives in the client store. Components read from both but write through hooks — they never `fetch()` directly, and server entities are never copied into the store.

```mermaid
flowchart LR
    API[("Backend API")]
    Client["shared/api-client<br/>(typed, only network entry)"]
    Query["Query / cache layer<br/>TanStack Query · RTK Query · SWR<br/><i>server state — source of truth</i>"]
    Store["Client store<br/>Zustand · Redux · MobX · Jotai<br/><i>UI state — dialogs, filters, drafts</i>"]
    Component["React / RN component"]

    API <--> Client
    Client <--> Query
    Query -->|"useInvoiceList()"| Component
    Store -->|"selector"| Component
    Component -->|"mutate()"| Query
    Component -->|"action"| Store

    Query -. "never mirror server data into the store" .-x Store
```

### 4. Component promotion — start local, move outward only when reused

A component is born in the narrowest scope and is promoted one level only when a **second** consumer appears. The same ladder applies to hooks, utils, and constants.

```mermaid
graph LR
    A["Used by ONE page<br/>pages/&#123;page&#125;/components/"]
    B["Used by 2+ pages in a module<br/>modules/&#123;feature&#125;/components/"]
    C["Used by 2+ modules<br/>shared/components/"]
    D["Used by 2+ apps/repos<br/>published package"]

    A -->|"2nd consumer in module"| B
    B -->|"2nd consuming module"| C
    C -->|"2nd consuming app"| D
```

### 5. How a request flows through the layers

Putting it together: a route mounts a page, the page reads server state via a hook and UI state via a selector, mutations go back through the query layer to the API, and styling is referenced from a co-located file.

```mermaid
sequenceDiagram
    participant Route as Route file
    participant Page as Page (module)
    participant Hook as Data hook
    participant Query as Query layer
    participant API as API client
    participant Store as UI store

    Route->>Page: mount <InvoiceListPage/>
    Page->>Hook: useInvoiceList(params)
    Hook->>Query: read cache / fetch
    Query->>API: GET /invoices
    API-->>Query: data
    Query-->>Page: invoices (server state)
    Page->>Store: read filters (UI state)
    Page->>Hook: useCreateInvoice().mutate()
    Hook->>Query: mutation + invalidate keys
    Query->>API: POST /invoices
```

---

## What's inside

```
skills/
├── frontend-architecture/
│   └── SKILL.md     ← architecture skill (frontmatter + instructions)
├── frontend-seo/
│   └── SKILL.md     ← SEO skill (frontmatter + instructions)
├── frontend-lighthouse/
│   └── SKILL.md     ← Lighthouse performance-gate skill (frontmatter + instructions)
├── frontend-data-contracts/
│   └── SKILL.md     ← typed network-boundary skill (frontmatter + instructions)
├── frontend-optimistic-mutations/
│   └── SKILL.md     ← write-path / optimistic-mutation skill (frontmatter + instructions)
└── frontend-observability/
    └── SKILL.md     ← field-side observability skill (frontmatter + instructions)
```

The **`frontend-architecture`** skill covers:

1. **Five core ideas** — feature modules, page directories, state-by-origin, barrel imports, promotion.
2. **Directory layout** — identical across frameworks.
3. **Feature modules** — the barrel as a contract.
4. **State split** — server (query layer) vs UI (client store), with examples for Zustand / Redux Toolkit / MobX / Jotai.
5. **Co-located styling** — Tailwind / CSS Modules / Tamagui / StyleSheet.
6. **Naming conventions** — `I`-prefixed interfaces and more.
7. **Framework adapters** — Next.js, Vite SPA, Remix, Expo / React Native.
8. **Review checklist** + **component promotion ladder**.

---

## The `frontend-seo` skill at a glance

A complete, framework-agnostic SEO system built as **pure builder functions plus a thin framework adapter**. Site identity lives in one constants module, every URL flows through a single `canonicalUrl()` function, and search engines get a generated sitemap, robots rules, RSS feed, and typed JSON-LD.

### How the pieces fit

```mermaid
graph TD
    Const["constants/seo<br/>SINGLE source of identity<br/>URL · name · description · OG · verification"]
    Canon["canonicalUrl(path)<br/>absolute + normalized URLs"]

    subgraph Builders["services/seo (pure builders)"]
        Meta["buildMetadata()"]
        Sitemap["sitemapEntries()"]
        Robots["robots()"]
        Rss["rssItems()"]
        Sd["structuredData() + per-type JSON-LD"]
    end

    subgraph Adapter["Thin framework adapter (route files)"]
        Layout["app/layout.tsx<br/>global metadata"]
        SitemapRoute["app/sitemap.ts"]
        RobotsRoute["app/robots.ts"]
        Feed["app/feed.xml/route.ts"]
        Page["page.tsx<br/>generateMetadata + JSON-LD script"]
    end

    Const --> Canon
    Const --> Builders
    Canon --> Builders
    Meta --> Layout
    Meta --> Page
    Sitemap --> SitemapRoute
    Robots --> RobotsRoute
    Rss --> Feed
    Sd --> Page
```

The skill covers:

1. **Five core ideas** — one source of identity, absolute canonical URLs, pure builders + thin adapter, typed reusable JSON-LD, content-generated discovery surfaces.
2. **`constants/seo`** — the single source of truth for site identity.
3. **`types/seo`** — typed data models (`RouteDescriptor`, `SitemapEntry`, `RobotsConfig`, `RssItem`, `JsonLd`, …).
4. **Canonical URLs** — one `canonicalUrl()` function used everywhere.
5. **Per-route metadata** — `buildMetadata`, global defaults in the layout, dynamic overrides.
6. **Discovery surfaces** — `sitemap.xml`, `robots.txt`, and an RSS feed generated from content.
7. **Structured data** — typed JSON-LD builders (Person, WebSite, BlogPosting, CreativeWork, BreadcrumbList, FAQPage) cross-referenced by stable `@id`.
8. **Framework adapters** — Next.js, Remix, Astro, Vite SPA, Expo Router (web).
9. **Review checklist** for SEO coverage.

---

## The `frontend-lighthouse` skill at a glance

A **CI performance gate** built around a single `lighthouserc.cjs`. It runs Lighthouse against the production build (median-of-N runs for stability) and **blocks pull requests** that miss Core Web Vitals budgets or category score floors.

### How the pieces fit

```mermaid
graph TD
    PR["Pull request<br/>(touches apps/web/**)"]
    Workflow[".github/workflows/lighthouse.yml<br/>build → start → lhci → upload"]
    Script["package.json<br/>lhci: lhci autorun"]
    Config["lighthouserc.cjs<br/>named budget constants"]

    subgraph Gate["lhci autorun"]
        Collect["collect<br/>prod server · mobile emulation · 3 runs"]
        Assert["assert<br/>median-run vs budgets"]
        Upload["upload<br/>filesystem report artifact"]
    end

    subgraph Budgets["Budgets (the contract)"]
        CWV["CWV: LCP ≤ 2500ms · TBT ≤ 200ms · CLS ≤ 0.1"]
        Cats["Categories: perf ≥ 0.9 · SEO/a11y ≥ 0.95 · best-practices ≥ 0.9"]
    end

    PR --> Workflow --> Script --> Config
    Config --> Collect --> Assert --> Upload
    Budgets --> Assert
    Assert -->|"any budget exceeded"| Fail["❌ PR blocked"]
    Assert -->|"all budgets met"| Pass["✅ PR green"]
```

The skill covers:

1. **Five core ideas** — one config source of truth, gate the production build, median-of-N stability, Google "good" CWV thresholds, blocking-in-CI with report artifacts.
2. **The config** — `lighthouserc.cjs` with named budget constants and collect/assert/upload sections.
3. **Budget severity & thresholds** — when to use `error` vs `warn`, and the CWV/category targets.
4. **The `lhci` script** — local reproduction of the CI run, mobile and desktop form factors.
5. **The GitHub Actions workflow** — PR-triggered, production-build gate, always-upload reports.
6. **Framework adapters** — Next.js, Remix, Astro, SvelteKit, Vite (server command + ready pattern).
7. **Debugging** — flaky runs, missing INP audit, server-ready timeouts, real regressions.
8. **Review checklist** for the performance gate.

---

## The `frontend-data-contracts` skill at a glance

Type safety at the **network edge**. One typed `apiClient` is the only place `fetch` is called; the moment data crosses in, it's parsed into a trusted domain type — or it becomes a single normalized `ApiError`.

### How the pieces fit

```mermaid
graph LR
    API[("Backend API<br/>{ data } / { error }")]

    subgraph Boundary["shared/api-client (the only fetch boundary)"]
        Client["apiClient.get/post/…<br/>buildUrl · headers · fetch"]
        Parse["parse envelope<br/>+ schema parse (Zod/Valibot)"]
        Err["ApiError<br/>code · status · fields"]
    end

    subgraph App["App (downstream)"]
        Hook["typed query/mutation hook"]
        Domain["trusted domain value<br/>(branded IDs)"]
        Form["form field errors"]
        Toast["localized toast (messageKey)"]
    end

    API <-->|fetch| Client
    Client --> Parse
    Parse -->|success| Domain
    Parse -->|failure| Err
    Domain --> Hook --> App
    Err -->|hasFieldErrors| Form
    Err -->|messageKey| Toast
```

The skill covers:

1. **Five core ideas** — one fetch boundary, parse-don't-validate, one envelope, one error type, branded IDs.
2. **The typed client** — verbs return unwrapped `data`, throw `ApiError`; framework-free.
3. **Parse, don't validate** — schema parse at the edge turns `unknown` wire JSON into trusted types.
4. **The envelope** — `{ data } / { error }` unwrapped once.
5. **Branded identifiers** — nominal `InvoiceId`/`CustomerId` so IDs can't be mixed.
6. **One normalized error** — server/status/network/abort all become `ApiError`; side effects live in the query layer; per-field errors map to forms.
7. **Library adapters** — TanStack Query, RTK Query, SWR, RN fetch hooks.
8. **Review checklist** for the data boundary.

---

## The `frontend-optimistic-mutations` skill at a glance

The **write path**. A mutation feels instant, is safe to retry, and leaves every cache coherent — built on the typed client from `frontend-data-contracts`.

### The optimistic lifecycle

```mermaid
sequenceDiagram
    participant UI as Component
    participant M as useMutation
    participant C as Query cache
    participant API as apiClient

    UI->>M: mutate({ id })
    M->>C: onMutate — cancelQueries (stop late refetch)
    M->>C: snapshot detail + every list page
    M->>C: patch detail + matching list rows (instant UI)
    M->>API: POST (Idempotency-Key from form init)
    alt success
        API-->>M: server entity
        M->>C: onSettled — invalidate detail + lists (refetch authoritative)
    else error
        API-->>M: ApiError
        M->>C: onError — restore snapshots verbatim
        M->>C: onSettled — invalidate detail + lists
    end
```

The skill covers:

1. **Five core ideas** — fixed lifecycle, verbatim rollback, idempotency-once, lock-step caches, server-state-stays-in-cache.
2. **When to be optimistic** (and when to use a pending state instead).
3. **The lifecycle** — cancel → snapshot → patch → rollback → invalidate (TanStack Query reference).
4. **Non-optimistic creates** — seed the cache from the server response.
5. **Cache coherence** — patch detail + every list page via a hierarchical key factory.
6. **Idempotency** — keys generated at intent time so retries replay, not double-charge.
7. **Retry policy** — reads vs writes vs conflicts.
8. **Library adapters** — TanStack Query, RTK Query, SWR.
9. **Review checklist** for the write path.

---

## The `frontend-observability` skill at a glance

The **field side** — the complement to the Lighthouse _lab_ gate. A typed event taxonomy and a best-effort fan-out tell you what real users do and experience, without analytics ever being able to crash the app.

### How the pieces fit

```mermaid
graph TD
    Const["constants/analytics<br/>ANALYTICS_EVENTS + union type"]
    Hook["useAnalytics().track<br/>(no-op outside provider / SSR)"]
    Consent["consent gate<br/>(checked once at fan-out)"]
    Vitals["web-vitals<br/>real-user LCP/INP/CLS"]

    subgraph FanOut["track() — best-effort, non-blocking"]
        direction LR
        GA["gtag adapter"]
        Clarity["clarity adapter"]
        PH["posthog adapter"]
        Err["error adapter (Sentry)"]
    end

    Const --> Hook --> Consent --> FanOut
    Vitals --> FanOut
    FanOut -.->|"each call try/caught — a broken provider can't throw"| FanOut
    Lab["frontend-lighthouse<br/>lab budget"] -. "same metrics & thresholds" .-> Vitals
```

The skill covers:

1. **Five core ideas** — typed vocabulary, best-effort fan-out, one SSR-safe entry point, field vitals complement lab budgets, consent gates everything.
2. **The event taxonomy** — canonical constants + union type, never inline strings.
3. **The fan-out** — `track()` guards each adapter; absent/throwing providers no-op.
4. **Provider + hook** — `'use client'` context, `useAnalytics()` no-ops outside a provider.
5. **Real-user Web Vitals** — reported through the same fan-out, matching the Lighthouse skill's thresholds.
6. **Consent & privacy** — gated once at the boundary; PII-light props.
7. **Error reporting** — at deliberate route/segment boundaries.
8. **Provider & framework adapters** — GA4, Clarity, PostHog, OpenPanel, Sentry; Next.js / Vite / RN.
9. **Review checklist** for observability.

---

## When the agent should use them

**`frontend-architecture`:**

- Scaffolding a new app.
- Adding a feature or module.
- Deciding where a component, hook, or piece of state belongs.
- Naming types and interfaces.
- Reviewing folder structure and import boundaries.

**`frontend-seo`:**

- Adding SEO to a site or wiring metadata into routes.
- Generating `sitemap.xml`, `robots.txt`, or an RSS feed.
- Adding schema.org JSON-LD structured data.
- Fixing canonical-URL or duplicate-content issues.
- Reviewing SEO coverage before release.

**`frontend-lighthouse`:**

- Adding a performance gate / Lighthouse CI to a project.
- Tuning Core Web Vitals budgets or category score thresholds.
- Wiring the Lighthouse GitHub Actions workflow.
- Debugging flaky or failing Lighthouse runs.
- Reviewing performance before release.

**`frontend-data-contracts`:**

- Designing or reviewing an API client.
- Validating or parsing API responses into typed domain values.
- Modeling request/response types and branded IDs.
- Handling API errors or mapping server validation to form fields.
- Stopping raw `fetch` calls from spreading through components.

**`frontend-optimistic-mutations`:**

- Writing create/update/delete mutations.
- Adding optimistic UI with rollback.
- Making writes idempotent and safe to retry.
- Keeping list and detail caches consistent after a write.
- Reviewing mutation and cache-invalidation code.

**`frontend-observability`:**

- Adding analytics or product tracking.
- Instrumenting user actions or designing an event schema.
- Reporting real-user Web Vitals (alongside the Lighthouse lab gate).
- Wiring error reporting or gating telemetry on consent.

---

## License

[MIT](./LICENSE) — use, copy, and adapt freely.
