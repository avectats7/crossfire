# Crossfire

Crossfire is a Claude Code skill. It sends a decision or a plan to an adversarial council and keeps only the findings that survive.

One model gives one answer, and you cannot tell how good that answer is. Crossfire sends the question to five advisors. Each advisor attacks the findings of the other advisors. Each advisor then defends or concedes its own findings. A judge with no stake in the result decides which findings survive.

## How a run works

1. **Check.** A run uses approximately 17 subagent calls. If a question is small or easy to undo, the skill answers it as usual and does not start a run.
2. **Evidence pack.** The lead agent collects the facts one time. Each fact has a source (`E1. Auth runs before rate limiting (src/server.ts:42)`). Known unknowns go in a separate list. All advisors argue from the same pack.
3. **Round 1.** Five advisors work in parallel: Contrarian, Simplifier, Reframer, Executor, and Outsider (decide mode) or Architect (plan mode). Each finding gives its support or the word "unverified".
4. **Round 2.** The findings get random anonymous labels. Each advisor marks the findings of the other advisors STANDS or CHALLENGE. A challenge must give a reason.
5. **Round 3.** Each advisor replies to the challenges against its own findings: DEFEND, REFINE or CONCEDE.
6. **Judge.** A new agent gives a ruling on each finding. The judge opens the cited files to check them. If a citation is false, its finding dies. A claim with no challenge and no evidence survives only as a labeled assumption.
7. **Output.**
   - Decide mode gives a verdict: agreement, clashes, assumptions, a recommendation and a first move.
   - Plan mode gives the surviving findings to a separate planner. The planner adds a verification step for each assumption. It makes each open question a gate that you answer before the work starts.
8. **Record.** The skill saves the full run to `crossfire-runs/YYYY-MM-DD-slug.md`.

## Install

To install it as a plugin, run these commands:

```
/plugin marketplace add avectats7/crossfire
/plugin install crossfire@crossfire
```

To install it manually, copy the skill folder:

```bash
git clone https://github.com/avectats7/crossfire.git
cp -r crossfire/skills/crossfire ~/.claude/skills/
```

## Use

```
crossfire this: should we move billing from Stripe Checkout to a custom flow before launch?
```

```
hyperplan: add offline sync to the savings app
```

Other triggers: `adversarial plan`, `red-team this plan`, `tear this plan apart`, `cross-examine this`.

## Origin

Crossfire combines two existing skills. It also adds the parts that each skill did not have.

From [LLM Council](https://github.com/karpathy/llm-council) (the method of Andrej Karpathy, adapted as a Claude skill): the parallel independent pass, the anonymous peer review and a verdict that gives a real answer.

From [Hyperplan](https://www.skillsdirectory.com/skills/code-yeongyu-hyperplan) (oh-my-opencode): the defend, refine or concede round, the open questions that become user gates, and the handoff to a separate planner.

New in Crossfire:

- **A shared evidence pack.** Hyperplan asked advisors for `file:line` citations but gave them no context. That causes invented sources.
- **An independent judge.** In Hyperplan, a finding survived if its own author said DEFEND.
- **Challenges that need support.** The advisors do not reject by default. STANDS is a valid answer. The judge rejects a challenge that has no reason.
- **Labeled assumptions.** An assumption keeps its label in the plan, and the plan gives it a verification step.
- **No hardcoded models**, no dependency on team mode, and a procedure for advisors that fail.

## License

MIT
