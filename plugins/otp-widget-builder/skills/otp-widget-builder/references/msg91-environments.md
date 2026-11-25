# MSG91 OTP environments (built-in defaults)

When the widget talks to MSG91's OTP widget API, use the values below. Production values are taken from MSG91's own published script (`https://verify.msg91.com/otp-provider.js`); test values come from the test build's env (confirmed by the skill owner). They are all **public** (API hosts and captcha *site* keys; no auth keys), so they may ship in frontend code.

**Do not ask the user for these values or for a `.env` file.** Ask only for the **widget id** and **tokenAuth** (the widget-scoped token from the MSG91 dashboard, meant for browser use; never the account's private authkey), then check which environment the widget belongs to (section 4).

## Contents

- 1. Values
- 2. Env var names in the reference (`production-js`)
- 3. How to use them in a generated widget
- 4. Widgets are per environment: check first

## 1. Values

| Purpose | Production (default) | Test |
| --- | --- | --- |
| API base (`widget/getWidgetProcess`, `widget/sendOtp`, …) | `https://control.msg91.com/api/v5` | `https://test.msg91.com/api/v5` |
| Fallback API (network error / 5xx, retried once) | `https://msg91backup.phone91.com/api/v5` | none |
| hCaptcha site key (`captcha_type` 1) | `3866024a-1680-4e12-b4e0-ff3625c661d0` | `05b5722e-9b2d-4c53-82bc-5b2609dddd0c` |
| reCAPTCHA Enterprise site key (`captcha_type` 2) | `6LemcdopAAAAACUyxk9h_mI_O-9fNxsWdrmA_N62` | not set in the test build |
| reCAPTCHA Enterprise script | `https://www.google.com/recaptcha/enterprise.js?render=:siteKey` | same |
| reCAPTCHA site key (`captcha_type` 3; the reference reads it from the var named `…_V3_SITE_KEY`) | `6LclMHEnAAAAAO3evidQfC39Uz7_tLFpKni0pDuT` | `6Lf5J3EnAAAAAHhdnO-nhjN9U4mZxRtObpmVZLZ1` (a v3 key: rendering it as a v2 checkbox fails with "ERROR for site owner") |

**Captcha keys are per environment.** A token made with the production key is rejected by `test.msg91.com` with `{"message":"Invalid Captcha Token.","type":"error"}`, and vice versa. Always pick the key for the widget's environment.

If the widget config returns `widgetMeta.captcha_site_key`, prefer it over the keys above (the reference does the opposite for hCaptcha; that is a defect, see `repository-architecture.md` §10).

## 2. Env var names in the reference (`production-js`)

| Var | Production value |
| --- | --- |
| `OTP_PROVIDER_API_URL` | `https://control.msg91.com` |
| `OTP_PROVIDER_MSG_MID_PROXY` | `/api/v5` |
| `OTP_PROVIDER_HCAPTCHA_ENTERPRISE_SITE_KEY` | `3866024a-1680-4e12-b4e0-ff3625c661d0` |
| `OTP_PROVIDER_RECAPTCHA_ENTERPRISE_SCRIPT_URL` | `https://www.google.com/recaptcha/enterprise.js?render=:siteKey` |
| `OTP_PROVIDER_RECAPTCHA_ENTERPRISE_SITE_KEY` | `6LemcdopAAAAACUyxk9h_mI_O-9fNxsWdrmA_N62` |
| `OTP_PROVIDER_RECAPTCHA_V3_SITE_KEY` | `6LclMHEnAAAAAO3evidQfC39Uz7_tLFpKni0pDuT` |

For test: `OTP_PROVIDER_API_URL=https://test.msg91.com`, `OTP_PROVIDER_BASE_URL=https://test.msg91.com`, `OTP_PROVIDER_MSG_MID_PROXY=/api/v5`, `OTP_PROVIDER_HCAPTCHA_ENTERPRISE_SITE_KEY=05b5722e-9b2d-4c53-82bc-5b2609dddd0c`, `OTP_PROVIDER_RECAPTCHA_V3_SITE_KEY=6Lf5J3EnAAAAAHhdnO-nhjN9U4mZxRtObpmVZLZ1`. When customizing the reference, write these into its `.env` (git-ignored) instead of asking the user.

## 3. How to use them in a generated widget

Put both environments in the config layer as **defaults**; env vars or init options only override them, so the widget works with no `.env`:

```ts
// config/msg91.ts
export const MSG91_ENVIRONMENTS = {
  production: {
    apiUrl: 'https://control.msg91.com/api/v5',
    fallbackApiUrl: 'https://msg91backup.phone91.com/api/v5',
    hcaptchaSiteKey: '3866024a-1680-4e12-b4e0-ff3625c661d0',
    recaptchaEnterpriseSiteKey: '6LemcdopAAAAACUyxk9h_mI_O-9fNxsWdrmA_N62',
    recaptchaSiteKey: '6LclMHEnAAAAAO3evidQfC39Uz7_tLFpKni0pDuT',
  },
  test: {
    apiUrl: 'https://test.msg91.com/api/v5',
    fallbackApiUrl: null,
    hcaptchaSiteKey: '05b5722e-9b2d-4c53-82bc-5b2609dddd0c',
    recaptchaEnterpriseSiteKey: null,
    recaptchaSiteKey: '6Lf5J3EnAAAAAHhdnO-nhjN9U4mZxRtObpmVZLZ1',
  },
} as const;

export const RECAPTCHA_ENTERPRISE_SCRIPT_URL = 'https://www.google.com/recaptcha/enterprise.js?render=:siteKey';
```

- Expose `environment: 'production' | 'test'` as an init option (default `production`) and accept `apiUrl` as an explicit override.
- `.env.example` lists the same variables pre-filled with the production values, commented as optional overrides.
- Only call the fallback host on network errors or HTTP 5xx, never on 4xx.

## 4. Widgets are per environment: check first

A widget exists in only one environment. A test widget returns `{"message":"AuthenticationFailure","type":"error","code":"401"}` from production, and vice versa.

Check with `GET {apiUrl}/widget/getWidgetProcess?widgetId=<widgetId>` and headers `Accept: application/json`, `tokenAuth: <tokenAuth>`:

- `data` with `widgetType`, `processType`, `otpLength`, `processes`, …: the widget belongs to that environment. Use this response as the real example for the config adapter.
- `type: "error"`, code `401`: try the other environment, then ask the user.

This only reads the widget's settings; it sends no OTP.
