---
title: "When the safety check can't parse its input"
description: "I moved a Claude Code plugin to a second agent harness to see how portable it was. Most of it was renames. The surprise was a push/merge safety hook: when its shell parser choked on an apostrophe, it fell back to keyword matching, blocked a harmless write, and the agent's turn ended without a word."
date: 2026-10-27
order: 25
rank: 14
tags: [agents, tooling]
cover: /img/fail-loud/gate-paths.svg
---

The agent was halfway through a run when it stopped talking. No error, no question, no summary. The turn just ended. The last thing it had tried to do was write a small file with a shell heredoc. Nothing about that is dangerous: no push, no merge, no network. Something had blocked it, and nobody had said why.

It took a while to find. The thing that stopped it was a safety hook, and the hook was working as written.

## The portability test

I wanted to know how portable a Claude Code plugin really is, so I ran one under a second agent harness. The plugin is mostly text: commands, skills, a few MCP server definitions, a couple of hooks. The other harness reads a very similar format.

Most of the port was renames. Fourteen skills and four MCP servers came across, and an end-to-end run took about an hour. Plugin-relative paths resolved, OAuth to the MCP servers worked, the "ask the user" tool mapped onto its equivalent. Two small surprises turned up on the way:

- **Commands are named by file name only.** `qa/plan.md` and a top-level `plan.md` both became `/plan`. We hit three collisions: `plan`, `status`, `help`. A prefix or a separate plugin would fix it; both are small.
- **Arguments were dropped.** A command invoked with a ticket ID ran without it. The fix is an explicit arguments placeholder in each command.

Both were noisy in a good way. You notice a wrong command or a missing ID immediately. The third one was quiet.

<figure>
<a href="/img/fail-loud/port.svg" target="_blank" rel="noopener"><img src="/img/fail-loud/port.svg" alt="What moved across to the second harness: fourteen skills, four MCP servers, paths, OAuth and the ask-user tool all ported with renames. Three surprises: command names collided, arguments were dropped, and a safety hook blocked a harmless write and ended the turn silently. The first two were loud, the third was silent."></a>
<figcaption>Most of the port was renames. Of the three surprises, only one was silent, and that was the one that mattered.</figcaption>
</figure>

## The wrong assumption

The environment had a safety hook in front of every shell command. Its job was simple: if the command pushes or merges, stop and ask a human. To decide, it parsed the command line with a shell tokenizer and looked at the actual program and arguments.

I assumed that when a hook like this is unsure, it says so. It didn't. When the tokenizer couldn't parse the command, the hook fell back to searching the raw text for words like "push" and "merge". The agent's heredoc had an apostrophe in it, a plain "don't", which a shell tokenizer can't always handle inside a heredoc. Parsing failed, the keyword search found a match somewhere in the text, and the hook blocked.

Then the second harness did what it does with a block that carries no usable reason: it ended the turn. From the outside, the agent simply went quiet.

<figure>
<a href="/img/fail-loud/gate-paths.svg" target="_blank" rel="noopener"><img src="/img/fail-loud/gate-paths.svg" alt="Before: the hook parses the command; if parsing works it allows or asks; if parsing fails it falls back to a keyword search, blocks a harmless write and the turn ends silently. After: if parsing fails the hook denies with a reason that says it could not read the command, and the agent passes that to the human."></a>
<figcaption>The guess was the problem. Without it, the same parse failure becomes one sentence the human can act on.</figcaption>
</figure>

## Two failures in one

There are really two problems here, and they stack.

**The fallback guessed.** A keyword match on raw text is a different check from the one the hook was built to run. It blocks harmless writes that mention the word, and a command that hides the word gets through. Falling back to it doesn't make the gate safer. It makes the gate's behaviour depend on how the command was quoted.

**The block was silent.** Even a correct block is useless if nobody hears about it. The agent can't recover, and the human can't help, because neither knows what happened.

The fix I wrote down is short, and it belongs to whoever owns the hook, not to the plugin. If the hook can't parse a command, it denies, and the deny message says so in plain words: "couldn't read this command, so I'm not running it; ask the human." It still fails closed, but now a person finds out. The agent passes the message on, the human looks at the command, and the run continues. That costs one extra question in a rare case, which is much better than a silent stop.

| | Fallback to a guess | Fail visibly |
|---|---|---|
| Command it can't parse | keyword search on raw text | deny, with "couldn't parse" as the reason |
| Harmless write mentioning "merge" | blocked | asked about, then allowed |
| Real push hidden by odd quoting | may slip through | denied |
| What the human sees | the agent goes quiet | one clear sentence |
| Where the time goes | reading logs to find the hook | answering one question |

## Why the port found it

Under the original harness this probably happened rarely, if at all. Different defaults, different commands, different quoting habits. Running the same plugin somewhere else changed the inputs, and the inputs found the branch nobody had looked at. A portability test is also a fuzz test for the parts you didn't write.

<div class="callout">
<p><strong>The rule:</strong> when a safety check can't read its input, it should refuse and say why. It should never fall back to a guess, and it should never block without a reason a human will see.</p>
</div>
