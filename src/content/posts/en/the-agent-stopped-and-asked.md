---
title: "The agent stopped and asked"
description: "Halfway through a batch run that books real test reservations, the agent stopped and asked a question. It was right to stop. The question just came too late: by then I had already said go to a summary that didn't contain the answer."
date: 2026-11-02
order: 22
rank: 22
tags: [agents, automation]
cover: /img/ask-first/step-chain.svg
---

I was trying out a new batch mode for an agent skill that tests marketing emails end to end. For each case it books a real reservation in a staging system, seeds a few customer signals, triggers the email, waits, and then checks what landed in a mailbox. The single-case version had been working for a while. Batch mode read a spreadsheet of cases, 72 rows of which 48 had a mapping, and I limited the first run to two.

The first turn went well. The agent showed a scope summary: 2 rows of 72, 48 mapped, 24 skipped with reasons, which mailbox it would use, which tools were present. Then it waited. I said go.

It ran the preflight, created the batch folders, mapped the two rows and picked the email copy. Then, before creating the first reservation, it stopped:

> The batch inputs default `language` to empty, but every earlier successful single run of these cases booked with `es`. Creating a reservation and triggering the email can't be repeated safely. Which language should I use?

I answered "es for both rows". The run finished, two reservations, two emails, a summary. A good outcome, and I almost moved on.

## The wrong assumption

I had treated "go" as approval for the whole run. The summary was the gate, I said yes, so everything after that was covered.

But the summary didn't show the language. Nobody had asked for it, so it wasn't on the screen, so my "go" approved a run with a blank in it. The agent caught the blank, which is to its credit. It caught it in the middle of the run, after four setup steps, at the last moment before the first step that can't be undone. If it had been a little less careful, it would have booked two reservations in the wrong language and sent two emails nobody could unsend.

The agent being careful at the edge is a nice extra. It can't be the design.

<figure>
<a href="/img/ask-first/step-chain.svg" target="_blank" rel="noopener"><img src="/img/ask-first/step-chain.svg" alt="The batch run as a chain of steps. The early steps (preflight, folders, map rows, pick copy) can be repeated. Creating the reservation and triggering the email cannot. The agent asked just before the first one that can't be repeated; the question belongs on the scope summary, before go."></a>
<figcaption>It asked at the last safe moment. The right moment was earlier, on the summary.</figcaption>
</figure>

## A second, smaller confession

In the same run the agent reported something about itself. Before producing the scope summary, it had run `head` on the spreadsheet to have a look. The design said only a script reads the user's input file; the agent reads what the script computed.

It sounds pedantic. It isn't. The counts on the summary (72, 48, 24, 2) are what I approve. If the agent forms its own impression of the file first, the summary becomes a mix of what the script computed and what the agent guessed from the first ten lines. I want the thing I say yes to to come from one place.

## What we changed

Two lines in the skill's instructions, each from one of those moments.

**Settle every input an irreversible step needs before the summary.** `language` is now one of the batch inputs. The agent either gets it from the request or derives it from earlier successful runs of the same cases, and it shows the value on the scope summary. If it can't decide, it asks there, before go, with nothing yet done.

**A script reads the user's input files, never the agent.** The plan step is the only read of the spreadsheet. The agent talks about what the plan returned.

<figure>
<a href="/img/ask-first/scope-summary.svg" target="_blank" rel="noopener"><img src="/img/ask-first/scope-summary.svg" alt="Before: the scope summary listed rows, mapping, skipped rows, the mailbox and tools, but not the language, so go approved a run with a blank in it. After: the summary also shows language es and where it came from, and the counts come from the plan script, so go covers everything the irreversible steps will use."></a>
<figcaption>Same summary, one more line. Now "go" means what I think it means.</figcaption>
</figure>

| | Before | After |
|---|---|---|
| Where the question came up | mid-run, before the first reservation | on the scope summary, before go |
| What "go" approved | a run with a blank input | every value the irreversible steps use |
| Where the counts came from | the plan script, plus the agent's own look | the plan script only |
| If the agent hadn't been careful | two wrong reservations, two emails sent | nothing to be careful about |

## Where the ask goes

Every agent that does real things has a few steps that can't be taken back. Here they were "create a reservation" and "trigger the email". Everything before them can be rerun for free. Everything after them depends on them.

So I now read a new skill backwards. Find the steps that can't be repeated, list every input they use, and check that each one is on the screen at the moment the human says yes. If one isn't, that's where a question belongs, and it belongs on the summary, not at the edge.

The whole try-out cost $7.02 over four turns. The turn where the agent stopped cost about a dollar. The two lines it produced will be read by every batch run after it.

<div class="callout">
<p><strong>The rule:</strong> put the agent's questions before the steps that can't be undone, and make the approval summary show every input those steps will use. An agent that stops at the last safe moment did well. A design that relies on that has a gap.</p>
</div>
