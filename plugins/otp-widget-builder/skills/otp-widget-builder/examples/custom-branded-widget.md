# Example: Customizing an existing widget (branding + behaviour)

Two short scenarios showing the "customize existing" workflow: inspect first, change the minimum, keep public APIs stable, re-verify.

## Scenario A: Rebrand a fork of the reference (`production-js`, vanilla JS)

User: "We forked OTP-widget-web (`production-js` branch). Make it match our brand: green primary `#0E7C66`, our logo in the header, 'Verify your phone' title, remove the MSG91 footer."

### Inspection findings Claude reported before planning

- Vanilla JS SDK in `core-js/`, Vite IIFE build to `dist/otp-widget/otp-provider.js`; styles in `core-js/src/styles/main.css` (classes prefixed `msg91-`) injected into a Shadow DOM.
- Primary colour `#1e5ead` (hover `#18509a`) is repeated across `main.css`; there are no CSS variables.
- Titles and labels are template strings in `core-js/src/core/UIManager.js` (`Mobile Verification`, `Email Verification`, `Enter OTP Sent to`, `Send` / `Generate`).
- The footer comes from `core-js/src/core/secured-by-logo.js`, rendered by `UIManager.footerHtml()`.
- Hosts cannot restyle it from outside: Shadow DOM blocks page CSS, and there is no `style`/`theme` option.
- OTP length, channels, retry time/count come from the MSG91 dashboard (`getWidgetProcess`), not code.
- `main.js` auto-initializes a demo with committed sample credentials whenever `#otp-widget-container` exists. Flagged to the user as a production risk to fix in the same change.
- No tests or lint. Claude did not run the branch's build: installing a downloaded repo's npm packages needs the user's go-ahead in this environment, so the baseline build was left for the user to run.

### Plan presented (approved by the user)

| Change | File(s) | Note |
| --- | --- | --- |
| Theme tokens | `main.css`: introduce `--msg91-primary`, `--msg91-primary-hover` on `.msg91-otp-overlay`, replace hard-coded `#1e5ead` / `#18509a` | One place to change; contrast of `#0E7C66` on white checked ≥4.5:1 |
| Brand config | `OTPWidget.js` / `UIManager.js`: optional `config.theme` (token map applied to the overlay element), `config.logoUrl`, `config.title` | Defaults keep the current look for other integrators |
| Logo + title | `UIManager.showInitialScreen()` header; logo rendered with `img.src`/`alt` set through the DOM, title via `textContent` | Avoids the branch's `innerHTML` pattern for dynamic text |
| Footer removal | `UIManager.renderShell()` (omit `footerHtml()`) | User owns the fork and asked for it |
| Demo auto-init | `main.js`: run `initDemo` only in dev (`process.env.NODE_ENV !== 'production'`), remove committed credentials | Asked first: it changes production behaviour |
| `requestVariables` | add `theme`, `logoUrl`, `title` to the skip list in `getRequestVariables()` | Otherwise the new options would be sent to MSG91 |
| Docs | `core-js/INTEGRATION.md` branding section | Only the new options |

Not changed: `OTPService`, captcha, Truecaller, flow logic.

### Verification reported

Claude added Vitest + happy-dom (user approved) with tests for theme application, logo/title rendering and the demo guard. Results reported as run. `npm run build:prod` and a manual check at 375 px and 1280 px were listed for the user to run, with the exact commands, because the build was not executed in this session.

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
