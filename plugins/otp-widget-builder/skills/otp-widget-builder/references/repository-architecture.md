# Reference Repository Architecture

**Reference branch: `production-js`** of `Walkover-Web-Solution/OTP-widget-web`, verified at commit `724c584` (November 2026). It contains the production widget as a vanilla JavaScript SDK in `core-js/`. Every statement below comes from reading that branch's source. The repo's `master` branch holds an older Angular 19 implementation of the same widget; see §12.

Clone it with:

```bash
git clone --branch production-js https://github.com/Walkover-Web-Solution/OTP-widget-web.git
```

If the user says the branch has changed, re-inspect it before relying on this file.

## Contents

1. What it is
2. Stack and dependencies
3. Directory map
4. Runtime architecture
5. Public JavaScript contract (`initSendOTP`)
6. Headless mode (`exposeMethods`)
7. Configuration, environment and build
8. Patterns worth reusing
9. Conventions
10. Known defects (do not copy)
11. Docs in the branch that disagree with the code
12. The `master` branch (Angular, legacy)

## 1. What it is

The browser SDK for MSG91's embeddable OTP widget. A host page loads one IIFE bundle (`otp-provider.js`) and calls `window.initSendOTP({...})`. The SDK fetches the widget configuration from MSG91 (`widget/getWidgetProcess`), renders a dialog in a Shadow DOM, collects a phone number or email, sends/resends/verifies the OTP through MSG91 widget APIs, and reports through `success` / `failure` callbacks. A headless mode exposes the same operations on `window` for custom UIs.

It does not send messages itself, manage widget settings, or validate tokens server-side.

## 2. Stack and dependencies

| Area | Choice |
| --- | --- |
| Language | Plain JavaScript (ES modules), no framework, no TypeScript |
| Build | Vite 5, library mode, one IIFE (`Msg91OtpProvider`) → `otp-provider.js` |
| State | Own `StateManager` (observer pattern) |
| UI | Template-string HTML + DOM APIs, `CustomEvent`s between UI and controller |
| Styling | `src/styles/main.css` imported as raw text and injected into the Shadow DOM; every class prefixed `msg91-` |
| Phone input | `intl-tel-input` 17 (bundled) + `google-libphonenumber` fallback validation; flag sprites bundled, CDN fallback |
| Captcha | hCaptcha, reCAPTCHA Enterprise, reCAPTCHA v2 (scripts loaded on demand) |
| Env | `dotenv` at build time → Vite `define` |
| Tests / lint | **None.** No test runner, no ESLint/Prettier config |

## 3. Directory map

```
package.json                 root scripts delegate to core-js (start, build:test, build:prod)
README.md                    still Angular CLI boilerplate (stale)
core-js/
├── package.json             vite, dotenv; intl-tel-input, google-libphonenumber, vanilla-hcaptcha
├── vite.config.js           IIFE lib build, env injection, writes dist artifacts
├── index.html               dev demo page (#otp-widget-container)
├── .env.example             OTP_PROVIDER_* variable names
├── README.md, INTEGRATION.md, CHANGELOG.md   SDK docs (partly out of date, §11)
└── src/
    ├── main.js              IIFE entry: initSendOTP, instance map, window methods, demo auto-init
    ├── config/config.js     env cleanup, API base URL, fallback URL, API_ENDPOINTS
    ├── services/OTPService.js   fetch wrapper, tokenAuth header, backup-host fallback
    ├── core/
    │   ├── OTPWidget.js     controller: init, send, verify, retry, one-tap, Truecaller, close, destroy
    │   ├── UIManager.js     screens, validation messages, retry timer, retry channel links
    │   ├── StateManager.js  loading / widgetData / requestId / error / currentScreen
    │   ├── CaptchaManager.js    hCaptcha, reCAPTCHA Enterprise / v2, token retrieval
    │   ├── IntlPhoneManager.js  intl-tel-input setup, geo-IP country, validation
    │   └── secured-by-logo.js   "Secured by MSG91" SVG footer
    ├── truecaller/          session, deeplink, status polling, iframe postMessage bridge
    ├── styles/main.css      widget styles (msg91- prefix)
    ├── captcha-handler.js   standalone background-captcha script (copied to dist as is)
    └── assets/img/          flags.png, flags@2x.png
```

