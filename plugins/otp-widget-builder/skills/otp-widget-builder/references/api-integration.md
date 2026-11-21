# API Integration

## Contents

1. Ground rules
2. API intake checklist (ask the user)
3. Security model
4. Client API layer pattern
5. Reference: MSG91 widget API as used by the reference client
6. Handling incomplete information

## 1. Ground rules

- Use the **user's** APIs. Do not build a backend unless the user explicitly asks for one.
- **Never invent** endpoints, methods, headers, field names, response shapes, status codes or auth formats. If something is unknown, ask, or leave a clearly marked `TODO(confirm-with-user)` and list it in the final report.
- Ask for **real request/response examples** (success and each error), not only prose descriptions.
- Keep all endpoint paths, base URLs and field mappings in one config/adapter module so a contract change touches one file.

## 2. API intake checklist (ask the user)

Collect one item at a time, skipping anything already provided (pasted docs, OpenAPI file, Postman collection, curl commands).

| # | Item | Why |
| --- | --- | --- |
| 1 | Base URL(s) per environment | Env config |
| 2 | Send OTP: method, path | Core call |
| 3 | Verify OTP: method, path | Core call |
| 4 | Resend OTP: separate endpoint, or re-call send? Channel switch supported? | Resend logic |
| 5 | Optional config endpoint (OTP length, channels, timers from server?) | Static vs dynamic config |
| 6 | Auth: none, public client key, session cookie, bearer from host app? Who issues it? | Security model |
| 7 | Required headers (content type, custom headers, CSRF) | Request builder |
| 8 | Request bodies for each call (exact field names, identifier format e.g. E.164 with/without `+`) | Payload mapping |
| 9 | Success responses (where is the request id? the token? expiry?) | Response mapping |
| 10 | Error responses: shape + codes for invalid OTP, expired OTP, too many attempts, resend too soon, rate limited, invalid identifier | Error mapping |
| 11 | Delivery channels and how a channel is chosen (field + values) | Channel UI |
| 12 | Limits: resend cooldown, max resends, max verify attempts, OTP expiry | Timer/limit UI |
| 13 | CORS: is the widget origin allowed? Cookies (`credentials: 'include'`)? | Avoid runtime failures |
| 14 | Captcha / bot protection required by the backend? Which provider, where does the token go? | Send flow |
| 15 | What the host needs after verification (token, user id) and how the host backend validates it | Success contract |
| 16 | Sandbox/test credentials or a test environment for integration testing | Testing |

## 3. Security model

Explain these to the user when relevant, and enforce them in code:

