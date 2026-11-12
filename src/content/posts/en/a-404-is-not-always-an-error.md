---
title: "A 404 is not always an error"
description: "I wanted clearer error messages from a service that creates test bookings. The first draft treated every upstream 404 and 5xx as a failure. It wasn't that simple: one 404 meant 'try the next date', the 'no flights' case was really a 200 with an empty body, and one quiet failure booked three extra reservations. Status codes alone can't tell you what happened."
date: 2026-10-24
order: 21
rank: 15
tags: [testing, engineering]
cover: /img/status-codes/matrix.svg
---

There is an internal service our test suites call to create test data: give it a route and a date, and it searches flights, books one, picks seats, and hands back a ready-to-use booking. When it fails, the caller gets one line and a 404, whatever went wrong. Automation retries three times, five seconds apart, and never reads the body. People staring at the test page have no idea whether to wait, change the date, or ask someone.

I set out to make the failures say something useful. The plan sounded simple: when an upstream call fails, say which service failed and how.

## The first draft

I walked the code with a coding assistant and ran the service locally against mocks, with a fault switch for every upstream. Thirty-six failure cases. The picture was bad. The processing chain returned `null` at every level and threw away the validator's own failure message. Four catch-all layers swallowed exceptions. The final response was rebuilt from counters, so most real reasons never reached the caller.

The assistant's draft followed naturally: every upstream 404 or 5xx is a failure, so stop and report it.

I stopped that draft. I know this service well, and I knew it was wrong: when flight search returns 404, that means there are no flights on that date, and the service is supposed to try the next one. Swallowing that 404 is the feature. A 404 from an external service, or even a 5xx, is not automatically an error. It depends on which endpoint said it, and sometimes on what the user asked for.

## Asking the real thing

So we stopped reasoning about codes and went to look. I put a fault proxy between the local service and the real staging environment. It could break a path, drop a parameter, strip a field, or inject a 5xx, a reset or a timeout, and the upstream answered for real.

<figure>
<a href="/img/status-codes/proxy.svg" target="_blank" rel="noopener"><img src="/img/status-codes/proxy.svg" alt="The local service calls real staging through a fault proxy that injects wrong paths, dropped fields, 5xx, resets and timeouts. Three findings: no flights is a 200 with an empty body, not a 404; a seat-map failure produced a success; a router failure booked three extra real reservations."></a>
<figcaption>Same service, real upstreams, faults injected one at a time. Three answers nobody had predicted.</figcaption>
</figure>

Three things came back that neither the assistant nor I had predicted:

- **"No flights" was a 200 with an empty body,** not a 404. My own rule from memory was out of date.
- **A seat-map failure ended in a success.** The booking came back looking fine, without the seats the test needed.
- **A routing failure booked three extra real reservations** in the partner's staging system before anything reported a problem.

There were smaller ones too. One "not initialised" 404 only showed its reason if the request asked for JSON. Reading the code, or remembering, would never have told us which of these the service did.

## The classification we ended up with

<figure>
<a href="/img/status-codes/matrix.svg" target="_blank" rel="noopener"><img src="/img/status-codes/matrix.svg" alt="A matrix of endpoint by status: flight search 200-empty is normal, try the next date; a seat-map 404 skips the item; a 5xx, reset or timeout from any service stops the run and names the service and who can help; repeated item failures also stop."></a>
<figcaption>The outcome is decided by endpoint, status and the request together, not by the status code alone.</figcaption>
</figure>

Every upstream answer now maps to one of three outcomes, decided by **endpoint × status × request context**:

- **Normal.** Expected answers like "no flights on this date". Carry on with the next date.
- **Skip the item.** One flight or one seat can't be used. Skip it, count it, keep going. If the same thing keeps failing, stop.
- **Stop.** The service itself is broken. Say which service and endpoint failed, and who can help.

There is no retry of the same call inside one request. A message only claims to know the reason when the answer has a shape we have actually seen. The service's own status codes to its callers stayed the same, so nothing downstream broke.

| | Before | After |
|---|---|---|
| How a failure is judged | by status code | endpoint × status × request |
| "No flights that day" | depends on which layer swallowed it | normal, try the next date |
| Seat map fails | quiet success, no seats | item skipped and counted |
| Routing fails | three extra bookings, then an error | stop at once, name the service |
| What the caller reads | one line and a 404 | which service, what happened, who helps |
| Logs per request | 85 KB | 6 KB |

One unglamorous fix along the way: the log search tool was cutting values at the first space, so half our new messages were truncated. Quoting them fixed it.

<div class="callout">
<p><strong>The rule:</strong> you can't classify errors from status codes, and you can't classify them from memory either. Put a fault proxy in front of the real dependency, break one thing at a time, and write down what each answer means for that endpoint. Then decide: normal, skip, or stop.</p>
</div>
