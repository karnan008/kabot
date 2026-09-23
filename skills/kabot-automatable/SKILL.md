---
name: kabot-automatable
description: |
  Stage 2 of kabot. Takes the test steps for a ticket — from the ticket itself if they are already
  written there, otherwise by generating them with kabot-ticket-steps — and splits them into what
  is worth automating and what stays manual, with a reason per case. Then offers to post the split
  to the ticket as a comment.

  Use this skill when the user asks: "what can be automated here", "which of these are
  automatable", "automation scope for this ticket", "can we automate this ticket", "add the
  automation coverage to the ticket".
---

# Steps → automatable / manual split

Answers one question honestly: **of the cases that prove this ticket, which ones should become
automated tests, and which should a human run once?**

Automating the wrong case is worse than not automating it — it buys a permanent maintenance cost
for a check that never catches a regression.

## Procedure

### 1. Get the steps

In this order:

1. `.kabot/runs/<KEY>/steps.md` — already generated in this repo, reuse it.
2. The ticket itself — a QA engineer may have written steps or acceptance criteria into the
   description or a comment. Use those as-is; they are the team's own words. Fetch via `kabot-jira`.
3. Neither → run `kabot-ticket-steps` now, then continue. Do not ask the user to run it.

Say which source you used.

### 2. Establish what the framework can actually reach

Read `.kabot/framework.md` (run `kabot-framework-detect` if missing). A case is only automatable in
*this* repo if the repo's driver can reach it. Concretely, check whether the framework has:

- a web driver, a mobile driver, an API/HTTP client, a DB client
- an existing page object / screen for the area under test, or a documented way to add one
- a way to create the required test data (fixture, API, existing helper) rather than needing a
  hand-built account
- for mobile: a real device, an emulator/simulator, or neither

A case needing a capability the repo does not have is **not** automatable today. Say what is
missing instead of promising it.

### 3. Classify every case

| Verdict | Meaning |
|---|---|
| **AUTOMATE** | deterministic, reachable by the repo's driver, and the assertion is machine-checkable |
| **AUTOMATE (needs groundwork)** | automatable, but first needs a new page object, a fixture, a test account, or a driver capability — name the groundwork |
| **MANUAL** | cannot be asserted by code, or the cost clearly exceeds the value — give the reason |

### 4. The reasons that make a case MANUAL

Use these, not vague discomfort. If none applies, the case is automatable.

- **Human judgement** — "looks right", visual polish, animation smoothness, copy tone. (A
  *specific* pixel/colour/order claim is automatable; "feels better" is not.)
- **Unautomatable channel** — push notification, SMS, printed output, phone call, third-party UI,
  payment terminal, camera/biometrics.
- **External system not controllable** — a real supplier or bank sandbox that cannot be reset.
- **Non-deterministic data** — the assertion depends on live data the test cannot pin.
- **One-shot state** — first-run onboarding, a migration, a licence upgrade that cannot be undone.
- **Cost exceeds value** — a one-line config change verified once and never regressing; setup
  longer than the assertion is worth.
- **Blocked by a known tooling gap** — record the gap; it may become automatable later.

"It's hard" is not a reason. "The element has no stable identifier" is a *finding* — report it as
groundwork with a suggested fix (ask dev for a test id), not as MANUAL.

### 5. Output format

```markdown
## <KEY> — automation scope
Steps source: <.kabot/runs/…/steps.md | ticket description | ticket comment | generated now>
Framework: <language / runner / driver, from the fingerprint>

| # | Case | Verdict | Reason / groundwork |
|---|---|---|---|
| 1 | <title> | AUTOMATE | reuses <ExistingPage>; assertion on <what> |
| 2 | <title> | AUTOMATE (needs groundwork) | no page object for <screen>; needs <fixture> |
| 3 | <title> | MANUAL | human judgement — "<quoted from the case>" |

**Automating: 2 of 3 cases.**

### Where the automated cases will live
- <path/to/ExistingTest> — append case 1 (<runner> ordering: next index N)
- <path/to/NewPage> — new page object for case 2

### Groundwork needed first
- <item> — <who/what unblocks it>

### Left manual, on purpose
- Case 3 — <one line>

### Findings for the team
- <e.g. "the total field has no test id — ask dev to add one; it would make case 2 stable">
```

Reuse before creation: if the repo already has a test class or page object covering the area, the
new cases go **into** it. Say so with paths. Never propose a parallel structure alongside an
existing one.

### 6. Save and offer to post

Write to `.kabot/runs/<KEY>/automatable.md`, then offer once to post it to the ticket via
`kabot-jira` — printed first, posted only on an explicit yes.

## Handing off

If the request was "automate this ticket", do not stop here: continue into `kabot-web-automate` or
`kabot-mobile-automate` with this file as the input. Stop and wait only when groundwork blocks
every AUTOMATE case, or when the user asked for the scope alone.
