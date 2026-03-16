---
name: solve
description: "Structured Decision Engineering framework. Use when facing decisions, trade-offs, architecture choices, or ambiguous problems with multiple viable paths. Synthesizes OODA, MECE, A3, Working Backwards, Cynefin, Mental Models, Cognitive Debiasing, Antifragility, Satisficing, Inversion, and PDCA into a single adaptive process."
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Write
  - Edit
user-invocable: true
---

# /solve — Structured Decision Engineering

The definitive problem-solving framework. Synthesizes 11 world-class methodologies into a single adaptive process that scales from 2-minute crisis response to multi-week architectural decisions.

**Lineage**: OODA Loop (Boyd, military) + MECE (McKinsey) + A3 (Toyota) + Working Backwards (Amazon) + Cynefin (Snowden) + Mental Models (Munger) + Cognitive Debiasing (Kahneman) + Antifragility (Taleb) + Satisficing (Simon) + Inversion (Jacobi) + PDCA (Deming)

---

## When to Use

Any time you face a decision, problem, trade-off, or ambiguity with more than one plausible path forward. If you're about to pick the first idea that comes to mind — stop and run this process.

---

## Speed Control: Read This First

Not every problem deserves the same rigor. Before starting, classify:

**Decision type** (Bezos):
- **Type 1 (Irreversible)**: Architecture, data migrations, public APIs, pricing, hiring. Full process. Sleep on it.
- **Type 2 (Reversible)**: Refactors, config, feature flags, internal tools. Decide in hours with 70% information. Move fast.

**Problem category** (Cynefin):

| Category | Signal | Process | Time |
|----------|--------|---------|------|
| **Simple** | Known solution, done before | Steps 1→5→6→10 | 5 min |
| **Complicated** | Multiple viable paths, needs analysis | Full 10 steps, moderate depth | 15-60 min |
| **Complex** | No clear solution, unknowns, emergent | Full 10 steps, heavy on 2, 7, 8 | 1-4 hours |
| **Chaotic** | Crisis, system down, bleeding | Step 8 FIRST (act), then 1→10 | 2 min to act, full loop after |

**Satisficing gate** (Simon, Nobel Prize): For low-stakes Type 2 decisions, the first adequate solution is often better than spending time finding the optimal one. If the stakes are low and the decision is reversible, pick the first solution that clears your constraints and move on. Analysis paralysis on small decisions is a worse failure mode than picking a B+ option.

---

## The 10 Steps

### Step 1: DEFINE — State the Real Problem

*OODA: Observe | McKinsey: Issue Tree | Toyota: Background*

Write one sentence: **"The problem is [X] because [Y], which causes [Z]."**

If you can't fill all three blanks, you don't understand the problem. Ask questions. Read actual code, data, logs — never theorize from memory.

**Separate symptom from problem**:
- Symptom: "The tests are failing"
- Problem: "The timing-safe hash check rejects mock hashes that don't match SHA256, blocking the pipeline"

If the problem doesn't actually exist after investigation — STOP. Document the non-finding. Solving phantom problems is worse than solving nothing.

---

### Step 2: DIAGNOSE — Find the Root Cause

*OODA: Orient | Toyota: Root Cause Analysis | 5-Whys*

Ask "why?" repeatedly until you reach a cause you can act on. Each answer must be evidence-based — read the code, check the data, verify the claim.

**5-Whys example**:
1. Why did the push fail? → Tests failed during the pipeline
2. Why did tests fail? → Security tests returned 401 instead of 200
3. Why 401? → timingSafeEqual rejected the hash comparison
4. Why rejected? → Mock hash "mock-api-key-hash" isn't valid hex, Buffer.from("hex") produces different bytes
5. Why hex encoding? → Implementation assumed all hashes are hex (true in prod, false in tests)

Root cause: **Encoding assumption mismatch between production and test data**

For **complex problems**, use a fishbone diagram across categories: People, Process, Technology, Environment, Data.

