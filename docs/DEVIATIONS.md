# Deviations

Every departure from an earlier draft of [00-TEMPLATE-BLUEPRINT.md](00-TEMPLATE-BLUEPRINT.md), with the reason and the evidence.

Two kinds of entry appear here. **V-01 to V-14** are the §6.2 verification gate: API assumptions checked against current documentation and shipped package source. **D-01 onwards** are everything else — version corrections, tooling changes, and defects found in the blueprint itself.

Gate completed **2026-07-27** against `astro@7.1.4`, `@astrojs/vercel@11.0.3`, `tailwindcss@4.3.3`.

---

## Part 1 — Verification gate (V-01 … V-14)

### V-01 — `output` modes and per-route `prerender`

**As documented.** `"static"` and `"server"` remain the only accepted values, `"static"` is still the default, and `export const prerender` behaves as described. `hybrid` was removed in Astro 5 and has not returned.

**Additional finding.** Astro 7 stabilised two top-level options the blueprint predates: `cache` and `routeRules`, promoted out of `experimental` by PR #17116. Astro 6 also replaced `experimental.failOnPrerenderConflict` with a top-level `prerenderConflictBehavior`. The first of these changed our architecture — see D-06.

Source: https://docs.astro.build/en/reference/configuration-reference/, https://docs.astro.build/en/guides/upgrade-to/v7/

---

### V-02 — `@astrojs/vercel` import path and ISR options

**Partially as documented; one deprecation, and the option is no longer used.**

`import vercel from "@astrojs/vercel"` is correct. The older `@astrojs/vercel/serverless` and `/static` entrypoints were removed in v10, so anything written against them fails outright.

`isr: { expiration, exclude, bypassToken }` is a real and correct option shape, verified against the adapter's `VercelISRConfig` interface. **We do not use it** — see D-06.

Three findings that still matter:

- `edgeMiddleware: true` is deprecated in favour of `middlewareMode: 'edge' | 'classic'`.
- The adapter has **no per-route runtime option**. It ships only a serverless entrypoint; the `/edge` entrypoint is long gone. This constrains `@vercel/og` (V-12).
- The adapter infers the production Node runtime **from the local Node major version**, falling back to a default when unsupported. The machine that runs the build therefore silently determines the production runtime unless it is pinned. Recorded as an operational requirement in §18.1.

Source: https://docs.astro.build/en/guides/integrations-guide/vercel/, adapter source `dist/index.js`

---

### V-03 — `astro:env` schema, `envField`, secret imports

**As documented, and stable since Astro 5.** The `env.schema` block, the `envField` helpers, the option names, and both `astro:env/server` and `astro:env/client` are all exactly as assumed.

**One assumed constraint does not exist.** An earlier draft implied a secret cannot carry a `default`. Verified against `packages/astro/src/env/schema.ts`: `default` is permitted on any field regardless of `access`. That rule was phantom and has been dropped.

**Three real constraints were missing and are now documented:**

- Only three `context`/`access` combinations are valid. `client` + `secret` throws "Secret client variables are not supported."
- Variable names must match `/^[A-Z0-9_]+$/` and cannot start with a digit.
- `astro:env` is a virtual module and is therefore **unusable inside `astro.config.ts`**. The config file reads `process.env` directly.

`envField.enum` exists and is now used for `RENDER_MODE`.

Source: https://docs.astro.build/en/guides/environment-variables/, `packages/astro/src/env/schema.ts`

---

### V-04 — Endpoint signature and `Response`

**As documented.** `import type { APIRoute } from "astro"` is valid; the context object still exposes `request`, `params`, `url`, `redirect`, `rewrite`, `locals`, `cookies`, `site` and `clientAddress`, plus newer additions including `cache`. Handlers return a standard `Response`, and `export const prerender = false` forces on-demand rendering. Extension-bearing filenames such as `sitemap.xml.ts` still work.

Source: https://docs.astro.build/en/guides/endpoints/

---

### V-05 — Built-in `i18n` configuration

**Config shape as documented; one default changed.**

`i18n: { defaultLocale, locales, routing: { prefixDefaultLocale } }` is correct, and `prefixDefaultLocale` still defaults to `false`. `Astro.currentLocale` still exists.