## 4. Runtime architecture

```
host page ── window.initSendOTP(config)                         (main.js)
   validate tokenAuth, widgetId, success() → destroy ALL existing instances
   resolve container: config.containerId | #otp-widget-container | new <msg91-otp-provider>
   new OTPWidget(container, {...envConfig, ...userConfig}).init()
        │
        ├─ attachShadow (default; useShadowDOM:false disables) + inject CSS, Inter font, intl CSS
        ├─ light-DOM <div id="captcha-container" slot="captcha"> so captcha iframes work
        ├─ OTPService(tokenAuth) ── GET widget/getWidgetProcess
        ├─ load TraceIQ script; iframe check (iFrame:0 → hide UI, keep captcha)
        ├─ UI mode: viewport meta, UIManager.render → initial screen
        ├─ Truecaller attempt (Android), CaptchaManager.init, auto-send prefilled identifier
        ▼
   UIManager dispatches CustomEvents → OTPWidget handlers
     sendOtp  → captcha token → POST widget/sendOtp → verification screen + retry timer
     retryOtp → POST widget/retryOtp → restart timer / hide resend / lock on 704
     verifyOtp→ POST widget/verifyOtp → success callback → TraceIQ track → hide
     closeWidget → failure({code:0, closeByUser:true, ...}) → hide
```

Layering: `config` (env + endpoints) → `OTPService` (HTTP) → `OTPWidget` (flow) → `UIManager` (DOM) with `CaptchaManager`, `IntlPhoneManager`, Truecaller runner as collaborators. Cleaner than master, though `OTPWidget` still mixes flow, captcha orchestration, one-tap polling and tracking.

## 5. Public JavaScript contract: `initSendOTP(config)`

| Key | Required | Effect in code |
| --- | --- | --- |
| `widgetId` | yes | Error `Widget Id is missing!` |
| `tokenAuth` | yes | Error `Token is missing!`; sent as `tokenAuth` header on every call and in send/retry/verify bodies |
| `success` | yes | Must be a function, else `success callback function missing !` |
| `failure` | no | Init errors, API errors, close-by-user payload |
| `identifier` | no | Prefill; auto-sends when it matches the widget's `processType` (unless Truecaller runs) |
| `exposeMethods` | no | `true`/`'true'` → headless, no UI |
| `captchaRenderId` | no | Element id for a visible captcha in headless mode; without it hCaptcha runs invisible |
| `hideMethod` | no | `'mobile'` or `'email'` when the widget allows both |
| `containerId` | no | Mount into this element (created if missing) instead of `<msg91-otp-provider>` |
| `disableInput` | no | Disables the phone input |
| `useShadowDOM` | no | `false` renders without Shadow DOM |
| any other key | no | Forwarded as `requestVariables` when `widgetType.value !== '1'` |

Errors in `initSendOTP` are caught, logged and passed to `failure` (not thrown). `window.destroyOTPWidget(instanceId)` destroys an instance. `initSendOTP` returns the instance only when `document.body` already exists; called earlier it returns `undefined`.

## 6. Headless mode (`exposeMethods: true`)

Re-assigned on every init (plain writable globals, so re-init rebinds them correctly):

| Method | Signature | Notes |
| --- | --- | --- |
| `sendOtp` | `(identifier, success?, failure?)` | Identifier sent as given (a leading `+` is kept; UI mode strips it) |
| `verifyOtp` | `(otp, success?, failure?, requestId?)` | Falls back to the last `requestId` |
| `retryOtp` | `(channel, success?, failure?, requestId?)` | `channel` = `'11'` SMS, `'12'` WhatsApp, `'4'` voice, `'3'` email; required only when `widgetType.value === '2'` |
| `getWidgetData` | `()` | Loaded config |
| `isCaptchaVerified` | `()` | `true` once a captcha token exists |
| `oneTapWhatsappVerification` | `(requestId, success?, failure?)` | Polls `isOneTapAuthenticated` every 2 s |

