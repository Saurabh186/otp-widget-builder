# Template: widget configuration schema and docs/CONFIGURATION.md

Use this as the checklist of configurable options. Implement only options in scope, and make every documented option actually work (the reference documents a `style` option that does nothing).

## Option catalogue

| Group | Option | Type | Default | Notes |
| --- | --- | --- | --- | --- |
| Mount | `target` | `string \| HTMLElement` | none → modal overlay | Inline vs modal |
| Mount | `mode` | `'modal' \| 'inline'` | derived from `target` | |
| API | `apiBaseUrl` | `string` | required | From env in the demo |
| API | `endpoints` | `Partial<{ send, verify, resend, config }>` | from contract | Paths only |
| API | `getAuthToken` | `() => string \| Promise<string>` | none | When the host app supplies a session/bearer token |
| API | `requestTimeoutMs` | `number` | 15000 | |
| Identifier | `identifierTypes` | `('phone' \| 'email')[]` | per requirements | |
| Identifier | `defaultCountry` | ISO 3166-1 alpha-2 | per requirements | Phone only |
| Identifier | `identifier` | `string` | none | Prefill |
| Identifier | `autoSend` | `boolean` | false | Send immediately when a valid prefill is given |
| Identifier | `lockIdentifier` | `boolean` | false | Disable editing the prefill |
| Channels | `channels` | `('sms' \| 'whatsapp' \| 'email' \| 'voice')[]` | per backend | Only those the backend supports |
| Channels | `defaultChannel` | channel | first of `channels` | |
| Code | `otpLength` | `4 \| 5 \| 6 \| 8` | 6 | Or from config endpoint |
| Code | `inputStyle` | `'single' \| 'segmented'` | `'single'` | |
| Code | `autoSubmit` | `boolean` | false | Verify when last digit entered |
| Timers | `resendCooldownSec` | `number` | per backend | Display; server enforces |
| Timers | `maxResends` | `number` | per backend | Display; server enforces |
| Timers | `showExpiry` | `boolean` | false | Needs expiry from API or config |
| Timers | `expirySec` | `number` | none | Only if API does not return it |
| Branding | `companyName`, `logoUrl`, `logoAlt` | `string` | none | |
| Branding | `theme` | `Record<CSS custom property, string>` | built-in tokens | See design-guidelines §3 |
| Branding | `customCss` | `string` | none | Injected inside shadow root |
| Text | `texts` | `Partial<Texts>` | English defaults | Every visible string |
| Text | `locale` | `string` | `'en'` | Only if i18n requested |
| Behaviour | `closable` | `boolean` | true (modal) | |
| Callbacks | `onSuccess` | `(result) => void` | required | Result shape agreed with user |
| Callbacks | `onError` | `(error: { code, message }) => void` | none | |
| Callbacks | `onClose` | `(reason) => void` | none | |
| Callbacks | `onStateChange` | `(state) => void` | none | Analytics hooks, only if requested |

`Texts` keys (suggested): `title`, `subtitle`, `identifierLabel`, `phonePlaceholder`, `emailPlaceholder`, `sendButton`, `codeTitle`, `codeLabel`, `verifyButton`, `changeIdentifier`, `resendIn` (with `{seconds}`), `resend`, `resendVia` (with `{channel}`), `invalidIdentifier`, `invalidOtp`, `expiredOtp`, `tooManyAttempts`, `resendLimit`, `networkError`, `timeout`, `genericError`, `success`.

## Validation rules

- Throw at init for missing required options with a message naming the option.
- Reject unknown `channels` values and `otpLength` outside the allowed set.
- Warn (dev builds only) for unknown keys, to catch typos like the reference's `exposedMethods`.

## docs/CONFIGURATION.md skeleton

~~~markdown
# Configuration

## Minimal example
```js
OtpWidget.init({
  apiBaseUrl: 'https://api.example.com',
  onSuccess: (result) => { /* send result.token to your backend */ },
});
```

## Full example
```js
<every option in scope with realistic values>
```

## Options
| Option | Type | Default | Description |
| --- | --- | --- | --- |
<only implemented options>

## Texts
| Key | Default |
| --- | --- |

## Theme tokens
| Token | Default | Used for |
| --- | --- | --- |

## Events / callbacks
| Callback | Payload | When |
| --- | --- | --- |
~~~
