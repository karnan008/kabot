---
name: kabot-web-automate
description: |
  Stage 3 of kabot. Takes the automatable cases for a web/browser ticket and writes the automation
  in the repo's own framework — same language, runner, layout, naming and house style — then
  burn-in verifies it and hands off to kabot-commit.

  Use this skill when the user asks to automate a ticket, case or flow that runs in a browser:
  "automate this ticket", "write the automation", "script these cases", "add a test for this",
  anything mentioning a URL, browser, Chrome/Firefox/Safari, or a web app screen.

  For an app on a phone, emulator or simulator use kabot-mobile-automate instead.
---

# Automatable cases → web automation, in the user's framework

The whole value of this stage is that the generated test is **indistinguishable from one the team
wrote by hand**. A test that works but reads foreign gets rewritten by the reviewer, which is worse
than no test.

## Step 1 — Load the two contracts

1. `.kabot/runs/<slug>/automatable.md` — the cases. Missing? Run `kabot-automatable` first (which
   will generate steps if needed). Never invent cases at this stage.
2. `.kabot/framework.md` — the fingerprint. Missing or stale? Run `kabot-framework-detect`.

Then read the **project's own instructions** — `CLAUDE.md`, `CONTRIBUTING.md`, lint config, the
fingerprint's "Project rules" section. Those outrank everything below.

Automate only the cases marked **AUTOMATE**. For **AUTOMATE (needs groundwork)**, do the named
groundwork if it is code you can write (a new page object, a fixture); stop and say so if it needs
someone else (a test id from dev, a test account).

## Step 2 — Read before you write

Never write a line until you have read:

- the test file the new cases belong in — to get the next index/priority, the imports, the shape
- the page object(s) for that area — for methods that already exist
- the locator definitions — for constants that already exist
- the base class / fixtures — for what setup is already done for you
- the most similar existing test in the repo — this is your template

**Reuse is the first rule.** If a page method already does what a step needs, call it. Writing a
second method that does the same thing in a slightly different way is the most common way a
generated test gets rejected.

## Step 3 — Locators come from the live page, never from imagination

For any element the repo does not already have a locator for:

1. Search the repo first — the locator may exist under a different name.
2. Still missing → **inspect the live page**. Use a browser MCP server if the session has one
   (navigate, snapshot the accessibility tree / DOM, read the real attributes), or the framework's
   own codegen/inspector.
3. Take the locator from what you actually saw. **Never guess a selector and never derive one from
   a screenshot.** A guessed selector is the single largest source of flake.

Follow the repo's locator strategy from the fingerprint. General order when the repo has no stated
preference: a stable test id → a stable `id`/`name` → an accessible role + visible text → a
structural CSS path. Avoid selectors built from framework-generated state classes, generated ids,
or absolute positional XPath — they change on interaction or rebuild.

Give a critical interaction a fallback selector only if the repo already does that; do not
introduce the pattern unasked.

## Step 4 — Write the code, in the repo's idiom

Match the fingerprint on every axis: file location and name, class/function naming, ordering
mechanism, logging prefixes, assertion library, retry annotation, exception signature, config
access, test-data convention.

Non-negotiables regardless of framework:

1. **The assertion is the test.** Every case ends asserting the case's expected result. Every
   input, selection and save is followed by a verification that it actually landed — never assume
   an action succeeded because it did not throw.
2. **Wait, do not sleep.** Use the framework's waiting mechanism. Never add a fixed sleep to fix a
   race; if the repo bans a specific wait strategy, honour that ban.
3. **No hardcoded environment.** URLs, credentials and accounts come from the repo's config
   mechanism. No secrets in test code, ever.
4. **Unique test data per run.** Generate distinct values (a run-scoped suffix) so you can prove
   *your* value landed in *that* field, instead of matching a leftover from a previous run. Reuse
   the repo's existing convention for this if it has one.
5. **Independent by default.** A case should set up what it needs and not depend on another case
   having run — unless the repo's own suite is deliberately an ordered chain, in which case follow
   that.
6. **Layer discipline.** Locators in the locator layer, interactions in the page layer, flow and
   assertions in the test layer. No raw selectors in a test.
7. **No new dependency** without telling the user and getting a yes.

Write files in this order, so nothing is duplicated: locators → page methods → test cases. Append
to existing files; create a file only when the area genuinely has none.

## Step 5 — Build, then burn in

Compile/lint first using the fingerprint's build command — fix your own errors before running.

Then hand the new tests to `kabot-burnin`: **3 isolated runs plus 1 in-suite run**. A test is not
finished until that passes. Fix what burn-in exposes — flaky waits, data collisions, order
dependence — and re-burn from run 1 after any code change.

Do not report success on a compile. Do not report success on one green run.

## Step 6 — Report and hand off

Write `.kabot/runs/<slug>/files.md`: every file created or modified, with the cases each covers.

Report to the user:

```
Automated <n> of <m> cases for <KEY>.
Files:   <paths, marked new/modified>
Reused:  <existing methods/locators you called instead of writing new ones>
Burn-in: 3/3 isolated + in-suite <PASS|FAIL>   (timings: 41s, 39s, 44s / 4m12s)
Not automated: <case + reason>
Follow-ups: <groundwork or findings for the team>
```

Then hand off to `kabot-commit`. Do not commit from this skill, and never push.
