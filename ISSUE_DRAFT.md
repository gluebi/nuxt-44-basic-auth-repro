# Issue draft — paste into https://github.com/nuxt/nuxt/issues/new?template=bug-report.yml

**Title**

> Router init crashes with `SecurityError` on `history.replaceState` when document URL contains userinfo (basic-auth in URL)

---

### Environment

```
Operating System: Darwin (macOS)
Node Version:     v20 (alpine, inside docker) / v24.15.0 (host)
Nuxt Version:     4.4.6 broken / 4.2.2 control (also reproduces on 4.2.2 if vue-router is forced to 5.0.7)
CLI Version:      —
Nitro Version:    2.13.4
Package Manager:  pnpm@10.28.2
Builder:          vite (default)
User Config:      defineNuxtConfig({}) — bare, no modules, no plugins
Runtime Modules:  none
Build Modules:    none
Browser:          Chromium 148 (via Playwright, headless). Same behaviour is expected in any current Chromium-based browser.
```

### Reproduction

Full minimal repro: **https://github.com/gluebi/nuxt-44-basic-auth-repro**

```bash
git clone https://github.com/gluebi/nuxt-44-basic-auth-repro.git
cd nuxt-44-basic-auth-repro
docker compose up --build
# wait for "Listening on http://0.0.0.0:3000"
```

Open Chrome with a fresh profile and navigate to `http://test:test@localhost:8080/`:

```bash
open -na "Google Chrome" --args --user-data-dir=/tmp/repro
# then visit the URL above
```

