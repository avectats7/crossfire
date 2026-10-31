# Crucible

A Claude Code skill that puts a decision or a plan through an adversarial council and keeps only what survives.

Ask one model and you get one answer, with no way to tell how good it is. Crucible runs the question through five advisors, makes them attack each other's findings, lets each one defend or concede, and has a judge with no stake in the outcome decide what's left standing.

## How a run works

1. **Worth it?** A run is about 17 subagent calls. Small or easily reversible questions get a normal answer instead.
2. **Evidence pack.** The lead gathers the facts once, each with a source (`E1. Auth runs before rate limiting (src/server.ts:42)`), and lists known unknowns separately. Every advisor argues from the same pack.
3. **Round 1.** Five advisors work in parallel: Contrarian, Simplifier, Reframer, Executor, plus Outsider (decide mode) or Architect (plan mode). Each finding cites its support or says "unverified".
4. **Round 2.** Findings get shuffled anonymous labels. Each advisor marks the others' findings STANDS or CHALLENGE, and a challenge needs a reason.
5. **Round 3.** Each advisor answers the challenges against its own findings: DEFEND, REFINE or CONCEDE.
6. **Judge.** A fresh agent rules on every finding and opens the cited files to check them. A false citation kills its finding. Unchallenged claims with no evidence survive only as labeled assumptions.
7. **Output.** Decide mode gives a verdict: agreement, clashes, assumptions, a recommendation and a first move. Plan mode hands the survivors to a separate planner, which turns assumptions into verification steps and open questions into gates you answer before work starts.
8. **Record.** The whole run is saved to `crucible-runs/YYYY-MM-DD-slug.md`.

## Install

As a plugin:

```
/plugin marketplace add avectats7/crucible
/plugin install crucible@crucible
```

Or copy the skill folder by hand:

```bash
git clone https://github.com/avectats7/crucible.git
cp -r crucible/skills/crucible ~/.claude/skills/
```

## Use

```
crucible this: should we move billing from Stripe Checkout to a custom flow before launch?
```

```
hyperplan: add offline sync to the savings app
```

Other triggers: `adversarial plan`, `red-team this plan`, `tear this plan apart`, `cross-examine this`.

## Where it comes from

Crucible combines two existing skills and fixes what each one lacked.

From [LLM Council](https://github.com/karpathy/llm-council) (Karpathy's method, adapted as a Claude skill): the parallel independent pass, anonymous peer review, and a verdict that commits to an answer.

From [Hyperplan](https://www.skillsdirectory.com/skills/code-yeongyu-hyperplan) (oh-my-opencode): the defend, refine or concede round, unresolved questions turned into user gates, and handing the plan to a separate planner.

What's new:

- **A shared evidence pack.** Hyperplan asked advisors for `file:line` citations without giving them any context, which invites invented sources.
- **An independent judge.** In Hyperplan a finding survived if its own author said DEFEND.
- **Calibrated challenges.** Advisors aren't told to reject by default. STANDS is a valid answer, and a challenge without a reason is thrown out.
- **Assumptions stay labeled** all the way into the plan, where each one gets a verification step.
- **No hardcoded models**, no team-mode dependency, and a failure path for advisors that drop out.

## License

MIT
