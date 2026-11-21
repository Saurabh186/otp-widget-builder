# OTP Lifecycle

Part A describes the flow exactly as the reference implements it on the `production-js` branch (`core-js/`, commit `724c584`); differences from the Angular `master` branch are noted where they matter. Part B is the generic, framework-neutral state machine to implement in new widgets. Part C lists the behaviour decisions you must confirm with the user.

## Contents

- A1. Initialization
- A2. Step 1: identifier entry and send
- A3. Step 2: code entry, resend, verify
- A4. Timer and limits
- A5. Alternate paths (invisible, magic link, one-tap, Truecaller)
- A6. Callback payloads
- B. Generic state machine for new widgets
- C. Decisions to confirm with the user

---

## A1. Initialization

1. `initSendOTP(config)` validates `tokenAuth`, `widgetId`, `success`, destroys existing instances, resolves the container and creates `OTPWidget`.
2. Shadow DOM attached (unless `useShadowDOM: false`), styles injected, a light-DOM captcha slot added.
3. `GET widget/getWidgetProcess?widgetId=…` with `tokenAuth` header. On failure: `failure(error)` and, in UI mode, "Failed to load widget. Please try again."
4. From the config (`response.data || response`) the widget derives:
   - mode: `processType.value` `'1'` both / `'2'` mobile / `'3'` email; `hideMethod` hides one when both are allowed.
   - `otpLength` (UI falls back to 4 if missing) → input `maxlength` and placeholder dots.
   - `verificationType` `'1'` OTP / `'2'` Magic Link (labels change, no code input).
   - `retryTime` (default 25 s) → resend cooldown; `retryCount` (default 2) → max resends.
   - retry channels: `processes` with `processVia.value === '5'`, filtered to email channels in email mode and non-email in mobile mode, de-duplicated by channel value.
   - captcha: `captchaValidations` + `widgetMeta.captcha_type` `1` hCaptcha / `2` reCAPTCHA Enterprise / `3` reCAPTCHA v2.
   - `iFrame === 0` inside an iframe → UI hidden, captcha still initialized.
   - `truecaller === 1` → Truecaller attempted on Android before the form is used.
5. TraceIQ script loaded; viewport meta added (UI mode); initial screen rendered; captcha initialized; prefilled `identifier` auto-sent when it matches `processType`.

## A2. Step 1: identifier entry and send

- Title "Mobile Verification" / "Email Verification"; radio buttons when both are allowed.
- **Mobile**: intl-tel-input with geo-IP default country (`db-ip` → `ipinfo.io`, fallback `widgetData.defaultCountry.iso2` or `in`); only digits and `+` accepted; error "Please enter valid mobile number" on blur or submit; live preview `+<dial code><number>`. UI mode sends the number without `+`; headless keeps what the host passed.
- **Email**: whitespace stripped while typing; "Email field is required" / "Enter valid email" on submit.
- Button: "Send OTP" (mobile) / "Generate OTP" (email); "Magic Link" instead of "OTP" when `verificationType === '2'`. Enter submits.
- **Captcha before send**: reCAPTCHA Enterprise `execute(siteKey, {action:'sendOtp'})`; hCaptcha/v2 must be solved first (visible), or hCaptcha runs invisible in headless mode without `captchaRenderId`. No token yet → send is silently skipped. Site-key failure → "Something went wrong! Please contact website support."
- Request: `POST widget/sendOtp` `{tokenAuth, widgetId, identifier, requestVariables?, token?}`.
- Success (`type === 'success'`): request id = `response.message` (or `request_id` / `requestId` / `reqId`); verification screen; cooldown starts; one-tap polling starts if enabled.
- Error: message shown at the top of the card (removed after 5 s); captcha reset; inputs re-enabled; `failure` called.

## A3. Step 2: code entry, resend, verify

- Title "Enter OTP Sent to" (or "Magic Link Sent to") + the identifier (with `+` for phones).
- Code input: `type="text"`, `inputmode="numeric"`, `autocomplete="one-time-code"`, digits only, `maxlength = otpLength`, paste stripped to digits and truncated. Verify disabled until the length is complete; Enter submits.
- Request: `POST widget/verifyOtp` `{tokenAuth, widgetId, otp, reqId}` (no TraceIQ `cookie_id` on this branch).
- Success: `type === 'success'` or `message === 'OTP verified successfully'` → per-call success, `config.success`, TraceIQ `track(...)`, widget hidden.
- Failure: message under the code input; code `704` → verify locked and resend hidden.
- **Resend** (after cooldown, while resends used < `retryCount`):
  - `widgetType '1'`: "Didn't receive OTP?" + one link: mobile "Get Via Call", email "Resend OTP"; sends `retryChannel: null`.
  - `widgetType '2'`: "Resend OTP via SMS OR WHATSAPP OR CALL…" from retry channels; sends the channel code.
  - Request: `POST widget/retryOtp` `{tokenAuth, widgetId, reqId, retryChannel, requestVariables?}`.
  - Success restarts the cooldown, or hides resend once the limit is reached. Response or error with code `704` → lock verify; message matching "already verified" → hide resend.
