# Project Structure Templates

Starting points, not mandates. Remove anything the requirements do not need (no empty folders, no unused files). Rename `otp-widget` / `otpw-` to the user's chosen name/prefix.

## Contents

1. Vanilla TypeScript Web Component (embed + optional npm)
2. React component / package
3. Vue 3 component / package
4. Angular standalone component
5. Documentation set (all projects)
6. README skeleton

## 1. Vanilla TypeScript Web Component

```
otp-widget/
├── package.json            scripts: dev, build, preview, test, typecheck, lint, test:e2e
├── tsconfig.json           strict: true
├── vite.config.ts          library mode: formats ['es', 'iife'], name 'OtpWidget'
├── .env.example            VITE_OTP_API_BASE_URL=https://api.example.com (public values only)
├── .gitignore              node_modules, dist, .env, coverage, playwright-report
├── index.html              demo page (loads src/demo.ts)
├── src/
│   ├── index.ts            public exports: init(), OtpWidgetElement, types
│   ├── config.ts           OtpWidgetConfig type, defaults, validateConfig()
│   ├── api/
│   │   ├── client.ts       request(): fetch + timeout + AbortController + normalize
│   │   └── endpoints.ts    endpoint map + response/error adapters (from the user's contract)
│   ├── core/
│   │   ├── otp-machine.ts  state machine (otp-flow.md §B), timers via injected clock
│   │   └── errors.ts       error codes + mapping
│   ├── ui/
│   │   ├── otp-widget.ts   custom element, Shadow DOM, render(state)
│   │   ├── templates.ts    HTML builders (escape all dynamic text)
│   │   └── styles.css      tokens + components (imported as string with ?inline)
│   └── demo.ts             demo wiring, optional ?mock=1 mode
├── tests/
│   ├── unit/               otp-machine.test.ts, client.test.ts, config.test.ts
│   ├── integration/        otp-widget.test.ts (DOM + MSW)
│   ├── mocks/handlers.ts   MSW handlers from confirmed API examples
│   └── e2e/                otp-flow.spec.ts (Playwright)
└── docs/                   see section 5
```

npm package additions (only if requested): `"exports"`, `"types"`, `"files": ["dist"]`, `vite-plugin-dts` or `tsc --emitDeclarationOnly`.

## 2. React component / package

```
otp-widget-react/
├── package.json            react/react-dom as peerDependencies when published
├── vite.config.ts          app mode for demo; library mode if packaged
├── .env.example
├── src/
│   ├── index.ts            export { OtpWidget, useOtp } and types
│   ├── config.ts
│   ├── api/client.ts, api/endpoints.ts
│   ├── core/otp-machine.ts        framework-free logic (reducer + effects)
│   ├── hooks/useOtp.ts            binds machine to React state, cleanup on unmount
│   ├── components/
│   │   ├── OtpWidget.tsx          composition + theming
│   │   ├── IdentifierStep.tsx
│   │   ├── CodeStep.tsx
│   │   └── OtpInput.tsx           single or segmented input
│   ├── styles/otp-widget.module.css
│   └── demo/main.tsx
├── tests/ (unit, integration with Testing Library + MSW, e2e with Playwright)
└── docs/
```

## 3. Vue 3 component / package

```
otp-widget-vue/
├── src/
│   ├── index.ts
│   ├── config.ts
│   ├── api/client.ts, api/endpoints.ts
│   ├── core/otp-machine.ts
│   ├── composables/useOtp.ts
│   ├── components/OtpWidget.vue, IdentifierStep.vue, CodeStep.vue, OtpInput.vue
│   └── demo/main.ts
├── tests/
└── docs/
```

## 4. Angular standalone component (new code in Angular 17+)

```
src/app/otp-widget/
├── otp-widget.component.ts|html|scss   standalone, OnPush, signals
├── identifier-step.component.ts
├── code-step.component.ts
├── otp-input.component.ts
├── otp-widget.config.ts                InjectionToken + defaults
├── otp-api.service.ts                  HttpClient, endpoint map, adapters
├── otp-state.service.ts                signals-based state machine, timers
└── *.spec.ts
```

For an embed from Angular, wrap with `@angular/elements` `createCustomElement` (as the reference does). Mention the bundle-size cost to the user.

## 5. Documentation set (all projects)

Only describe behaviour that exists. Combine files when the project is small (e.g. README + docs/INTEGRATION.md), but every topic below must be covered somewhere.

| File | Must contain |
| --- | --- |
| `README.md` | What it is, features actually implemented, quick start, scripts, links to docs |
| `.env.example` | Every env var with placeholder value + comment; note which are public |
| `docs/API.md` | Fill from `templates/api-documentation.md` |
| `docs/CONFIGURATION.md` | Fill from `templates/widget-configuration.md` |
| `docs/INTEGRATION.md` | Script tag usage, npm usage, framework usage (only the formats built), callbacks/events, server-side token validation step for the host backend |
| `docs/CUSTOMIZATION.md` | Theme tokens table, logo/name, texts/i18n, custom CSS, layout options, examples |
| `docs/ARCHITECTURE.md` | Layer diagram, state machine diagram, file map, design decisions |
| `docs/TESTING.md` | How to run tests + actual results table (`testing-guidelines.md` §7) |
| `docs/DEPLOYMENT.md` | Build commands, outputs, hosting the embed (CDN, cache headers, versioned filenames, SRI hash command), CORS reminder |
| Demo | `index.html` / demo route reachable via `npm run dev` and from the production build preview |

## 6. README skeleton

~~~markdown
# <Widget name>

<One sentence: what it does and for whom.>

## Features
- <Only implemented features: identifier types, channels, resend, timer, theming, formats>

## Quick start
```bash
npm install
cp .env.example .env   # fill in public values
npm run dev            # demo at http://localhost:5173 (add ?mock=1 for mock API)
```

## Usage
### Script tag
### npm
### <Framework>

## Configuration
Short table of the most used options → docs/CONFIGURATION.md

## Security
- No secrets in this package. <Where the private key lives.>
- Verify the returned token on your server: docs/INTEGRATION.md#server-side-validation

## Scripts
| Script | Purpose |

## Docs
API · Configuration · Integration · Customization · Architecture · Testing · Deployment
~~~
