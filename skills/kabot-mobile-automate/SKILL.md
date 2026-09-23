---
name: kabot-mobile-automate
description: |
  Stage 4 of kabot. Takes the automatable cases for a mobile ticket and writes the automation in the
  repo's own mobile framework — Appium, Espresso, XCUITest, Detox, Maestro or whatever is detected —
  matching its language, layout and house style, then burn-in verifies it on a real device,
  emulator or simulator and hands off to kabot-commit.

  Use this skill when the user asks to automate a ticket, case or flow on a phone or tablet:
  anything mentioning Android, iOS, app, APK, IPA, device, emulator, simulator, Appium, Espresso,
  XCUITest, Detox, Maestro, or a screen of a mobile app.

  For a browser target use kabot-web-automate instead.
---

# Automatable cases → mobile automation, in the user's framework

Same contract as the web stage, with the differences that actually matter on a device: the target
is physical, the element tree is platform-specific, and a run costs minutes rather than seconds.

## Step 1 — Load the contracts

1. `.kabot/runs/<slug>/automatable.md` — the cases (run `kabot-automatable` if absent).
2. `.kabot/framework.md` — the fingerprint (run `kabot-framework-detect` if absent or stale).
3. The project's own instructions — `CLAUDE.md`, `CONTRIBUTING.md`, any reference docs the repo
   keeps on locators or device setup. **They outrank everything below.**

Confirm the platform (Android / iOS / both) from the ticket, the request, or the repo. If the repo
supports both and nothing says which, ask — porting later costs more than asking now.

## Step 2 — Confirm a target exists, before writing anything

A mobile test cannot be verified without a device. Check what the session actually has, using the
repo's own tooling where it has some:

- Android: `adb devices -l` — a device/emulator listed and authorised
- iOS: a connected device (`xcrun devicectl list devices` / the repo's detection helper) or a
  booted simulator (`xcrun simctl list devices booted`)
- Appium: a server reachable, and the driver the repo uses installed
- the app under test installed, and at the build the ticket refers to

If no target is available, write the code but say plainly that burn-in could not run and the test
is **unverified**. Never present unverified mobile code as done.

If the repo drives a live inspection session (an Appium MCP server, an inspector), **quit any live
session before starting a test run** — a leftover session takes the port the runner needs.

## Step 3 — Read before you write

The most similar existing test, the screen/page object for that area, the locator definitions, the
base class and its device-lifecycle hooks (who launches the app, who resets state between tests).
Reuse an existing screen method rather than adding a near-duplicate.

Respect the repo's session model. Many mobile suites share one driver session per class because a
fresh session costs 15–30s, and relaunch the app between tests instead. Write tests that fit that
model rather than assuming a clean session per method.

## Step 4 — Locators come from the live element tree

Never guess a mobile locator and never derive one from a screenshot. For every element the repo does
not already have:

1. Search the repo first.
2. Inspect the **live** tree — an Appium MCP server's page source, Appium Inspector, `uiautomator
   dump`, Xcode's accessibility inspector — whatever the repo uses.
3. Take exactly what you saw.

Platform notes that cost people days when ignored:

- Prefer an **accessibility id / resource-id / name** over any XPath. Use XPath only when nothing
  stable exists, and anchor it on an attribute, not on position.
- The same logical screen has **different trees per platform** — an Android locator is not an iOS
  locator. Keep them separated the way the repo separates them.
- A label can appear **several times in the tree** (offscreen copies, reused cells). When it does,
  disambiguate on a real attribute or index deliberately, and write down *why* in a comment.
- **No hardcoded coordinates.** Compute from screen size at runtime; a tap at `(540, 1200)` breaks
  on the next device.
- Scrolling is part of finding: an element absent from the tree may simply be below the fold. Scroll
  with the framework's scroll action, do not fail on first miss.
- Keyboards, permission dialogs and system alerts intercept taps. Handle them explicitly the way
  the repo already does.

## Step 5 — Write the code, in the repo's idiom

Match the fingerprint exactly on layout, naming, ordering, logging, assertions and retry. Plus the
non-negotiables:

1. **Assert the expected result**, and verify every input/selection/save actually landed.
2. **Wait, do not sleep** — use the framework's waits; never paper over a race with a fixed delay.
3. **Device-agnostic** — no fixed coordinates, no device-specific timing constants, no
   environment-specific ids. Compute at runtime.
4. **No hardcoded environment or credentials.**
5. **Unique test data per run**, so you can prove your value landed in that field.
6. **Layer discipline** — locators / screen objects / tests stay separate; no raw locators in tests.
7. **No new dependency** without asking.
8. **Never reset app data** unless the repo's setup already does; wiping a signed-in state can cost
   a long re-provisioning.

Order of writing: locators → screen methods → test cases. Append to existing files where the area
already exists.

## Step 6 — Build, then burn in on the real target

Compile first. Then hand to `kabot-burnin`: **3 isolated runs + 1 in-suite run** on the actual
device/emulator.

Mobile burn-in is slow and the environment is a real participant, so:

- Separate **test failure** from **environment failure** — a dropped session, an unauthorised
  device, a lost tunnel, a network blip, an OS dialog. An environment failure does not count as a
  burn-in pass *or* a script defect: fix the environment and re-run that iteration.
- Record each run's duration. Large variance between green runs is a flake signal even at 3/3.
- If the repo's suite runs on multiple platforms, burn in on the platform the ticket targets and say
  the other platform is unverified.

Re-burn from run 1 after any code change.

## Step 7 — Report and hand off

Write `.kabot/runs/<slug>/files.md`, then report:

```
Automated <n> of <m> cases for <KEY>  (<platform>, <device/emulator name + OS version>).
Files:   <paths, new/modified>
Reused:  <existing screen methods/locators called>
Burn-in: 3/3 isolated + in-suite <PASS|FAIL>   (timings: …)   [environment retries: <n>]
Not automated: <case + reason>
Follow-ups: <groundwork, findings, unverified platform>
```

Then hand off to `kabot-commit`. Do not commit here, and never push.
