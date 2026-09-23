---
name: kabot-ticket-steps
description: |
  Stage 1 of kabot. Reads a ticket in full — description, every comment, attachments — extracts the
  claims it actually makes, and returns the mandatory test steps that prove those claims. Then
  offers to post them back to the ticket as a comment.

  Use this skill when the user pastes a ticket key or URL and asks: "test steps", "how do I test
  this", "what should I verify", "QA steps", "test cases for this ticket", "I'm picking this up for
  QA". Also use when another kabot stage needs steps for a ticket that has none.

  Not for creating tickets, and not for writing automation code.
---

# Ticket → mandatory test steps

Produces the **minimum set of cases that must pass for the ticket to be signed off** — and nothing
more.

This skill exists because the default instinct when asked for test cases is to brainstorm:
rotation, offline, low battery, accessibility, 500 records, rapid double-taps, localisation,
SQL injection. **That output is not wanted here.** The reader is a QA engineer verifying one
specific ticket this afternoon, not authoring a feature test plan.

## The one law

> **Every case must trace to a claim written in the ticket. If it does not trace to a claim, it
> does not get written.**

A **claim** is a statement in the ticket's summary, description, acceptance criteria, comments or
attachments saying what the software does wrong, or what it should do instead. Claims are quoted
from the ticket — never inferred from what you imagine the feature should also do.

**One claim → one case. Same count, no more.** Most bug tickets carry 1–3 claims. A case count
above 4 is a strong signal the law was broken: recount the claims.

## Procedure

### 1. Fetch the ticket

Use `kabot-jira`. Read the description **and every comment** — the acceptance bar is often in a
comment, and the newest statement on a point wins. If the fetch fails, say so and stop.

### 2. Write the claims down first, before any step

```
Claims found:
  C1 — "the list still shows deleted items after a refresh"
  C2 — "the total should exclude cancelled rows"  (from comment, dev, 12 Aug)
```

Quote them. Mark where each came from (description / comment / attachment / acceptance criteria)
and note when a comment narrows or overrides the description.

If the ticket is too vague to yield a single testable claim ("the settings screen needs
improvement"), **do not invent claims to fill the gap.** Say what is unclear and ask the one
question that would unblock it.

### 3. Resolve the navigation path — do not guess it

You need the exact route to the thing under test and the exact field and button labels, in the
product's own words. Get them from, in order:

1. Existing test code in the repo — a page object or an older test usually already documents the
   route and the verbatim labels. Reuse those strings.
2. Project documentation: `docs/`, a feature reference, a use-case file, the repo `README`.
3. Other skills available in this session that describe the product's domain.
4. The ticket's own screenshots and steps-to-reproduce.

If nothing documents the path, write it as **unverified** and say so. Never invent a menu name,
tab name or button label — a QA engineer following a fabricated path loses more time than one
following an honest gap.

### 4. Write the cases

One case per claim, in the order the claims appear:

- **The shortest sequence of steps that reaches the claim's assertion point.** Setup collapses into
  preconditions; it is not spelled out as numbered steps.
- **Exactly one expected result**, phrased as the claim's *fixed* behaviour.
- Runnable by someone who has never read the ticket.
- Concrete values, not placeholders — a real-looking name, a real-looking amount.

### 5. The exclusion gate — apply it before output

Delete any case that fails **any** row:

| Gate | Question | Drop if |
|---|---|---|
| Traceability | Which claim (C1/C2/…) does this prove? | no claim fits |
| Detection | If this case passed while the defect was still present, would we notice? | no — it cannot detect the bug |
| Sourcing | Does the expected result come from the ticket, or from my idea of good behaviour? | it came from me |
| Duplication | Does another case already assert this? | yes |

### 6. Output format

```markdown
## <KEY> — <summary>
Status: <status> · Type: <type> · Platform: <web | android | ios | unclear>
Verifying: a fix (dev comment present) | a new feature | unclear

### Claims
C1 — "<quoted>"  (description)
C2 — "<quoted>"  (comment — <who>, <when>)

### Preconditions
- <account / role / data / build / feature flag needed>

### Case 1 — <short title>   [proves C1]
1. <step>
2. <step>
**Expected:** <single expected result>

### Case 2 — <short title>   [proves C2]
…

### Not covered here, deliberately
- <anything a reader might expect and why it is out of scope — e.g. "C2 mentions an export;
  the ticket makes no claim about its contents">

### Unverified
- <any navigation path or label that no source confirmed>
```

### 7. Save and offer to post

Write the block to `.kabot/runs/<KEY>/steps.md`.

Then offer — in one line — to post it to the ticket as a comment, and post it only on an explicit
yes, via `kabot-jira`. Print the body first. Do not post silently, and do not ask twice.

If the ticket already has a kabot steps comment, say so and ask whether to post an updated one
rather than adding a near-duplicate.

## Handing off

If the user's original request was really "automate this", the steps are only the first stage:
say the steps are ready and continue into `kabot-automatable`. Do not stop and wait unless the
request was for steps alone.
