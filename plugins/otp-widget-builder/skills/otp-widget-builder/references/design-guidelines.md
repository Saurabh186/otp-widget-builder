# Design Guidelines

## Contents

1. Choosing a design approach
2. Reference design (measured from the repo)
3. Theming and branding system
4. Required states
5. OTP input design
6. Responsive rules
7. Accessibility checklist
8. Embedding and style isolation

## 1. Choosing a design approach

Ask the user which applies:

| Approach | What to collect | How to work |
| --- | --- | --- |
| **A. Reference design** | Brand overrides only (logo, primary colour, name) | Rebuild the layout in section 2 with theme tokens. Do not copy MSG91's "Secured by MSG91" footer or logo for other brands. |
| **B. User-provided design** | Screenshots, Figma link/export, spec (sizes, fonts, colours) | Read the images; extract tokens (colours, radius, spacing, type scale) into the theme first, then build components. If a Figma link cannot be opened, ask for exported frames or a spec. List every point where the design is ambiguous (missing error/loading states) and propose a choice. |
| **C. Generated design** | Brand colour, tone (minimal, playful, enterprise), light/dark | Produce a clean card layout with the state set in section 4; show the token table for approval before building. |

Never drop a functional state (error, loading, disabled, cooldown) to match a design. If the design lacks it, design it in the same visual language.

## 2. Reference design (measured from the repo)

From `otp-global.scss`, `send-otp*.scss`, `app_theme-colors.scss`:

| Element | Value |
| --- | --- |
| Presentation | Full-screen fixed overlay `rgb(33 37 40 / 95%)`, centred card, `z-index` 1000 / 999999 |
| Card | 380 px wide, min-height 350 px, padding 48 px 32 px, radius 10 px, white, layered shadow |
| Content width | 316 px |
| Font | Inter (Google Fonts), weights 400/500 |
| Title | 20 px / 24 px, weight 500, `#3f4346`, margin-bottom 24 px |
| Subtitle/helper | 14 px / 16 px, `#3f4346`; identifier on step 2: 18 px weight 500 |
| Secondary text | `#5d6164` (resend line 12 px, loading text 14 px) |
| Links | `#1e75ba`, weight 500, 12 px |
| Error | `#f44336` (messages), `#cc5229` (field error text 12 px) |
| Primary colour | Material azure palette generated from `#1157A6` |
| Buttons | Full width, radius 4 px, no shadow, 24 px top margin |
| Progress | Indeterminate bar pinned to the card top during send/resend/verify; spinner overlay (48 px) during config load |
| Close | 25 px icon button, top-right 16 px |
| Mobile (≤768 px) | Card 90 % width, 80 % height |
| Footer | "Secured by MSG91" SVG, fixed bottom-left (MSG91 branding) |

## 3. Theming and branding system

Implement theming with **CSS custom properties** on the widget root (works across frameworks and through Shadow DOM, where host-page CSS cannot reach). Minimum token set:

```css
:host, .otpw-root {
  --otpw-color-primary: #1157a6;
  --otpw-color-primary-contrast: #ffffff;
  --otpw-color-text: #3f4346;
  --otpw-color-text-muted: #5d6164;
  --otpw-color-error: #d32f2f;
  --otpw-color-success: #2e7d32;
  --otpw-color-surface: #ffffff;
  --otpw-color-border: #c1c5c8;
  --otpw-color-overlay: rgb(33 37 40 / 0.9);
  --otpw-radius: 10px;
  --otpw-radius-input: 6px;
  --otpw-font-family: Inter, system-ui, sans-serif;
  --otpw-space: 8px;          /* spacing scale = multiples of this */
  --otpw-card-width: 380px;
}
```

Expose branding through the widget config (typed, documented):

- `theme`: partial token map → written to the root element's `style`.
- `logoUrl`, `logoAlt`, `companyName`.
- `texts`: every user-facing string (titles, buttons, helper, errors, success) with defaults; supports localization.
- `className` / `customCss`: optional extra CSS injected inside the shadow root (document that it is the user's responsibility).

Make sure every config option is actually wired (the reference's `style` option is dead).

## 4. Required states

Each must be visually distinct and announced to assistive tech:

| State | UI |
| --- | --- |
| Loading config | Skeleton or spinner; inputs not interactive |
| Idle identifier | Label, input, helper text, primary button |
| Invalid identifier | Error border + message after blur/submit (not on first keystroke) |
| Sending / verifying / resending | Button shows spinner + disabled; inputs disabled; prevent double submit |
| Code entry | Masked identifier with "Change" action, code input focused |
| Cooldown | "Resend in 0:24" (non-interactive), then a resend button/link |
| Resend limit reached | Explanation text, no resend control |
| Invalid / expired code | Error message tied to the input; input cleared or selected for retyping |
| Too many attempts | Locked message with next step (restart/contact) |
| Network/server error | Retry-able message, keeps entered data |
| Success | Confirmation (icon + text) before handing back, or immediate close if the host prefers |

## 5. OTP input design

Two valid patterns. Pick per design, keep accessibility:

- **Single input** (reference): `inputmode="numeric"`, `autocomplete="one-time-code"`, `maxlength=otpLength`, digits-only filtering, paste of full code. Most robust for SMS autofill and screen readers.
- **Segmented boxes**: one visual box per digit. Prefer one hidden-but-focusable real input driving the boxes, or several inputs with: auto-advance, backspace to previous, full-code paste distribution, arrow keys, a single group label (`role="group"` + `aria-label`). Ensure `autocomplete="one-time-code"` on the first input.

Do not use `type="number"` (strips leading zeros, shows spinners).

## 6. Responsive rules

- Mobile first. Breakpoints: ≤480 px phone (full-width card or bottom sheet, 16 px side padding), 481–1024 px tablet (centred card), >1024 px desktop.
- Touch targets ≥44×44 px. Input font-size ≥16 px on mobile (prevents iOS zoom).
- No horizontal scroll at 320 px width. Test in both orientations.
- Overlay mode: lock body scroll while open; restore on close.

## 7. Accessibility checklist

- Every input has a visible `<label>` (or `aria-label` when the design hides it).
- Errors: `aria-invalid="true"` + `aria-describedby` pointing at the message; message container `role="alert"` or `aria-live="assertive"`.
- Timer: announce only at start and when resend becomes available (`aria-live="polite"`), not every second.
- Modal: `role="dialog"`, `aria-modal="true"`, labelled by the title; focus moves in on open, is trapped, returns to the trigger on close; `Esc` closes when closing is allowed.
- Contrast ≥4.5:1 for text, ≥3:1 for UI borders and focus rings. Check brand colours; derive a darker shade if needed.
- Visible focus style on every control. Respect `prefers-reduced-motion`.

## 8. Embedding and style isolation

- Script/embed builds: render inside a Shadow DOM (custom element) so host CSS does not leak in and widget CSS does not leak out. Load fonts by `<link>` inside the shadow root or fall back to system fonts.
- In-app components (React/Vue/Angular inside the user's app): no Shadow DOM by default; scope styles (CSS modules, scoped styles, BEM prefix `otpw-`) and respect the host app's design system if the user asks.
- Never inject global styles into the host page (the reference sets `html, body { overflow: hidden }` in `styles.scss`; avoid this in embeds except for an explicit scroll lock while a modal is open).
