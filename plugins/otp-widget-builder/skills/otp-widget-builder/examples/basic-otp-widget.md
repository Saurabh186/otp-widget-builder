# Example: Basic embeddable OTP widget (new project)

A condensed walkthrough of one skill run. The code blocks marked **verified** were type-checked with `tsc --strict` and their tests passed with Vitest (11 tests) when this skill was written. Treat them as patterns to adapt, not files to paste blindly: the API contract in every real run comes from the user.

## Contents

- 1. Requirement gathering (one question per turn)
- 2. Summary and plan shown for approval
- 3. Core types (verified)
- 4. State machine (verified)
- 5. Unit tests (verified)
- 6. UI wiring outline
- 7. Verification reported to the user

## 1. Requirement gathering (one question per turn)

| # | Claude asks | User answers |
| --- | --- | --- |
| 1 | New widget or customize an existing one? | New |
| 2 | Where will it run? | Marketing site (WordPress) and a React dashboard |
| 3 | Delivery formats? | Script tag for WordPress; React team will use the same script for now |
| 4 | Identifier and channels? | Phone only; SMS, WhatsApp as resend option |
| 5 | API docs? | Pastes curl examples + JSON for send / verify / resend (see `api-integration-example.md`) |
| 6 | Auth? | Same-site session cookie set by their backend; CORS allows the site origin |
| 7 | OTP length, cooldown, max resends? | 6 digits; server returns `resendAfter`; max 2 resends |
| 8 | Design? | Reference layout, brand colour `#0E7C66`, logo URL, company name |
| 9 | What does the host need on success? | The `token` to post to `/session/otp-login` |

Skipped questions: framework (script embed for unknown hosts → vanilla TS Web Component is the clear fit; stated in the plan instead of asked).

## 2. Summary and plan shown for approval

```
Building: <acme-otp> Web Component + window.AcmeOtp.init(), vanilla TypeScript, Vite library build (IIFE + ES).
Flow: phone (E.164, default country US) → POST /auth/otp/send → 6-digit code → POST /auth/otp/verify
      resend after server resendAfter, max 2, channel SMS or WhatsApp → POST /auth/otp/resend
Security: no secrets in bundle; cookie auth (credentials: 'include'); token validated by host backend.
Design: reference layout, tokens --otpw-color-primary #0E7C66, logo + name in header.
Outputs: dist/acme-otp.iife.js, dist/acme-otp.js, demo index.html (?mock=1), docs/*, tests.
Deps: vite, typescript, vitest, happy-dom, @testing-library/dom, msw, @playwright/test, libphonenumber-js.
Not building: npm publish config, backend, i18n.
Approve?
```

## 3. Core types (verified)

`src/core/types.ts`

```ts
export type Channel = 'sms' | 'whatsapp';

export type ErrorCode =
  | 'INVALID_OTP'
  | 'OTP_EXPIRED'
  | 'TOO_MANY_ATTEMPTS'
  | 'RESEND_TOO_SOON'
  | 'NETWORK_ERROR'
  | 'TIMEOUT'
  | 'UNKNOWN';

export interface WidgetError {
  code: ErrorCode;
  message: string;
  status?: number;
  retryable: boolean;
}

export type ApiResult<T> = { ok: true; data: T } | { ok: false; error: WidgetError };

export interface SendResult {
  requestId: string;
  resendAfterSec?: number;
}

export interface VerifyResult {
  token: string;
}

export interface OtpApi {
  send(identifier: string, channel?: Channel): Promise<ApiResult<SendResult>>;
  resend(requestId: string, channel?: Channel): Promise<ApiResult<SendResult>>;
  verify(requestId: string, code: string): Promise<ApiResult<VerifyResult>>;
}
```

## 4. State machine (verified)

`src/core/otp-machine.ts`: no DOM, no `fetch`, injectable clock. The UI re-renders on `onChange` and runs a 1 s interval only while `secondsUntilResend() > 0`, clearing it on close/destroy.