For **simple problems**: one sentence, move on.

---

### Step 3: CONSTRAIN — Map the Boundaries

*OODA: Orient | McKinsey: Constraints*

Solutions that ignore constraints are fantasies. Map these before generating options:

| Constraint | Question to Answer |
|------------|-------------------|
| **Time** | Deadline? Blocking others? How urgent vs how important? |
| **Effort** | Hours, days, weeks? Who does the work? What's the opportunity cost? |
| **Dependencies** | What must exist first? What else breaks if we change this? |
| **Reversibility** | Can we undo it? How fast? At what cost? |
| **Blast radius** | How many files, services, users, customers affected? |
| **Risk tolerance** | What's the worst case? Can the business survive it? |
| **Compatibility** | Does it break APIs, contracts, user expectations? |
| **Cost of delay** | What does each day/week of indecision cost? Is NOT deciding worse than a wrong but reversible decision? |

---

### Step 4: APPLY MENTAL MODELS — See the Problem Through Multiple Lenses

*Munger: Latticework of Mental Models*

Before generating solutions, run the problem through relevant models from different disciplines. This prevents narrow thinking. Not all models apply to every problem — pick 3-5 that fit.

**Engineering**: First principles, redundancy, feedback loops, failure modes
**Economics**: Opportunity cost, marginal utility, incentive alignment, sunk cost
**Psychology**: Incentives, social proof, loss aversion, commitment bias
**Biology**: Evolution, niche competition, Red Queen effect, adaptation
**Physics**: Entropy, critical mass, leverage
**Systems**: Second-order effects, emergence, bottleneck theory

**Selection guide** — match models to problem type:
- Technical/engineering → Engineering + Systems models
- Business/strategic → Economics + Psychology models
- People/process → Psychology + Biology models
- Novel/unprecedented → Physics (first principles) + all disciplines (wide search)

Output: Name 3-5 models that apply. For each, one sentence on what it reveals about this specific problem. If a model contradicts another, note the tension — that's where the real insight lives.

---

### Step 5: INVERT — How Would This Fail?

*Munger/Jacobi: "Invert, always invert"*

Instead of "how do I solve this?", ask: **"How would I guarantee this problem gets WORSE?"**

List 3-5 ways to guarantee failure. Then ensure your solutions avoid every single one.

**Example** (dashboard rate limiting):
- Guarantee failure: Don't rate limit at all → abuse escalates
- Guarantee failure: Rate limit by IP only → authenticated attackers bypass with session tokens
- Guarantee failure: Set limits too low → legitimate users get locked out
- Guarantee failure: No bypass for admins → can't debug production issues

This surfaces risks that positive thinking misses. Munger attributes more of his success to avoiding stupidity than to being clever.

---

### Step 6: GENERATE — Produce Solutions (Full Spectrum MECE)

*McKinsey: Mutually Exclusive, Collectively Exhaustive*

Generate solutions across the **full intervention spectrum** — from doing nothing to transforming everything. Anchoring on the first idea is the #1 decision failure; clustering solutions in one region of the spectrum is the #2 failure.

Solutions must be:
- **Mutually exclusive**: Each is a distinct path (no "A but with a bit of B")
- **Collectively exhaustive**: Together they cover the full solution space from zero intervention to maximum intervention

**The 5 mandatory solution slots** (produce all 5, then cut any that are genuinely inapplicable):

```
Do nothing ←————————————————————————————————→ Transform everything
   |            |            |            |            |
 Option 0    Option A     Option B     Option C     Option D
```

- **Option 0: Do Nothing / Status Quo** — What happens if we don't act? What's the cost of inaction? This option is ALWAYS worth evaluating because it forces you to quantify why action is needed. Sometimes "do nothing" is genuinely the best choice. Explicitly rejecting it is more valuable than never considering it.

