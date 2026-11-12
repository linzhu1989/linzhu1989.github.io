---
title: "Two sessions, one repo"
description: "I run several Claude Code sessions against the same personal git repo, often at the same time. They cannot see each other. The only thing they share is the files, so the coordination has to live in the files too: a claim line, a scoped commit, a note for whoever comes next."
date: 2026-11-11
order: 17
rank: 4
tags: [agents, engineering]
cover: /img/two-sessions/lifecycle.svg
---

This morning I opened a new Claude Code session in my personal repo to write this post. Before touching anything, it ran `git status`. Two changes were waiting there: a deleted Excel lock file in the car-maintenance folder and a new, untracked file at the root. I hadn't made either of them in this session. Another session had, at some point, and left them behind.

A month ago that would have been a small accident waiting to happen. A session finishing its own work would run `git add -A`, and someone else's half-done change would ride along in a commit called "blog: new post".

## The setup

The repo is one folder with about a dozen topics in it: the blog, family paperwork, a photo cleanup, a side project's launch checklist. Each topic has its own folder and a `notes.md`. One `INDEX.md` at the top has a row per topic. The whole thing is backed up to a private remote.

I usually have two or three sessions open, one per topic. A long photo job runs in one window while I draft something in another. Sometimes a session from yesterday is still sitting in a terminal tab.

## The wrong assumption

I assumed the sessions would notice each other. They are capable, they read the files, they run `git status`. Surely one would see the other's work and step around it.

They don't, and it isn't a capability problem. Each session sees its own conversation and the files on disk. It has no idea another conversation exists. Whatever another session did shows up only as an unexplained change in the working tree. Agents are eager to finish, so an unexplained change usually gets swept into the commit or "cleaned up".

If the files are the only thing they share, then the files are where the coordination has to live.

## What every session now does

The rules sit in the repo's `CLAUDE.md`, which every session loads automatically. There are three moments.

<figure>
<a href="/img/two-sessions/lifecycle.svg" target="_blank" rel="noopener"><img src="/img/two-sessions/lifecycle.svg" alt="A session's lifecycle in three stages. Start: pull with autostash, read the index, claim the topic and push the claim at once. During: work only in your own folder, commit small, stage only your own paths. End: write a pick-up-here note, release the claim, push and check nothing is left ahead."></a>
<figcaption>Start, during, end. Each step is something another session can see in git.</figcaption>
</figure>

**Start.** Pull with `--rebase --autostash`, because a plain `pull --rebase` refuses to run while anyone has uncommitted changes in the tree, and someone usually does. Read `INDEX.md`. Find the topic and claim it: the status column becomes `🔄 in progress (what, when)`. Commit and push that single line before doing anything else. The claim is a lock, and it only works if the other sessions can see it.

If the topic is already claimed and the claim is less than 24 hours old, the session stops and tells me. Older than that, it's a session I forgot to close, so the claim is cleared.

**During.** Work only inside your own topic folder and your own INDEX row. Commit small, at natural checkpoints. Never `git add -A`. Run `git status`, stage your own paths by name, and if something outside your folder changed, leave it alone and mention it. That is what happened this morning: the session listed the two strangers, said they weren't ours, and moved on.

**End.** Update the topic's `notes.md` with a "next session, pick up here" section. Replace the claim with a real status. Commit, pull, push, and check `git status -sb` shows nothing ahead. Never force-push.

## What it bought

<figure>
<a href="/img/two-sessions/before-after.svg" target="_blank" rel="noopener"><img src="/img/two-sessions/before-after.svg" alt="Before: two sessions write into the same tree, one runs git add -A and commits the other's half-done change. After: each session claims a topic, stages only its own folder, and the stranger's change stays where it was."></a>
<figcaption>The second session doesn't have to know the first one exists. It only has to stay in its own lane.</figcaption>
</figure>

| | Before | After |
|---|---|---|
| How a session knows a topic is busy | it doesn't | a pushed 🔄 line in `INDEX.md` |
| What gets committed | everything in the tree | the session's own paths, named |
| Someone else's change | swept in or "cleaned up" | left alone, reported to me |
| Picking up tomorrow | re-read everything, re-derive | read the pick-up-here note |
| Worst case | a mixed commit, a force-push | a stale claim, cleared after 24h |

None of this is new. It is what a team does with a shared repo: say what you're working on, don't commit other people's files, leave a note. The difference is that a human team does most of it out loud, in chat and standups. The sessions have no channel like that. They read the files when they start, so anything they need to know has to be written there.

The handoff note turned out to matter as much as the lock. A fresh session doesn't remember yesterday. With a "pick up here" section it starts on the right step in a minute instead of rebuilding the context from the commit log.

<div class="callout">
<p><strong>The rule:</strong> parallel agents coordinate through what they all read, not through being clever. Put the claim, the commit scope and the handoff in the repo, push the claim before the work, and treat any change you didn't make as somebody else's.</p>
</div>
