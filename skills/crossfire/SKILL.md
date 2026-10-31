---
name: crossfire
description: "Adversarial council for high-stakes decisions and implementation plans. Five advisors use different lenses and work from one shared evidence pack. They cross-examine the findings of the other advisors anonymously. Then each advisor defends, refines or concedes its own findings. An independent judge decides which findings survive. The result is a verdict (decide mode) or a gated, executable plan that a separate planner writes (plan mode). Triggers: 'crossfire', 'crossfire this', 'hyperplan', 'adversarial plan', 'red-team this plan', 'tear this plan apart', 'cross-examine this'. Use it for decisions or plans where a wrong answer is expensive and the user wants to find weak assumptions. Do NOT use it for factual lookups, small or easily reversible changes, or creation tasks (write me X). For a lighter multi-perspective review without the defense round, use llm-council."
---

# Crossfire

One model gives one answer, and you cannot tell how good that answer is. Crossfire sends the question to five advisors. Each advisor attacks the findings of the other advisors. Each advisor then defends or concedes its own findings. A judge who took no side decides which findings survive. Only the surviving findings go into the output.

Crossfire uses three parts of LLM Council: the parallel independent pass, the anonymous peer review, and the final verdict. It uses three parts of Hyperplan: the defend, refine or concede round, the open questions that become user gates, and the handoff to a planner. It adds two parts that neither skill has:

- A shared evidence pack, so that all advisors argue from the same facts.
- An independent judge, so that the author of a finding does not decide if that finding survives.

In this skill, "you" is the lead agent that runs the session. A "seat" is the role of one advisor.

## Step 0: Decide if the question needs a run

A run uses approximately 17 subagent calls. Before you start, answer these two questions:

- Does the question have real stakes, more than one possible path, and real uncertainty?
- Does a wrong answer cost more than a few minutes to undo?

If the answer to either question is no, tell the user in one line. Then answer or plan as usual. If both answers are yes, tell the user in one line that you start a crossfire run, and name the mode.

**Select the mode:**

- **decide**: a choice between options, a strategy question, or a "should I" question. The output is a verdict.
- **plan**: a question about how to build or change something. The output is an executable plan with gates.

If the mode is not clear, ask the user one question. Do not ask more than one.

## Step 1: Make the evidence pack

All advisors must argue from the same facts. If they do not, the debate is only a set of guesses.

Collect the context yourself. Keep this step short.

- Read the files that the user named or attached.
- Read `CLAUDE.md`, the project memory, and any file that gives constraints, numbers or past decisions.
- In plan mode, read the relevant code. If the code is in more than five files, use an Explore subagent. Record the entry points, the data flow, the existing tests and the conventions.
- If the decision depends on a library or a service, read its documentation. Do this only if it is fast.

Write the evidence pack as a numbered list. Give a source for each entry:

```
E1. Auth middleware runs before rate limiting (src/server.ts:42)
E2. No test covers the token refresh path (searched tests/, none found)
E3. User wants this shipped before the Dec 1 launch (user message)
U1. Unknown: expected request volume. Nobody stated it.
```

An `E` entry is a fact that you verified. A `U` entry is a known unknown. Do not put a guess in an `E` entry. Keep the pack to approximately 60 lines or fewer, because every agent gets the pack in every round.

## Step 2: Write the brief

Write one neutral prompt. The prompt contains the core question, the mode, the stakes and the hard constraints from the user. Do not add your opinion. Do not push the advisors toward an answer. This prompt and the evidence pack together are the brief.

## Step 3: Round 1, independent findings (5 agents in parallel)

Start all five advisors in one message, so that no advisor sees the output of another advisor. The fifth seat changes with the mode.

| Seat | Lens | Looks for |
|------|------|-----------|
| Contrarian | What fails | Fatal flaws, failure modes, edge cases, blast radius, downstream breakage |
| Simplifier | What can go | Over-engineering, scope creep, premature abstraction. Asks "what is the smallest thing that works?" |
| Reframer | Is this the right question | First principles, the real need behind the literal request, at least two real alternatives to the obvious path |
| Executor | Can it be done | Sequence, first step, effort, dependencies, what happens on Monday morning |
| Outsider (decide) | Fresh eyes | Curse of knowledge, jargon, assumptions that only insiders think are obvious |
| Architect (plan) | Structure | Hidden coupling, leaky interfaces, debt that the plan creates. Rejects patterns that do not pay for themselves |

