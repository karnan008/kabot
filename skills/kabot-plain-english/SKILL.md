---
name: kabot-plain-english
description: |
  Stage 6 of kabot. Turns a test case written in plain English — free prose, numbered steps, a
  use-case table, BDD Given/When/Then, or a recorded script pasted in any language — into automation
  in the repo's own framework, burn-in verified and ready to commit. No ticket needed.

  Use this skill when the user describes what to test in their own words and wants code: "write a
  test that logs in and…", "automate these steps", "convert this to a test", "here's UC-12 with its
  steps", "turn this recording into our framework", "here's a Gherkin scenario".
---

# Plain English → automation

The ticket path (stages 1–2) exists to find out *what* to test. Here the user already knows. This
skill's only job is to turn their words into a real, verified test without losing or inventing
anything.

## Step 1 — Normalise the input

Whatever form it arrives in, restate it as: **preconditions, numbered steps, expected result(s)**.
Handle each shape on its own terms:

- **Prose** — split into atomic steps; one action per step.
- **Numbered steps** — keep the user's numbering and wording.
- **Use-case / test-case table** — read every column; a "test data" or "expected result" column is
  part of the spec, not decoration.
- **BDD** — `Given` → preconditions, `When` → steps, `Then` → assertions. One `Then` clause is one
  assertion; do not merge them.
- **A recorded script** (codegen from any language, a Selenium IDE export, a Maestro flow) — read
  what it actually does, then **discard its structure**. Recorded scripts are flat, brittle and full
  of generated selectors; keep the *intent* and the *values*, and rebuild it in the repo's page-object
  idiom. Never paste a recording in as a test.

**Show the normalised version back to the user in a compact block and get a nod** when anything is
ambiguous — a missing expected result, an unnamed screen, a value like "some customer". One question
now beats a wrong test later. If it is unambiguous, say the assumption in one line and keep going.

## Step 2 — Fill the gaps honestly

Two things are missing from almost every plain-English case:

- **The expected result.** If the user wrote steps but no assertion, ask what proves it worked. A
  test without an assertion is a script, not a test, and kabot does not write one.
- **The starting state.** Which account, role, data, feature flag, build. Ask if a step cannot be
  reached without it.

Never fill either with a guess. Everything else — a label you can look up, a route you can read out
of an existing page object — you resolve yourself without asking.

## Step 3 — Decide web or mobile

From the user's words (URL/browser vs app/device), then the fingerprint (`.kabot/framework.md` — if
the repo has only one kind of driver, that is the answer), then ask. Run
`kabot-framework-detect` first if the fingerprint is missing.

## Step 4 — Check it is automatable, briefly

Apply the same test as `kabot-automatable`, without the ceremony: if a step needs human judgement,
an unautomatable channel (push notification, SMS, print, payment terminal, camera), or a system you
cannot control, say so **before** writing code and offer to automate the rest. Do not silently drop
a step, and do not weaken an assertion to make an unautomatable step look automated.

## Step 5 — Hand to the code-writing stage

Write the normalised case to `.kabot/runs/<slug>/steps.md` (`<slug>` = a short kebab-case name from
the case title, or the user's own UC/TC id), then run `kabot-web-automate` or
`kabot-mobile-automate`. Those stages own the code rules: read-before-write, reuse first, locators
from live inspection only, repo idiom, no sleeps, assert everything.

Burn-in applies exactly as it does to the ticket path: **3 isolated runs + 1 in-suite**.

## Step 6 — Report and offer the commit

Report what was generated, which files, what was reused, the burn-in verdict, and anything you
left out with the reason. Then offer `kabot-commit` — and let the user decide about pushing.