```ts
import type { Channel, OtpApi, VerifyResult, WidgetError } from './types';

export type Step = 'identifier' | 'sending' | 'code' | 'verifying' | 'verified' | 'closed';

export interface OtpState {
  step: Step;
  identifier: string;
  requestId: string | null;
  resendAvailableAt: number | null; // epoch ms, not a tick counter
  resendsUsed: number;
  resending: boolean;
  locked: boolean;
  error: WidgetError | null;
}

export interface OtpMachineOptions {
  api: OtpApi;
  cooldownSec: number; // used when the API does not return resendAfter
  maxResends: number;
  now?: () => number;
  onChange: (state: OtpState) => void;
  onSuccess: (result: VerifyResult) => void;
  onError?: (error: WidgetError) => void;
}

const initialState: OtpState = {
  step: 'identifier',
  identifier: '',
  requestId: null,
  resendAvailableAt: null,
  resendsUsed: 0,
  resending: false,
  locked: false,
  error: null,
};

export function createOtpMachine(opts: OtpMachineOptions) {
  const now = opts.now ?? Date.now;
  let state = initialState;

  // Read through a function: `state` changes across awaits, so TS narrowing must not apply.
  const step = () => state.step;
  const set = (patch: Partial<OtpState>) => {
    state = { ...state, ...patch };
    opts.onChange(state);
  };
  const fail = (error: WidgetError, patch: Partial<OtpState>) => {
    set({ ...patch, error });
    opts.onError?.(error);
  };
  const cooldownEnd = (sec?: number) => now() + (sec ?? opts.cooldownSec) * 1000;

  async function send(identifier: string, channel?: Channel) {
    if (state.step !== 'identifier') return; // ignore double submit
    set({ step: 'sending', identifier, error: null });
    const res = await opts.api.send(identifier, channel);
    if (step() !== 'sending') return; // closed while in flight
    if (!res.ok) return fail(res.error, { step: 'identifier' });
    set({
      step: 'code',
      requestId: res.data.requestId,
      resendsUsed: 0,
      resendAvailableAt: cooldownEnd(res.data.resendAfterSec),
    });
  }

  function canResend() {
    return (
      state.step === 'code' &&
      !state.resending &&
      !state.locked &&
      state.resendsUsed < opts.maxResends &&
      state.resendAvailableAt !== null &&
      now() >= state.resendAvailableAt
    );
  }

  async function resend(channel?: Channel) {
    if (!canResend() || !state.requestId) return;
    set({ resending: true, error: null });
    const res = await opts.api.resend(state.requestId, channel);
    if (step() !== 'code') return;
    if (!res.ok) return fail(res.error, { resending: false });
    set({
      resending: false,
      resendsUsed: state.resendsUsed + 1,
      requestId: res.data.requestId,
      resendAvailableAt: cooldownEnd(res.data.resendAfterSec),
    });
  }

  async function verify(code: string) {
    if (state.step !== 'code' || state.resending || state.locked || !state.requestId) return;
    set({ step: 'verifying', error: null });
    const res = await opts.api.verify(state.requestId, code);
    if (step() !== 'verifying') return;
    if (!res.ok) return fail(res.error, { step: 'code', locked: res.error.code === 'TOO_MANY_ATTEMPTS' });
    set({ step: 'verified' });
    opts.onSuccess(res.data); // host backend must validate res.data.token
  }

  const secondsUntilResend = () =>
    state.resendAvailableAt === null ? 0 : Math.max(0, Math.ceil((state.resendAvailableAt - now()) / 1000));

  return {
    getState: () => state,
    send,
    resend,
    verify,
    canResend,
    secondsUntilResend,
    changeIdentifier: () => state.step === 'code' && set({ ...initialState, identifier: state.identifier }),
    close: () => set({ step: 'closed' }),
  };
}
```

## 5. Unit tests (verified)

`tests/otp-machine.test.ts`

