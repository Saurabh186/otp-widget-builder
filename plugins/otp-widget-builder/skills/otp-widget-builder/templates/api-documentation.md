# Template: docs/API.md for a generated widget

Fill every `<…>` from the user's confirmed contract. Delete sections that do not apply (e.g. no separate resend endpoint). Do not add fields that the user did not confirm; mark unconfirmed items `⚠️ Unconfirmed`.

~~~markdown
# API Integration

This widget calls the following backend endpoints. It does not contain any server code or private credentials.

## Environments

| Environment | Base URL | Env var |
| --- | --- | --- |
| Production | <https://api.example.com> | `<VITE_OTP_API_BASE_URL>` |
| Staging | <…> | same var, different `.env` |

## Authentication

<How requests are authorized: none / public client id header / session cookie / bearer token supplied by the host via `config.getAuthToken()`. State explicitly that no private key is shipped in the browser.>

CORS: the backend must allow origin(s) <…> with headers <…>. Cookies: <credentials: 'include' | omitted>.

## Send OTP

`<POST> <path>`

Headers:
| Header | Value |
| --- | --- |
| Content-Type | application/json |
| <…> | <…> |

Request body:
```json
<confirmed example>
```

| Field | Type | Required | Source in widget |
| --- | --- | --- | --- |
| <identifier> | string | yes | user input, formatted as <E.164 without +> |

Success response (`<status>`):
```json
<confirmed example>
```
Widget reads: `<path.to.requestId>` → request id; `<path.to.expiry>` → expiry (optional).

## Resend OTP

<Same structure. Include channel field and allowed values if supported.>

## Verify OTP

<Same structure.> On success the widget passes `<fields>` to `onSuccess`.

## Error handling

| HTTP status | Body code / message | Widget error code | Message shown (configurable key) |
| --- | --- | --- | --- |
| 400 | <invalid_otp> | INVALID_OTP | texts.invalidOtp |
| 400 | <otp_expired> | OTP_EXPIRED | texts.expiredOtp |
| 429 | <…> | TOO_MANY_ATTEMPTS / RESEND_TOO_SOON | … |
| network failure | – | NETWORK_ERROR | texts.networkError |
| timeout (<15 s>) | – | TIMEOUT | texts.timeout |
| anything else | – | UNKNOWN | server message or texts.genericError |

## Server-side validation (required)

After `onSuccess`, the host application must send `<token/request id>` to its own backend, which validates it with <provider/your API: endpoint confirmed by the user> before trusting the user. The widget's success callback alone is not proof of verification.

## Changing the contract

All endpoint paths and field mappings live in `<src/api/endpoints.ts>`. Update that file and the mock handlers in `<tests/mocks/handlers.ts>` together.
~~~
