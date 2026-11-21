---
name: otp-widget-builder
description: Builds or customizes an OTP verification widget on the user's own APIs in any frontend stack, with branding, tests and docs. Use when asked to create, embed, restyle or extend an OTP widget.
---

# OTP Widget Builder

Interactively create a new, independent OTP verification widget, or customize an existing one, using Walkover's OTP-widget-web repository as the technical reference for OTP behaviour. The reference repo is documented in `references/`; generated projects never depend on it at runtime.

## Reference files: when to read them

| File | Read when |
| --- | --- |
| `references/repository-architecture.md` | Always, before planning (stack, contract, known defects not to copy) |
| `references/otp-flow.md` | Always, before planning (lifecycle, timers, limits, state machine) |
| `references/api-integration.md` | Collecting API details; writing the API layer; any security question |
| `references/design-guidelines.md` | Discussing design; building UI, theming, accessibility |
| `references/development-guidelines.md` | Choosing technology/output formats; before writing code; customizing existing code |
| `references/testing-guidelines.md` | Writing tests; verifying; writing the test report |
| `templates/project-structure.md` | Planning files and the documentation set |
| `templates/api-documentation.md` | Writing the project's `docs/API.md` |
| `templates/widget-configuration.md` | Designing config options; writing `docs/CONFIGURATION.md` |
| `examples/basic-otp-widget.md` | A full run for a new widget, with verified core code |
| `examples/api-integration-example.md` | Turning a pasted API contract into adapter + tests |
| `examples/custom-branded-widget.md` | Customizing an existing widget |

## Non-negotiable rules

1. **No invented contracts.** Never make up endpoints, methods, headers, payloads, response fields, error codes or auth formats. Ask. Unconfirmed items get `TODO(confirm-with-user)` and appear in the final report.
2. **No secrets in frontend code**, no client-side-only verification, no OTPs in storage or logs. The host's backend validates the verification result. Details: `references/api-integration.md` §3.
3. **No false claims.** Only report checks you actually ran; label tests Mocked vs Real API.
4. **Ask before major decisions**: framework, output formats, state library, adding a backend, replacing existing components, breaking public APIs.
5. **Minimal, clean, production-ready code**; no unneeded dependencies, files, fallbacks or abstractions; no changes to unrelated code. Full list: `references/development-guidelines.md` §1.
6. **Don't copy reference defects** (listed in `references/repository-architecture.md` §10) or its committed sample credentials.

## Workflow

Follow the phases in order. Keep a visible checklist of phases for the user.

### Phase 1: Understand the request and the reference

1. Read `references/repository-architecture.md` and `references/otp-flow.md`.
2. If the user is customizing an existing widget, get its code (path, repo URL or ZIP) and inspect it now (framework, build, config flow, styles, tests). If customizing the reference repo itself and network access exists, clone `https://github.com/Walkover-Web-Solution/OTP-widget-web.git` and confirm the documented commit (`7e3c798`) or note differences.
3. If a repository the user points to is private or unreachable, ask for access or a ZIP. Do not guess its contents.

### Phase 2: Gather requirements interactively

