# kabot

**Ticket → test steps → automation → burn-in → commit.** In *your* framework, not kabot's.

kabot is a Claude Code plugin for QA engineers. Point it at a ticket and it reads the whole thing
(including the comments, where the real acceptance bar usually hides), works out the test steps,
decides honestly what is worth automating, writes the automation **in the language, layout and house
style your repo already uses**, proves it is stable with a burn-in, and commits it only when you say
so.

It ships no framework of its own and no company-specific knowledge. It detects your stack —
Playwright, Selenium, Cypress, Appium, Espresso, XCUITest, Detox, Maestro; Java, TypeScript, Python,
C#, Ruby — and matches it.

---

## Install

```bash
/plugin marketplace add karnan008/kabot
/plugin install kabot
```

Or clone and add it as a local marketplace:

```bash
git clone https://github.com/karnan008/kabot.git
# in Claude Code:
/plugin marketplace add ./kabot
/plugin install kabot
```

## Set up a project (once per repo)

```bash
mkdir -p .kabot
cp <kabot>/templates/config.example.json       .kabot/config.json
cp <kabot>/templates/config.local.example.json .kabot/config.local.json   # then fill in your token
printf '.kabot/config.local.json\n.kabot/runs/\n' >> .gitignore
```

Edit `.kabot/config.json` with your tracker URL and project keys. Credentials go in
`.kabot/config.local.json` (gitignored) or in the environment:

```bash
export JIRA_BASE_URL="https://your-site.atlassian.net"
export JIRA_EMAIL="you@example.com"
export JIRA_API_TOKEN="…"
```

If your Claude Code session already has an **Atlassian MCP server**, kabot uses that and you can skip
the token entirely.

Then let it learn your framework:

```
/kabot what framework do I have
```

It writes `.kabot/framework.md` — language, runner, driver, where tests/pages/locators live, your
naming convention, your waiting and assertion style, and the commands to run one test and one suite.
Every later stage obeys that file, so generated code reads like yours.

---

## Use it

One entry point. Say what you want in your own words.

```
/kabot ABC-123 give me the test steps
/kabot what can we automate in ABC-123
/kabot automate ABC-123
/kabot automate this: log in as an admin, create an invoice for £250, check it appears in the list
/kabot run the new test 3 times and tell me if it's flaky
/kabot commit it
```

kabot picks the stage, and runs forward through the chain when your request spans several —
"automate ABC-123" walks steps → scope → code → burn-in → commit without asking you to drive each
step.

For analysis you want off your main thread, there is a read-only agent:

```
@kabot-analyst analyse ABC-123, ABC-124 and ABC-125 and tell me what's automatable
```

---

## The pipeline

| Stage | Skill | Does |
|---|---|---|
| 1 | `kabot-ticket-steps` | Reads the ticket and every comment, extracts the **claims**, writes the mandatory test steps — one claim, one case, no invented corner cases. Offers to post them to the ticket. |
| 2 | `kabot-automatable` | Splits the cases into **automate / automate-with-groundwork / manual**, with a real reason each, judged against what your framework can actually reach. Offers to post the scope to the ticket. |
| 3 | `kabot-web-automate` | Writes the web automation in your framework. Reuses your existing page objects and locators; new locators come from **live page inspection**, never a guess. |
| 4 | `kabot-mobile-automate` | Same for Android / iOS, on a real device, emulator or simulator. Platform-specific trees, accessibility ids over XPath, no hardcoded coordinates. |
| 5 | `kabot-commit` | Stages by explicit path, refuses secrets and build output, matches your commit style, and **asks before every push**. Opens a PR/MR if you want one. |
| 6 | `kabot-plain-english` | No ticket needed — prose, a use-case table, Gherkin, or a pasted recording becomes a proper page-object test. |
| — | `kabot-burnin` | The gate: **3 isolated runs + 1 in-suite run**, all green, or it is not done. |
| — | `kabot-framework-detect` | The fingerprint every code stage reads. |
| — | `kabot-jira` | Tracker access: Atlassian MCP first, REST API fallback. |

---

## What kabot will not do

These are deliberate, and they are why its output is trustworthy:

- **It will not invent ticket content.** Every case traces to a claim quoted from the ticket. If the
  fetch fails, it stops rather than guessing.
- **It will not guess a locator.** Selectors come from the live page or element tree. Guessed
  selectors are the single biggest source of flake.
- **It will not impose a framework.** No new dependency, base class or helper appears without asking.
- **It will not call a test done on a compile**, or on one green run.
- **It will not make a test pass by weakening it** — no deleted assertions, no widened waits, no
  swallowed errors, no retry bolted on to hide a race.
- **It will not publish anything silently.** Posting a ticket comment and `git push` each need an
  explicit yes, every time.
- **It will not stage a secret.** No `git add -A`, ever; `.gitignore`, `.env*` and credential files
  are refused even if you ask.
- **It will not claim automation it could not verify.** No device, no environment → the verdict is
  `NOT RUN` and the test is labelled unverified.

Your repo's own rules — `CLAUDE.md`, `CONTRIBUTING.md`, lint config, a pre-push hook — outrank
everything kabot prefers.

---

## State it keeps

All inside your project, so a chain survives a new session or a handover to a teammate:

```
.kabot/
├── config.json          committed — tracker URL, project keys, defaults (no secrets)
├── config.local.json    gitignored — API token
├── framework.md         cached framework fingerprint
└── runs/<KEY>/          steps.md · automatable.md · files.md · burnin.md · comment-N.md
```

## Requirements

- Claude Code
- A tracker if you use the ticket stages: Atlassian MCP server, or a Jira API token. Other trackers
  (GitHub/GitLab Issues, Linear, Azure Boards) work through the same contract.
- For web locator inspection: a browser MCP server, or your framework's own codegen/inspector.
- For mobile: a device, emulator or simulator, plus your framework's driver.

## Licence

MIT — see [LICENSE](LICENSE).
