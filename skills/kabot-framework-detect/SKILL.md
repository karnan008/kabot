---
name: kabot-framework-detect
description: |
  Fingerprints the test automation framework in the current repo — language, test runner, driver,
  directory layout, naming convention, run commands and house style — and caches the result to
  .kabot/framework.md so generated code matches what is already there.

  Use this skill when: another kabot stage needs the fingerprint and .kabot/framework.md is
  missing or stale; the user asks "what framework do I have", "re-detect my framework",
  "the generated code doesn't match my style"; or the repo's test layout has changed.

  Loaded automatically by kabot-web-automate, kabot-mobile-automate and kabot-plain-english.
---

# Framework fingerprint

kabot writes code in the user's framework, never its own. That is only possible if the framework
is read off the repo first. This skill produces `.kabot/framework.md` — the contract every
code-writing stage obeys.

**Never skip this and never guess a value.** An unknown field is written as `UNKNOWN` with a note,
not filled with a plausible default. A wrong guess here corrupts every test kabot writes
afterwards.

## Step 1 — Identify language and build system

Look for, in this order, and record every one that hits (a repo can be polyglot):

| Marker | Language | Notes |
|---|---|---|
| `pom.xml` | Java (Maven) | read `<dependencies>` for the runner and driver |
| `build.gradle` / `build.gradle.kts` | Java / Kotlin (Gradle) | |
| `package.json` | JS / TS | read `devDependencies` and `scripts` |
| `pyproject.toml` / `requirements.txt` / `setup.py` | Python | |
| `*.csproj` / `*.sln` | C# / .NET | |
| `Gemfile` | Ruby | |
| `go.mod` | Go | |
| `composer.json` | PHP | |

If several hit, the one whose tree contains the test files wins; note the others as secondary.

## Step 2 — Identify the runner and the driver

Read the dependency list, do not infer from folder names.

- **Runners:** TestNG, JUnit 4/5, pytest, unittest, Playwright Test, Jest, Vitest, Mocha,
  NUnit, xUnit, MSTest, RSpec, `go test`, Cucumber/Behave/SpecFlow (BDD layer on top of a runner).
- **Web drivers:** Playwright, Selenium, Cypress, Puppeteer, WebdriverIO, TestCafe.
- **Mobile drivers:** Appium (+ client language), Espresso, XCUITest, Detox, Maestro, Flutter
  integration_test, Robot Framework + AppiumLibrary.

Record versions — API shape differs between Selenium 3 and 4, JUnit 4 and 5, Appium 1 and 2.

## Step 3 — Map the layout

Find where each kind of file actually lives, by reading the tree rather than assuming:

- test / spec classes
- page objects (or screens / components / robots)
- locator or selector definitions — a separate file/class, or inline in the page object?
- base class / fixtures / conftest / hooks
- config files, environment definitions, test data
- suite definitions (`testng.xml`, `playwright.config.ts`, `pytest.ini`, tags/groups)

Record the naming convention verbatim from real examples: `LoginPage.java`, `login.page.ts`,
`login_page.py`, `LoginSteps`, `login.spec.ts`.

## Step 4 — Read the house style off 2–3 existing tests

Pick the most recently modified test files and record what they actually do. This is the section
that makes generated code indistinguishable from hand-written code:

- **Waiting** — explicit waits, auto-waiting assertions, custom helpers, any banned pattern
  (e.g. a repo that forbids `sleep`, or forbids `networkidle` for SPA reasons)
- **Assertions** — which library, and whether assertions live in tests only or also in pages
- **Logging / reporting** — prefixes, step annotations (`@Step`, Allure, Extent, ChainTest)
- **Retry** — retry analyzer, `test.describe.configure({retries})`, `flaky` markers
- **Ordering** — priorities, `dependsOn`, alphabetical, independent
- **Test data** — fixtures, factories, faker, hardcoded, unique-value conventions
- **Environment/config access** — how a URL or credential is read; confirm nothing is hardcoded
- **Locator strategy preference** — ids, data-testid, role-based, accessibility id, XPath rules
- **Exception / error handling signature** used on test methods
- Anything the repo's own `CLAUDE.md`, `CONTRIBUTING.md`, `README` or lint config mandates —
  **project instructions outrank every default in this skill**

## Step 5 — Find the commands, and verify them

Extract from `scripts`, Makefile, CI config (`.github/workflows`, `.gitlab-ci.yml`, `Jenkinsfile`)
or the runner's convention:

- run a **single** test by name
- run a single **class/file**
- run a **suite/tag/group**
- build/compile only
- how a device/browser target is selected

Verify the build-only command actually runs (e.g. compile with tests skipped). If it fails, record
the failure — do not report a command you have not seen work.

## Step 6 — Write the fingerprint

Write `.kabot/framework.md` using this shape. Keep it short enough that every later stage can read
it whole:

```markdown
# Framework fingerprint
Detected: <ISO date> · Repo: <name> · Confidence: high | medium | low

## Stack
- Language / build: Java 17 / Maven
- Runner: TestNG 7.9
- Web driver: Playwright Java 1.45   (or: none)
- Mobile driver: Appium Java client 9.2 — iOS XCUITest, Android UiAutomator2   (or: none)

## Layout
- Tests:      src/test/java/testcases/<area>/<UseCase>Test.java
- Pages:      src/test/java/pages/<Area>Page.java
- Locators:   src/test/java/locators/<Area>Locators.java  (public static final String)
- Base:       src/test/java/base/BaseTest.java
- Suites:     src/test/resources/testng-<area>.xml  (one per area)
- Config:     config/config.properties + -Denv override

## House style   (observed in <file A>, <file B>)
- Waits: explicit waitForSelector before every action; sleep() is banned
- Assertions: <library>, at the end of every test method
- Logging: System.out.println("[ACTION] …") / "[ASSERT] …"
- Retry: retryAnalyzer = <Class>.class on every @Test
- Ordering: @Test(priority = N), incrementing per class
- Test data: every text input must be a unique value per run
- Locators: prefer <attribute>; never use <banned pattern>

## Commands
- Single test:  <command>
- Single class: <command>
- Suite:        <command>
- Build only:   <command>            [verified: yes/no]
- Target select: <how a device/browser is chosen>

## Project rules that override kabot defaults
- <quoted from CLAUDE.md / CONTRIBUTING.md / lint config>

## Unknowns
- <field>: UNKNOWN — <why, and what would resolve it>
```

## Step 7 — Report

Give the user a 6–10 line summary and the confidence level. If confidence is **low**, or any
field the next stage needs is `UNKNOWN`, say which one and ask before generating code — writing a
test into the wrong layout costs more than one question.

## Staleness

The fingerprint is reused as-is unless: it is older than 30 days, the build file's mtime is newer
than the fingerprint, or the user says the style has changed. Then re-run and overwrite. Always
report which one you used.