The companion repo [`gluebi/nuxt-42-basic-auth-control`](https://github.com/gluebi/nuxt-42-basic-auth-control) (Nuxt 4.2.2, transitively pulls vue-router 4.6.4) ships an identical setup on `:8082` and **does not** crash, isolating the regression to Nuxt 4.4 — the release that bumped the declared vue-router peer from `^4.6.4` to `^5.0.3`.

> The docker-compose + nginx fronting is required because Chrome only preserves userinfo in `document.URL` after a real `401 / WWW-Authenticate` challenge. StackBlitz / CodeSandbox cannot reproduce this — there is no way to inject a 401-issuing proxy in front of their preview hosts.

### Describe the bug

When the client-side bundle boots on a page whose `document.URL` carries userinfo (RFC 3986 `user:pass@host` form), Nuxt's initial router setup calls `history.replaceState(state, '', newUrl)` with a `newUrl` built from `base + path` that omits the userinfo. Chromium's `replaceState` security check enforces **userinfo equality**, not just origin equality, and rejects the call with `SecurityError`.

The throw escapes the router-init code path, so `useRouter()` returns `undefined` for every plugin that runs after it. This produces the visible cascade:

1. `SecurityError: Failed to execute 'replaceState' on 'History'`
2. `[nuxt] error caught during app initialization Error: Context conflict`
3. `Hydration completed but contains mismatches.`
4. `[nuxt] error caught during app initialization TypeError: Cannot read properties of undefined (reading 'beforeEach')`

User-visible effect: hydration fails, `<NuxtLink>` clicks fall back to full page loads, and any composable that depends on the runtime router breaks.

The same Nuxt app works on subsequent visits once Chrome has cached the credentials, because the document URL no longer carries userinfo — so the bug only affects the very first navigation. That makes it intermittent and easy to miss in dev, but it bites real users who are sent basic-auth links (staging environments, internal tooling, etc.).

### Additional context

**Regression boundary observed:**

| Nuxt    | declared peer    | natural install | First-load URL                          | Result   |
|---------|------------------|-----------------|-----------------------------------------|----------|
| 4.2.x   | `vue-router ^4.6.3`  | 4.6.4         | `http://test:test@localhost:8082/`      | ✅ clean   |
| 4.3.x   | `vue-router ^4.6.4`  | 4.6.4         | _(not run; expected clean)_              | —          |
| **4.4.x**   | **`vue-router ^5.0.3`** | **5.0.7**       | `http://test:test@localhost:8080/`      | ❌ crashes |
| 4.2.2 (forced override) | `vue-router 5.0.7` | 5.0.7 | `http://test:test@localhost:8080/`      | ❌ crashes |

**The regression starts in Nuxt 4.4** — the release that bumped the declared vue-router peer from `^4.6.x` to `^5.0.x`. Forcing `vue-router@5.0.7` under Nuxt 4.2.2 also reproduces the crash, so the actual breaking change appears to be on the vue-router side; but the cascade into `Context conflict` and `useRouter() === undefined` is a Nuxt symptom — vue-router's throw isn't caught in Nuxt's plugin pipeline, so subsequent plugins run without a router. Defensively wrapping the initial `replaceState` in Nuxt would stop the cascade independent of any vue-router change.

**Why `document.URL` looks innocent from JS:** the `document.URL` and `location.href` getters strip userinfo per the URL spec, but the underlying `NavigationEntry` URL that Chromium's History API checks against still has it. That mismatch is what trips `replaceState`.

**Suggested fix directions:**

1. **Preserve userinfo.** When constructing the `newUrl` for the initial `replaceState`, copy userinfo from the current document URL onto `base + path`. The call then no-ops.
2. **Defensive try/catch.** Wrap the initial `replaceState` in `try { … } catch (e) {}` so a userinfo-mismatch throw doesn't leave the router undefined for downstream plugins.

(1) is the principled fix and probably belongs in vue-router. (2) is the cheaper cascade-stopper and could live in either project — it would also harden Nuxt against any future first-call `replaceState` rejection by Chromium.

**Browser scope:** Firefox sanitises userinfo from `document.URL` earlier in its pipeline, so the `SecurityError` doesn't fire there — but the underlying logic (Nuxt/vue-router building a `newUrl` that drops userinfo) is browser-independent. The Chromium-side observation just happens to be the loudest symptom.

### Logs

Console output captured verbatim from Chromium 148 against Nuxt 4.4.6 (full file in `console-errors-44.log` in the repro repo):

```
SecurityError: Failed to execute 'replaceState' on 'History': A history state object with URL 'http://localhost:8080/' cannot be created in a document with origin 'http://localhost:8080' and URL 'http://test:test@localhost:8080/'.
    at o (http://localhost:8080/_nuxt/DB_mcCIs.js:4:100621)
    at Xp (http://localhost:8080/_nuxt/DB_mcCIs.js:4:100374)
    at Qp (http://localhost:8080/_nuxt/DB_mcCIs.js:4:101045)
    at setup (http://localhost:8080/_nuxt/DB_mcCIs.js:4:122198)
    at http://localhost:8080/_nuxt/DB_mcCIs.js:4:68826
    at r (http://localhost:8080/_nuxt/DB_mcCIs.js:4:69670)
    at Object.runWithContext (http://localhost:8080/_nuxt/DB_mcCIs.js:4:13268)
    at ci (http://localhost:8080/_nuxt/DB_mcCIs.js:4:69707)
    at http://localhost:8080/_nuxt/DB_mcCIs.js:4:67584
    at Qi.run (http://localhost:8080/_nuxt/DB_mcCIs.js:2:4939)

[nuxt] error caught during app initialization Error: Context conflict
    at r (http://localhost:8080/_nuxt/DB_mcCIs.js:4:65828)
    at Object.set (http://localhost:8080/_nuxt/DB_mcCIs.js:4:66212)
    at ci (http://localhost:8080/_nuxt/DB_mcCIs.js:4:69691)
    ...

Hydration completed but contains mismatches.

[nuxt] error caught during app initialization TypeError: Cannot read properties of undefined (reading 'beforeEach')
    at setup (http://localhost:8080/_nuxt/DB_mcCIs.js:4:132702)
    ...
```
