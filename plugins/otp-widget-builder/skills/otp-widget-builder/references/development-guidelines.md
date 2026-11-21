# Development Guidelines

## Contents

1. Mandatory rules
2. Technology selection
3. Output format selection
4. Code organization
5. Customizing an existing widget
6. Dependency policy
7. Quality gates

## 1. Mandatory rules

1. Read `repository-architecture.md` and `otp-flow.md` before writing code. Re-inspect the reference repo only if the user says it changed or you need detail not covered.
2. Understand the existing flow (and, when customizing, the user's existing code) before changing anything.
3. Write minimal, clean, production-ready code. No placeholder logic left in shipped paths.
4. Reuse existing logic in the target project; do not duplicate helpers that already exist.
5. Add no dependency without a clear reason (section 6).
6. Create no file or component without a reason; no empty files.
7. No speculative fallbacks, feature flags or abstraction layers the requirements do not need.
8. Follow the chosen framework's current conventions (e.g. React hooks + function components; Angular standalone components and signals for new Angular 17+ code; Vue 3 Composition API with `<script setup>`).
9. Separate UI, flow/state, API client and configuration (section 4).
10. Never expose secrets (see `api-integration.md` §3).
11. Do not modify unrelated files or behaviour.
12. Fix every lint, type, build and test error your change introduces.
13. Ask before major architectural decisions (framework, state library, output formats, adding a backend, replacing an existing component).
14. Verify the final output (build, tests, demo) before saying it is done; report what was not verified.

Work like a senior engineer: the simplest design that meets the requirements, readable names, small functions, comments only where intent is not obvious.

## 2. Technology selection

Decide from the integration environment, then confirm with the user:

| Situation | Recommended stack | Reason |
| --- | --- | --- |
| Must embed in any website via `<script>` (unknown host stacks) | Vanilla TypeScript + Web Component (custom element, Shadow DOM), bundled with Vite library mode (IIFE + ES) | Smallest bundle, no framework runtime clash, style isolation |
| Host app is React / Next.js | React + TypeScript component (hooks), optional headless hook `useOtp` | Native integration, no custom-element friction |
| Host app is Angular | Angular standalone component + service, signals for state | Matches host DI/forms |
| Host app is Vue / Nuxt | Vue 3 SFC + composable `useOtp` | Native integration |
| Needed in several frameworks | Framework-agnostic TS core (API client + state machine) + thin framework adapters, or one Web Component | One flow implementation |
| Customizing the reference itself (`production-js` branch, `core-js/`) | Keep vanilla JS + Vite + its class structure | Minimal-change rule |

Notes:

- Do not force the reference's stack on every project. Its `production-js` branch is plain JavaScript in one IIFE, which is a good fit for script embeds; the older Angular + Material + NgRx build on `master` is heavy for an embed. For new code prefer TypeScript even when the reference is plain JS.
- Phone input: use `libphonenumber-js` (smaller than `google-libphonenumber`) for validation/formatting; add a country picker only if required. `intl-tel-input` is acceptable when the user wants the reference's look.
- Captcha: integrate only if the backend requires a token; load the provider script on demand.

## 3. Output format selection

Generate only what the user needs:

| Format | Build output | When |
| --- | --- | --- |
| Standalone web app | Vite app (`index.html` + assets) | Hosted verification page, or a demo |
| Embeddable script | `dist/otp-widget.iife.js` exposing `window.OtpWidget.init(config)` and/or `<otp-widget>` element | Third-party sites, CMS, no build step |
| npm package | ES module + type declarations (`.d.ts`), `exports` map, `peerDependencies` for frameworks | Teams installing via npm |
| In-app integration | Component/files added to the user's repo | User has an existing frontend |

A working demo page is always delivered (it can be the standalone app or `demo/index.html` loading the embed build).

Script-embed contract (adapt names to the user's brand):

```js
const widget = OtpWidget.init({
  target: '#otp',            // or omit for modal overlay
  apiBaseUrl: 'https://api.example.com',
  otpLength: 6,
  theme: { '--otpw-color-primary': '#0a7' },
  onSuccess(result) {},      // result shape agreed with the user
  onError(error) {},         // { code, message }
  onClose() {},
});
widget.destroy();            // removes DOM, timers, listeners, aborts requests
```

Fail fast on missing required config with a clear `Error` (the reference does this for `widgetId`/`tokenAuth`/`success`).

## 4. Code organization

Responsibilities, whatever the framework:

| Layer | Contains | Must not contain |
| --- | --- | --- |
| `config` | Typed config schema, defaults, validation | DOM or HTTP code |
| `api` | Endpoint map, request builder, `fetch` + timeout + abort, response/error adapters | UI strings, state |
| `core` / `state` | State machine (`otp-flow.md` §B), timers, limits | `fetch`, DOM |
| `ui` | Components/templates/styles, accessibility | Business rules, endpoint paths |
| `entry` | Embed bootstrap, custom element registration, package exports | Logic |

Write the core so it is testable without a browser (inject the API client and a clock).

## 5. Customizing an existing widget

1. Get the code: path on disk, repo URL or ZIP. Do not guess.
2. Inspect: framework, build, how config flows in, where styles live, existing tests, lint setup, how the widget is consumed.
3. Run the existing build/tests first and record the baseline (including pre-existing failures).
4. Map each requested change to the exact files. Present that map in the plan.
5. Make the smallest change that satisfies each requirement. Keep public APIs backward compatible unless the user approves a breaking change; if breaking, document migration.
6. Re-run build/tests; compare with the baseline; fix only what your change broke, report pre-existing issues separately.
7. Update the docs that describe changed behaviour (only those).

When customizing the reference repo specifically: config-driven behaviour (OTP length, channels, retry time/count, captcha) comes from MSG91's widget dashboard via `getWidgetProcess`, so changing it in code is usually wrong; UI text, layout, styles and host config handling are client-side.

## 6. Dependency policy

Before adding a package, confirm: (a) the platform or existing deps cannot do it in a few lines, (b) it is maintained, (c) bundle-size impact is acceptable for an embed. Typical justified deps: framework itself, `libphonenumber-js`, a test runner, Testing Library, Playwright, MSW. Usually unjustified: lodash, moment/dayjs (timer formatting is trivial), state libraries for a single widget, UI kits inside embeds.

Pin versions that exist; check with the package manager rather than guessing version numbers.

## 7. Quality gates

Run what the project defines; add only what is missing and needed:

- Type check (`tsc --noEmit` or framework equivalent), strict mode on for new projects.
- Lint (ESLint with the framework's recommended config) if the project has or needs it.
- Unit + integration tests (`testing-guidelines.md`).
- Production build succeeds; check embed bundle size and that the IIFE works when loaded by a plain `<script>`.
- Demo page runs and the full flow works against the mock (and the real API when credentials exist).
- `.env.example` complete; no secrets in the repo (`git grep` for keys/tokens before handing off).
