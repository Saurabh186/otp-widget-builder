# Example: Mapping a user's API contract into the widget

Shows how a pasted API description becomes an endpoint adapter, error mapping, tests and docs. The contract below is the **example user's** backend, used for illustration only; never reuse it for a real project.

## Contents

- 1. What the user provided
- 2. HTTP client (verified)
- 3. Endpoint adapter (verified)
- 4. Client tests (verified)
- 5. Docs produced
- 6. Variant: MSG91 widget backend

## 1. What the user provided

```
POST /auth/otp/send     { "phone": "+14155550123", "channel": "sms" }
  200 { "requestId": "r_9f2", "resendAfter": 30 }
POST /auth/otp/resend   { "requestId": "r_9f2", "channel": "whatsapp" }
  200 { "requestId": "r_9f3", "resendAfter": 30 }
  429 { "error": { "code": "RESEND_TOO_SOON", "message": "Try again in 12s" } }
POST /auth/otp/verify   { "requestId": "r_9f2", "code": "123456" }
  200 { "verified": true, "token": "<opaque>" }
  400 { "error": { "code": "OTP_INVALID", "message": "Incorrect code" } }
  410 { "error": { "code": "OTP_EXPIRED", "message": "Code expired" } }
  429 { "error": { "code": "TOO_MANY_ATTEMPTS", "message": "Too many attempts" } }
Auth: session cookie (SameSite=Lax) set by the user's backend. CORS allows https://www.example.com with credentials.
After verify: the site POSTs the token to /session/otp-login (server validates it).
```

Questions Claude still asked before coding: identifier format (`+` included? yes), timeout preference (default 15 s accepted), whether `resend` returns a new `requestId` (yes, use the new one), and whether `verified: false` with 200 can happen (user: "should not, treat as invalid").

## 2. HTTP client (verified)

`src/api/client.ts`: timeout, external abort (widget destroy), non-JSON safety, normalized errors.

```ts
import type { ApiResult, WidgetError } from '../core/types';

export interface PostOptions {
  timeoutMs: number;
  credentials?: RequestCredentials;
  headers?: Record<string, string>;
  signal?: AbortSignal; // aborted by widget.destroy()
  mapError: (status: number, body: unknown) => WidgetError;
}

export async function postJson<T>(url: string, body: unknown, opts: PostOptions): Promise<ApiResult<T>> {
  const controller = new AbortController();
  let timedOut = false;
  const timer = setTimeout(() => {
    timedOut = true;
    controller.abort();
  }, opts.timeoutMs);
  const onExternalAbort = () => controller.abort();
  opts.signal?.addEventListener('abort', onExternalAbort);

  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Accept: 'application/json', ...opts.headers },
      body: JSON.stringify(body),
      credentials: opts.credentials ?? 'same-origin',
      signal: controller.signal,
    });
    const data: unknown = await res.json().catch(() => null); // non-JSON error pages
    return res.ok ? { ok: true, data: data as T } : { ok: false, error: opts.mapError(res.status, data) };
  } catch {
    return {
      ok: false,
      error: timedOut
        ? { code: 'TIMEOUT', message: 'The request timed out. Please try again.', retryable: true }
        : { code: 'NETWORK_ERROR', message: 'Network error. Check your connection and try again.', retryable: true },
    };
  } finally {
    clearTimeout(timer);
    opts.signal?.removeEventListener('abort', onExternalAbort);
  }
}
```

## 3. Endpoint adapter (verified)

`src/api/endpoints.ts`: the only file that knows paths and field names.

```ts
import type { ApiResult, Channel, ErrorCode, OtpApi, SendResult, VerifyResult, WidgetError } from '../core/types';
import { postJson } from './client';

// Contract confirmed by the user (see docs/API.md). Change paths/fields here only.
interface SendResponse { requestId: string; resendAfter?: number }
interface VerifyResponse { verified: boolean; token: string }
interface ErrorBody { error?: { code?: string; message?: string } }

const ERROR_CODES: Record<string, ErrorCode> = {
  OTP_INVALID: 'INVALID_OTP',
  OTP_EXPIRED: 'OTP_EXPIRED',
  TOO_MANY_ATTEMPTS: 'TOO_MANY_ATTEMPTS',
  RESEND_TOO_SOON: 'RESEND_TOO_SOON',
};

function mapError(status: number, body: unknown): WidgetError {
  const err = (body as ErrorBody | null)?.error;
  const code = (err?.code && ERROR_CODES[err.code]) || 'UNKNOWN';
  return {
    code,
    status,
    message: err?.message ?? 'Something went wrong. Please try again.',
    retryable: code !== 'TOO_MANY_ATTEMPTS',
  };
}

const toSendResult = (r: ApiResult<SendResponse>): ApiResult<SendResult> =>
  r.ok ? { ok: true, data: { requestId: r.data.requestId, resendAfterSec: r.data.resendAfter } } : r;

export function createOtpApi(config: { baseUrl: string; timeoutMs?: number; signal?: AbortSignal }): OtpApi {
  const opts = { timeoutMs: config.timeoutMs ?? 15000, credentials: 'include' as const, signal: config.signal, mapError };
  const url = (path: string) => `${config.baseUrl.replace(/\/$/, '')}${path}`;

  return {
    async send(phone: string, channel: Channel = 'sms') {
      return toSendResult(await postJson<SendResponse>(url('/auth/otp/send'), { phone, channel }, opts));
    },
    async resend(requestId: string, channel: Channel = 'sms') {
      return toSendResult(await postJson<SendResponse>(url('/auth/otp/resend'), { requestId, channel }, opts));
    },
    async verify(requestId: string, code: string): Promise<ApiResult<VerifyResult>> {
      const r = await postJson<VerifyResponse>(url('/auth/otp/verify'), { requestId, code }, opts);
      if (!r.ok) return r;
      return r.data.verified
        ? { ok: true, data: { token: r.data.token } }
        : { ok: false, error: { code: 'INVALID_OTP', message: 'Incorrect code.', retryable: true } };
    },
  };
}
```