**Advisor prompt:**

```
You are the [Seat] on a crossfire council. Your lens: [lens and "looks for" from the table].

Brief:
---
[brief]

Evidence pack:
[E and U entries]
---

Write 3 to 6 numbered findings from your lens. For each finding, give:
- The claim, in 3 sentences or fewer.
- The support: E-numbers from the pack, a file:line or a document that you checked, or the word "unverified".

You can read files to check a claim. Do not invent a citation. "Unverified" is an acceptable answer. An invented source is not acceptable.

Use only your lens. The other seats cover the other lenses. Do not write a preamble, a summary or a plan.
```

Record the agent ID of each advisor. In rounds 2 and 3, you continue the same agents.

## Step 4: Round 2, anonymous cross-examination (5 agents in parallel)

Give each finding a random label (`F01` to `F30`). Shuffle the labels, so that the labels of one seat are not in sequence. Keep the map from labels to seats. Do not show the map to the advisors.

Send the full anonymous list to each advisor. Tell each advisor which labels are its own. Use SendMessage to continue the existing agent, so that the agent keeps its round 1 reasoning. If you cannot continue the agent, start a new agent and paste its round 1 output into the prompt.

**Cross-examination prompt:**

```
Here are all the findings from the council. The authors are hidden. Your findings are [labels]. Do not review them.

[anonymous findings with their support lines]

Give one verdict for each of the other findings:
- STANDS: you have no real argument against it. This is a good answer and it has no cost.
- CHALLENGE: give the specific reason that the finding is wrong, too strong or incomplete. Use 3 sentences or fewer. Give your own support (E-number, file:line, document or reasoning).

Challenge a finding only if you have support. The judge rejects a challenge that has no reason.

Then answer one more question: what did all the findings miss? Give 2 items or fewer, each with support.
```

The labels must stay anonymous. If an advisor knows that the Contrarian wrote a finding, the advisor argues against the role, not against the claim.

## Step 5: Round 3, defend, refine or concede (5 agents in parallel)

Sort the challenges by the finding that each challenge attacks. Send each advisor only the challenges against its own findings. Continue the same agents again.

**Defense prompt:**

```
These challenges attack your findings:

[F07] your claim: ...
  - challenge: ...
  - challenge: ...

For each finding that has a challenge, select one answer:
- DEFEND: reply with real support. If you only repeat your claim, that is not a defense.
- REFINE: the challenge is partly correct. Give the narrower or corrected claim.
- CONCEDE: the challenge is correct. Say which part of the finding, if any, survives.

If your finding is wrong, concede it. A concession counts for you. A defense of a wrong finding does not.
Format: "[F07] DEFEND/REFINE/CONCEDE: explanation (3 sentences or fewer)"
```

Findings that have no challenge do not go into this round.

## Step 6: The judge (1 new agent)

The judge is a new agent that wrote none of the findings. Do not do this step yourself. You wrote the brief and you assembled the bundles, so you are not neutral.

Give the judge the brief, all findings with their support, all challenges, all defenses and the "what all the findings missed" items. Keep the authors hidden.

**Judge prompt:**

```
You are the judge of a crossfire council. You wrote none of these findings. You have no side.

[brief, findings, challenges, defenses, missed items]

Give a ruling for each finding:
- SURVIVES: the finding has support and no challenge, or the author defended it successfully.
- SURVIVES-REFINED: the finding survives in the narrower form from its REFINE answer. Quote that form.
- ASSUMPTION: the finding has no challenge or has a defense, but its support is "unverified". It survives only as a labeled assumption.
- DIES: the author conceded it, or a challenge with support defeated a defense without real support.
- UNRESOLVED: both sides have real support, and the evidence does not decide between them.

Check the citations. If a ruling depends on a file:line or a document, open it. Make sure that it says what the finding claims. If a citation is false, the finding that depends on it dies.

Give a ruling for each "missed" item with the same rules.

Write one line for each finding: "[F07] RULING: reason (one sentence)". Then list each citation that you found to be false.
```

