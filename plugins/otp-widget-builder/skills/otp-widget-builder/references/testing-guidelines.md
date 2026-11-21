# Testing Guidelines

The reference repository has **no tests**. Every generated or customized widget must ship with tests, and the final report must state exactly what was run.

## Contents

1. Honesty rules
2. Tooling per stack
3. Required scenario matrix
4. Mocking strategy
5. Responsive and integration checks
6. Verification commands
7. Test report format

## 1. Honesty rules

- Never say tests passed unless you ran them in this session and saw the output.
- Label every result **Mocked** (fake API) or **Real API** (user's backend). Without credentials/backend access, only mocked tests exist; say so.
- If the environment cannot install packages or run a browser (e.g. no network, no Playwright browsers), say which checks were skipped and give the exact commands for the user to run.
- Report failures with their output; do not delete or skip failing tests to get green.

## 2. Tooling per stack

| Stack | Unit / integration | E2E |
| --- | --- | --- |
| Vanilla TS / Web Component (Vite) | Vitest + `happy-dom` or `jsdom`, `@testing-library/dom` | Playwright |
| React | Vitest + `@testing-library/react` + `@testing-library/user-event` | Playwright |
| Vue 3 | Vitest + `@vue/test-utils` or `@testing-library/vue` | Playwright |
| Angular 17+ | The project's configured runner (Karma/Jasmine default, or Jest/Vitest if set up) + `TestBed`, `HttpTestingController` | Playwright |
| Existing project | Its existing runner; do not add a second one | Its existing E2E tool, else Playwright if E2E is required |

API mocking: MSW (`msw`) for component/integration tests and for the demo's mock mode; or inject a fake API client into the core.

Timers: fake timers (`vi.useFakeTimers()`, `jasmine.clock()`, `fakeAsync/tick`) and an injectable clock.

## 3. Required scenario matrix

Cover each row at the lowest level that proves it (core unit tests for logic, component tests for UI, E2E for the integrated flow):

| # | Scenario | Assert |
| --- | --- | --- |
| 1 | Send OTP success | Correct request (URL, method, headers, body); request id stored; moves to code step; cooldown starts; focus on code input |
| 2 | Send OTP API error | Error mapped and shown; stays on identifier step; inputs re-enabled; `onError` called |
| 3 | Verify success | Correct request with request id; success state; `onSuccess` payload matches contract; timers stopped |
| 4 | Invalid OTP | Mapped message shown on the input; can retry; verify re-enabled after edit |
| 5 | Expired OTP | Expired message; resend offered (if allowed) |
| 6 | Too many attempts / locked | Verify disabled; resend hidden if server says so; message shown |
| 7 | Resend success | Correct request (channel if any); cooldown restarts; counter increments |
| 8 | Resend restrictions | Hidden/disabled during cooldown; hidden after max resends; "resend too soon" server error handled |
| 9 | Timer behaviour | Countdown text updates; reaches 0 → resend available; accurate after simulated tab sleep (clock jump); cleared on close/unmount |
| 10 | Network error | `fetch` rejects → `NETWORK_ERROR` message, retry possible, no crash |
| 11 | Timeout | Request aborted after timeout → `TIMEOUT` message |
| 12 | Non-JSON / 5xx response | Generic error, no crash |
| 13 | Identifier validation | Invalid phone/email blocked client-side with message; valid formats pass; identifier formatted as the API expects |
| 14 | OTP input validation | Digits only, max length, paste full code, paste with non-digits rejected/sanitized, verify disabled until complete |
| 15 | Loading states | Buttons disabled + spinner while in flight; double submit ignored (one request only) |
| 16 | Success / error messages | Custom `texts` overrides render |
| 17 | Config validation | Missing required config throws a clear error |
| 18 | Theming | Theme tokens applied to the root element |
| 19 | Accessibility basics | Labels present; error linked via `aria-describedby`; focus management on step change |
| 20 | Integration | Embed: `<script>` build loads, `init()` mounts, `destroy()` cleans up. npm: package imports and types resolve. In-app: component renders in the host app's test setup |

Add rows for optional features in scope: channel switch, magic link, captcha token forwarding, auto-submit, close-by-user payload.

## 4. Mocking strategy

- Build mock handlers from the **user's confirmed examples** (copy their JSON). Do not invent extra fields.
- One handler set per scenario (success, invalid, expired, locked, resend-too-soon, 500, network failure, delay for timeout).
- The demo page may offer a `?mock=1` mode using the same handlers so stakeholders can click through every state without a backend. Mock code must be excluded from production bundles (dynamic import behind a dev flag, or a separate demo entry).

## 5. Responsive and integration checks

- Playwright projects or viewport loops at 360×740, 768×1024, 1280×800: no horizontal scroll, primary button visible without scrolling in the code step, touch targets ≥44 px.
- Screenshot each state at mobile + desktop when visual review matters; attach paths in the report.
- Embed check: a plain HTML page loading the built IIFE from `dist/` (served over HTTP, not `file://`).

## 6. Verification commands

Run whichever exist (adapt to the package manager in use):

```bash
npm run lint
npm run typecheck   # or: npx tsc --noEmit
npm test            # unit + integration
npm run build
npx playwright test # E2E, if configured
```

Then serve the demo (`npm run preview` or `npx vite preview`) and exercise the flow at least once in mock mode.

## 7. Test report format

Write `docs/TESTING.md` in the generated project and summarize it in the final message:

```markdown
## Test results: <date>

Environment: Node <version>, <OS>, <browser(s)>

| Check | Command | Result | Mode |
| --- | --- | --- | --- |
| Lint | npm run lint | ✅ pass | n/a |
| Types | npm run typecheck | ✅ pass | n/a |
| Unit/integration | npm test | ✅ 34 passed | Mocked |
| E2E | npx playwright test | ⚠️ not run: browsers not installed | Mocked |
| Build | npm run build | ✅ pass (embed 14.2 kB gzip) | n/a |
| Real API | manual | ❌ not run: no test credentials | Real API |

Scenario coverage: rows 1–20 of the matrix → test file names.
Known gaps: …
```