- **Option A: Minimal Intervention** — The lightest touch that could plausibly work. Band-aid fix. Incremental improvement on what exists. The question is: "What's the absolute least we could do?" Not a clever alternative — the laziest credible option. If the current approach has 6 warning scripts that don't prevent the problem, Option A might be "add a 7th warning script." This lets you explicitly reject the minimal path with evidence.

- **Option B: Conventional Best Practice** — What experts in this domain would recommend. The standard, proven, widely-adopted solution. Search for prior art — someone has almost certainly solved this before. This is usually the front-runner.

- **Option C: Unconventional / Creative** — Challenge an assumption. What if we solved the upstream problem instead? What if we made the problem irrelevant? What if we turned the weakness into a strength? What would a competitor do? What would someone with 10x our budget do? What would someone with 1/10th our budget do?

- **Option D: Eliminate the Problem** — What if we removed the need entirely? Can we restructure so this problem cannot exist? This is the "don't manage complexity — remove it" option. Example: "Don't isolate parallel agents — don't run parallel agents." This challenges whether the constraint that created the problem is actually necessary.

**Feed inversion into generation**: Review your failure modes from Step 5. At least one solution should directly address the #1 failure mode identified by inversion.

**Antifragility lens** (Taleb): For at least one solution, evaluate: does this get BETTER when surprises happen? Prefer options with capped downside and uncapped upside (convexity) over options that are merely robust.

**Stakeholder perspectives**: Before finalizing, ask: would the user/customer choose differently than the implementer? Would a security auditor? Would someone optimizing for speed vs. quality? If any perspective produces a new solution, add it.

**Spectrum validation check** (do this AFTER generating): Plot your solutions on the intervention spectrum. If all solutions cluster in one region, you have a coverage gap. Add solutions to fill empty regions. The goal is that after reading all options, there is no obvious "but what about...?" left unaddressed.

**Hybrid solution check**: After evaluating all options in Step 7, if the best answer combines elements from multiple options (e.g., B's architecture with C's rollout strategy), describe the hybrid as a new option and evaluate it. MECE applies to the initial generation — but the best real-world solution often synthesizes across options. Don't let the framework prevent the obvious best answer.

For each solution: one paragraph covering what it is, how it works, and what changes.

---

### Step 7: EVALUATE — Weighted Decision Matrix

*McKinsey: Evaluation Matrix | Toyota: Select Countermeasure*

Score each solution against weighted criteria:

| Criteria | Weight | What It Measures |
|----------|--------|-----------------|
| Effectiveness | 25% | How completely does it solve the root cause? |
| Effort | 15% | Implementation cost (time, complexity, risk of bugs) |
| Risk | 15% | What's the worst case? Probability x impact |
| Reversibility | 10% | How easily can we undo this if wrong? |
| Maintenance | 10% | Ongoing burden after ship? Tech debt? |
| Second-order effects | 10% | What else changes? Positive and negative side effects |
| Antifragility | 10% | Does it get stronger under stress/surprise? |
| Regret minimization | 5% | At age 80, would you regret NOT trying this? (for strategic decisions) |

**Evaluation table**:

| Criteria (Weight) | Opt 0: Status Quo | Opt A: Minimal | Opt B: Best Practice | Opt C: Creative | Opt D: Eliminate |
|--------------------|:-:|:-:|:-:|:-:|:-:|
| Effectiveness (25%) | X/10 | X/10 | X/10 | X/10 | X/10 |
| ... | ... | ... | ... | ... | ... |
| **Weighted Total** | **X.X** | **X.X** | **X.X** | **X.X** | **X.X** |

Drop any column where you determined the option was genuinely inapplicable in Step 6. But you should have at least 4 columns — if you have fewer, your solution generation was too narrow.

**Antifragility scoring guide**: Score 1-3 for fragile (breaks under stress), 4-6 for robust (survives stress), 7-10 for antifragile (improves under stress). If no option scores above 6, reconsider whether you're missing a creative solution with convex payoff — limited downside, potentially unlimited upside.

