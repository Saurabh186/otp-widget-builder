# Reference Repository Architecture

Verified against `Walkover-Web-Solution/OTP-widget-web` at commit `7e3c798` (master, November 2026). Every statement below comes from reading the source. If the user says the repo has changed, re-inspect it before relying on this file.

## Contents

1. What the repo is
2. Stack and dependencies
3. Directory map
4. Runtime architecture
5. Public JavaScript contract (`initSendOTP`)
6. Headless mode (`exposeMethods`)
7. Environment and build
8. Shared libraries worth knowing
9. Conventions
10. Known defects (do not copy)
11. What to reuse in a new widget

## 1. What the repo is

The browser client for MSG91's embeddable OTP widget. A host page loads one script bundle (`otp-provider.js`) and calls `window.initSendOTP({...})`. The widget fetches its configuration from MSG91 (`widget/getWidgetProcess`), collects a phone number or email, sends and verifies an OTP through MSG91 widget APIs, and reports the result through `success` / `failure` callbacks.

It does **not** send messages itself, manage widget settings, or verify tokens on a server. Those live in MSG91's backend.

## 2. Stack and dependencies

| Area | Choice |
| --- | --- |
| Framework | Angular 19.2 (NgModule-based, `standalone: false` components) |
| Embedding | `@angular/elements` custom element `msg91-otp-provider`, `@webcomponents/custom-elements` polyfill |
| State | NgRx Store + Effects 19 (single `otp` feature slice) |
| UI kit | Angular Material 19 (button, form-field, input, radio, progress bar/spinner, icon, dialog, tooltip) |
| Phone input | `intl-tel-input` 17 (script loaded at runtime from MSG91's domain) + `google-libphonenumber` |
| Captcha | `ng-hcaptcha` / `vanilla-hcaptcha` (hCaptcha), Google reCAPTCHA Enterprise loaded by script tag |
| Utilities | `lodash-es`, `dayjs`, `rxjs` 7.8, `ngx-cookie-service` |
| Build | `ngx-build-plus` (single bundle) + custom webpack (`DefinePlugin` for env), TypeScript 5.5 |
| Tests / lint | **None.** No `*.spec.ts`, no Karma config, no root ESLint/Prettier config. `ng test` is listed in `package.json` but has nothing to run. |

`ng-otp-input` and `ng-recaptcha` are installed but not used for rendering (`ReCaptchaV3Service` is imported only so `grecaptcha` typings resolve).

## 3. Directory map (OTP-relevant parts)

```
src/
├── main.ts / main.element.ts      Bootstraps ElementModule (identical files)
├── app/
│   ├── element.module.ts          Defines <msg91-otp-provider>, installs window.initSendOTP
│   ├── app.module.ts, app.component.*   Dev-only shell; auto-calls initSendOTP outside production
│   ├── otp/
│   │   ├── otp.module.ts          Material imports, NgRx store/effects, DI tokens for base URL
│   │   ├── send-otp/              Host component: Shadow DOM, overlay, loader, footer, captcha/TraceIQ script loading
│   │   ├── component/send-otp-center/   All flow logic + form UI (≈1270 lines)
│   │   ├── service/otp.service.ts       HTTP calls
│   │   ├── service/urls/otp-urls.ts     Endpoint path builders (the client's API contract)
│   │   ├── model/otp.ts                 Request/response/config types, RetryChannelTypes enum
│   │   ├── store/                       actions, effects, reducer, selectors
│   │   └── truecaller/                  Truecaller session, deeplink, polling runner
│   └── libs/                       Vendored @msg91/* shared libs (path aliases in tsconfig.json)
│       ├── services/http-wrapper   HttpClient wrapper (adds proxy headers for other MSG91 apps)
│       ├── models/root-models      BaseResponse, errorResolver, DI token enum
│       ├── utils                   IntlPhoneLib, inIframe, iframeDetection, many unrelated helpers
│       ├── regex                   EMAIL_REGEX, ONLY_INTEGER_REGEX, others
│       ├── directives/remove-character-directive   Keystroke filter for inputs
│       └── service/lib/utils/admin/*   Email/RCS/voice/WhatsApp URL builders: unrelated carry-over
├── assets/js/captcha-handler.js   Standalone script: background captcha execution with backoff
├── environments/                  env-variables.ts reads process.env.OTP_PROVIDER_*
├── styles.scss, otp-global.scss, app_theme-colors.scss   Global + widget styles, Material theme
tools/postbuild.js                 Renames main.js → otp-provider.js, copies captcha-handler.js
docs/VISION.md, OPEN_QUESTIONS.md  Product notes (no API spec exists in the repo)
```

## 4. Runtime architecture

```
host page ── window.initSendOTP(config)
                │  validates widgetId, tokenAuth, success()
                ▼
<msg91-otp-provider>  (SendOtpComponent, ViewEncapsulation.ShadowDom)
   │  dispatch resetAll, getWidgetData  ──► OtpEffects ──► OtpService.getWidgetData ──► GET widget/getWidgetProcess
   │  loads Inter font + viewport meta (UI mode only)
   │  loads reCAPTCHA Enterprise script if widget needs it; loads TraceIQ script
   ▼
<msg91-send-otp-center>  (SendOtpCenterComponent)
   step 1: choose mobile/email → validate → captcha → dispatch getOtpAction ─► POST widget/sendOtp
   step 2: OTP input, resend countdown, retry channels → POST widget/retryOtp
           verify → POST widget/verifyOtp → success callback → close
   side paths: Truecaller (Android), WhatsApp one-tap polling, magic link, invisible verification
```

Layering in the reference:

- **UI**: Angular templates + Material, all styles inside Shadow DOM.
- **Flow logic**: in the center component (too much in one class; see defects).
- **State**: NgRx reducer holds `widgetData`, in-progress flags, `otpGenerateData.reqId`, `resendCount`, `errors`, `apiErrorResponse`.
- **API**: `OtpService` + `otp-urls.ts`; effects translate responses into actions and invoke per-call callbacks.
- **Config**: build-time env (`OTP_PROVIDER_*`) + runtime config from `getWidgetProcess` + host config passed to `initSendOTP`.

## 5. Public JavaScript contract: `initSendOTP(config)`

Defined in `element.module.ts`. Runs after DOM ready, removes any existing `<msg91-otp-provider>`, creates a new one and appends it to `<body>`.

| Key | Required | Effect in code |
| --- | --- | --- |
| `widgetId` | yes | Throws `Widget Id is missing!` if absent |
| `tokenAuth` | yes | Throws `Token is missing!` if absent; sent as `tokenAuth` header on config/Truecaller/one-tap calls and in the body of send/retry/verify |
| `success` | yes | Must be a function, else throws `success callback function missing !` |
| `failure` | no | Called with API error responses and with the close-by-user object |
| `identifier` | no | Prefills phone/email; a leading `+` is stripped; if valid, OTP is sent automatically (unless the Truecaller flow will run) |
| `disableInput` | no | Disables the phone input |
| `exposeMethods` | no | `true`/`'true'` → headless mode, no UI rendered |
| `captchaRenderId` | no | Headless mode: element id where the hCaptcha widget is rendered |
| `hideMethod` | no | `'mobile'` or `'email'`: hides that option when the widget allows both |
| `style` | no | Stored but **never applied** (see defects) |
| any other key | no | For widgets with `widgetType.value === '2'`, the whole config minus reserved keys is sent as `requestVariables` (template variables) |

## 6. Headless mode (`exposeMethods: true`)

The center component defines these on `window` with `Object.defineProperties` (non-writable, non-configurable), only if `sendOtp`/`retryOtp`/`verifyOtp`/`oneTapWhatsappVerification` are not already present:

| Method | Signature | Notes |
| --- | --- | --- |
| `sendOtp` | `(identifier, success?, error?)` | Errors via callback if identifier missing; captcha handled internally (invisible hCaptcha or reCAPTCHA Enterprise) unless `captchaRenderId` is set |
| `retryOtp` | `(channel, success?, error?, reqId?)` | `channel` only used for `widgetType '2'`; required there |
| `verifyOtp` | `(otp, success?, error?, reqId?)` | Sends `cookie_id` from `window.traceiq_cookie_id` when set |
| `getWidgetData` | `()` | Returns the loaded widget config |
| `isCaptchaVerified` | `()` | `true` once an hCaptcha token was captured |
| `oneTapWhatsappVerification` | `(requestId, success?, error?)` | Polls `isOneTapAuthenticated` every 2 s |
| `checkVerifiedSession` | `(widgetId?, cookieId?) → Promise` | Always exposed (also in UI mode); tries mobile then email |

## 7. Environment and build

Env vars read at build time through `process.env` (webpack `DefinePlugin`, `.env` loaded by `dotenv`):

`OTP_PROVIDER_API_URL`, `OTP_PROVIDER_BASE_URL`, `OTP_PROVIDER_MSG_MID_PROXY`, `OTP_PROVIDER_RECAPTCHA_V3_SITE_KEY`, `OTP_PROVIDER_RECAPTCHA_ENTERPRISE_SCRIPT_URL` (contains `:siteKey`), `OTP_PROVIDER_RECAPTCHA_ENTERPRISE_SITE_KEY`, `OTP_PROVIDER_HCAPTCHA_ENTERPRISE_SITE_KEY`, `OTP_PROVIDER_TRACE_IQ_SCRIPT_URL` (contains `:timestamp`), `OTP_PROVIDER_TRACE_IQ_API_URL`.

API base URL = `apiUrl + msgMidProxy` (commented history shows `/api/v5`).

Scripts: `start` (`ng serve`), `build:test` / `build:master` (development config), `build:prod` (production config). Each build runs `tools/postbuild.js`, which copies `main.js` to `otp-provider.js`, copies `captcha-handler.js` to the output root and deletes `assets/`. `singleBundle: true` makes one JS file; global styles are injected.

Deployment target and CDN URL are not in the repo (OPEN_QUESTIONS OQ-002).

## 8. Shared libraries worth knowing

- `createUrl(base, path)` → `${base}/${path}`.
- `errorResolver(x)` → always `string[]` (string → `[x]`, array → as is, object → flattened values, otherwise `['Something went wrong.']`). A good pattern for normalizing error shapes.
- `IntlPhoneLib`: wraps intl-tel-input; `isRequiredValidNumber` (libphonenumber validation), `phoneNumber` (E.164 with `+`), `selectedCountryData`. Initial country uses `widgetData.defaultCountry.iso2`, falling back to `'in'`, after IP lookups.
- `RemoveCharacterDirective`: blocks keys not matching `allowedRegex`, strips whitespace on input. The OTP field uses `allowedRegex="[0-9]"`.
- `inIframe()` (`window.self !== window.top`, `true` on cross-origin error) and `iframeDetection()`.

## 9. Conventions

- Component selectors prefixed `msg91-`; NgRx action types `[OTP] ...`; `*InProcess` / `*Success` boolean pairs per request.
- `BaseComponent` supplies `destroy$` for `takeUntil` cleanup.
- 4-space indentation in source (despite `.editorconfig` saying 2), single quotes, SCSS.
- `strict: false`, `strictTemplates: false`, heavy `any` usage.

## 10. Known defects (observed in code: do not copy)

| Defect | Where | Consequence |
| --- | --- | --- |
| `config.style` stored in `css$` but never bound in the template | `send-otp.component.ts` | Host styling via config has no effect |
| `config.exposedMethods` typo (should be `exposeMethods`) | `onCaptchaVerified` / `onError` | `config.captchaVerified` callback never fires |
| `+this.widgetData?.retryTime ?? 25` | center component | `+undefined` is `NaN`, so the 25 s default never applies |
| `window` methods defined non-configurable once | `exposeMethodsToWindow` | Calling `initSendOTP` again leaves methods bound to the first (destroyed) component |
| `OtpService.options` is one shared mutable object | `otp.service.ts` | Headers leak between calls; `tokenAuth` header depends on call order |
| Inconsistent HTTP error extraction (`err.error.message` vs `err.errors`) | `otp.effects.ts` | Resend/verify network errors collapse to "Something went wrong." |
| Whole host config forwarded as `requestVariables` | `makeAdditionalConfig` | Unrelated keys (`style`, `hideMethod`, ...) go to the server |
| `geoIpLookup` calls `http://ip-api.com` (plain HTTP) | `libs/constant` | Mixed content on HTTPS hosts; third-party IP lookup without consent |
| Sample `widgetId` / `tokenAuth` committed | `app.component.ts` | Never copy credentials from the reference into generated projects |
| `installTruecallerParentMessageListener` never called | `truecaller.utils.ts` | Parent-page deeplink bridge is not active |
| `expiryTime` in the model is never read | `model/otp.ts` | No client-side OTP expiry; expiry is server-enforced only |
| One 1,270-line component holds UI, flow, captcha, polling | center component | Hard to test; split responsibilities in new projects |
| `libs/service/lib/utils/admin/*`, `curl-to-json`, etc. | `libs/` | Unrelated code bundled into the repo |

## 11. What to reuse in a new widget

Reuse the **ideas**, re-implement cleanly in the chosen stack:

- Two-step flow (identifier → code) with a server-driven config step.
- Per-request in-progress/success/error state and a `reqId` carried from send to retry/verify.
- Resend cooldown + max-retry count + server "limit reached" code handling.
- Channel-specific retry (SMS, voice, email, WhatsApp) when the backend supports it.
- Embeddable custom element + global init function + Shadow DOM isolation.
- Headless API for teams that bring their own UI.
- Paste handling and digits-only filtering for the OTP field; E.164 phone validation.
- Error normalization into a string list.
- Callback contract: `success(result)`, `failure(error)`, explicit "closed by user" payload.

Do not reuse: NgRx (unless the host app already uses it), the vendored `@msg91/*` libs, Angular Material inside an embeddable bundle (size), the defects above.
