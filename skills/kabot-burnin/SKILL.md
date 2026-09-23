---
name: kabot-burnin
description: |
  Stability gate for kabot — runs a newly written or changed test 3 times in isolation plus once
  inside its suite, to prove it passes for the right reason rather than by luck. Reports per-run
  timings, separates environment failures from script defects, and gives a single PASS/FAIL.

  Use this skill when: a kabot stage has just generated or fixed a test; or the user asks "run it 3
  times", "burn it in", "is this test flaky", "prove it's stable", "run the stability check".
---

# Burn-in

One green run proves a test can pass. It does not prove the test passes **because the behaviour is
correct**. Burn-in is the cheapest available proof.

## The bar

| Run | What | Catches |
|---|---|---|
| 1–3 | the test alone, three consecutive times | race conditions, first-run-only state, leftover data from its own previous run, timing flake |
| 4 | the test inside its class/suite/tag | order dependence, shared-state collisions, a sibling test's leftovers |

**PASS = 4/4 green.** Anything else is FAIL. There is no "2 of 3 is fine".

## Procedure

### 1. Get the commands

From `.kabot/framework.md`: the single-test command and the suite command. If either is `UNKNOWN`,
derive it from the runner's convention, verify it runs, and update the fingerprint.

### 2. Run three times in isolation

Run them **sequentially, not in parallel** — parallel runs hide the data collisions run 2 and 3
exist to expose. Capture for each run: exit status, duration, and the failure output if any.

Do not clean state between runs beyond what the framework itself does. Run 2 hitting a unique
constraint on data run 1 created is exactly the defect this stage is for — fix the test's data
generation, do not clear the database and pretend.

### 3. Run once in-suite

Run the whole class, tag or suite the test now belongs to. A test that is green alone and red in
suite is order-dependent — the most expensive kind of flake to debug later.

If the full suite is impractically long (an hours-long end-to-end chain), run the narrowest real
grouping that contains the test — its class or its tag — and say in the report that the full suite
was not run.

### 4. Classify every failure before reacting

| Kind | Signals | Action |
|---|---|---|
| **Script defect** | assertion failed, element not found, wrong value, timeout on an element that is present | fix the test/page/locator, then restart burn-in **from run 1** |
| **Product defect** | the app genuinely does the wrong thing | stop. Report it as a finding — do not weaken the assertion to make it green |
| **Environment failure** | device disconnected, driver/server not reachable, network or DNS error, expired login, missing test data owned by someone else, CI runner died | fix the environment and re-run **that iteration**; it counts as neither pass nor defect. Record the count |

Never make a test pass by deleting an assertion, widening a wait until it happens to work, adding a
retry, or catching and swallowing the failure. If a test only passes with a retry, it has not passed.

### 5. Watch the timings, even at 3/3

Record each duration. If the slowest isolated run is more than roughly twice the fastest, the test
is racing something even though it was green — say so in the report and name the step that varies.
That is a warning, not a FAIL.

### 6. Report

```
Burn-in: <test name>
  Run 1 (isolated): PASS  38s
  Run 2 (isolated): PASS  41s
  Run 3 (isolated): PASS  39s
  Run 4 (in suite <name>): PASS  4m06s
  Environment retries: 1 (device reconnect before run 2)
  Verdict: PASS
  Notes: <timing variance, anything the runs revealed>
```

On FAIL, give the verdict, the failing run, the classification, the actual error, and what you
changed or what you need. Save the report to `.kabot/runs/<slug>/burnin.md`.

## Never do this

- Never report a burn-in you did not run. If no device, no environment or no credentials were
  available, the verdict is **NOT RUN** and the test is **unverified** — say that plainly.
- Never count a compile or a lint pass as a run.
- Never reduce the bar because runs are slow. Say the cost instead and let the user decide.