**Confidence calibration**: After scoring, assign your overall confidence in the recommendation: **Low** (<50%), **Medium** (50-75%), **High** (75-90%), or **Very High** (>90%). If Very High — you're probably overconfident; look harder for what you're missing. If Low — the analysis needs more evidence before deciding. Record this level; you'll compare it against actual outcomes at the review date.

**Advantages / Disadvantages** (for each non-eliminated option):

| | Advantages | Disadvantages |
|---|-----------|---------------|
| 0: Status Quo | ... | ... |
| A: Minimal | ... | ... |
| B: Best Practice | ... | ... |
| C: Creative | ... | ... |
| D: Eliminate | ... | ... |

---

### Step 8: VALIDATE — Stress-Test the Winner

*Amazon: Working Backwards | Klein: Pre-Mortem | Military: Red Team*

Three validation techniques, in order:

**1. Working Backwards** (Amazon): Write the "press release" for the completed solution in 2 sentences. If you can't clearly articulate the benefit, the solution may be wrong.

**2. Pre-mortem** (Klein — research shows 30% improvement in decision quality): It's 6 months from now and this solution **failed catastrophically**. Write down exactly why it failed. What went wrong? What did we miss? This surfaces risks that positive evaluation systematically misses.

**3. Red Team** (Military): Actively try to **destroy** the solution. Not passively "assume it failed" — actively attack it. What's the weakest link? How would a hostile actor exploit it? How would Murphy's Law hit it? If you can't break it, it might actually be good.

**Cheapest validation** before full commitment:
- **Code**: Spike/prototype in 30 min
- **Architecture**: Walk through the data flow with edge cases
- **Process**: Do it manually once before automating
- **Type 1 decisions**: Sleep on it — literally. Overnight processing catches what daytime analysis misses.

**For CHAOTIC problems**: Act first (stabilize the crisis), THEN validate.

---

### Step 9: BIAS CHECK — Verify Your Own Thinking

*Kahneman: Cognitive Debiasing Checklist*

Before finalizing, run this checklist. Check each item honestly:

| Bias | Check | If Triggered |
|------|-------|-------------|
| **Confirmation** | Did I seek evidence AGAINST my preferred solution? | Actively search for disconfirming evidence |
| **Anchoring** | Am I weighting the first option I considered too heavily? | Re-evaluate Solution B and C with fresh eyes |
| **Sunk cost** | Am I choosing this because of past investment (time, code, effort)? | Decide as if starting from zero today |
| **Availability** | Am I overweighting a recent/vivid experience? | Check base rates and actual data |
| **Overconfidence** | How confident am I? (>80% confident = probably overconfident) | Widen uncertainty range, add buffer |
| **Status quo** | Am I choosing "do nothing" because change is uncomfortable? | Evaluate "do nothing" as an active choice with real costs |
| **Groupthink** | Is everyone agreeing too easily? | Assign a devil's advocate role |
| **Planning fallacy** | Is my time/effort estimate based on best case? | Multiply estimate by 1.5x (for complicated) or 3x (for complex) |

If 2+ biases triggered: revisit Step 7. The evaluation may be compromised.

---

### Step 10: DOCUMENT & MONITOR — Close the Loop

*Toyota A3: Report | Deming: PDCA (Plan-Do-Check-Act)*

**Document** the decision so future-you doesn't re-analyze the same problem:

For architectural decisions — ADR:
```
# ADR-NNN: [Title]
## Status: Accepted
## Context: [Problem and why it needed a decision]
## Decision: [What we chose]
## Alternatives Considered: [What we rejected and why]
## Consequences: [Positive and negative effects]
## Confidence: [Low/Medium/High/Very High — from Step 7]
## Predicted Outcome: [What I expect to happen by review date]
## Review Date: [When to revisit — 30/60/90 days]
```

For operational decisions — structured summary:
```
## Decision: [One sentence]
## Root Cause: [From step 2]
## Solution: [From step 7]
## Trade-off: [What we gave up]
## Success Criteria: [How we'll know it worked]
## Confidence: [Low/Medium/High/Very High]
## Predicted Outcome: [Expected result by review date]
## Review: [Date to check results]
```