**Breaking change we would have missed.** Astro 6 flipped `i18n.routing.redirectToDefaultLocale` from `true` to `false` (PR #14406). `astro.config.ts` now sets it explicitly rather than relying on a default that changed under us.

Also noted: `Astro.preferredLocale` is `undefined` on prerendered pages, which is a trap for a mostly-static template.

Source: https://docs.astro.build/en/reference/modules/astro-i18n/, astro CHANGELOG 6.0.0

---

### V-06 — Rendering a component held in a variable

**Works, but the type we would have imported does not exist.**

`const Component = lookup(name)` followed by `<Component {...props} />` works, with the usual constraints: the variable must be capitalized, and `client:*` directives are unsupported on dynamic tags.

**`AstroComponentFactory` is not exported from the `astro` root.** An import from `"astro"` fails. The public path is `AstroInstance`, whose `default` property is exactly that factory type:

```ts
import type { AstroInstance } from "astro"
type SectionComponent = AstroInstance["default"]
```

**Consequence: the `as any` cast an earlier draft placed in `AnySection.astro` is unnecessary.** `src/lib/sections.ts` types the registry as `Record<string, AstroInstance["default"]>` and props spread as `Record<string, unknown>`, with no cast anywhere. This is strictly better than the original specification.

Astro's documentation does not cover typing a dynamic component, so this was derived from shipped type declarations.

Source: `astro@7.1.4/dist/types/public/{context,content}.d.ts`, https://docs.astro.build/en/reference/astro-syntax/

---

### V-07 — `import.meta.glob` with `eager: true`

**As documented, including the key shape.** Verified against Vite 8.1.5's `importMetaGlob.ts`: when a pattern is absolute (starts with `/`), keys are computed as `relative(root, file)` and prefixed with `/`, producing exactly `/src/components/sections/SectionHero.astro`. Relative patterns instead yield the specifier as written. Our pattern is absolute, so `nameFromPath()` works unchanged.

`as: "raw"` is deprecated in favour of `query`, but not removed. Patterns must still be static string literals.

Source: https://vite.dev/guide/features, Vite 8.1.5 source

---

### V-08 — `set:html`

**As documented, with no new escaping.** Astro's docs are explicit that the value is not escaped. This makes `sanitize-html` load-bearing rather than defensive, which §11.11 now states directly.

Source: https://docs.astro.build/en/reference/directives-reference/

---

### V-09 — Tailwind 4 via `@tailwindcss/vite`

**As documented.** `vite: { plugins: [tailwindcss()] }` plus `@import "tailwindcss"` is the current recommendation. `@theme` and `@custom-variant` are correct as written.

**`@astrojs/tailwind` is unusable here.** Its peer range is `astro: ^3 || ^4 || ^5`, so it cannot be installed against Astro 7 without forcing overrides.

**One Astro-specific trap, now a rule.** In Tailwind 4, theme variables are not visible from separately-processed stylesheets, which includes `<style>` blocks in `.astro` components. `@apply` there fails with "Cannot apply unknown utility class" unless a `@reference` line is added. Rather than requiring that line everywhere, §10.5 and §12.1 forbid `@apply` in component style blocks entirely.

Also noted: Tailwind 4 has no `darkMode` config key; `@custom-variant dark` is the only mechanism.

Source: https://tailwindcss.com/docs/installation/framework-guides/astro, https://github.com/tailwindlabs/tailwindcss/discussions/16429

---

### V-10 — `getStaticPaths` for a rest route

**Return shape as documented; two corrections.**

The root path of a `[...slug]` route is `params: { slug: undefined }`, **not an empty string**. Param values must be strings. `Astro` is deprecated inside `getStaticPaths()` as of v6 — use `import.meta.env.SITE`.

**The i18n assumption was wrong, and it shaped the architecture.** Astro's built-in i18n is folder-based: it expects real `src/pages/<locale>/` directories, and `prefixDefaultLocale`, `i18n.fallback` and `fallbackType: "rewrite"` all assume that layout. There is no documented way to combine it with a root `src/pages/[...slug].astro` catch-all — which is exactly what a CMS-driven site needs.

**Adaptation.** This template derives the locale from URL segments itself in `parseRoute()` rather than depending on Astro's i18n middleware. Shipping one unprefixed locale means this costs nothing today; adding a locale requires setting `i18n.routing: "manual"` but no page-tree restructuring. Recorded in §7.6 and ADR-0006, and as risk R-14.

Source: https://docs.astro.build/en/reference/routing-reference/, https://docs.astro.build/en/guides/internationalization/

---

### V-11 — 404 signalling under on-demand rendering

**As documented, and better than assumed.** `Astro.rewrite()` exists with signature `(payload: string | URL | Request) => Promise<Response>`. It does not generically preserve status, but `astro@7.1.4`'s `dist/runtime/server/render/page.js` hardcodes `status = 404` when the rewritten route is the 404 route:

```js
if (route?.route && isRoute404(route.route)) {
  status = 404
  if (statusText === "OK") statusText = "Not Found"
}
```

So `if (!page) return Astro.rewrite("/404")` produces a genuine 404. FR-07 works as specified. `Astro.originPathname` exposes the pre-rewrite path inside `404.astro`.

Source: `astro@7.1.4/dist/runtime/server/render/page.js`, https://docs.astro.build/en/guides/routing/

---

### V-12 — `@vercel/og` from an Astro endpoint

**Works, with two caveats that were not anticipated.**

`import { ImageResponse } from "@vercel/og"` is correct, and it **does not require the Edge runtime** — there is a full Node build. That matters, because `@astrojs/vercel@11` offers no per-route runtime option (V-02), so the Node/serverless runtime is the only option available.

Two frictions:

- **`@types/react` must be installed for `@vercel/og` to typecheck at all.** Its declarations import from `react`. This is types-only with no runtime footprint, so it does not violate ADR-0003, but it is an odd dependency in a no-framework template and is now listed explicitly in §6.1.
- Passing plain object element trees instead of JSX is documented by Satori but **not by Vercel**, so it is an unsupported contract. Recorded as risk R-10 and to be verified on a real preview deployment in Phase 7.

No newer replacement exists as of 2026-07: `@vercel/functions@3.7.6` exposes no `og` export.

Source: https://vercel.com/docs/og-image-generation, `@vercel/og@0.11.1` type declarations

---

### V-13 — Content collections

**As documented, and now mandatory when used.** Astro 7 content collections config lives in a root `content.config.ts` (path naming per Astro docs), collections use `defineCollection()` with a `loader`, `glob()` and `file()` come from `astro/loaders`, and `getCollection()` queries. This template does **not** ship content collections (out of scope).

Breaking changes worth knowing if a project opts in: legacy collections are fully removed in v6 with no compatibility flag; `z` must be imported from `astro/zod`, not `astro:content`; `entry.slug` is gone in favour of `entry.id`; and `entry.render()` is replaced by `render(entry)` imported from `astro:content`.

Content collections remain optional and out of scope for the template itself (§4.2).

Source: https://docs.astro.build/en/guides/content-collections/, https://docs.astro.build/en/guides/upgrade-to/v6/

---

### V-14 — `astro add`

**As documented.** Still the recommended way to wire integrations.

One note: `npm create astro@latest` now writes an `AGENTS.md` by default, skippable with `--no-ai`. The `basics` template ships `astro` as its only dependency and includes neither `typescript` nor `@astrojs/check`.

Source: https://docs.astro.build/en/install-and-setup/

---

## Part 2 — Other deviations

### D-01 — TypeScript pinned to `^6.0.3`, not `^7.0.2`

**Blueprint said:** `typescript@^7.0.2`, described as verified against the registry.

**Reality:** the version exists, but nothing in the toolchain accepts it.

- `@astrojs/check@0.9.10` — the latest published version, with no newer dist-tag — declares `peerDependencies: { typescript: "^5.0.0 || ^6.0.0" }`.
- `typescript-eslint@8.65.0` declares `peerDependencies: { typescript: ">=4.8.4 <6.1.0" }`. Its canary `8.65.1-alpha.8` has the same bound, so no unreleased version helps either.

This is not merely a peer-range technicality. TypeScript 7 is the native compiler port and no longer exposes the programmatic API the Astro language server depends on. Astro issues [#17268](https://github.com/withastro/astro/issues/17268) and [#17336](https://github.com/withastro/astro/issues/17336) are both open and labelled `triage: unable to fix`, because the root cause is upstream.

**Adaptation:** `typescript@^6.0.3`, the single latest version satisfying both constraints simultaneously. Keeping TypeScript 7 would have broken `pnpm run typecheck` and `pnpm run lint`, contradicting Phase 1's own acceptance criteria. Revisit when `@astrojs/check` publishes TS 7 support.

---

### D-02 — `eslint-plugin-jsx-a11y` added to the stack

**Blueprint said:** `eslint`, `eslint-plugin-astro`, `typescript-eslint`, "latest majors".

**Reality:** `eslint-plugin-astro@3.0.1` declares three required peers, one of which the blueprint omits entirely:

```
{ "@typescript-eslint/parser": ">=8.61.0", "eslint": ">=10.0.0", "eslint-plugin-jsx-a11y": ">=6.10.2" }
```

**Adaptation:** `eslint-plugin-jsx-a11y@^6.10.2` added, and `eslint` pinned `^10.8.0` since the plugin requires ESLint 10. `astro-eslint-parser@^3.0.0` added explicitly.

---

### D-03 — Prettier stays on 3.x

Prettier 4 exists only as an alpha. `@astrojs/language-server`, `pretty-quick` and `prettier-plugin-tailwindcss` all peer on `^3.0`. Pinned `prettier@^3.9.6`.

---

### D-04 — pnpm replaces npm, and its settings have moved twice

**Owner decision (§23.1 item 6).** Consequences beyond a find-and-replace:

- **The `pnpm` field in `package.json` is dead.** pnpm 11 ignores it and warns that the keys were dropped. All settings move to `pnpm-workspace.yaml`.
- **The build-scripts allowlist was also renamed.** pnpm 10's `onlyBuiltDependencies` (a list) became pnpm 11's `allowBuilds` (a map). The old name is not merely deprecated — it fails silently and misleadingly: `pnpm config get onlyBuiltDependencies` echoes the value back, so it looks configured, while every install still prints `ERR_PNPM_IGNORED_BUILDS` and leaves the packages unbuilt. `pnpm approve-builds <pkg>` writes the correct key.
- This matters because `astro@7.1.4` depends on `esbuild@^0.28` and on `sharp` for image optimisation, both of which have install scripts.
- **`pnpm setup` is a built-in pnpm command**, so a script named `setup` is shadowed. The script keeps its name for vocabulary parity, but every invocation and every piece of documentation uses `pnpm run setup` with the explicit `run`.
- `pnpm-lock.yaml` is committed; `npm ci` becomes `pnpm install --frozen-lockfile`.
- `strictPeerDependencies: true` is set deliberately — it is exactly what surfaces problems like D-01 and D-24 at install time rather than at first typecheck. **No `.npmrc` is committed**; a second config file expressing the same settings would only drift.
- pnpm 11 appends a generated `minimumReleaseAgeExclude` list to the same file as part of its supply-chain quarantine. It is machine-maintained; leave it alone.

---

### D-05 — `astro.config.ts` instead of `astro.config.mjs`

The locale list must be a single source of truth shared between the Astro config and the type layer (§11.1). A `.ts` config can import it directly. Astro supports TypeScript config files, so this costs nothing.

---

### D-06 — Native CDN caching replaces adapter ISR

**Blueprint said:** `output: "server"` with the adapter's `isr: { expiration, bypassToken, exclude }` block, recorded as ADR-0005.

**Reality:** Astro 7 stabilised top-level `cache` and `routeRules`, and `@astrojs/vercel@11` added `cacheVercel()` to back them. Comparing the two revealed a disqualifying property of ISR.

**Adapter ISR strips every query parameter.** The adapter writes a `.prerender-config.json` whose `allowQuery` lists only Astro's internal `x_astro_path` parameters, and Astro's own documentation confirms "ISR function requests do not include search params." FR-13 requires `/posts` to paginate. Under ISR, `?page=2` would be discarded and every visitor would see page 1 — and that limitation would ship to every site built from this template.

Three supporting reasons: ISR has a single global `expiration` with no per-route TTLs; excluding `/api/**` requires a regex matched against Astro's internal _route pattern strings_ rather than request paths, which is easy to get silently wrong; and ISR offers no tag invalidation, which is the natural shape for a CMS where one entry maps to several URLs.

**Adaptation:** `cache: { provider: cacheVercel() }` plus per-route `routeRules`. Both mechanisms serve genuinely CDN-cached HTML with no function invocation on a hit, so NFR-12 holds either way. ADR-0005 rewritten.

**Accepted costs, recorded as R-11.** `cacheVercel()` shipped five weeks before this gate and Astro's own documentation labels the CDN providers experimental. On a cache _miss_ ISR is stronger — durable store, cache shielding, request collapsing, ~300ms global purges — so the native regional cache hits the SQL API roughly once per region per TTL instead of once per TTL globally. Mitigations: `@astrojs/vercel` is pinned exactly rather than with a caret, and two tests assert caching behaviour. Reverting to ISR is a config-level change.

---

### D-07 — Two undocumented traps in `routeRules`

Both found by reading `astro@7.1.4` source rather than documentation.

**A rule with only `swr` and no `maxAge` emits no headers at all.** The runtime gate in `dist/core/cache/runtime/cache.js` checks `maxAge` and `tags`, never `swr`:

```js
if (finalOptions.maxAge === void 0 && !finalOptions.tags?.length) return
```

The example in Astro's own configuration reference and caching guide is affected by this. **Every `routeRules` entry in this template sets `maxAge`.**

**A catch-all rule matches `/api/**`, and there is no declarative opt-out.** Rules compile through Astro's real routing matcher and the first match wins, so `/[...slug]` also matches `/api/anything`. Adding `'/api/[...path]': {}` does not shadow it — `normalizeRouteRuleCacheOptions()` drops any rule whose cache fields are all undefined. The only reliable guard is `cache.set(false)` at runtime.

**Adaptation:** `src/middleware.ts` is mandatory, not optional, and a test asserts `/api/**` carries no CDN cache header (NFR-13).

---

### D-08 — `X-XSS-Protection` omitted

Present in the predecessor's header set. Deprecated and ignored by modern browsers; omitted deliberately from `vercel.json`.

---

### D-09 — `uid()` installer behaviour (revised after live probe)

**Earlier claim:** `uid()` is not Orbitype-provided; the installer must create it or installation fails.

**Live finding (2026-07-27):** on a fresh empty connector, `uid()` already existed and was byte-identical to the blueprint’s 6-character function. `CREATE OR REPLACE FUNCTION uid()` returns **500** with `must be owner of function uid`. Tables still create successfully because the function is present.

**Adaptation:** the installer still attempts `CREATE OR REPLACE`, then on failure runs `SELECT uid()` and treats a working function as success. Idempotent insurance remains correct; the premise that Orbitype never provisions `uid()` was too strong. ADR-0011 updated.

---

### D-10 — `pages` gains `lead` and `img`

**Blueprint said:** `pages` has `id`, `title`, `slug`, `sections`, `keywords`, `head`, `created_at`, `updated_at`.

**Reality:** the real table also has `lead json` and `img text`.

Since both templates read the same databases, omitting them would have broken the cross-stack compatibility that §22 item 8 exists to prove. Both added to §8.4.

---

### D-11 — The seed lookup returns `null` for unknown slugs

**Blueprint said:** `getPage()` returns `welcomePage(slug)` whenever a row is missing.

**Reality: that cannot satisfy FR-07.** If every miss returns the welcome page, no slug ever 404s in mock mode — and §17.2's "unknown slug returns HTTP 404" test runs in mock mode, so it would have failed. The predecessor gets this right by matching the slug against known seeds and returning `null` otherwise.

**Adaptation:** `findSeedPage(slug)` returns the matching page or `null`, and `getPage()` propagates that. Adopting the seed system (D-13) makes this fall out naturally.

---

### D-12 — `normalizeSections()` adopted

Not in the blueprint at all. Orbitype sometimes stores `sections` as a nested array — `[[{...}]]` — which the predecessor's own CMS documentation calls out as a known pitfall. A plain `sections.map()` would render nothing useful for such a payload.

**Adaptation:** `src/lib/normalize-sections.ts` recursively flattens and discards anything without a string `_orbi.component`. Every `sections` read passes through it. Added as FR-24 with a test.

---

### D-13 — Five features adopted from the predecessor

Owner-approved scope additions the blueprint omitted.

- **Seed system.** The predecessor builds its mock pages from the same builders that populate the database. Adopting that gives one definition of starter content serving mock mode, the fallback, and `pnpm run cms:seed` — and fixes D-11 as a side effect. ADR-0012.
- **`templates` table.** `sections_before` / `sections_after` for chrome shared across pages, changing composition to `[...before, ...page, ...after]`. ADR-0013, FR-25.
- **Migration action.** `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` statements alongside the DDL, exposed via `pnpm run cms:migrate` (CLI only — never HTTP).
- **`phrases` catalogue.** The typed chrome-string object §4.2 alludes to but never specified. Now §11.18.
- **MCP and Figma verification scripts.** `mcp:env`, `mcp:verify`, `figma:verify` (Figma is REST, not MCP).

`contacts` keeps generic columns; the predecessor's project-specific fields are dropped.

---

### D-14 — ADR-0010's premise was false

**Blueprint said:** `SectionSpacer` does not exist in the predecessor, so a freshly installed page renders the debug fallback, and shipping the component fixes that defect.

**Reality:** it does exist, with a `height` prop. There is no upstream defect.

**Adaptation:** ADR-0010 rewritten as parity rather than a fix. The reason to ship the component is unchanged — the schema default references it — but the stated justification was wrong.

---

### D-15 — ADR-0009 reframed

**Blueprint said:** the predecessor's installer never creates `contacts`, so this template adds the DDL as an improvement.

**Reality:** the DDL exists upstream. It is simply absent from the installer's hardcoded `["pages","posts","settings"]` list, so the table is defined but never created.

The decision stands — the installer must create it — but the reason is narrower than stated. ADR-0009 updated.

---

### D-16 — Orbitype SQL API error shapes

**Blueprint said:** nothing about error responses.

**Reality**, measured against the live API:

| Condition       | Status    | Content-Type       | Body                                           |
| --------------- | --------- | ------------------ | ---------------------------------------------- |
| No API key      | `400`     | `text/plain`       | `E_HTTP_EXCEPTION: connector required`         |
| Invalid API key | **`404`** | `application/json` | `{"message":"E_ROW_NOT_FOUND: Row not found"}` |
| Unrouted method | `404`     | `application/json` | `E_ROUTE_NOT_FOUND`                            |

Two consequences. Any code that classifies a bad key by a 401 status would never fire, so the welcome-screen fallback would not trigger when it should. And any code calling `.json()` on an error body throws on the 400 case, because it is plain text.

**Adaptation:** §8.3 documents the real shapes; `client.ts` reads error bodies with `.text()` and never assumes 401. Also documented: `PUT` is not routed on `/api/sql/v1`, and every response carries an undocumented `x-ratelimit-limit: 6000`.

---

### D-17 — Every mutation requires `RETURNING`

**Reality:** Orbitype mutation responses contain only the rows named by a `RETURNING` clause. There is no rowcount, no affected-rows field, and no envelope. A mutation without `RETURNING` returns nothing to verify against.

**Adaptation:** the installer, seeder, contacts insert and comments insert all include `RETURNING`, and §8.9's canonical snippets were updated to show it. Added to the database rule in §20.3.

---

### D-18 — `/api/revalidate` ships in v1

**Blueprint said:** out of scope for v1, listed as a candidate v2 feature.

**Reality:** the owner made it conditional on Orbitype supporting row-change triggers, and it does. Workflows offer a "Database (table events)" trigger firing on create, update and delete, handing a code node `[oldRow, newRow]`, and code nodes can `fetch()` any URL.

**Adaptation:** `/api/revalidate` accepts `{ tags?, path? }`, guarded by `REVALIDATE_SECRET`. §8.8 documents the three constraints: the triggering statement must run through the Orbitype API with `RETURNING`; admin-GUI saves appear to fire it but this is documented only indirectly; and a workflow writing back to its trigger row loops without an atomic lock (R-13). Risk R-04 largely dissolves.

---

### D-19 — Single locale, self-derived routing

**Owner decision (§23.1 item 2).** The template ships `LOCALES = ["en"]` with the list in `src/config/locales.ts`.

FR-03 was rewritten: locale resolution is driven by the configured list, and with one locale there is no URL prefix. §17.2's locale-route test applies only when a project configures more than one. `I18nString` keeps the multi-locale shape with `en` required, so a row authored as `{ en, de }` still renders — preserving §22 item 8.

See V-10 for why routing is self-derived rather than delegated to Astro's i18n middleware.

---

### D-20 — `I18nString` requires `en`

**Blueprint said:** `Partial<Record<Locale, string>> & { en?: string }` — everything optional.

**Reality:** the predecessor's type requires `en` and treats it as the terminal fallback, which is what makes resolution total. An all-optional type would allow a value that resolves to nothing in every locale.

**Adaptation:** `en: string` required, with an open index signature for other locales.

---

### D-21 — Email is provider-agnostic

**Owner decision (§23.1 item 3).** No provider chosen. §11.16 defines `EmailProvider` and `EmailMessage` and ships a stub that throws an actionable error until one is wired. Whichever provider a project adopts is called over its HTTP API with `fetch`, never an SDK.

---

### D-22 — Client neutrality

**Owner decision (§23.1 item 8).** §2.3's checklist previously covered third-party companies and individual authors. It now also forbids any client, project or site name anywhere in this repository, because the template is cloned per project and any such name would propagate to unrelated clients.

Scoped to the template only — a clone is expected to carry its own name. Phase 9 enforces it with a repository-wide grep.

---

### D-23 — The reference implementation is unreachable, and §24 no longer cites one

**Blueprint said:** consult a named GitHub repository whenever the document is silent on CMS behaviour, listing four files as most useful.

**Reality:** that repository returns 404 and its organisation reports zero public repositories. Every "consult the reference" instruction pointed nowhere.

A predecessor-stack checkout was made available locally for extraction, but it is a client project, so neither its name nor its path may appear here (D-22).

**Adaptation:** the facts were extracted instead of cited. §8 is now normative and self-contained — verbatim DDL, the `uid()` function, the sections contract, the nested-array behaviour, the real error shapes, the Workflow model and the canonical SQL. §1 no longer tells an agent to consult a reference, §24 has no reference row, and the predecessor is described generically with no name or link.

---

### D-24 — `eslint-plugin-astro` has a self-contradictory peer set

Found by `strictPeerDependencies` failing the first install — the guard justifying its own existence.

`eslint-plugin-astro@3.0.1` declares:

```
{ "eslint": ">=10.0.0", "eslint-plugin-jsx-a11y": ">=6.10.2", "@typescript-eslint/parser": ">=8.61.0" }
```

But `eslint-plugin-jsx-a11y@6.10.2` — the latest published release, with nothing newer available — declares `eslint: "^3 || ^4 || ^5 || ^6 || ^7 || ^8 || ^9"`. The plugin therefore requires ESLint 10 while simultaneously requiring a package that refuses it. Every `eslint-plugin-astro` from 2.0.0 onward carries this; `1.7.0` is the last version without it, and it peers only `eslint >=8.57.0`.

**Adaptation:** a `peerDependencyRules.allowedVersions` entry scoped to exactly that edge:

```yaml
peerDependencyRules:
  allowedVersions:
    eslint-plugin-jsx-a11y>eslint: "10"
```

Two alternatives were rejected. Setting `strictPeerDependencies: false` would have silenced the guard globally, discarding the protection that caught D-01. Pinning `eslint-plugin-astro@1.7.0` would have avoided any override but given up two majors of Astro lint support. jsx-a11y is a rules-only plugin with no real ESLint 10 incompatibility, so a narrowly scoped override is the honest description of the situation. Remove it when jsx-a11y publishes ESLint 10 support.

Verified after the override: install clean, `pnpm run lint` passes with `--max-warnings=0`.

---

### D-25 — Four dependencies the blueprint's stack table omitted

Each surfaced as a hard failure during Phase 1, not as a preference.

| Package                     | Why it is required                                                                                                                                                                                                                                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `@types/node`               | `astro.config.ts` reads `process.env`, because `astro:env` is a virtual module unavailable in the config file (V-03). Without it `astro check` reports `Cannot find name 'process'` and Phase 1's typecheck criterion fails. Pinned `^24` to track the runtime major rather than the newest published major.                                     |
| `@eslint/js`                | `js.configs.recommended` is the base of the flat config. pnpm's strict `node_modules` layout does not expose transitive dependencies, so it must be declared directly. Note it is versioned independently of `eslint` — `eslint@10.8.0` pairs with `@eslint/js@10.0.1`, and assuming the versions match produces `ERR_PNPM_NO_MATCHING_VERSION`. |
| `globals`                   | `scripts/**/*.mjs` are plain JavaScript, so `no-undef` applies and flags `console`, `process` and `fetch`. TypeScript files are exempt because the compiler handles those.                                                                                                                                                                       |
| `@typescript-eslint/parser` | A direct peer of `eslint-plugin-astro@3`. The `typescript-eslint` meta-package depends on it but does not satisfy a root-level peer requirement.                                                                                                                                                                                                 |

`@types/luxon` and `@types/sanitize-html` are also present: neither library ships its own declarations, and both are imported by Phase 2 modules.

---

### D-26 — `cacheVercel()` emits nothing at build time

Expected the Vercel build output to show cache configuration, by analogy with ISR's `.prerender-config.json`.

**Reality:** `.vercel/output/config.json` after a `server`-mode build contains only routing entries. The native cache provider works at **runtime**. Additionally, `@astrojs/vercel` does **not** support `astro preview`, so local CDN-hit tests cannot use the preview server. Caching e2e therefore runs the API-not-cached middleware assertion under `astro dev`; a true CDN HIT is verified on a Vercel deployment (Phase 10).

---

### D-27 — `.cursor/mcp.json` is committed (env placeholders only)

**Earlier claim:** gitignore `.cursor/mcp.json` because it holds live keys.

**Adaptation (ADR-0014):** the file holds only `${env:VAR}` references, so it is safe to commit and clones inherit MCP. `.cursor/mcp.json.example` documents multi-connector scope suffixes. `scripts/check-mcp-safety.mjs` plus the pre-commit hook reject any inlined key.

---

### D-28 — `htmlparser2` pinned to `9.1.0` for Vercel serverless

**Symptom on Vercel Production/Preview:** every SSR request returns 500 with:

```
Error [ERR_REQUIRE_ESM]: require() of ES Module .../htmlparser2@12.0.0/...
from .../sanitize-html@2.17.x/... not supported.
```

**Cause:** `sanitize-html@2.17+` depends on `htmlparser2@^12`, which is ESM-only. The published `sanitize-html` entry is still CommonJS and does `require("htmlparser2")`. Vite/dev often masks this; the `@astrojs/vercel` Node function does not. The crash surfaces as soon as the server bundle loads [`src/lib/sanitize.ts`](../src/lib/sanitize.ts) (via `SafeHtml`), including catch-all hits for missing static assets.

**Adaptation:** pnpm override in `pnpm-workspace.yaml` (pnpm 11 — not `package.json#pnpm`):

```yaml
overrides:
  htmlparser2: "9.1.0"
```

`9.1.0` is dual CJS/ESM (`exports.require` + `exports.import`). Downgrading `sanitize-html` to `2.16.x` also works but drops 2.17 patches; the override is narrower. Remove when `sanitize-html` ships an ESM entry that no longer `require()`s ESM-only `htmlparser2`.

---

## Still unverified

These need a live Orbitype connector and remain open (§23.3):

- The response body shape for DDL statements. Only throw-versus-no-throw is currently reliable.
- Error shapes for SQL syntax errors and constraint violations.
- Whether Workflow Database triggers fire on admin-GUI saves, or only on API writes with `RETURNING`.
- Create and delete payload shapes for Workflow Database triggers — only the update shape is documented.
- The rate-limit window length behind `x-ratelimit-limit: 6000`.
- End-to-end confirmation of a CDN cache hit from an Astro route using `cacheVercel()`. Not observed on a real deployment; scheduled for Phase 10.