- Channel codes: SMS `'11'`, WhatsApp `'12'`, Voice `'4'`, Email `'3'`.

## A4. Timer and limits

- Cooldown: `setInterval` every 1 s decrementing from `retryTime`; label "Retry in N Seconds"; at 0 the resend options appear.
- Restarted after each successful resend; cleared on hide, close and destroy.
- Resend limits are counted in the UI (`retryClickCount`) and in the controller (`retryAttempts`) in UI mode; headless mode leaves limits to the server.
- **OTP expiry is not handled client-side** on either branch; the server rejects expired codes and the UI shows its message.

## A5. Alternate paths

| Path | Trigger | Behaviour |
| --- | --- | --- |
| Invisible verification | send response has `invisibleVerified` and `access-token` | Success immediately with `message` = access token, `referenceId` = original message |
| Magic link | `verificationType === '2'` | No code input; closing after a send reports `magicLinkSent: true`, "Magic Link Sent." |
| WhatsApp one-tap | widget `oneTapVerification` after a send, or `oneTapWhatsappVerification()` | `POST widget/isOneTapAuthenticated` every 2 s while `message === 'Verification Pending'`; JWT (`eyJ…`) or "Already Verified" → success; "Verification Expired" → failure |
| Truecaller | `truecaller === 1`, Android (or mobile in desktop-site mode), not iOS | `GET widget/getTruecallerSession` → open `truecallersdk://` at top level (iframe: `postMessage` to the parent page, whose listener is installed by the bundle) → poll `GET widget/truecallerStatus/{requestId}` every 1 s for 10 s (+ on tab visibility) → JWT → success `{token, message, mobile}`; failure/timeout → OTP form |
| Verified session | master only | `checkVerifiedSession` is not on `production-js` |

## A6. Callback payloads

- `success`: the verify response object; invisible path: response with `message` = access token; one-tap UI path: the JWT string itself; Truecaller: `{token, message, mobile}`.
- `failure`: API error body (`apiResponse`), HTTP errors as `{status, message, data, error}`, init errors, or on close `{code: 0, closeByUser: true, magicLinkSent, message: 'User cancelled the verification process.' | 'Magic Link Sent.'}`.

The token returned to the browser is **not** proof of verification by itself. The integrator's backend must validate it server-side before trusting the user. The reference contains no server-side step.

---

## B. Generic state machine for new widgets

```
idle ──init──► loadingConfig ──ok──► enterIdentifier ──submit(valid)──► sending
                    │ fail                    ▲                           │ ok        │ fail
                    ▼                         │ change identifier         ▼           ▼
                configError                   └───────────────── enterCode ◄── (show error, back to enterIdentifier)
                                                                   │  │
                                            resend (cooldown=0,    │  │ submit(len ok)
                                            count<max) ► resending ┘  ▼
                                                                    verifying ──ok──► verified (emit success, stop timers)
                                                                       │ fail
                                                                       ▼
                                                     enterCode + error  (limit code → locked)
any state ──close──► closed (emit failure {code:'CLOSED_BY_USER'} or the agreed shape)
```

Skip `loadingConfig` when configuration is static (provided by the host instead of an API).

Rules every implementation must follow:

1. **One source of truth** for state (store, reducer, signal, or a plain state object); UI renders from it.
2. **Carry the request id** from send to resend/verify; never derive it client-side.
3. **Guard double submits**: ignore submit while `sending`/`resending`/`verifying`.
4. **Cooldown timer** is driven by an end timestamp (`Date.now() + seconds*1000`), not tick counting, so background tabs stay accurate. Clear it on verify, close and unmount.
5. **Resend limits**: enforce the server's answer; the client counter only hides the button.
6. **Expiry**: show a countdown only if the API gives an expiry; treat it as display only. Server decides.
7. **Errors**: normalize every failure (HTTP status, API error body, network error, timeout) into `{ code, message, retryable }` before it reaches the UI.
8. **Cleanup**: abort in-flight requests (`AbortController`) and stop polling on close/unmount.
9. **Verification result** goes to the host via callback/event; the host's backend validates it.

## C. Decisions to confirm with the user

- OTP length and whether it is fixed or comes from the API.
- Identifier types (phone, email, both) and default country for phone.
- Resend cooldown seconds, max resends, whether resend can switch channel.
- Whether expiry is displayed, and where its value comes from.
- Auto-submit when the last digit is entered (default in the reference: no).
- Behaviour on max attempts (lock, restart, contact support).
- What the success payload must contain for the host (token, identifier, request id).
- Close behaviour: allowed? What does the host receive?