API failures go to both the per-call callback and `config.failure`. Missing inputs call the per-call callback with `{message}` or throw if none was given. `checkVerifiedSession` (present on master) does not exist on this branch.

## 7. Configuration, environment and build

Env variables (read from `process.env`, then `../.env`, then `core-js/.env`; later wins): `OTP_PROVIDER_API_URL`, `OTP_PROVIDER_MSG_MID_PROXY` (default `/api/v5`), `OTP_PROVIDER_HCAPTCHA_ENTERPRISE_SITE_KEY` (or `VITE_HCAPTCHA_SITE_KEY`), `OTP_PROVIDER_RECAPTCHA_ENTERPRISE_SCRIPT_URL` (or `VITE_RECAPTCHA_ENTERPRISE_URL`, default Google's `enterprise.js?render=:siteKey`), `OTP_PROVIDER_RECAPTCHA_ENTERPRISE_SITE_KEY`, `OTP_PROVIDER_RECAPTCHA_V3_SITE_KEY`, `OTP_PROVIDER_BASE_URL` (picked up but unused by `config.js`). Production values: `msg91-environments.md`.

`config.apiUrl` = `OTP_PROVIDER_API_URL` + proxy, defaulting to `https://test.msg91.com/api/v5` when unset. `fallbackApiUrl` = `https://msg91backup.phone91.com/api/v5`.

Build (`vite.config.js`, not run while writing this file): `npm run build:prod` at the root installs and builds `core-js` into `<repo>/dist/otp-widget/` with `otp-provider.js` (IIFE, minified in production, sourcemaps in development), `captcha-handler.js`, `assets/flags*.png` and a minimal `index.html`. The comments name AWS Amplify as the deploy target. `npm start` runs the Vite dev server on port 3000.

## 8. Patterns worth reusing

- **Single IIFE bundle + global init**, internal instance map, `destroy()` that clears timers, polling, Truecaller, captcha and DOM.
- **Shadow DOM by default, with an opt-out**, and CSS shipped inside the bundle (`?raw` import) so the host page is untouched; all classes prefixed (`msg91-`).
- **Captcha through a light-DOM slot**, because captcha iframes do not work inside a shadow root.
- **UI → controller via `CustomEvent`s** (`sendOtp`, `verifyOtp`, `retryOtp`, `closeWidget`, `composed: true`), keeping DOM code out of the flow logic.
- **Re-bindable headless globals** (fixes master's non-configurable `defineProperties`).
- **Server-driven config** (`getWidgetProcess`) for OTP length, channels, retry time/count, captcha, iframe policy.
- **OTP field**: `inputmode="numeric"`, `autocomplete="one-time-code"`, digits-only on input and paste, Verify enabled only at full length.
- **Phone validation** with intl-tel-input plus a libphonenumber fallback that tries several candidate formats (leading 0, dial code typed twice).
- **Resend handling**: cooldown timer, max resends, hide resend on 704 or "already verified", lock verify on 704.
- **Viewport meta injection** so the dialog is not scaled down on phones; removed on close.
- **Backup API host** on network failure or 5xx (reuse only if the user's backend has one).

## 9. Conventions

- Classes per responsibility (`OTPWidget`, `UIManager`, `CaptchaManager`, ...), ES module imports, 4-space indent, single quotes.
- Log prefix `[MSG91 OTP]`; CSS prefix `msg91-`; element `msg91-otp-provider`; viewport meta id `meta-tag-id-msg91-otp-provider`.

## 10. Known defects (do not copy)

| Defect | Where | Consequence |
| --- | --- | --- |
| Demo auto-init ships in the production bundle | `main.js` `initDemo` | Any host page with `#otp-widget-container` (the id `INTEGRATION.md` tells hosts to use) auto-initializes with hard-coded sample credentials |
| Sample `widgetId` / `tokenAuth` committed | `main.js`, `INTEGRATION.md` | Never copy credentials from the reference |
| "Multiple instances" claimed, but every `initSendOTP` destroys all instances first | `main.js` `clearExistingWidgets` | Only one widget can live on a page |
| `initSendOTP` returns `undefined` when called before `<body>` exists | `main.js` `documentReady` | `widget.instanceId` (as documented) fails |
| Verbose `console.log` in production, including shadow-root HTML and request ids; `testSendOTP`/`testVerifyOTP`/`testRetryOTP` globals | `main.js`, `OTPWidget.js` | Noise and data in host consoles; global namespace pollution |
| Dynamic text inserted with `innerHTML` (identifier, some error messages, retry labels) | `UIManager.js` | HTML injection risk; always use `textContent` for dynamic strings |
| Send and Verify buttons stay enabled during requests | `UIManager.js` | Double submits possible |
| Errors removed after 5 s, no `aria-live`/`role="alert"` | `UIManager.showError` | Screen-reader users and slow readers miss errors |
| Countdown uses tick counting (`setInterval` decrement) | `UIManager.startRetryTimer` | Drifts in background tabs; use an end timestamp |
| TraceIQ script always loaded from `pass.hostnsoft.com`; after verify sends phone/email, domain, company id, user agent | `OTPWidget.initTraceIq` / `trackUser` | Personal data to a third party without host opt-in; generated widgets must not do this unless the user explicitly asks |
| Backup-host switch is sticky; network-error branch condition is dead code; non-JSON responses throw | `OTPService.js` | Hard-to-debug failures; guard JSON parsing |
| Non-reserved config keys (`containerId`, `hideMethod`, `captchaRenderId`, `disableInput`, ...) forwarded as `requestVariables` | `getRequestVariables` | Unrelated data sent to the server; use an explicit allow-list |
| `postMessage(..., '*')` and a listener that does not check `event.origin` | `truecaller.utils.js` | Only the deeplink scheme is validated; check origins in new code |
| hCaptcha prefers the env site key over the widget's; reCAPTCHA v2 uses the env var named `..._V3_SITE_KEY` | `CaptchaManager` | Confusing configuration |
| No tests, lint or types | whole branch | Generated projects must add them |

## 11. Docs in the branch that disagree with the code

Trust the code. Specifically:

- `core-js/README.md` lists endpoints `GET /widget/configuration/widgetId`, `POST /otp`, `/otp/verify`, `/otp/retry`; the code calls `widget/getWidgetProcess`, `widget/sendOtp`, `widget/verifyOtp`, `widget/retryOtp`, `widget/isOneTapAuthenticated`.
- README's `VITE_API_URL` is not read anywhere; the API URL comes from `OTP_PROVIDER_API_URL` + `OTP_PROVIDER_MSG_MID_PROXY`.
- README/INTEGRATION show `retryOtp('mobile' | 'email', ...)`; the code expects channel codes (`'11'`, `'12'`, `'4'`, `'3'`).
- INTEGRATION's multiple-instances example does not work (§10).
- The root `README.md` is Angular boilerplate.

## 12. The `master` branch (Angular, legacy)

`master` (commit `7e3c798`) implements the same widget in Angular 19 with NgRx, Angular Material and `@angular/elements`, built with `ngx-build-plus` into the same `otp-provider.js` name. Same MSG91 endpoints and flow, plus `checkVerifiedSession` and TraceIQ `cookie_id` on verify. Its notable defects: the `style` option is never applied, the `exposedMethods` typo breaks `captchaVerified`, the resend default is `NaN`, and the `window` methods are non-configurable, so re-init leaves stale bindings. Use `master` only if the user is customizing that Angular code specifically.
