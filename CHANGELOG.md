# Changelog

## 0.1.0 — 2026-09-23

First release.

- `kabot` — router skill, the single entry point for the pipeline
- `kabot-ticket-steps` — ticket (+ comments) → mandatory test steps → tracker comment
- `kabot-automatable` — steps → automatable / manual split → tracker comment
- `kabot-web-automate` — automatable cases → web automation in the repo's own framework
- `kabot-mobile-automate` — same for Android / iOS
- `kabot-plain-english` — plain English, use-case tables, BDD or a recorded script → automation
- `kabot-burnin` — 3 isolated runs + 1 in-suite run as the stability gate
- `kabot-commit` — stage by name, refuse secrets, ask before pushing
- `kabot-framework-detect` — framework fingerprint cached to `.kabot/framework.md`
- `kabot-jira` — tracker access via Atlassian MCP, falling back to the REST API
- `kabot-analyst` — read-only agent for ticket analysis
