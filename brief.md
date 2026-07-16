# Greenlight — product brief

*Draft 1 · July 2026*

## The problem, concretely

At any org that's serious about Terraform, infra changes move through a
funnel that looks like this: an engineer opens a PR, CI runs a plan plus a
stack of policy checks (OPA, Checkov, tflint, some homegrown scripts), and
then the PR waits for a human from the platform or security team. Three
things go wrong with that funnel, and they compound:

**The checks cry wolf.** Policy scanners flag everything at the same
volume — a public S3 bucket and a missing `cost-center` tag look identical
in a wall of CI output. Engineers learn to scroll past. I've watched teams
where the policy job had a *documented* "just re-run it and ping someone
for an override" workflow. At that point the checks are theater.

**The humans are the bottleneck.** Because the checks can't be trusted to
say "this one's fine," a person reviews everything. Most of what they
review is routine — a new SQS queue, a replica count bump, an IAM policy
copied from the last service. The reviewer skims, approves, and the real
judgment calls get the same thirty seconds as the trivial ones. Review
latency is measured in days; reviewer attention is spent evenly across
changes that deserve wildly uneven attention.

**The policy isn't anywhere.** What's actually allowed lives in three
places at once: partially in rego files, partially in a wiki page from
2024, and mostly in the heads of two staff engineers. When a check blocks
you, nothing explains *why* or what the compliant version looks like — so
you DM one of those two people, and now they're the bottleneck twice.

The cost is a slow, grumpy infra change process where the controls that
exist are simultaneously annoying and not actually protective. That's the
worst quadrant.

## Where I'd start

Greenlight is an agent that reviews the **plan diff on every infra PR**
against policies written in plain language, and sorts each change into one
of three lanes:

- **Green — cleared.** The change matches known-routine patterns and
  violates nothing. Greenlight posts its reasoning and the specific
  policies it checked, with citations. A human still clicks approve (more
  on that below), but they're approving a verdict, not re-deriving one.
- **Red — blocked.** The change trips a hard rule — public data exposure,
  wildcard IAM on production, deleting a stateful resource without a
  migration note. The block comes with a plain-English explanation of the
  policy, *why it exists*, and a suggested compliant diff. A block that
  doesn't show the way out is just friction.
- **Yellow — needs a human.** The change is neither routine nor forbidden:
  a security group that's broad but maybe justified, a cost jump, a new
  pattern nobody's written a rule for. Greenlight routes it to the right
  reviewer with a prepared packet: what changed, why it couldn't decide,
  and how similar past changes were decided. The reviewer's decision gets
  recorded — and repeated identical decisions become candidate policies.

The whole idea rests on one observation: **most infra PRs don't need a
human's judgment, and the ones that do currently get too little of it.**
Greenlight's job is to make that split explicit, with evidence, so reviewer
attention lands where it matters.

The product bet: **if Greenlight can correctly clear the routine majority
of infra PRs and its cleared verdicts survive audit, review latency drops
from days to hours and the platform team gets their calendar back —
without loosening a single actual control.** If the audits find false
greens, the bet is off, and I mean that literally (see risks).

## Who it's for

**Infra change authors** — every product engineer who touches Terraform.
They want to know three things in one minute: can this merge, if not why
not, and what do I change. Today they get a wall of CI noise and a wait.

**Reviewers** — platform and security engineers who currently approve
everything by default. This is the group with veto power over the tool.
Greenlight only works if they believe a green verdict means something —
which is why they audit it (metrics section) and why they, not the agent,
decide what counts as routine.

**Whoever owns compliance** gets a byproduct they currently can't have:
a queryable log of every infra change, the policy verdicts on it, and who
approved what — instead of archaeology through PR comments at audit time.
Useful, but a byproduct. I wouldn't let audit-trail requirements drive v1
design.

## What v1 is

### 1. The review comment

One structured comment per PR, posted when the plan lands: the lane, the
reasoning in plain language, the policies checked (each linked to its
written text), and — for red — the suggested fix as an actual diff you
can apply. Not a dashboard you have to go visit — the decision happens on
the PR, so the verdict shows up there.

### 2. Policies as documents

