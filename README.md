# OTP Widget Builder: a Claude skill

A Claude skill that walks you through building your own **OTP verification widget**, or customizing an existing one. Claude asks you questions one at a time, shows a plan for your approval, then writes the code, tests and docs.

- **Any stack:** vanilla TypeScript Web Component, React, Angular, Vue, and others. Claude recommends one based on where the widget will run.
- **Your APIs:** it uses your own send / verify / resend endpoints. It never invents API details and never puts secrets in frontend code.
- **Your design:** the reference layout, your screenshots or Figma, or a generated design, all themed with your brand colours, logo and text.
- **Delivery formats:** embeddable `<script>`, npm package, standalone page, or a component inside your existing app. It builds only the ones you need.
- **Tested and documented:** unit, integration and E2E tests, plus README, API, configuration, integration, customization, architecture, testing and deployment docs and a working demo page. Test results are labelled as mocked or real API.

The OTP flow (timers, resend limits, channel retry, error handling, embedding) is based on [Walkover-Web-Solution/OTP-widget-web](https://github.com/Walkover-Web-Solution/OTP-widget-web). The widgets it generates are independent projects with no runtime dependency on that repo.

## Install

### Claude Code (recommended)

In a Claude Code session:

```
/plugin marketplace add Saurabh186/otp-widget-builder
/plugin install otp-widget-builder@otp-widget-builder
```

Or from your shell:

```bash
claude plugin marketplace add Saurabh186/otp-widget-builder
```

```bash
claude plugin install otp-widget-builder@otp-widget-builder
```

Then ask Claude for an OTP widget, e.g. *"Create an OTP login widget for our React app"*, or run `/otp-widget-builder:otp-widget-builder`.

**Manual install (no plugin system):** download `otp-widget-builder.zip` from [Releases](https://github.com/Saurabh186/otp-widget-builder/releases) and unzip it into `~/.claude/skills/` (just you) or `<your-repo>/.claude/skills/` (shared with your team via git). It then runs as `/otp-widget-builder`.

### Claude.ai and Claude Desktop

1. Download `otp-widget-builder.zip` from [Releases](https://github.com/Saurabh186/otp-widget-builder/releases).
2. Turn on **Settings → Capabilities → Code execution and file creation** (newer layouts may call it "Create and edit files").
3. Go to **Customize → Skills**, upload the ZIP, and make sure the skill is switched on.
4. Ask for an OTP widget in any chat.

On claude.ai, Claude runs code in a sandbox that may not have network or browser access. The skill tells you which checks it couldn't run and gives you the commands to run locally.

## How a session goes

1. **Questions:** new or existing widget, where it runs, formats, identifiers and channels, your API contract (with real request/response examples), OTP rules, design, texts, and what your backend needs on success.
2. **Plan:** a summary, architecture, file list, dependencies and test plan. Nothing is built until you approve it.
3. **Build:** config, API client, state machine, UI and theming, entry points, and a demo page with a mock mode.
4. **Verify:** lint, type check, tests, production build, demo run and responsive checks.
5. **Docs and handoff:** the full documentation set, plus a final checklist covering open items and anything not verified.

## Repository layout

```
.claude-plugin/marketplace.json                   marketplace definition
plugins/otp-widget-builder/
├── .claude-plugin/plugin.json                    plugin manifest
└── skills/otp-widget-builder/
    ├── SKILL.md                                  workflow and rules
    ├── references/                               reference-repo architecture, OTP flow, API/security, design, dev and testing guidelines
    ├── templates/                                project structure, API docs, configuration docs
    └── examples/                                 new widget, API mapping, customization walkthroughs
```

## Security notes

- The skill refuses to put private API keys in browser code and treats a successful verification in the widget as a signal only. Your backend must validate the returned token.
- Never paste production secrets into a chat. Use test credentials or a staging environment for real-API testing.