## 4. Client tests (verified)

```ts
import { afterEach, describe, expect, it, vi } from 'vitest';
import { createOtpApi } from '../src/api/endpoints';

afterEach(() => { vi.unstubAllGlobals(); vi.useRealTimers(); });
const json = (status: number, body: unknown) =>
  new Response(JSON.stringify(body), { status, headers: { 'Content-Type': 'application/json' } });

describe('api client', () => {
  it('maps send success', async () => {
    const fetchMock = vi.fn().mockResolvedValue(json(200, { requestId: 'r1', resendAfter: 30 }));
    vi.stubGlobal('fetch', fetchMock);
    const api = createOtpApi({ baseUrl: 'https://api.example.com/' });
    expect(await api.send('+14155550123')).toEqual({ ok: true, data: { requestId: 'r1', resendAfterSec: 30 } });
    const [url, init] = fetchMock.mock.calls[0];
    expect(url).toBe('https://api.example.com/auth/otp/send');
    expect(JSON.parse(init.body)).toEqual({ phone: '+14155550123', channel: 'sms' });
    expect(init.credentials).toBe('include');
  });

  it('maps known error codes', async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue(json(410, { error: { code: 'OTP_EXPIRED', message: 'Code expired' } })));
    const r = await createOtpApi({ baseUrl: 'x' }).verify('r1', '123456');
    expect(r).toEqual({ ok: false, error: { code: 'OTP_EXPIRED', status: 410, message: 'Code expired', retryable: true } });
  });

  it('handles non-JSON 500', async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue(new Response('<html>oops</html>', { status: 500 })));
    const r = await createOtpApi({ baseUrl: 'x' }).send('a');
    expect(r.ok === false && r.error.code).toBe('UNKNOWN');
  });

  it('handles network errors', async () => {
    vi.stubGlobal('fetch', vi.fn().mockRejectedValue(new TypeError('Failed to fetch')));
    const r = await createOtpApi({ baseUrl: 'x' }).send('a');
    expect(r.ok === false && r.error.code).toBe('NETWORK_ERROR');
  });

  it('times out', async () => {
    vi.useFakeTimers();
    vi.stubGlobal('fetch', vi.fn((_u: string, init: RequestInit) => new Promise((_r, reject) => {
      init.signal!.addEventListener('abort', () => reject(new DOMException('Aborted', 'AbortError')));
    })));
    const p = createOtpApi({ baseUrl: 'x', timeoutMs: 1000 }).send('a');
    await vi.advanceTimersByTimeAsync(1000);
    const r = await p;
    expect(r.ok === false && r.error.code).toBe('TIMEOUT');
  });
});
```

Component and E2E tests use MSW handlers that return the same JSON bodies the user pasted (one handler per scenario).

## 5. Docs produced

- `docs/API.md` from `templates/api-documentation.md`, with the table of error codes above and the "Server-side validation" section pointing at `/session/otp-login`.
- `.env.example`:

```bash
# Public: base URL of the OTP API (no secrets belong in this file)
VITE_OTP_API_BASE_URL=https://api.example.com
```

## 6. Variant: MSG91 widget backend

If the user is integrating MSG91's widget APIs instead, the same adapter shape maps to the endpoints listed in `references/api-integration.md` §5:

| Adapter method | MSG91 call (as used by the reference client) | Mapping |
| --- | --- | --- |
| `send` | `POST widget/sendOtp {tokenAuth, widgetId, identifier}` | `requestId ← message`; `type === 'error'` → error |
| `resend` | `POST widget/retryOtp {tokenAuth, widgetId, reqId, retryChannel}` | channel values `'11'` SMS, `'4'` voice, `'3'` email, `'12'` WhatsApp |
| `verify` | `POST widget/verifyOtp {tokenAuth, widgetId, otp, reqId}` | `token ← message` on `type === 'success'`; `code 703` → `INVALID_OTP`, `704` → `TOO_MANY_ATTEMPTS` |

MSG91 returns errors in the body (`type: 'error'`), so the adapter must check the body even when HTTP status is 200. Confirm the current contract with MSG91's documentation and the user's widget settings before shipping.
