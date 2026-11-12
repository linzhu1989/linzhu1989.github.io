---
title: "The grader was wrong"
description: "A new agent skill scored 0.33 on the smallest model, three runs out of three. The obvious move was to fix the agent. The transcripts said the agent was right every time. Four of the failures were in the graders, one was real, and the only way to tell them apart was to read what the agent actually did."
date: 2026-10-31
order: 19
rank: 19
tags: [agents, testing]
cover: /img/grader/two-columns.svg
---

The eval came back at 0.33 on the smallest model, three runs out of three. On the two bigger models it was 0.67. The skill was new, so the obvious next step was to tighten its instructions. Small model, low score, add more guidance.

Before changing a word, I opened the three transcripts. In every run the agent had done the job: the right counts, the right rows, the right list of what it would skip, a clear note that one integration was missing, and then it stopped and waited for a go. One run had even gone further and checked the network first. Every run was correct, and every run had failed.

## Where the points went

The eval had three graders. Two of them were wrong.

The first was a regex. It wanted the reply to name the example email address from the test fixture. But the eval harness runs as the real user, so every reply used the real address and called it "your own address". Correct behaviour, wrong pattern.

The second was an LLM judge with a rubric. The rubric said the reply must state that a particular tool server was not available. The harness, though, serves a mock of that server, so it was available, and the agent said so. The rubric was written for the production setup, not for the place the test actually runs.

The bigger models had passed the regex by repeating the example address. They lost points on the rubric too.

Two days later, a second skill went through the same process and failed the same way. This time the graders broke in two new ways:

- A regex used `(?i)` for case-insensitive matching. That is Python syntax. The harness evaluates regexes in JavaScript, where `(?i)` is invalid, so the grader threw an error on every single run. Nothing the agent wrote could have passed it.
- A long rubric listed five or six facts that all had to appear. The judge was the small model. It failed a reply from the largest model that contained every fact. Same reply, three judge runs: fail, fail, pass.

<figure>
<a href="/img/grader/two-columns.svg" target="_blank" rel="noopener"><img src="/img/grader/two-columns.svg" alt="Two columns. Left, what the agent did: used the real user's address, said the tool server was present as a mock, case-insensitive answer, every required fact present. Right, what the grader checked: the fixture's example address, 'tool server not available', a Python-only regex the JavaScript harness rejects, an all-or-nothing rubric judged by the small model. Each row is marked as a grader fault."></a>
<figcaption>Left: what the transcripts showed. Right: what the graders demanded. Every mismatch was on the right-hand side.</figcaption>
</figure>

## One failure was real

Not every failure was the grader's fault. On a request that didn't match anything the skill knew, the small model went ahead and ran a check anyway, because the user had asked for it. That is the behaviour the skill was supposed to refuse. The grader caught it correctly, and the fix belonged in the skill: one sentence saying that no match means stop, whatever the user's wording.

If I had started by editing the skill, that real bug would have been buried under four fake ones, and I would have "fixed" the four by making the agent copy a fixture address and claim a server was missing when it wasn't.

## What we changed

<figure>
<a href="/img/grader/triage.svg" target="_blank" rel="noopener"><img src="/img/grader/triage.svg" alt="A triage flow. A failing eval leads first to reading the transcript. If the behaviour was right, fix the grader and rerun. If the behaviour was wrong, fix the skill and rerun. Before: the score went 0.33 three times. After the grader fix: 1.0 three times."></a>
<figcaption>The order matters: transcript first, then decide which side is broken.</figcaption>
</figure>

The graders were rewritten to match how the harness runs:

- The address regex accepts either the example address or "your own address".
- The rubric only requires what the test can actually observe: the missing integration must be named, and the mocked server can be described as present or absent, as long as the reply says which.
- Case-insensitive matching is spelled out as `[Dd][Rr][Aa][Ff][Tt]`, which works in any regex dialect.
- Long rubrics became one short pass sentence, with "either form passes" written out where two answers are both fine.

The same agent behaviour, unchanged, went from 0.33 to 1.0 on the small model, three runs out of three. The bigger models went to 1.0 as well. The one real bug got its one-sentence fix and stayed fixed.

| | Fix the agent first | Read the transcript first |
|---|---|---|
| What the score means | "the agent is weak" | a question to answer |
| Fake failures | patched into the agent's behaviour | fixed in the grader |
| The one real bug | lost in the noise | found and fixed in one sentence |
| Cost of the habit | nothing up front, a worse skill later | a few minutes reading per failing run |

Graders are code. They're written quickly, by the same people who wrote the skill, often for an environment slightly different from the one the eval runs in. They have bugs just like the thing they test. An eval score tells you the grader and the agent disagreed. It doesn't tell you which of them is wrong.

<div class="callout">
<p><strong>The rule:</strong> when an eval fails, read the transcript before touching the agent. If the behaviour was right, the grader is the bug. Write graders for the harness they actually run in, keep rubrics short enough for the judge model to apply, and use regex syntax that every dialect accepts.</p>
</div>