1. **No secrets in the browser.** Anything shipped in a JS bundle, `.env` consumed by a frontend build (`VITE_*`, `NG_APP_*`, `NEXT_PUBLIC_*`, webpack `DefinePlugin`) is public. Only put *public* identifiers there (public widget id, public site keys, base URLs).
2. **Private API keys** (e.g. an SMS provider auth key) must stay on the user's server. The widget calls the user's backend; the backend calls the provider. If the user only has a private key and no backend, stop and explain; offer a minimal proxy only if they ask for one.
3. **Verification is server-side.** The browser never compares codes and never decides "verified" on its own. A client-side "success" is only a signal; the host's backend must validate the returned token/session before granting access.
4. **Do not store OTPs** in `localStorage`, `sessionStorage`, cookies, URLs or logs. Do not log identifiers or tokens in production builds.
5. **Transport**: HTTPS only; no `http://` third-party calls (the reference's IP lookup is an anti-example).
6. **Rate limiting and lockouts** are server responsibilities; the client mirrors them in the UI.
7. `.env.example` lists variable **names** with placeholder values. Real `.env` files are git-ignored. Never copy credentials from the reference repo (it has sample ones committed).
8. Embeds: if `postMessage` is used, validate `event.origin` and send to a specific target origin, not `'*'`.
9. Phone/email are personal data: send them only to the configured API; no analytics or third-party lookups without the user's approval.

MSG91 note: in the reference model `tokenAuth` is a widget-scoped token issued for client-side use and is sent from the browser. Treat any other provider's keys as private unless the user's provider documents them as public.

## 4. Client API layer pattern

Keep three pieces separate:

```
config (endpoints, base URL, field names)  →  api client (fetch, timeout, abort, normalize)  →  flow/state (no HTTP details)
```

Minimum behaviour of the API client:

- `fetch` with `AbortController` and a timeout (default 15 s unless the user specifies).
- JSON parse guarded (non-JSON error pages must not crash the widget).
- Map HTTP status + body into a normalized result:

```ts
type ApiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: string; message: string; status?: number; retryable: boolean } };
```

- Error codes: map known backend codes to stable widget codes (`INVALID_OTP`, `OTP_EXPIRED`, `TOO_MANY_ATTEMPTS`, `RESEND_TOO_SOON`, `NETWORK_ERROR`, `TIMEOUT`, `UNKNOWN`). Only map codes the user confirmed; everything else becomes `UNKNOWN` with the server message.
- Response adapters extract the request id, token and expiry from the confirmed fields only. The reference's `extractWidgetProcessPayload` (accepts `data.widgetId` or `widget_id`) and `errorResolver` (any error shape → `string[]`) are good models.
- No global mutable headers object (a defect in the reference). Build headers per request.

## 5. Reference: MSG91 widget API as used by the reference client

Use this **only** when the user is integrating MSG91's OTP widget backend. It is derived from client code, not from an official specification (none exists in the repo). Confirm against MSG91's current documentation before relying on it.

Base: `{apiUrl}{msgMidProxy}` (historically `https://api.msg91.com/api/v5` for production). All calls send `Accept` and `Content-Type: application/json`, `withCredentials: false`.

| Purpose | Method + path | Request | Response fields used |
| --- | --- | --- | --- |
| Widget config | `GET widget/getWidgetProcess?widgetId=` | header `tokenAuth` | `{status|type, data:{...IWidgetResponse}}` or flat; `widgetId`/`widget_id` |
| Send OTP | `POST widget/sendOtp` | `{tokenAuth, widgetId, identifier, token?, requestVariables?}` | `type`, `message` (= reqId), `invisibleVerified`, `access-token` |
| Resend | `POST widget/retryOtp` | `{tokenAuth, widgetId, reqId, retryChannel, requestVariables?}` | `type`, `message` |
| Verify | `POST widget/verifyOtp` | `{tokenAuth, widgetId, otp, reqId, cookie_id?}` | `type`, `message` (token on success), `code` (703/704 on error) |
| One-tap status | `POST widget/isOneTapAuthenticated` | `{requestId, tokenAuth}` + header | `type`, `message`, `verified` |
| Truecaller session | `GET widget/getTruecallerSession?widgetId=&identifier=&isMobileSdk=1` | header `tokenAuth` | `type`, `message`, `redirection_url`, `deeplink` (may be nested in `data`) |
| Truecaller status | `GET widget/truecallerStatus/{requestId}` | header `tokenAuth` | `status` (`pending`, `processing`, `verified`, `failed`, ...), token |
| Verified session | `POST widget/checkVerifiedSession` | `{widgetId, cookie_id, mobile, email}` | `type`, `verified`, `message` / `access-token` |

`IWidgetResponse` fields the client reads: `otpLength`, `processType.value`, `verificationType`, `widgetType.value`, `retryCount`, `retryTime`, `captchaValidations`, `widgetMeta.captcha_type`, `widgetMeta.captcha_site_key`, `processes[].channel{value,name}`, `processes[].processVia.value`, `iFrame`, `oneTapVerification`, `truecaller`, `defaultCountry.iso2`.

Server-side token validation: MSG91 documents a server-side access-token verification API for the integrator's backend. It is not in this repo; get the exact endpoint and auth from MSG91's docs or the user. Never call it from the browser with a private auth key.

## 6. Handling incomplete information

- Missing response examples → ask for them; if the user cannot provide them, write the adapter against the fields they describe, mark assumptions with `TODO(confirm-with-user)`, and list them in the final report as unverified.
- No test environment → run mocked tests only and label them "mocked"; state that real-API integration is untested.
- Contract conflicts (docs vs example) → show both to the user and ask which is right.
