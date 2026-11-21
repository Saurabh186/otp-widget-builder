# Example: Customizing an existing widget (branding + behaviour)

Two short scenarios showing the "customize existing" workflow: inspect first, change the minimum, keep public APIs stable, re-verify.

## Scenario A: Rebrand a fork of the reference repo (Angular)

User: "We forked OTP-widget-web. Make it match our brand: green primary `#0E7C66`, our logo in the header, 'Verify your phone' title, remove the MSG91 footer."

### Inspection findings Claude reported before planning

- Angular 19, NgModule, styles in `src/otp-global.scss`, `src/app/otp/send-otp/send-otp.component.scss`, Material theme in `src/app_theme-colors.scss` (azure palette from `#1157A6`).
- Titles are hard-coded in `send-otp-center.component.html` (`Mobile Verification` / `Email Verification`, `Enter OTP Sent to`).
- Footer SVG "Secured by MSG91" is inline in `send-otp.component.html`.
- `config.style` is accepted but never applied, so it cannot be used for theming without a fix.
- OTP length, channels, retry time/count come from the MSG91 dashboard (`getWidgetProcess`), not code.
- No tests or lint config exist; baseline `npm run build:prod` result recorded before changes.

### Plan presented (approved by the user)

| Change | File(s) | Note |
| --- | --- | --- |
| Primary colour | `app_theme-colors.scss` (regenerate palette with `ng generate @angular/material:theme-color` from `#0E7C66`) | Keeps Material buttons/progress consistent |
| Text/link colours | `otp-global.scss` (`.resend-otp a`, titles) | Contrast checked ≥4.5:1 |
| Logo + title | `send-otp-center.component.html` header; new optional `config.logoUrl`, `config.title` read in the component | Defaults keep current behaviour for other integrators |
| Footer removal | `send-otp.component.html` | User owns the fork and asked for it |
| Optional: make `config.style` work | bind `[ngStyle]="css$ \| async"` on `.otp-verification-dialog` | Asked user first: it changes behaviour for anyone already passing `style` |
| Docs | README section "Branding options" | Only the new options |

Not changed: store, effects, API service, Truecaller, captcha.

### Verification reported

`npm run build:prod` ✅ (same warnings as baseline), manual check in `ng serve` at 375 px and 1280 px with screenshots, unit tests: none exist; Claude added a component test only for the new `logoUrl`/`title` inputs after the user agreed to add a test runner config.

## Scenario B: Existing React widget (generated earlier) → new brand + 4-digit OTP

User: "Use our new design (screenshot attached), switch to 4-digit codes, and change the button to 'Continue'."

### Steps

1. Read the project: `src/components/OtpInput.tsx` takes `length` from config; theme tokens in `src/styles/otp-widget.module.css`; texts in `src/config.ts` defaults.
2. Ask one question: "Does your verify API already accept 4-digit codes, or does the backend need a change too?" User: "Backend already sends 4 digits for this tenant." (The widget never decides OTP length on its own.)
3. Extract tokens from the screenshot and show them for approval:

   | Token | Value |
   | --- | --- |
   | `--otpw-color-primary` | `#5B3DF5` |
   | `--otpw-radius` | `16px` |
   | `--otpw-radius-input` | `12px` |
   | `--otpw-font-family` | `"Plus Jakarta Sans", system-ui, sans-serif` |
   | Input style | segmented, 56×56 px boxes, 12 px gap |

   Ambiguities raised: no error state in the screenshot (proposed red border + message under boxes), no disabled button style (proposed 40 % opacity).
4. Implement:
   - `config.ts`: default `otpLength: 4`, `texts.verifyButton: 'Continue'` (host can still override).
   - CSS tokens updated; segmented input variant already existed, so only `inputStyle: 'segmented'` default changes.
   - No new dependencies; font loaded by the host page (documented).
5. Re-run `npm run lint`, `npm run typecheck`, `npm test`, `npm run build`, Playwright at 360/768/1280. Updated snapshot tests only where the visual change is intended; listed them in the report.
6. Update `docs/CUSTOMIZATION.md` (token values) and `docs/CONFIGURATION.md` (new defaults). Changelog note: default `otpLength` changed from 6 to 4 (breaking for hosts relying on the default).

### Things Claude did not do

- Did not touch the API adapter or state machine (no requirement).
- Did not rename public props or events.
- Did not add a design-system dependency to reproduce the screenshot.
