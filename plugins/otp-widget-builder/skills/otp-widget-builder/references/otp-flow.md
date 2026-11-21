# OTP Lifecycle

Part A describes the flow exactly as the reference repository implements it (commit `7e3c798`). Part B is the generic, framework-neutral state machine to implement in new widgets. Part C lists the behaviour decisions you must confirm with the user.

## Contents

- A1. Initialization
- A2. Step 1: identifier entry and send
- A3. Step 2: code entry, resend, verify
- A4. Timer and limits
- A5. Alternate paths (invisible, magic link, one-tap, Truecaller, verified session)
- A6. Callback payloads
- B. Generic state machine for new widgets
- C. Decisions to confirm with the user

---

## A1. Initialization

1. `initSendOTP(config)` validates `widgetId`, `tokenAuth`, `success`; mounts `<msg91-otp-provider>`.
2. Store reset, then `GET widget/getWidgetProcess?widgetId=…` with `tokenAuth` header.
3. A full-panel loader ("Loading…") shows while config loads. On failure, `errors` holds the message (special text when an extension blocks XHR: "An external script is blocking the API call required for widget execution.").
4. From the config the widget derives:
   - mode: `processType.value` `'1'` both / `'2'` mobile / `'3'` email (`hideMethod` can force one when both).
   - `otpLength` → input min/max length and placeholder dots.
   - `verificationType` `'1'` OTP / `'2'` Magic Link (changes labels and hides the code input).
   - `retryTime` → resend cooldown seconds; `retryCount` → max resends (default 2).
   - retry channels: `processes` whose `processVia.value === '5'`, de-duplicated by `channel.value`.
   - captcha: `captchaValidations` + `widgetMeta.captcha_type` (`2` = reCAPTCHA Enterprise, otherwise hCaptcha).
   - `iFrame === 0` and running inside an iframe → UI hidden.
   - `truecaller === 1` → attempt Truecaller on Android before showing the form.
5. TraceIQ script is loaded; its token endpoint sets `window.traceiq_cookie_id`.

## A2. Step 1: identifier entry and send

- **Mobile**: intl-tel-input field; Enter key sends only when valid; error "Please enter valid mobile number" after blur. Identifier sent = E.164 without `+` (e.g. `919876543210`). A live preview line shows `+<dial code><number>`.
- **Email**: whitespace stripped while typing; errors "Email field is required" / "Enter valid email" once touched.
- Button label: "Send OTP" (mobile) / "Generate OTP" (email); with magic link: "Send Magic Link" / "Generate Magic Link".
- **Captcha before send**: hCaptcha token from the visible widget, or reCAPTCHA Enterprise `execute(siteKey, {action: 'sendOtp'})`. Token goes into the request as `token`. Invalid site key → "Something went wrong! Please contact website support."
- Inputs are disabled while the request is in flight and re-enabled on success or error.
- Request: `POST widget/sendOtp` `{ tokenAuth, widgetId, identifier, token?, requestVariables? }`.
- Success (`type !== 'error'`): `reqId = response.message`; move to step 2; focus the OTP input; start the cooldown timer; clear the captcha token.
- Error: messages shown under the form; `failure` callback receives the error response.

## A3. Step 2: code entry, resend, verify

- Header: "Enter OTP Sent to" + the identifier.
- OTP input: digits only, `maxlength`/`minlength` = `otpLength`, paste accepted only if all digits (truncated to length). Typing clears the previous error.
- **Verify** button enabled only when the value is digits, length equals `otpLength`, no resend is in flight, and the last error code is not `704`.
- Request: `POST widget/verifyOtp` `{ tokenAuth, widgetId, otp, reqId, cookie_id? }`.
- Success (`type === 'success'`): `success(response)` then close.
- Failure: the response message is shown below the input (`invalidOtpError`), `failure` is called.
- Error codes used by the UI: `703` invalid OTP, `704` verification limit reached (hides resend, disables verify).
- **Resend** (UI mode, only while `retryCount < widget.retryCount ?? 2` and code ≠ 704, after cooldown):
  - `widgetType '1'`: one link. Email → "Resend OTP"; mobile → "Get Via Call". Both send `retryChannel: null` (server picks the channel).
  - `widgetType '2'`: "Resend OTP via SMS OR CALL OR WHATSAPP…" built from retry channels (email channel only for email mode; `VOICE` shown as `CALL`). Sends `retryChannel: <channel.value>`.
  - Request: `POST widget/retryOtp` `{ tokenAuth, widgetId, reqId, retryChannel, requestVariables? }`.
  - Success restarts the cooldown and increments `resendCount`. Retry is not captcha-gated.
- `RetryChannelTypes`: SMS `'11'`, Voice `'4'`, Email `'3'`, WhatsApp `'12'`.

## A4. Timer and limits

- Cooldown: RxJS `timer(1000, 1000)`; `timeRemain = timerSec - tick`; label "Retry in N Seconds"; stops at 0, then the resend link appears.
- Started after a successful send and after each successful resend.
- Max resends enforced client-side only in UI mode (`retryCount` counter). Headless mode leaves limits to the server.
- **OTP expiry is not handled client-side.** `expiryTime` exists in the type but is never read; the server rejects expired codes and the UI shows the server message.

## A5. Alternate paths

| Path | Trigger | Behaviour |
| --- | --- | --- |
| Invisible verification | `sendOtp` response has `invisibleVerified: true` and `access-token` | Treated as verified immediately; `message` replaced by the access token, `referenceId` = original message |
| Magic link | `verificationType === '2'` | No code input; closing after a send reports `magicLinkSent: true`, message "Magic Link Sent." |
| WhatsApp one-tap | widget `oneTapVerification` and a `reqId` exists | `POST widget/isOneTapAuthenticated` every 2 s while `message === 'Verification Pending'`; JWT (`eyJ…`) or "Already Verified" → success; "Verification Expired" → failure |
| Truecaller | `truecaller === 1`, Android (or mobile in desktop-site mode), not iOS | `GET widget/getTruecallerSession` → open `truecallersdk://` deeplink (top window) → poll `GET widget/truecallerStatus/{requestId}` every 1 s for 10 s (+ on tab visibility) → JWT → success `{token, message, mobile}`; failure/timeout → fall back to the OTP form |
| Verified session | host calls `window.checkVerifiedSession()` | `POST widget/checkVerifiedSession` with `{widgetId, cookie_id, mobile:true, email:false}`, then email; resolves on success/verified/JWT, rejects otherwise |

## A6. Callback payloads

- `success`: the verify response object (`{ type: 'success', message, request_id?, ... }`); for invisible/one-tap paths `message` is the access token (JWT); Truecaller: `{ token, message, mobile }`.
- `failure`: raw API error responses, HTTP error objects, or on user close `{ code: 0, closeByUser: true, magicLinkSent, message: 'User cancelled the verification process.' | 'Magic Link Sent.' }`.

The access token returned to the browser is **not** proof of verification by itself. The integrator's backend must validate it server-side before trusting the user. The reference repo contains no server-side step.

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
