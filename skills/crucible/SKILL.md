---
name: crucible
description: "Adversarial council for high-stakes decisions and implementation plans. Five advisors with different lenses work from one shared evidence pack, cross-examine each other's findings anonymously, then defend, refine or concede. An independent judge rules on what survived, and the result becomes either a verdict (decide mode) or a gated, executable plan written by a separate planner (plan mode). Triggers: 'crucible', 'crucible this', 'hyperplan', 'adversarial plan', 'red-team this plan', 'tear this plan apart', 'cross-examine this'. Use for decisions or plans where being wrong is expensive and the user wants weak assumptions exposed. Do NOT use for factual lookups, small or easily reversible changes, or creation tasks (write me X). For a lighter multi-perspective take without the defense round, llm-council is enough."
---

# Crucible

One model gives you one answer and no way to tell how good it is. Crucible runs the question through five advisors, makes them attack each other's findings, lets each one defend or concede, and has a judge who took no side decide what survived. Only survivors reach the output.

It takes the parallel independent pass and anonymous peer review from LLM Council, and the defend/refine/concede round, open questions as user gates, and the planner handoff from Hyperplan. It adds two things neither has: a shared evidence pack so nobody argues from vibes, and a judge so a finding's author doesn't get to decide whether it survives.

## Step 0: is this worth it?

A run costs about 17 subagent calls. Before anything else, check:

- Is there a real decision or plan with stakes, more than one plausible path, and uncertainty?
- Would being wrong cost more than a few minutes to undo?

If not, say so in one line and answer or plan normally. If yes, tell the user in one line that you're starting a crucible run and which mode it is.

**Pick the mode:**

- **decide**: a choice between options, a strategy question, "should I". Output is a verdict.
- **plan**: "how should we build or change X". Output is an executable plan with gates.

If it's unclear, ask one question. Only one.

## Step 1: build the evidence pack

Everyone argues from the same facts, or the debate is just competing guesses.

Gather the context yourself, briefly:

- Files the user named or attached.
- `CLAUDE.md`, project memory, and anything that states constraints, numbers or past decisions.
- In plan mode: the relevant code. Use an Explore subagent if it spans more than a handful of files. Note the actual entry points, data flow, existing tests and conventions.
- Docs for any library or service the decision hinges on, if it's cheap to check.

Write it as numbered entries with a source on each:

```
E1. Auth middleware runs before rate limiting (src/server.ts:42)
E2. No test covers the token refresh path (searched tests/, none found)
E3. User wants this shipped before the Dec 1 launch (user message)
U1. Unknown: expected request volume. Nobody stated it.
```

`E` entries are verified facts. `U` entries are known unknowns. Never put a guess in an `E` entry. Keep the pack under about 60 lines; it goes to every agent in every round.

## Step 2: frame the question

Write one neutral prompt: the core question, the mode, the stakes, and any hard constraints from the user. Don't add your opinion and don't steer. This framed question plus the evidence pack is the shared brief.

## Step 3: round 1, independent findings (5 in parallel)

Spawn all five advisors in one message so none sees another's output. The fifth seat depends on the mode.

| Seat | Lens | Looks for |
|------|------|-----------|
| Contrarian | What fails | Fatal flaws, failure modes, edge cases, blast radius, what breaks downstream |
| Simplifier | What can go | Over-engineering, scope creep, premature abstraction. Asks "what's the smallest thing that works?" |
| Reframer | Is this the right question | First principles, the underlying need versus the literal request, at least two real alternatives to the obvious path |
| Executor | Can it actually be done | Sequencing, first step, effort, dependencies, what happens Monday morning |
| Outsider (decide) | Fresh eyes | Curse of knowledge, jargon, assumptions that are obvious only to insiders |
| Architect (plan) | Structure | Hidden coupling, leaky interfaces, debt the plan creates. Rejects patterns that don't pay for themselves |

**Advisor prompt:**

```
You are the [Seat] on a crucible council. Your lens: [lens and "looks for" from the table].

Brief:
---
[framed question]

Evidence pack:
[E and U entries]
---

Produce 3 to 6 numbered findings from your lens. For each:
- The claim, in at most 3 sentences.
- Its support: E-numbers from the pack, a file:line or doc you checked yourself, or the word "unverified".

You may read files to check a claim. Don't invent citations: "unverified" is an acceptable answer and an invented source is not.

Lean fully into your lens. The other seats cover what you don't. No preamble, no summary, no proposed final plan.
```

Record each advisor's agent ID. You'll continue the same agents in rounds 2 and 3.

## Step 4: round 2, anonymous cross-examination (5 in parallel)