Each policy is a short written page: the rule, the reason it exists, the
lane it maps to (hard-block vs. needs-judgment), examples of compliant and
non-compliant changes, and its enforcement stats (how often it fires, how
often it's overridden). The agent enforces *from these documents* — which
means the policy set is readable by the people subject to it, and writing
a new policy is writing a paragraph, not learning rego. The existing rego
and Checkov rules don't get thrown away; they become inputs the agent
cites, with the document as the source of truth for intent.

### 3. The judgment queue

Where yellow-lane changes land, routed to the right reviewer with the
packet attached. Every decision made here is recorded with its rationale.
When the same shape of decision shows up three times, Greenlight drafts a
policy document from those precedents and proposes it to the platform
team. Over time this is how the policy set grows — out of decisions that
actually happened, rather than rules written top-down in advance.

### 4. The exception path

Sometimes a red is legitimately overridable — a break-glass fix, a
migration that needs a temporary wildcard. Requesting an exception is
one click, pre-filled with context, requiring a named approver and an
**expiry date**. Expired exceptions reopen as issues on the owning team.
No expiry, no exception.

## What v1 does not do

- **No merge rights.** Greenlight clears, blocks, and routes; a human's
  approval is still the thing GitHub requires, even on green. Removing
  the human click is a cheap follow-up *after* the audit numbers earn it —
  and an unrecoverable trust loss if done before.
- **No runtime scanning.** Drift detection, live cloud posture, CSPM —
  real problems, different product. V1 governs *changes*, at the PR, where
  the fix is a code review comment instead of a production incident.

## Day-one metrics

Instrumented before the pilot, with a baseline from historical PRs
(backlog issue #1).

**The headline numbers**

- **Median time from PR open to merge for infra changes**, against the
  historical baseline. This is the number that pays for the product.
- **Lane distribution** — % of PRs green / yellow / red. The bet assumes
  green is the majority; if it's 30%, the premise is wrong and we should
  know fast.

**Is it trustworthy** (these gate everything else)

- **False-green rate.** Weekly audit: platform reviewers re-review a
  random sample of ~15 auto-cleared PRs, blind to the verdict. Target is
  zero findings that a careful human would have blocked or escalated.
  This is the metric with a kill switch attached — a confirmed false
  green on a security-relevant change pauses the green lane, full stop.
- **Escalation precision** — of yellow-lane PRs, what fraction did the
  human reviewer agree actually needed them? Too low means Greenlight is
  hedging and just re-creating the old review-everything world with extra
  steps.
- **Override rate on reds**, per policy. A policy overridden more than
  it's obeyed is a bad policy, and the stats page should shame it into
  revision.

**Are people better off**

- Reviewer hours spent on infra review per week (self-reported, imperfect,
  still worth tracking against the baseline interviews).
- **Fix acceptance rate** — % of red verdicts where the author applies the
  suggested compliant diff rather than arguing or abandoning. This is the
  best proxy for "the blocks are helpful, not hostile."
- Author sentiment, crudely: a one-click "this verdict was fair / unfair"
  on every comment. Unfair-flags on greens and reds get read weekly.

**Is the system learning**

- Count of policies codified from yellow-lane precedents (vs. written
  top-down). The ratio tells you whether the judgment-queue loop works.
- Yellow-lane repeat rate — the same shape of change escalating over and
  over means a policy is missing and the loop isn't closing.
- Open exceptions past expiry. Target: zero, enforced by reopened issues.

## Risks, and what I'd do about them

**A false green.** The failure that kills the product: if Greenlight
clears a PR that opens a data exposure, nobody will trust a green verdict
again, and they'd be right not to. Mitigations:
the green lane starts narrow (an explicit allowlist of routine change
shapes, expanded deliberately, not a general "looks fine to me"); hard-rule
domains — IAM, public exposure, data deletion — can never be
green-by-inference, only by exact-match to a vetted pattern; the weekly
blind audit; and the standing rule that one confirmed security-relevant
false green pauses the lane while we figure out why. The brief promises
that in writing so nobody has to argue for it during the incident.

**Rubber-stamping the green lane.** Humans approve what a machine
pre-approved without looking — automation complacency is thoroughly
documented. Partial answer: the verdict comment is designed to be
*checkable in ten seconds* (here's what changed, here's the pattern it
matched), and the audit measures the agent, not the approver, so the
system doesn't depend on every approver staying vigilant forever.

**Engineers gaming the agent.** Splitting a risky change across three
innocuous PRs, or wording things to slip past. Mitigations: verdicts are
computed from the *plan diff*, not the PR description, which closes the
cheap version; and cross-PR patterns on the same resources within a window
get flagged to the yellow lane. This won't catch a determined insider,
but neither does today's skimming human.

**The policy documents drift from the enforcement.** If the written page
says one thing and the agent enforces another, the whole "policies you can
read" pitch collapses. Every verdict links to the exact policy version it
applied, and policy edits re-run against a regression set of past PRs so
you can see what a wording change *does* before merging it. Policy edits
go through review like any other code change.

**The pilot org has unusually clean Terraform.** Same sampling bias as any
pilot — whoever volunteers is whoever's proud of their repo. Flag it in
the readout, and pick a messier org for the second round.

## Rollout shape

One platform team's review domain, two to four Terraform repos with real
change volume (~40+ infra PRs a month between them), eight weeks. Phase
zero is silent: Greenlight runs on every PR but posts nothing, and its
verdicts are compared against what human reviewers actually did — that's
the eval that decides whether it goes visible at all (backlog #7–8). Then
comment-only (verdicts visible, changing nothing about required
approvals), then the green lane, then — only if the audits are clean — a
conversation about reduced human approval on green. Each gate has a number
attached, and the last issue in the backlog is a written go/narrow/kill
memo, same as every pilot I'd run.

## Open questions I'd want to argue about

- Where does the green-lane allowlist start? My lean: the ten most common
  change shapes from the historical analysis, and nothing security-touching
  in the first cut, even if that keeps green under 50% for a while.
- Do yellow-lane routings go to the existing review rotation or to a named
  domain owner (networking changes → networking person)? Named owners give
  better decisions and better precedents; rotations spread load. Probably
  named owners for the pilot, revisit at scale.
- One agent verdict or several? A security-lens pass and a cost-lens pass
  disagree sometimes, and showing that disagreement in the yellow packet
  might be more honest than forcing a single voice. Undecided; prototype
  shows a single voice for simplicity.
- What model of "policy version pinning" do exceptions use — pinned to the
  policy as-written at grant time, or floating? Pinned, probably, but it
  makes expiry cleanup subtler.
