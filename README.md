# Greenlight

A small package showing how I'd approach AI-driven infrastructure-as-code
governance as a product engineer. There's no production code here on purpose — the deliverable is the product
thinking: where to start, what earns trust, what gets measured from day
one, and how a team would sequence the work.

## What's in here

| File | What it is |
|---|---|
| [brief.md](https://github.com/KumarappanA/greenlight/blob/main/brief.md) | The product brief. The problem, where I'd start, the three-lane review model, risks, and day-one metrics. |
| [Prototype](https://kumarappana.github.io/greenlight/prototype/) | A click-through HTML prototype, hosted live — no download needed. |
| [Issues](https://github.com/KumarappanA/greenlight/issues) | The backlog — 15 issues across 4 milestones, each with a why, what, and done-when. |

## The idea in one paragraph

Every infra change at a company with Terraform discipline goes through the
same funnel: open a PR, wait for a platform or security reviewer, get a
mix of real feedback and nitpicks, merge a day or three later. Meanwhile
the CI policy checks that were supposed to automate this are so noisy that
everyone treats a red X as background radiation. Greenlight is an agent
that reviews the *plan diff* on every infra PR against policies written in
plain language: it auto-clears the routine majority with cited evidence,
hard-blocks the few genuinely dangerous patterns with a suggested compliant
fix, and routes everything in between to a human reviewer with the context
already assembled. The goal is that human reviewers only ever see changes
that actually need judgment.

## How to review it

1. Read [brief.md](https://github.com/KumarappanA/greenlight/blob/main/brief.md)
   first — particularly the three-lane model and the "false green" part of
   the risks section, since everything else depends on that holding up.
2. Open the [prototype](https://kumarappana.github.io/greenlight/prototype/)
   and click through the review queue. Look at all three verdicts. The
   yellow one ("needs a human") is the screen I'd argue matters most — it
   shows what the agent does when it shouldn't decide on its own.
3. Skim the [issues](https://github.com/KumarappanA/greenlight/issues). The
   first is measurement, the agent is graded against historical PRs before
   it touches a live one, and the last is an explicit go/no-go decision.

## A few honest notes

- The Terraform snippets and policies in the prototype are invented but
  shaped like the real thing.
- The agent never gets merge rights in v1. It clears, blocks, or routes —
  a human's approval is still what GitHub requires. The brief explains why
  I'd hold that line even though auto-merge is the obvious ask.
- The backlog ends with a kill-switch issue on purpose. If the audit finds
  false greens, the auto-clear lane gets turned off — that's issue #11,
  wired in before the green lane ever goes live.
