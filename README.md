# Nuxt 4.4 — SecurityError on `replaceState` when document URL contains userinfo

Minimal reproduction for a client-side router-init crash that fires when a page is loaded with HTTP basic-auth credentials embedded in the URL (e.g. `http://user:pass@host/`).

## Summary

When the browser's current document URL carries userinfo (username/password segments per RFC 3986), Nuxt's initial client-side router setup calls `history.replaceState(state, '', newUrl)` with a `newUrl` built from `base + path` that **omits** the userinfo. Chromium's `replaceState` security check enforces userinfo equality (not just origin equality), so the call throws a `SecurityError`. The throw escapes router init, leaving `router` undefined for subsequent plugins, which produces a cascade:

```
SecurityError: Failed to execute 'replaceState' on 'History':
  A history state object with URL 'http://localhost:8080/' cannot be created in a
  document with origin 'http://localhost:8080' and URL 'http://test:test@localhost:8080/'.

[nuxt] error caught during app initialization Error: Context conflict

Hydration completed but contains mismatches.

[nuxt] error caught during app initialization
  TypeError: Cannot read properties of undefined (reading 'beforeEach')
```

The user-visible effect: client-side hydration fails, `<NuxtLink>` clicks fall back to full page loads, and any code that depends on the runtime router (route guards, navigation middlewares, `useRouter()`) breaks.

## Affected versions

Observed while reproducing this report (all on Chromium 148, macOS):

