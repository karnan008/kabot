---
name: kabot
description: |
  kabot router — the single entry point for the whole ticket-to-automation pipeline. Reads the
  request, works out which stage of the pipeline it belongs to, and runs that stage.

  Use this skill whenever the user:
  - pastes a ticket key or ticket URL (Jira or any tracker) and asks anything about testing it
  - says "test steps", "how do I test this", "what should I verify", "QA steps for this ticket"
  - asks "what can be automated here", "which cases are automatable", "automation scope"
  - says "automate this", "automate this ticket", "write the automation", "script this"
  - describes a test case in plain English and wants it turned into code
  - asks for a burn-in, a stability run, "run it 3 times", "is this test flaky"
  - asks to commit or push generated test code
  - types /kabot with anything after it

  This is the only kabot skill a user should need to invoke by name. It loads the others.
---

# kabot — router

kabot takes a ticket (or plain English) and walks it to committed, burn-in-verified automation
**in the framework the user already has** — never in a framework kabot prefers.

## The pipeline

```
          ticket key / URL
                 │
   ┌─────────────┴──────────────┐
   │  1. kabot-ticket-steps     │  ticket (+ comments) -> mandatory test steps -> post to ticket
   └─────────────┬──────────────┘
                 │
   ┌─────────────┴──────────────┐
   │  2. kabot-automatable      │  steps -> automatable / manual split -> post to ticket
   └─────────────┬──────────────┘
                 │
   ┌─────────────┴──────────────┐       plain English ──► same entry point
   │  3. kabot-web-automate     │  or   4. kabot-mobile-automate
   │     (web / browser)        │          (Android / iOS)
   └─────────────┬──────────────┘
                 │  every generated test goes through
   ┌─────────────┴──────────────┐
   │     kabot-burnin           │  3 isolated runs + 1 in-suite run
   └─────────────┬──────────────┘
                 │
   ┌─────────────┴──────────────┐
   │  5. kabot-commit           │  detect remote, stage by name, commit, ask before push
   └────────────────────────────┘
```

Two support skills are loaded by the stages, never by the user:

- **kabot-framework-detect** — fingerprints the target repo (language, runner, layout, run
  command, house style) and caches it. Every code-writing stage depends on it.
- **kabot-jira** — fetches tickets and posts comments, via MCP when available and REST otherwise.

## Routing table

Read the user's message and pick **one** entry point. When the request spans several stages
("give me the steps and automate it"), start at the earliest stage and run forward through the
chain — each stage hands its artifact to the next, so nothing is derived twice.

| The request is… | Route to | Notes |
|---|---|---|
| ticket + "test steps / how do I test / what to verify" | `kabot-ticket-steps` | |
| ticket + "what's automatable / automation scope" | `kabot-automatable` | it calls `kabot-ticket-steps` itself if the ticket has no steps yet |
| ticket + "automate this" for a **web/browser** target | `kabot-web-automate` | it pulls the automatable list first |
| ticket + "automate this" for a **mobile** target (Android/iOS/app/device/emulator/simulator) | `kabot-mobile-automate` | |
| plain English steps, no ticket | `kabot-plain-english` | it decides web vs mobile, then delegates to stage 3 or 4 |
| "run it 3 times / burn it in / is it flaky" for an existing test | `kabot-burnin` | |
| "commit / push this" | `kabot-commit` | |
| "what framework do I have / re-detect" | `kabot-framework-detect` | |

### Choosing web vs mobile when it is not stated

In order: an explicit word in the request (browser/Chrome/URL vs app/device/APK/simulator) →
the ticket's project key, components or labels → the fingerprint in `.kabot/framework.md` (if the
repo only has one kind of driver, that is the answer) → ask. Never guess when the repo supports
both and the ticket says nothing.

## Hard rules for every stage

1. **Never invent a framework.** Read `.kabot/framework.md`; if it is missing, run
   `kabot-framework-detect` first. Generated code matches the detected language, runner,
   directory layout, naming convention and house style — even when kabot would have chosen
   differently. Never introduce a new dependency, base class or helper library without saying so
   and getting a yes.
2. **Never invent ticket content.** Every test case traces to something written in the ticket.
   If the fetch fails, say so and stop.
3. **Two actions publish and both need explicit approval**: posting a ticket comment, and
   `git push`. Show the exact payload or path list, then wait for a yes. A yes for one is never a
   yes for the other.
4. **Never stage a secret.** `kabot-commit` stages by explicit path and refuses anything
   `.gitignore`d, any `.env*`, and any config file holding credentials.
5. **No test is "done" before burn-in.** A test that compiles is not a passing test.
6. **Artifacts, not memory.** Each stage writes its output under `.kabot/runs/<slug>/` so a later
   stage, a new session or a different person can pick the chain up mid-way.

## Working directory and state

Everything kabot stores lives in the **target project**, not in the kabot install:

```
.kabot/
├── config.json          committed — tracker base URL, project keys, defaults (no secrets)
├── config.local.json    gitignored — API tokens
├── framework.md         cached framework fingerprint
└── runs/<slug>/         per-ticket artifacts: steps.md, automatable.md, files.md, burnin.md
```

`<slug>` is the ticket key (`ABC-123`) or, for plain English, a short kebab-case name.

On the first run in a repo, if `.kabot/` does not exist: create it, run
`kabot-framework-detect`, and add `.kabot/config.local.json` and `.kabot/runs/` to the project's
`.gitignore` (ask before editing `.gitignore`).

## Output discipline

State which stage you are in, in one line, then do the work. Do not narrate the routing table
back to the user and do not print this pipeline diagram.