Then use your map to replace the labels with the seats.

## Step 7a: Decide mode, the verdict

Write the verdict yourself from the rulings of the judge. Use only the findings that survived. Show an assumption only with its label.

```
## Where the council agrees
Surviving findings that two or more seats found independently. These have the highest confidence.

## Where it clashed
UNRESOLVED findings. Give both sides, and the evidence that can decide between them.

## What appeared only under attack
Surviving "missed" items, and refined findings that changed shape.

## Assumptions this rests on
Every ASSUMPTION ruling, so that the user knows what nobody verified.

## Recommendation
A real answer. "It depends" is not a real answer. You can agree with a minority if its finding survived and the majority finding did not.

## First move
One specific next step.
```

## Step 7b: Plan mode, the handoff to a planner

Do not write the plan yourself. Make the insight bundle and give it to a Plan subagent. If you start to write tasks before the handoff, stop. A draft from you pushes the planner toward your draft.

```
# Crossfire insight bundle: [title]

## Request
[the request of the user, word for word]

## Evidence pack
[E and U entries]

## Hard constraints (survived)
- [constraint] (seat, ruling)

## Decisions (survived)
- [decision]: who proposed it, what challenged it, how it survived

## Risks and mitigations
- [risk]: [mitigation], (seat)

## Assumptions (unverified, no challenge)
- [assumption]

## Open questions (UNRESOLVED)
- [question]: both positions, and the evidence that can decide between them

## Killed
- [finding]: why it died, in one line. The planner must not add these findings again.
```

**Planner prompt:**

```
Make an executable work plan from this insight bundle.

Rules:
- Obey every hard constraint.
- Put each mitigation into the task that it protects.
- Make each open question a user gate. Put the gate before the tasks that depend on it.
- Give each assumption a verification step before any task uses it. Do not treat an assumption as a fact.
- Do not add killed findings again.
- Give every task clear success criteria and a method to check them.
- Mark the tasks that can run in parallel.

You can read code to put the work in sequence. Do not open a settled decision again unless you find evidence against it. If you find such evidence, report it at the top.

If you need answers from the user before you can make the plan, return only the questions.

[bundle]
```

Show the output of the planner without changes. Put one provenance line on top:

```
*Plan from a crossfire run ([N] advisors, 3 rounds, independent judge), written by a separate planner.*
```

`[N]` is the number of advisors that completed the run. If the planner returned questions, give them to the user without changes.

## Step 8: Save the record

Write the full run to `crossfire-runs/[YYYY-MM-DD]-[slug].md` in the current working directory. The record contains the brief, the evidence pack, the round 1 findings, the label map, the challenges, the defenses, the rulings of the judge and the final output. Tell the user the path in one line.

With the record, a later run can see how the reasoning changed, and the user can check each ruling. Make an HTML report only if the user asks for one.

## Problems during a run

- **An advisor fails or returns nothing:** Continue with the other advisors. Tell the user which seat failed. Use the real number of advisors everywhere (the provenance line and the record). If fewer than three advisors remain, stop. Tell the user that the run is not reliable.
- **An advisor goes off task** (it writes a plan, ignores the format or invents citations): Send one correction. If it goes off task again, remove its findings and tell the user.
- **No advisor challenges any finding:** The question was too easy, or the advisors were too polite. Say this in the verdict. Do not invent a conflict.
- **The bundles become too large:** Cut each finding to its claim and its support line. Do not summarize a challenge or a concession.

## Prohibited actions

- Do not skip round 3 or the judge to save time. These two steps are the difference between this skill and five separate questions.
- Do not tell the advisors to reject by default. A challenge with support is better than a show of hostility.
- Do not let the lead give rulings on findings. Only the judge gives rulings.
- Do not change an unverified claim into a fact at a later step. This includes the plan.
- Do not hardcode models. A seat is a lens, not a model assignment.