| Nuxt    | declared peer    | natural install | First-load URL                         | Result |
|---------|------------------|-----------------|----------------------------------------|--------|
| 4.2.x   | `vue-router ^4.6.3`  | 4.6.4 | `http://test:test@localhost:8082/`     | ✅ clean (companion repo's control) |
| 4.3.x   | `vue-router ^4.6.4`  | 4.6.4 | _(not tested in this repro)_            | _(expected clean)_ |
| **4.4.x**   | **`vue-router ^5.0.3`** | **5.0.7** | `http://test:test@localhost:8080/`     | ❌ **CRASHES** — all four errors below |
| 4.4.x   | `vue-router ^5.0.3`  | 5.0.7 | `http://localhost:8080/` (auth via dialog) | ✅ clean |

> **The regression starts in Nuxt 4.4.** That is the exact release where Nuxt bumped its declared vue-router peer from `^4.6.3` (Nuxt 4.2.x) → `^4.6.4` (Nuxt 4.3.x) → `^5.0.3` (Nuxt 4.4.x). With a forced `vue-router@5.0.7` override under Nuxt 4.2.2 the bug also fires — so the breaking change appears to live in vue-router 5's initial-history setup. The cascade (`Context conflict`, then `Cannot read properties of undefined (reading 'beforeEach')`) is a Nuxt symptom: vue-router's throw isn't caught in Nuxt's plugin pipeline, leaving `useRouter()` undefined for every plugin that runs after it. Either project can fix it.

## Environment

- **Nuxt**: 4.4.6 (broken) / 4.2.2 (control)
- **vue-router**: 5.0.7 (broken) / 4.6.4 (control, transitively)
- **Node**: 20-alpine (inside docker), 24.15.0 (host)
- **pnpm**: 10.28.2
- **nginx**: alpine (basic_auth)
- **Browser**: Chromium 148 (via Playwright, headless). Captured `console-errors-44.log` and `console-errors-44.png` come from this Chromium. The same bug is expected in any current desktop Chromium-based browser (Chrome, Edge, Arc, Brave) provided the document URL retains userinfo — see Reproducer notes below.
- **OS**: macOS (Darwin)

## How to reproduce

```bash
git clone https://github.com/<you>/nuxt-44-basic-auth-repro.git
cd nuxt-44-basic-auth-repro
docker compose up --build
```

Wait until the `nuxt-1` container prints `Listening on http://0.0.0.0:3000` and nginx is up on `:8080`. Then, in a **fresh** Chrome profile (so cached credentials don't defeat the repro):

```bash
# macOS
open -na "Google Chrome" --args --user-data-dir=/tmp/repro
```

Navigate to:

```
http://test:test@localhost:8080/
```

Open DevTools → Console. You will see the four errors quoted above (`console-errors-44.log` and `console-errors-44.png` in this repo are captures from Chromium 148 via Playwright).

The link rendered as `<NuxtLink to="/about">` does still change pages, but as a full page reload, not a client-side navigation — proof that the runtime router was never created.

### Control case — same URL, no userinfo

In the same fresh profile, close the tab and open `http://localhost:8080/` directly. Chromium will show a basic-auth dialog; enter `test` / `test`. The console will be clean and `<NuxtLink>` navigates client-side normally. This confirms the bug is specifically about the **first** navigation carrying userinfo.

### Regression check — older Nuxt + vue-router 4

A companion repo, [`gluebi/nuxt-42-basic-auth-control`](https://github.com/gluebi/nuxt-42-basic-auth-control) (pinned to Nuxt 4.2.2, which transitively pulls vue-router 4.6.4), ships an identical setup on port `:8082`. Clone it, `docker compose up --build`, then visit `http://test:test@localhost:8082/` in the same fresh Chrome profile. Console is clean; `<NuxtLink>` navigates client-side; no `SecurityError`. The repo's `console-clean-42.log` is the capture from this verification run.

## Reproducer notes (read these — the bug is fiddly)

- **A real 401 challenge is required.** Chrome only preserves userinfo in `document.URL` after the server actually issues `WWW-Authenticate`. That is why this repro fronts Nuxt with nginx + `auth_basic`. If you remove nginx and try `http://test:test@localhost:3000/` against Nuxt directly, Chrome strips the userinfo before navigation and the bug doesn't fire.
- **Fresh Chrome profile.** Once Chrome caches credentials for the host, subsequent visits don't carry userinfo in `document.URL` and the bug doesn't reproduce. Use `--user-data-dir=/tmp/repro` (or a guest profile / incognito with cache cleared).
- **`document.URL` from JS lies.** Inside the page, `document.URL` and `location.href` getters strip userinfo per the URL spec — but the underlying NavigationEntry URL the History API checks against still has it. That mismatch is what trips `replaceState`.
- **Firefox is different.** Firefox sanitises userinfo from `document.URL` earlier in the pipeline, so the `SecurityError` doesn't fire there — but the underlying Nuxt/vue-router behaviour (building a `newUrl` that strips userinfo from the document URL) is browser-independent.

## Suspected cause

> Vue Router's initial `history.replaceState` builds `newUrl` from `base + path` without preserving the document URL's userinfo. Chrome's `replaceState` security check enforces userinfo equality (not just origin), so the call throws `SecurityError`. The throw escapes router init, leaving `router` undefined for subsequent plugins.

The stack trace points into the minified Vite chunk (`/_nuxt/DB_mcCIs.js`); the unminified call is around vue-router's `createWebHistory` initialisation where it normalises the current location and replaces history state. The same code path runs once per page load before any user navigation.

## Suggested fix directions

1. **Preserve userinfo.** When constructing the `newUrl` for the initial `replaceState`, copy userinfo from the current `document` URL onto `base + path`. Strictly more correct — `replaceState` would then no-op.
2. **Defensive try/catch.** Wrap the initial `replaceState` in `try { … } catch (e) { /* userinfo mismatch is benign on first navigation */ }`. Cheaper, ships behind the existing init code path, would also prevent the cascade where the throw leaves `router` undefined.

(1) is the principled fix and probably belongs in vue-router. (2) is the safer cascade-stopper and could live in either project.

## Workaround

Strip userinfo from the document URL on the server before the SPA bundle runs — e.g. a Nitro `render:html` hook that rewrites `<head>`'s base / inline state. Not pretty; ideally not needed.

## Files in this repo

```
.
├── README.md
├── package.json        # Nuxt 4.4.6 (no other deps)
├── nuxt.config.ts      # bare defineNuxtConfig({})
├── app.vue             # <NuxtPage />
├── pages/
│   ├── index.vue       # "Hello" + <NuxtLink to="/about">
│   └── about.vue       # "About"  + <NuxtLink to="/">
├── docker-compose.yml  # nuxt + nginx (basic-auth) on :8080
├── nginx/
│   ├── default.conf    # auth_basic + proxy_pass to nuxt:3000
│   └── .htpasswd       # test:test (bcrypt)
├── pnpm-lock.yaml      # checked in so docker compose can use --frozen-lockfile
├── console-errors-44.log   # captured console output from Chromium 148
├── console-errors-44.png   # full-page screenshot at moment of crash
├── ISSUE_DRAFT.md      # paste-into-Nuxt-issue draft
└── .gitignore
```

No experimental flags, no modules, no SSR tweaks, no custom plugins. Stripped to the bare minimum so the surface area for "is it really Nuxt?" is as small as possible.