```ts
import { describe, expect, it, vi } from 'vitest';
import { createOtpMachine } from '../src/core/otp-machine';
import type { OtpApi } from '../src/core/types';

function setup(api: Partial<OtpApi> = {}) {
  let t = 1_000_000;
  const fullApi: OtpApi = {
    send: vi.fn().mockResolvedValue({ ok: true, data: { requestId: 'r1' } }),
    resend: vi.fn().mockResolvedValue({ ok: true, data: { requestId: 'r2' } }),
    verify: vi.fn().mockResolvedValue({ ok: true, data: { token: 'tok' } }),
    ...api,
  };
  const onSuccess = vi.fn();
  const onError = vi.fn();
  const m = createOtpMachine({
    api: fullApi, cooldownSec: 30, maxResends: 2, now: () => t,
    onChange: () => {}, onSuccess, onError,
  });
  return { m, api: fullApi, onSuccess, onError, advance: (ms: number) => (t += ms) };
}

describe('otp machine', () => {
  it('sends, starts cooldown, verifies', async () => {
    const { m, api, onSuccess } = setup();
    await m.send('+14155550123');
    expect(m.getState()).toMatchObject({ step: 'code', requestId: 'r1' });
    expect(m.secondsUntilResend()).toBe(30);
    await m.verify('123456');
    expect(api.verify).toHaveBeenCalledWith('r1', '123456');
    expect(onSuccess).toHaveBeenCalledWith({ token: 'tok' });
    expect(m.getState().step).toBe('verified');
  });

  it('ignores double submit', async () => {
    const { m, api } = setup();
    await Promise.all([m.send('a'), m.send('a')]);
    expect(api.send).toHaveBeenCalledTimes(1);
  });

  it('send error returns to identifier step', async () => {
    const error = { code: 'NETWORK_ERROR' as const, message: 'x', retryable: true };
    const { m, onError } = setup({ send: vi.fn().mockResolvedValue({ ok: false, error }) });
    await m.send('a');
    expect(m.getState()).toMatchObject({ step: 'identifier', error });
    expect(onError).toHaveBeenCalledWith(error);
  });

  it('enforces cooldown and max resends', async () => {
    const { m, api, advance } = setup();
    await m.send('a');
    await m.resend();
    expect(api.resend).not.toHaveBeenCalled();
    advance(30_000);
    await m.resend();
    expect(m.getState()).toMatchObject({ resendsUsed: 1, requestId: 'r2' });
    advance(30_000);
    await m.resend();
    advance(30_000);
    expect(m.canResend()).toBe(false);
  });

  it('locks after too many attempts', async () => {
    const error = { code: 'TOO_MANY_ATTEMPTS' as const, message: 'locked', retryable: false };
    const { m } = setup({ verify: vi.fn().mockResolvedValue({ ok: false, error }) });
    await m.send('a');
    await m.verify('000000');
    expect(m.getState()).toMatchObject({ step: 'code', locked: true });
  });

  it('drops results that arrive after close', async () => {
    let resolve!: (v: unknown) => void;
    const { m } = setup({ send: vi.fn(() => new Promise((r) => (resolve = r))) as OtpApi['send'] });
    const p = m.send('a');
    m.close();
    resolve({ ok: true, data: { requestId: 'late' } });
    await p;
    expect(m.getState()).toMatchObject({ step: 'closed', requestId: null });
  });
});
```

## 6. UI wiring outline

Not compiled here; shows the responsibilities only.

```ts
class AcmeOtpElement extends HTMLElement {
  #machine?: ReturnType<typeof createOtpMachine>;
  #abort = new AbortController();
  #tick?: number;

  connectedCallback() {
    const root = this.attachShadow({ mode: 'open' });
    // inject styles (tokens from config.theme), render header with logo/companyName
    this.#machine = createOtpMachine({
      api: createOtpApi({ baseUrl: config.apiBaseUrl, signal: this.#abort.signal }),
      cooldownSec: 30, maxResends: 2,
      onChange: (s) => { this.#render(root, s); this.#syncTicker(); },
      onSuccess: config.onSuccess, onError: config.onError,
    });
  }
  disconnectedCallback() { this.#abort.abort(); clearInterval(this.#tick); this.#machine?.close(); }
  #syncTicker() { /* start setInterval(render, 1000) while secondsUntilResend() > 0, clear at 0 */ }
  #render(root: ShadowRoot, s: OtpState) { /* textContent only for dynamic strings; aria-live for errors */ }
}
```

## 7. Verification reported to the user

```
Lint ✅  Types ✅  Unit/integration ✅ 38 passed (Mocked)  E2E ✅ 6 passed (Mocked, Chromium)
Build ✅ dist/acme-otp.iife.js 11.8 kB gzip   Real API ❌ not run: no staging credentials
```

(The numbers above illustrate the format; report the real output of your run.)
