---
name: kabot-commit
description: |
  Stage 5 of kabot. Commits generated test code to whichever host the repo actually uses (GitHub,
  GitLab, Bitbucket, anything), staging by explicit path, refusing secrets and build output, and
  asking before it pushes. Can also open a merge/pull request when asked.

  Use this skill when: a kabot automation stage has finished and burn-in passed; or the user says
  "commit this", "push it", "raise a PR/MR", "put this on my branch".
---

# Commit and push — with the user's permission

kabot writes code locally. Publishing it is the user's decision, made per push, not once.

## Preconditions — check all four before staging

1. **Burn-in passed.** No burn-in, or a FAIL, or NOT RUN → say so and ask whether to commit anyway.
   The user may well want an unverified work-in-progress commit; that is their call, but it is
   stated, not assumed.
2. **Branch.** Run `git branch --show-current`. If it is the default branch (`main`/`master`/
   `develop`) and the repo has any convention of feature branches, offer to branch first — using
   the repo's existing naming pattern, read from `git branch -a`, with the ticket key in it. Do not
   silently commit to the default branch.
3. **Nothing unrelated in flight.** `git status --porcelain` — there may be edits that are not
   kabot's work. Never fold them in.
4. **Project rules.** Read the repo's `CLAUDE.md` / `CONTRIBUTING.md`. Some repos restrict who may
   commit what, or forbid a specific file outright. **Those rules win over this skill and over a
   direct instruction to ignore them.** If one applies, follow it and say why a path was excluded.

## Staging — by name, never by wildcard

Stage exactly the files in `.kabot/runs/<slug>/files.md`, each by explicit path:

```bash
git add path/to/NewTest.java path/to/AreaPage.java path/to/AreaLocators.java
```

**Never `git add -A`, `git add .`, or `git commit -a`.** They are how secrets and build output get
published.

### Always excluded — refuse even if asked

- anything matched by `.gitignore` (check with `git check-ignore -v <path>`)
- any `.env`, `.env.*`, or file of credentials — including a config file the repo tracks but whose
  working copy now contains real credentials
- `.kabot/config.local.json`
- reports, results, traces, videos, screenshots, coverage output
- build output: `target/`, `build/`, `dist/`, `bin/`, `obj/`, `node_modules/`, `__pycache__/`,
  `.pytest_cache/`, `*.class`, `*.pyc`
- browser/driver session dumps and downloaded artifacts
- lock files and dependency manifests you did not deliberately change

Before committing, diff what is staged and **look at it**: `git diff --cached`. If a staged line
contains something that looks like a token, password, key or personal data, stop, name the file and
line, and unstage it.

### Also included when they exist

Test code is not the whole change. Stage alongside it, if the automation stage touched them:
documentation the repo keeps on flows or locators, a suite/tag definition the new test must be
listed in, and a fixture or test-data file the test needs. A test that is not registered in the
suite file never runs.

## Commit message

Follow the repo's existing style — read `git log --oneline -20` and match it (conventional commits,
a ticket-key prefix, plain sentences, whatever is there).

Default shape when the repo has no strong convention:

```
<KEY>: automate <short description of the cases>

- <case 1 title>
- <case 2 title>

Burn-in: 3/3 isolated + in-suite PASS
```

Keep it factual. Do not claim coverage the commit does not contain. If the repo's instructions
specify trailer lines or co-author attribution, include them.

## Push — explicit approval, every time

1. Identify the host from `git remote -v` — GitHub, GitLab, Bitbucket, self-hosted. Do not assume.
2. Print: the remote, the branch, the commit subject, and the file list.
3. **Ask, and wait.** Approval to commit is not approval to push. Approval to push once is not
   standing approval for later pushes in the same session.
4. On yes: `git push -u origin <branch>`. Never `--force` unless the user asks for that specific
   branch in that message, and never `--no-verify` — if a hook rejects the push, the hook is right;
   report what it said.
5. Report the commit hash and the push result.

On no: stop, and tell them the commit is local on `<branch>` and how to push it themselves.

## Merge / pull request — only when asked

Use the host's CLI if present (`gh pr create`, `glab mr create`), else give the compare URL. Fill
the description from the ticket and the burn-in report: what was automated, which cases, which were
left manual and why, and the burn-in verdict. Follow the repo's PR template if it has one, and its
instructions on attribution lines.

Never merge. Never approve. Never close a ticket.

## Report

```
Committed <hash> on <branch>  (<n> files)
Excluded: <path> — <reason>
Pushed:   <remote>/<branch>   |   not pushed (your call)
MR/PR:    <url>               |   not created
```