**Monitor** (the step most frameworks skip): Set a concrete review date. After implementation, check:
- Did it actually solve the root cause?
- Were there unexpected side effects?
- Is the maintenance burden acceptable?
- How does the actual outcome compare to the predicted outcome? (This calibrates future decisions.)
- Should we adjust, double-down, or revert?

**Disagree and commit** (Amazon): Once the decision is made through this process, execute it wholeheartedly — even if you would have chosen differently. Half-hearted execution kills good decisions more often than bad analysis does. If new evidence emerges that invalidates the decision, trigger a re-evaluation at the review date (or immediately if critical) — don't silently undermine the current plan.

---

## Completeness Gate

Before writing the RECOMMENDATION, verify every required artifact is present. If any is missing, go back to that step.

| Step | Required Artifact | Check |
|------|-------------------|:-----:|
| 1 | Problem statement: "The problem is [X] because [Y], which causes [Z]" | [ ] |
| 2 | Root cause identified (or documented as non-existent) | [ ] |
| 3 | At least 5 constraints mapped (including cost of delay) | [ ] |
| 5 | At least 3 failure modes from inversion | [ ] |
| 6 | At least 4 solutions spanning the full intervention spectrum | [ ] |
| 7 | Weighted evaluation table with scores and totals | [ ] |
| 7 | Confidence level assigned (Low/Medium/High/Very High) | [ ] |
| 8 | Pre-mortem: at least 2 specific failure scenarios | [ ] |
| 8 | Red team: at least 1 active attack on the chosen solution | [ ] |
| 9 | Bias checklist: all 8 items checked with honest answers | [ ] |

For **Simple** problems (from Speed Control): only Steps 1, 6, 7 artifacts are required.

---

## Output Format

Present each step with a clear header. Use tables for comparisons. End with:

```
## RECOMMENDATION

**[Solution name]** — [one-sentence summary]

**Why this wins**: [2-3 sentences — root cause alignment, evaluation score, validation results]
**Confidence**: [Low/Medium/High/Very High] — [one sentence on what could change this]
**Key trade-off**: [What we're accepting]
**Risk mitigation**: [How we handle the biggest risk from pre-mortem/red team]
**Next action**: [The immediate concrete step]
**Predicted outcome**: [What I expect to happen]
**Review date**: [When to evaluate if this worked]
```

---

## Framework Lineage Map

| Step | Boyd (OODA) | McKinsey (MECE) | Toyota (A3/PDCA) | Amazon | Cynefin | Munger | Kahneman | Taleb | Simon | Jacobi | Deming |
|------|-------------|-----------------|-------------------|--------|---------|--------|----------|-------|-------|--------|--------|
| 1. Define | Observe | Issue tree | Background | Customer problem | Sense | — | — | — | — | — | Plan |
| 2. Diagnose | Orient | Root cause | Root cause | — | Probe | — | — | — | — | — | Plan |
| 3. Constrain | Orient | Constraints | Goal/Target | Constraints | — | — | — | — | — | — | Plan |
| 4. Mental Models | Orient | — | — | Tenets | — | Latticework | — | — | — | — | — |
| 5. Invert | — | — | — | — | — | Inversion | — | — | — | Invert | — |
| 6. Generate | — | MECE hypotheses | Countermeasures | Solutions | Respond | — | — | Optionality | Satisfice | — | — |
| 7. Evaluate | Decide | Decision matrix | Select | Tenets | — | — | — | Convexity | — | — | — |
| 8. Validate | Act | Pilot | Plan + Follow-up | Working Backwards | Probe | — | Pre-mortem | — | — | — | Do |
| 9. Bias Check | — | — | — | Disagree & commit | — | Psychology | Full checklist | — | — | — | Check |
| 10. Document | — | Recommendation | A3 report | PRFAQ | — | — | — | — | — | — | Act |
