---
title: "Waiting for a dead process"
description: "A coding assistant started a long eval run in the background and set a loop to wait for the word ALL_DONE in its log. The run had died in its first second. The loop waited almost two hours, until I asked whether anything was still running."
date: 2026-11-06
order: 24
rank: 24
tags: [agents, engineering]
cover: /img/waiter/two-waiters.svg
---

Just before one in the afternoon, my coding assistant started a long eval run. It would take a while, so the assistant launched it in the background with `nohup`, sent the output to a log file, and started a second loop to watch that log. When the run printed `ALL_DONE` at the end, the loop would wake up and the assistant would read the results.

I went to do something else. Close to quarter to three I came back, saw nothing had changed, and asked the obvious question: is it actually still running?

It wasn't. It had never run. The launch line had the directory twice, something like `evals/evals/run.sh`. The shell printed "No such file or directory" and exited in under a second. The log had exactly one line in it, and it was never going to have `ALL_DONE`. The waiter had spent almost two hours checking a file that a dead process would never write to again.

Nothing was lost except the afternoon. The run hadn't started, so it had cost nothing. The assistant fixed the path and relaunched a minute later.

## The wrong assumption

The waiter assumed one thing: *if the marker isn't there yet, the job is still working.* That is true only while the job is alive. A job can end in three ways: it finishes and writes the marker, it crashes before writing it, or it never starts. A loop that only looks for the marker can tell the first case from "not yet", and that's all it can tell. The other two look exactly like "not yet", forever.

This isn't really an agent bug. People write this loop all the time: `tail -f` and wait, or `until grep -q DONE log; do sleep 60; done`. The difference with an agent is that nobody is watching the terminal. When I write that loop myself, I notice after twenty minutes that the log hasn't moved. The assistant was patient and didn't get bored, so it kept waiting.

## What a waiter has to check

<figure>
<a href="/img/waiter/two-waiters.svg" target="_blank" rel="noopener"><img src="/img/waiter/two-waiters.svg" alt="Before: the producer dies at second one, the waiter keeps polling the log for ALL_DONE from 12:47 to 14:41. After: on every poll the waiter checks the marker, then whether the process is alive, then a deadline; the dead producer is caught on the first poll."></a>
<figcaption>Same dead job, two waiters. One asks a single question, the other asks three.</figcaption>
</figure>

On every pass, a waiter needs to ask three things, in this order:

1. **Is the marker there?** Then the job finished. Read the results.
2. **Is the producer still alive?** If not and there's no marker, it died. Stop and show the last lines of the log.
3. **Is it past the deadline?** If it's still alive but has been going much longer than it should, stop and tell a person.

```sh
# before: waits forever if the job dies
until grep -q ALL_DONE run.log; do sleep 60; done

# after: marker, then liveness, then a deadline
nohup ./run.sh > run.log 2>&1 &
pid=$!
deadline=$(( $(date +%s) + 3*3600 ))
while ! grep -q ALL_DONE run.log; do
  if ! kill -0 "$pid" 2>/dev/null; then
    echo "run exited without ALL_DONE"; tail -20 run.log; exit 1
  fi
  if [ "$(date +%s)" -gt "$deadline" ]; then
    echo "still running after 3h, check it"; exit 2
  fi
  sleep 60
done
```

`kill -0` sends no signal. It only asks whether the process exists. Keeping `$!` right after the launch is the cheap part; the waiter needs it, and you can't get it back later. With this loop, the doubled directory would have been caught on the first poll, at one minute instead of two hours.

The assistant now launches background jobs this way: record the pid at launch, check it on every poll, set a deadline from the expected duration.

| | Marker only | Marker + liveness + deadline |
|---|---|---|
| Job finishes | wakes up | wakes up |
| Job crashes halfway | waits forever | stops on the next poll, shows the log |
| Job never starts | waits forever | stops on the first poll |
| Job hangs | waits forever | stops at the deadline |
| Who notices otherwise | a person, eventually | nobody needs to |

<div class="callout">
<p><strong>The rule:</strong> a loop that waits for a background job must check that the job is still alive, not only look for its end marker, and it needs a deadline. Waiting is the easy part. The loop has to be able to tell "not done yet" from "never going to finish".</p>
</div>
