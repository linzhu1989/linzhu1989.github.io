---
title: "The gate only sees new code"
description: "A Java 25 upgrade passed the pull-request quality gate cleanly. The main branch then showed 63 issues in code nobody had touched. After we cleaned them up, it showed 63 again, because an older pipeline finished last. Two blind spots of a gate I still believe in."
date: 2026-11-09
order: 23
rank: 23
tags: [testing, engineering]
cover: /img/new-code-gate/two-views.svg
---

A few weeks ago I wrote about [holding new code to 100% coverage and zero new issues](/posts/one-hundred-percent-on-purpose/). I still do. This is about the part of the codebase that rule doesn't look at.

We upgraded an old Java service to Java 25. The pull request was small: build files, a base image, a couple of API changes. The quality gate on the PR went green. 100% coverage on new code, zero new issues, thousands of tests passing. We merged it and it deployed without trouble.

Then someone opened the main branch's dashboard. 63 issues, all in code the PR never touched.

## Where the 63 came from

Moving to a new Java version also switches on the analyser rules written for that version. Three of them fired across the old code:

- 51 places where a parameter is never used and Java now lets you name it `_`
- 11 unused local variables, newly reported under the updated rule set
- 1 call to an API that is marked for removal in the new release

None of it is dangerous. All of it had been there for years, and none of it was in the diff. The PR gate is set to judge **new code**, meaning the lines this change added or modified. That is exactly what makes it fair to hold at 100%: you only answer for what you wrote. It also means a change that alters how the whole codebase is judged gets judged on its few edited lines.

<figure>
<a href="/img/new-code-gate/two-views.svg" target="_blank" rel="noopener"><img src="/img/new-code-gate/two-views.svg" alt="The same upgrade seen twice. The pull-request gate looks only at the changed lines: zero issues, green. The main-branch analysis looks at all the code under the new rules: 63 issues in files the upgrade never edited."></a>
<figcaption>Same commit, two views. The PR gate is right about its slice. The slice is the problem.</figcaption>
</figure>

## The wrong assumption

I treated "the gate is green" as "the branch is clean". On a normal feature change those are close enough. On a toolchain change, like a new language version, a new analyser version or a new rule profile, they come apart. The change edits a few lines but moves the standard for every line.

So we did the cleanup as its own small PR: rename the 51 parameters, delete the 11 variables, replace the one deprecated call. Same bar as always: the gate on that PR green, all tests passing. Merged.

## Then it said 63 again

The main dashboard still showed 63.

For a few minutes it looked like the cleanup hadn't worked. It had. Two PRs had gone into main within minutes: an unrelated feature first, the cleanup right after. Each merge started its own pipeline on main. The cleanup's pipeline finished first and reported 0. Then the earlier pipeline, analysing the earlier commit, finished a few minutes later and reported its own result: 63. The dashboard shows whatever arrived last.

<figure>
<a href="/img/new-code-gate/last-wins.svg" target="_blank" rel="noopener"><img src="/img/new-code-gate/last-wins.svg" alt="Two pipelines on main. Pipeline 1 analyses the older commit and is slow. Pipeline 2 analyses the newer cleanup commit and finishes first with 0 issues. Pipeline 1 then finishes and overwrites the dashboard with 63. A re-run on the latest commit restores 0."></a>
<figcaption>The dashboard shows the analysis that finished last, not the one for the newest commit.</figcaption>
</figure>

We re-ran the analysis on the latest commit of main. 0 issues. The number on the dashboard had been honest about an old commit and wrong about the current one.

## What I check now

| | What I assumed | What is true |
|---|---|---|
| PR gate green after an upgrade | the branch is clean | the edited lines are clean |
| Who finds rule-driven issues in old code | the gate | whoever looks at the whole branch |
| Main dashboard after two quick merges | the newest commit | whichever analysis finished last |
| How to trust the number | it's on the dashboard | it names the commit you expect |

Two habits came out of it. After any change to the toolchain, I look at the full-branch analysis on main, not only the PR gate, and if the new rules found old debt, I clean it in a separate PR right away while the context is fresh. And before I believe a number on a dashboard, I check which commit it belongs to.

The new-code rule is still right. It keeps every change honest without making anyone fix the past. It just doesn't cover the one kind of change that rewrites what "the past" means.

<div class="callout">
<p><strong>The rule:</strong> a new-code gate judges the lines you changed, not the standard you changed. After a language, analyser or rule-set upgrade, read the whole-branch analysis. And a dashboard number is only as current as the commit it names: the analysis that finishes last wins.</p>
</div>