Tag every finding with a random, meaningless label (`F01` to `F30`, shuffled so labels don't group by seat). Keep the mapping to yourself.

Send each advisor the full anonymized list, telling it which labels are its own. Continue the existing agent with SendMessage so it keeps its round 1 reasoning. If continuation isn't available, spawn a fresh agent and paste its own round 1 output in.

**Cross-examination prompt:**

```
Here are all findings from the council, anonymized. Yours are [labels]; skip them.

[anonymized findings with their support lines]

For every other finding, give one verdict:
- STANDS: you have nothing substantive against it. This is a fine answer and costs nothing.
- CHALLENGE: give the specific reason it's wrong, overstated or incomplete, in at most 3 sentences, with your own support (E-number, file:line, doc, or reasoning).

Only challenge what you can back up. A challenge without a reason will be thrown out.

Then answer one more question: what did every finding miss? At most 2 items, each with support.
```

Anonymity matters here. If reviewers know the Contrarian wrote a finding, they argue with the role instead of the claim.

## Step 5: round 3, defend, refine or concede (5 in parallel)

Group challenges by the finding they target. Send each advisor only the challenges against its own findings, again continuing the same agent.

**Defense prompt:**

```
These challenges target your findings:

[F07] your claim: ...
  - challenge: ...
  - challenge: ...

For each challenged finding, choose one:
- DEFEND: rebut with concrete support. Restating your claim louder is not a defense.
- REFINE: the challenge landed partly. State the narrower or corrected claim.
- CONCEDE: the challenge was right. Say what, if anything, survives.

Conceding a wrong finding counts in your favor. Defending one does not.
Format: "[F07] DEFEND/REFINE/CONCEDE: explanation (at most 3 sentences)"
```

Findings nobody challenged skip this round.

## Step 6: the judge (1 fresh agent)

The judge is a new agent that wrote none of the findings. Don't do this step yourself: you framed the question and assembled the bundles, so you're not neutral either.

Give it the brief, every finding with its support, all challenges, all defenses, and the "what everyone missed" items. Still anonymized.

**Judge prompt:**

```
You are the judge of a crucible council. You wrote none of these findings and have no side.

[brief, findings, challenges, defenses, missed items]

Rule on every finding:
- SURVIVES: unchallenged and supported, or challenged and successfully defended.
- SURVIVES-REFINED: survives in the narrowed form from its REFINE answer. Quote that form.
- ASSUMPTION: never challenged, or defended, but rests on "unverified" support. It survives only as a labeled assumption.
- DIES: conceded, or defended without real support against a substantiated challenge.
- UNRESOLVED: both sides have real support and the evidence doesn't settle it.

Spot-check citations. For every finding whose ruling depends on a file:line or doc, open it and confirm it says what's claimed. A citation that doesn't hold up kills the finding it supports.

Rule on the "missed" items the same way.

Output one line per finding: "[F07] RULING: reason (one sentence)". Then list any citation you found to be false.
```

Then de-anonymize the rulings with your mapping.

## Step 7a: decide mode, verdict

Write the verdict yourself from the judge's rulings. Use only findings that survived. Assumptions appear only labeled as assumptions.

```
## Where the council agrees
Survivors that several seats reached independently. Highest confidence.

## Where it clashed
UNRESOLVED findings. Both sides, and what evidence would settle it.

## What only came out under attack
Survivors from the "what everyone missed" items and refined findings that changed shape.

## Assumptions this rests on
Every ASSUMPTION ruling, so the user knows what wasn't verified.

## Recommendation
A real answer, not "it depends". You may side with a minority if its finding survived and the majority's didn't.

## First move
One concrete next step.
```

## Step 7b: plan mode, handoff to a planner

You don't write the plan. Build the insight bundle and hand it to a Plan subagent. If you notice yourself drafting tasks first, stop: a draft anchors the planner.

```
# Crucible insight bundle: [title]

## Request
[user's request, verbatim]

## Evidence pack
[E and U entries]

## Hard constraints (survived)
- [constraint] (seat, ruling)

## Decisions (survived)
- [decision]: who proposed it, what challenged it, how it held

## Risks and mitigations
- [risk]: [mitigation], (seat)

## Assumptions (unverified, not challenged)
- [assumption]

## Open questions (UNRESOLVED)
- [question]: both positions, what would settle it

## Killed
- [finding]: why it died, in one line. The planner must not reintroduce these.
```

**Planner prompt:**

```
Turn this insight bundle into an executable work plan.

Rules:
- Respect every hard constraint.
- Put each mitigation into the task it protects.
- Each open question becomes a user gate placed before the tasks that depend on it.
- Each assumption gets a verification step before anything is built on it. Don't treat it as fact.
- Don't reintroduce killed findings.
- Every task has explicit success criteria and a way to check them.
- Mark which tasks can run in parallel.

You may read code to sequence the work, but don't reopen settled decisions unless you find evidence that contradicts one. If you do, report it at the top.

If you need answers from the user before you can plan, return only the questions.

[bundle]
```

Present the planner's output as is, with one provenance line on top:

```
*Plan from a crucible run ([N] advisors, 3 rounds, independent judge), written by a separate planner.*
```

`[N]` is the number of advisors that actually finished. If the planner returned questions, pass them to the user without changes.

## Step 8: save the record

Write the full run to `crucible-runs/[YYYY-MM-DD]-[slug].md` in the current working directory: the brief, evidence pack, round 1 findings, the anonymization mapping, challenges, defenses, judge rulings, and the final output. Tell the user the path in one line.

That record lets a later run see how the thinking changed, and lets the user check any ruling. Only build an HTML report if the user asks for one.

## When things go wrong

- **An advisor fails or returns nothing:** continue with the rest, tell the user which seat dropped, and use the real count everywhere (provenance line, record). Below three advisors, stop and say the run isn't trustworthy.
- **An advisor drifts** (writes a plan, ignores the format, invents citations): send one correction. If it drifts again, drop its findings and say so.
- **Every finding stands and nothing gets challenged:** that's a sign the question was too easy or the seats too polite. Say so in the verdict; don't manufacture conflict.
- **Bundles get too large:** cut findings to their claim and support line. Never summarize away a challenge or a concession.

## Don't

- Don't skip round 3 or the judge to save time. Those two steps are the difference between this and asking five times.
- Don't tell advisors to reject by default. Calibrated challenges beat performed hostility.
- Don't let the lead rule on findings. That's the judge's job.
- Don't promote an unverified claim to fact anywhere downstream, including the plan.
- Don't hardcode models. Seats are lenses, not model assignments.
