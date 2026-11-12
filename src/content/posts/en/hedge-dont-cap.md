---
title: "Hedge, don't cap"
description: "A tools server was slow to come back after a dropped connection, and the obvious fix was an 8-second cap per attempt. Fourteen days of logs said almost half the slow loads would have succeeded. We started a second try instead of killing the first, and the 30-second stalls went from 36 to zero."
date: 2026-11-04
order: 18
rank: 18
tags: [engineering, agents]
cover: /img/hedge/distribution.svg
---

Our agents get their tools from a shared MCP tools server. When an agent's connection to it drops, the next call has to load the server again: open a new connection, run `initialize`, fetch the tool list. Usually that takes a few seconds. Sometimes it took long enough that the agent's harness gave up at its own 40-second limit, and the agent started its task with no tools at all.

The fix already sitting in review looked reasonable. Put an 8-second cap on each load attempt. If it hasn't answered by then, kill it and retry. A stuck attempt shouldn't be allowed to eat the whole 40 seconds.

Before merging it, I pulled the logs.

## What the logs said

Fourteen days, 121 fresh loads after a dropped connection.

- median: 6.5 seconds
- 90th percentile: 55 seconds
- slowest: 108 seconds
- **47% took longer than 8 seconds, and every one of those still succeeded**

So the cap was about to kill almost half of the slow loads, and they weren't stuck. They were slow, and they were going to finish. Each retry would start a fresh load from zero, which is just as likely to be slow. On a bad day the agent would cycle through three dead attempts and still miss the deadline.

<figure>
<a href="/img/hedge/distribution.svg" target="_blank" rel="noopener"><img src="/img/hedge/distribution.svg" alt="A histogram of 121 load times. The median is 6.5 seconds; a long tail reaches 108 seconds. An 8-second line cuts through it: 47% of the loads sit to the right of the line, and all of them succeeded."></a>
<figcaption>Everything right of the 8-second line would have been killed by the cap. All of it succeeded.</figcaption>
</figure>

## The wrong assumption

A cap assumes that slow means dead: past some point, an attempt is never coming back, so you may as well stop it. That's often true for a hung TCP connection. It wasn't true here. The distribution had a long tail of loads that were slow and healthy, and only the logs could show which kind we had.

## What we did instead

We kept the 8 seconds, but changed what happens when they pass. Instead of killing the first attempt, start a second one on a new connection and let both run. Whichever succeeds first wins, and the other is dropped. The load fails only when every attempt that was started has failed.

<figure>
<a href="/img/hedge/cap-vs-hedge.svg" target="_blank" rel="noopener"><img src="/img/hedge/cap-vs-hedge.svg" alt="Two timelines. Cap: attempt one is killed at 8 seconds, attempt two starts from zero and is killed at 16, attempt three is killed at 24, and the agent starts with no tools. Hedge: attempt one keeps running, attempt two starts at 8 seconds on a new connection, attempt one answers at 14 seconds and wins."></a>
<figcaption>The cap throws away progress every 8 seconds. The hedge keeps it and adds a second chance.</figcaption>
</figure>

This is the "hedged request" idea from large-scale serving, applied to one client. If the slow first try was simply unlucky, the second one usually overtakes it. If the first try was about to finish, it still does. The only cost is one extra connection in the slow cases.

We tested it on the production bundle with fake upstreams: healthy, hung, slow, two slow in a row, everything hanging, and the connection-drop cases from the earlier fix. Then we merged and compared about ten hours before with about fourteen hours after.

| | Before (no hedge) | After (hedge at 8 s) |
|---|---|---|
| `initialize` over 30 s | 36 | 0 |
| harness traces hitting its 40 s limit | 25% | 2% |
| sessions that needed a retry | 117 | 30 |
| crashes | 0 | 0 |

One more thing showed up in the after window. Twenty-nine first tool calls took between 11 and 148 seconds, and all 29 returned 200. A cap on those would have turned 29 slow successes into 29 failures.

## The habit

Before choosing a timeout, look at the distribution of how long the thing actually takes, and at whether the slow cases succeed. If slow ones fail, a cap is right: stop waiting and move on. If slow ones mostly finish, a cap only throws work away, and a hedge gets you the speed without the losses.

It took about twenty minutes to pull the numbers, and they changed the design.

<div class="callout">
<p><strong>The rule:</strong> pick a timeout from the latency distribution, not from a feeling. When slow doesn't mean dead, don't kill the slow attempt. Start a second one beside it and take whichever answers first.</p>
</div>
