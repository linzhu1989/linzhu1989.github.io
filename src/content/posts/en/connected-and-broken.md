---
title: "Connected, green, and broken"
description: "Our issue tracker moved to a new domain at five in the morning. By lunch every dashboard was green and the bot still couldn't create a ticket. The connection said Connected, the jobs said success, and the errors said permission denied. None of the three was telling the truth."
date: 2026-11-07
order: 20
rank: 11
tags: [engineering, automation]
cover: /img/connected/layers.svg
---

At five in the morning our issue tracker moved to a new domain. The vendor did it for every customer at once, and it was announced. By nine, reads still worked. The Slack bot could search tickets, add comments and move them across the board. It could not create or update one. Creates came back 500 or 302, depending on which tool you called.

Updating the URLs fixed the wiki side by mid-morning. The tracker side then changed its error. Writes now failed with 400 and 404, which looked exactly like a permissions problem. Every surface we owned said things were fine. The connection screen in the integration platform said **Connected**. The job history showed **success** for every failed create.

## The wrong assumption

We assumed that a green status meant the thing behind it worked. "Connected" meant the credentials were good. "Success" meant the ticket existed. A 404 meant the thing wasn't there and a 400 meant the body was wrong. So we spent the late morning looking at request bodies and field permissions.

All of those were wrong, and each for its own reason.

<figure>
<a href="/img/connected/layers.svg" target="_blank" rel="noopener"><img src="/img/connected/layers.svg" alt="A request path through four layers. The connection screen says Connected, because it only checks that a credential is stored. The job says success, because the flow ignores HTTP errors. Jira answers 400 or 404, because bad credentials are treated as an anonymous user. Only the last box, no ticket created, is the truth."></a>
<figcaption>Every layer reported green or something reasonable. Only the missing ticket was honest.</figcaption>
</figure>

**"Connected" checked that a credential was saved,** not that Jira accepted it. The stored Basic-auth credential no longer matched the new site.

**The 400 and 404 were an authentication failure in disguise.** Jira Cloud treats bad Basic-auth credentials as an anonymous user. An anonymous user can't see the project, so the request fails as if the project or the field doesn't exist. The only clue was a response header, `x-seraph-loginreason: AUTHENTICATED_FAILED`.

**"Success" was our own doing.** The integration flows had been built with HTTP errors ignored, so any response, 400 included, ended the job as a success. The job history was a list of calls made, not tickets created.

## How we found it

What cracked it was one boring test. I sent the same request, same URL, same body, with my own API token instead of the shared credential, and it created a ticket on the first try. That ruled out the body, the URL and the field mapping in one go. Only the credential was left. Once the credential was fixed in the afternoon, the original flows worked unchanged, pointed at the new domain. We had built a replacement flow on a different connection type as a fallback. It stayed switched off.

## The second service, quietly broken

That evening we checked who else still pointed at the old domain. A test-data service did. Its Jira client used the old base URL in config, and the old domain now answered every path with a 302 redirect to the new one.

<figure>
<a href="/img/connected/redirect.svg" target="_blank" rel="noopener"><img src="/img/connected/redirect.svg" alt="A GET to the old domain receives a 302 and the client follows it to the new domain by default. A POST or PUT receives the same 302, but the client doesn't follow redirects for writes by default, so the write never reaches the new domain and the caller gets an error or nothing."></a>
<figcaption>The same redirect, two behaviours: clients follow it for reads and stop at it for writes.</figcaption>
</figure>

HTTP clients follow a redirect on a GET without fuss. For POST and PUT they usually don't, because re-sending a write to a different place isn't safe to do on your behalf. So this service's searches, updates and issue links had most likely been failing since five in the morning. Nobody noticed, because nothing about it was loud. The fix was a one-line config change and a deploy. On staging, the call that had been returning 500 came back 200.

| | What it said | What it meant |
|---|---|---|
| Connection screen | Connected | a credential is stored |
| Job history | success | the call returned, any status |
| Jira 400 / 404 | bad request, not found | you're anonymous, check the auth header |
| Old domain on GET | 200 after a redirect | fine for now |
| Old domain on POST / PUT | 302 | the write never arrived |

## What changed

Three habits came out of that day. The flows should raise on 4xx and 5xx, so a failed create shows up as a failed job. After any change to a connection, verify with a real write and read back the result. A test-connection button doesn't count. And when a domain moves, search every service's config for the old host, not just the one that is shouting. The quiet ones are the ones still failing at midnight.

<div class="callout">
<p><strong>The rule:</strong> a green badge describes the badge. To know a connection works, make it do a real write and look for the result. Let errors surface as errors, and when you're stuck, swap one variable, such as the credential, while keeping everything else the same.</p>
</div>