Ask **one question per message** (use the host's question/choice tool when available, otherwise plain chat). Skip any question already answered by the user, their files, or their existing code. Offer sensible options with a recommended default where one exists, but never default API contracts, security or business behaviour.

Question bank, in priority order:

1. **Mode**: new widget, or customize an existing one?
2. **Environment**: where will it run (any website via script tag, a specific React/Angular/Vue/other app, mobile WebView)? Existing code to integrate with?
3. **Output formats**: standalone app, embeddable script, npm package, in-app component (only what is needed; see `references/development-guidelines.md` §3).
4. **Technology**: recommend from the environment using `references/development-guidelines.md` §2 and ask for confirmation rather than asking open-ended.
5. **Identifiers and channels**: phone, email or both; default country; SMS / WhatsApp / email / voice; can the user switch channel on resend?
6. **APIs**: run the intake checklist in `references/api-integration.md` §2 (base URLs, send/verify/resend, methods, headers, auth, request/response examples, error formats, channels, limits, CORS, captcha). Ask for real examples.
7. **OTP behaviour**: length, resend cooldown, max resends, expiry display, auto-submit, attempt limits, what happens on lock (`references/otp-flow.md` §C).
8. **Design**: reference design, user-provided (screenshots/Figma/spec), or generated; brand colours, logo, company name, fonts, modal vs inline (`references/design-guidelines.md` §1).
9. **Texts**: button labels, titles, success/error copy, languages.
10. **Success contract**: what the host receives (token, identifier, request id) and how its backend validates it.
11. **Deliverables and constraints**: hosting/CDN, browser support, bundle-size limits, test credentials or staging environment, deadline-driven scope cuts.

Ask follow-ups whenever an answer is ambiguous or reveals something unconfirmed (e.g. the user has only a private provider key and no backend: explain the risk and ask how to proceed).

### Phase 3: Summarize and plan, then get approval

Present one concise message containing:

- Requirements summary (everything confirmed, grouped as above).
- Open items / assumptions, explicitly marked.
- Architecture: stack, output formats, layers (config / api / core / ui / entry), state approach.
- File plan (from `templates/project-structure.md`, trimmed to need) or, for customization, the file-by-file change map.
- Dependencies with one-line justification each.
- Test plan (scenario matrix rows in scope, tools, mocked vs real).
- Deliverables list.

Do not start implementation until the user approves. If they change something, update the plan and re-confirm only the changed part.

### Phase 4: Implement

1. Read `references/development-guidelines.md` (and `references/design-guidelines.md` for UI work).
2. Scaffold with the framework's official tooling where one exists (e.g. `npm create vite@latest`), then remove unused boilerplate.
3. Build in this order: config + validation → API client and adapters from the confirmed contract → state machine (`references/otp-flow.md` §B) → UI and theming → entry points (embed bootstrap, package exports) → demo page (with an optional mock mode excluded from production bundles).
4. Keep UI, flow logic, API and configuration separated; every documented config option must work.
5. For customization: smallest diff per requirement, backward compatible unless approved; record a baseline build/test run before editing.

### Phase 5: Test and verify

Follow `references/testing-guidelines.md`:

1. Write unit, integration and (where practical) E2E tests covering the scenario matrix rows in scope.
2. Run lint, type check, tests, production build; run the demo and exercise the full flow at least in mock mode; check responsive widths.
3. Fix failures your work caused. Report pre-existing failures separately.
4. If something cannot run in this environment (no network, no browser), say exactly what was skipped and give the commands.

### Phase 6: Document

Produce the documentation set in `templates/project-structure.md` §5 (README, `.env.example`, API, configuration, integration for each built format, customization, architecture, testing with actual results, deployment, usage examples, demo). Document only behaviour that exists.

### Phase 7: Final check and handoff

Before saying the work is done, confirm each item and tell the user the result:

- [ ] Requirements in the approved plan implemented; deviations explained
- [ ] No invented API details; all `TODO(confirm-with-user)` items listed
- [ ] No secrets in code, config or git history; `.env.example` present
- [ ] Verification result is validated server-side (documented for the host)
- [ ] Lint / types / tests / build results stated with Mocked vs Real labels
- [ ] Demo runs; responsive check done
- [ ] Docs complete and accurate
- [ ] Only requested formats built; no unused files or dependencies

End with: what was built, where the files are, how to run the demo, test results, open items, and suggested next steps (e.g. run real-API tests once credentials exist).

## Environment notes

- **Claude Code**: you can run commands, install packages and execute tests; do so.
- **Claude.ai / Claude Desktop**: code execution may lack network access or browsers. Generate the full project, run what the sandbox allows (often type check and unit tests if packages install), clearly list what was not executed, and provide the project as downloadable files.
