---
title: "A vLLM Deep Dive: How a Multimodal Cache Bug Froze Our Model Server"
description: "A deep dive into vLLM's multimodal cache, how one bug froze the engine while every health check passed, and what we changed."
date: 2026-11-17
tags:
  - vllm
  - llm-infra
  - observability
  - reliability
  - health-checks
draft: false
---

One afternoon earlier this year, our document AI stopped working. It didn't crash, it didn't throw errors, and it didn't set off a single alert. It just quietly stopped doing any work.

For a while, almost every document sent to it either hung or failed. Through all of that, every dashboard we had said things were fine. Docker had been calling the container `(healthy)` for weeks, and it kept saying so. We found out the way nobody wants to: from outside our own monitoring.

The fix, once we found it, was quick. This post is about the stretch before it, and why none of our checks saw it.

> A quick heads-up before we start. I don't enjoy long articles that are just walls of text either, and this one is long. So I've kept the incident as simple as I could and drawn a few diagrams to show what's happening under the hood. Hopefully it saves you some trouble somewhere, someday (it probably won't but it's a fun read!)

## A bit of context

We extract data from documents with a vision-language model that we host ourselves on [vLLM](https://github.com/vllm-project/vllm).

Our backend services never talk to the model directly. They send documents to a FastAPI gateway that I've done a lot of the work on. For each document, the gateway:

- checks the caller's API key,
- turns the document into page images and a prompt for the task, and sends that to vLLM,
- wraps the request in an async job, so the backend submits it and polls until the result is ready.

It keeps a fixed-size pool of connections open to vLLM, and it exports Prometheus metrics and health snapshots along the way.

In short: **backend service → gateway** (auth, page images and prompt, async job) **→ vLLM on the GPU**. The result goes back to the gateway, and the backend picks it up on its next poll.

## The first clue

The first thing I did once we knew something was wrong was the simplest test I could think of. I sent a tiny 5-token text request straight to vLLM. No images, nothing clever. It hung. The engine wasn't slow, it was dead, and nothing in our monitoring had noticed.

To explain why, I need to explain one optimisation inside vLLM.

## Two processes and a shortcut

vLLM runs as two separate programs on the same machine:

- **The API server** receives HTTP requests and does the CPU-heavy image preparation (decoding, resizing, and cutting the image into the patches the model expects). The result is a block of numbers much bigger than the original JPEG.
- **The engine** runs the model on the GPU, and needs that block to do its work.

![Our gateway sends HTTP to the vLLM API server, which passes a fingerprint (plus the image if it is new) to the engine, which drives four GPU workers](blog/diagrams/vllm-multimodal-cache-bug-deep-dive/01-vllm-processes.svg)
*Drawn with four GPUs. The number of workers doesn't matter for this story.*

Sending that block across for every request is expensive, so vLLM takes a shortcut. It gives every image a **fingerprint**, a hash of its bytes:

- **The first time** it sees an image, the API server sends the fingerprint **and** the prepared image, and the engine stores it under that fingerprint.
- **Every time after that**, it sends **only the fingerprint**, and the engine reuses its stored copy.

So each process keeps its own cache (the engine holds the prepared images, and the API server tracks which ones it has sent).

When I was going through the incident, one question kept nagging me: why does vLLM have this cache at all? Why would the same image ever arrive twice?

The answer is chat. Chat APIs are stateless, so in a multi-turn conversation about one photo, the client resends that photo on every turn. Ten turns, ten copies of the same image. That is the case the cache was built for, and on repeats like that vLLM's own benchmark showed [up to 70% more throughput](https://github.com/vllm-project/vllm/pull/20452).

We don't do chat, but it applies to us too. Several steps in our pipeline look at the same page, and every retry sends it again.

## The shelf and the notebook

The engine's cache is like a **shelf with limited space** (4 GB by default). When it's full and a new image arrives, it throws out the one that was **least recently used**, the one nobody has asked for in the longest time.

Now the important bit. How does the API server know whether the engine still has an image, or has already thrown it out? **It doesn't ask.** Asking would mean a round trip between the processes for every image, which is exactly the cost the shortcut exists to avoid.

Instead, the API server keeps **its own notebook** (a list of fingerprints it believes are on the engine's shelf). The notebook has the same size limit and the same "throw out the least recently used" rule. If both sides see the same images in the same order, they throw out the same ones at the same moment, and the notebook always matches the shelf.

Here's how it's meant to work, with a shelf that holds three images:

![The API server and the engine cache images A, B and C, reuse A by fingerprint, and both throw out B when D arrives, so notebook and shelf stay in step](blog/diagrams/vllm-multimodal-cache-bug-deep-dive/02-cache-in-step.svg)

## How the notebook and the shelf drift apart

The whole scheme rests on one assumption: **both sides see images in the same order.** Well, they don't.

The API server prepares many requests at the same time, and updates its notebook in whatever order that work happens. The engine runs many requests together on the GPU too, but it takes them in through a single intake step, one message at a time, and that's where it checks and updates its shelf. So the shelf is always updated in arrival order, and the notebook isn't. vLLM's maintainers describe it this way in [the fix for this bug](https://github.com/vllm-project/vllm/pull/46747): the API server *"processes multi-modal items concurrently while P1 ingests them serially"* (in vLLM's naming, P0 is the API server and P1 is the engine). Because "least recently used" depends on order, two different orders can make the two sides throw out different images.

A simplified example. Both sides hold A, B and C, with A the oldest. Two requests arrive close together (one repeats image A, the other brings a new image D).

![Two requests reach the engine in the opposite order to the one the API server recorded, so the engine throws out A while the API server still sends only its fingerprint, and the lookup fails with an AssertionError](blog/diagrams/vllm-multimodal-cache-bug-deep-dive/03-cache-out-of-step.svg)

The notebook says A is there; the shelf threw it out. The API server sends a fingerprint with nothing behind it.

That's what happened to us. A large batch arrived with a few hundred jobs, many of them sending the same page images again and again, the engine was handling dozens of requests at once, and somewhere in that burst the notebook and the shelf disagreed. The engine hit this:

```
AssertionError: Expected a cached item for mm_hash=...
```

It's a known vLLM bug, [#31404](https://github.com/vllm-project/vllm/issues/31404). That issue had been closed a few months earlier, but people kept reporting it afterwards.

## Why one missing image killed everything

A cache miss like that could have meant one failed request. It didn't, because of where the check sits. The assertion fires inside the engine's loop that **takes in new requests**, and an assertion there doesn't just fail one request. It stops that loop.

For a short while the engine kept working through what it already had, and its throughput fell to a trickle and then to zero. After that it accepted nothing at all, not even a text-only request with no image in it. That was my 5-token test.

## Why everything looked fine

I think this is the most useful part of the story, because none of our checks was broken. Each one was honestly answering a narrower question than we thought it was.

- **`/health` returned 200** because the API server answers it, and the API server was perfectly happy. Docker's healthcheck called `/health`, so Docker stayed green.
- **vLLM's queue showed zero waiting** because nothing could get into the queue. It really was empty.
- **The gateway was fast** the whole time, answering in well under half a second. Requests only stopped once they reached the model.

> Every check asked "is the process up?" (**liveness**). None of them asked "can it do the job?" (**readiness**).

![The Docker healthcheck stops at the API server, which is up, while real requests go on to the engine, which is stuck](blog/diagrams/vllm-multimodal-cache-bug-deep-dive/04-health-check-blind-spot.svg)

## How one dead loop became thousands of failed jobs

Our gateway made it worse. Every request sent to the frozen engine held one of the gateway's pooled connections for its full ten-minute timeout. Dead requests piled up until every connection in the pool was taken. From then on every new job failed in five seconds, because there was no connection left to give it.

By the end, a few thousand jobs had failed. The one mercy was that the failures were clean. Nothing got corrupted and nothing wrong was written. Those documents just came back without their AI fields and had to be reprocessed.

Recovery really was turning it off and on again. We restarted vLLM, then restarted the gateway to drop the stuck connections. Even that came with a small sting. The calling service resubmitted all of its timed-out jobs at the same moment, while vLLM was still warming up, and we got a second, shorter burst of errors. A textbook [thundering herd](https://en.wikipedia.org/wiki/Thundering_herd_problem).

## What we changed

> We didn't fix the vLLM bug that week. What we fixed was not knowing.

**We made the health check do real work.** A watchdog now sends a real one-token completion to vLLM every 30 seconds, over its own small connection pool, so a jammed main pool can't make a healthy engine look dead. The rule is simple. If the model can't produce one token, it isn't healthy, whatever `/health` says.

**We added a circuit breaker, and made it a little stubborn.** Three failed probes in a row open it; it takes two successes in a row to close it again. That asymmetry is deliberate. While the engine was limping, some requests still got through, and a breaker that reacts to every result would have flipped back and forth the whole time. While it's open, the gateway fails new jobs before they ever reach vLLM. Clients learn about it on their first poll, in about a second instead of ten minutes, and the connection pool never fills up.

![Circuit breaker: closed to open after 3 failed probes in a row, open to closed after 2 good probes in a row](blog/diagrams/vllm-multimodal-cache-bug-deep-dive/05-circuit-breaker.svg)

**We turned an accident into a decision.** That connection cap? It was httpx's default, and we had never revisited it. Now the pool size and timeouts are explicit settings, chosen on purpose, with the cap sitting comfortably above what the engine actually handles.

**We started alerting on outcomes.** We added Prometheus rules for "jobs are coming in but nothing is completing", for the job failure rate, and for "requests running with zero throughput". When we replayed the outage against that last rule, it would have fired long before we first heard about the problem.

**We wrote the restart down.** Restart vLLM, wait until it's really ready, then restart the gateway. It's boring, and it helps with the herd.

## How vLLM fixed it

Some time after our outage, vLLM merged [#46747](https://github.com/vllm-project/vllm/pull/46747), "Recover from P0/P1 processor cache drift", and shipped it in **v0.28.0**. I like the approach, because it doesn't pretend the drift can be prevented. Its description says the mirroring assumption:

> "cannot be guaranteed across a concurrent, separate-process front-end, so a drift should be recoverable rather than fatal."

So instead of asserting:

1. The engine raises a normal, typed error that lists every missing fingerprint in the request.
2. The API server crosses those fingerprints out of its notebook.
3. The retry sends the full image again, and the engine's shelf is refilled.

One bad lookup now costs one retry instead of the whole engine. The normal path, a cache hit, doesn't change at all.

There are two other ways out:

- `--mm-processor-cache-type shm` replaces the two mirrored caches with one shared cache behind a lock, so there's nothing to drift apart.
- `--mm-processor-cache-gb 0` turns the cache off, and you pay to prepare repeated images again.

## What I took away from it

- **Alive isn't the same as ready.** Liveness means the process is up; readiness means it can serve a request right now. A model server can be completely alive and completely useless. Kubernetes keeps those two apart for a reason, and you should too, whatever you deploy on.
- **The honest health check is the real request, made small.** For an LLM server, that's a one-token completion. Anything cheaper is checking something else.
- **A connection cap only protects you if requests give up quickly.** A pool of connections with ten-minute timeouts means one dead dependency can hold everything for ten minutes.
- **Alert on what your users would notice.** A busy GPU is a normal Tuesday. Zero completions while work keeps arriving never is.
- **Two copies kept in step by prediction will eventually disagree.** vLLM's fix and our watchdog are the same lesson from opposite ends. Assume the disagreement will happen, and make it cheap and loud when it does.

## Food for thought

Our fixes make this kind of failure visible and fast. They don't change the pattern underneath, where a gateway accepts any amount of work and passes it straight on, so one stuck engine can turn into thousands of failed jobs.

So if you were building a system like this to be reliable from the start, what else would you put in? I'd start with three things.

### 1. A bounded queue

*So the gateway can say "not now".*

- **What:** a queue in front of the model with two limits, how many jobs run at once and how many may wait.
- **How big:** [Little's law](https://en.wikipedia.org/wiki/Little%27s_law) gives the wait limit. Multiply the jobs the engine finishes per minute by the minutes a caller is willing to wait. Queueing more than that is pointless.
- **When it's full:** answer straight away with `429 Too Many Requests` and a `Retry-After` header. That is backpressure. Callers feel the slowdown immediately, and it doesn't build up inside the gateway.
- **In our outage:** jobs would have been turned away in milliseconds, long before they tied up the whole connection pool.

### 2. Rate limits measured in work

*So one heavy caller can't crowd out the rest.*

- **Why not count requests:** a short text request and a 20-page document aren't the same load.
- **What:** a token bucket per caller, charged by estimated prompt size. Hosted model providers limit their customers the same way, in tokens per minute.

### 3. Priorities

*So bulk work can't starve live traffic.*

- **What:** separate queues, with live traffic served first and a smaller share for batch work.
- **In our outage:** the batch that set this off would have waited its turn.
- **The alternative:** separate hardware is the cleaner fix. Priorities are the cheap one.

### And a few smaller ones

| Idea | Why |
|---|---|
| Retries with backoff and jitter, respecting `Retry-After` | The burst of retries during our recovery was a thundering herd we caused ourselves. |
| Deadlines that travel with the job | If a caller has already given up, its job should leave the queue, not still run on the GPU. |
| Queue depth and wait time as metrics | vLLM's own "waiting" counter read zero throughout, because requests never got that far. A queue you own is a signal you can trust. |
| vLLM 0.28 or later | A cache drift costs one retry and not the engine. |
| A cap on pages sent per document | One big document can't flood the engine. |

### The watchdog stays

A queue protects the engine from too much work. The watchdog tells you when the engine has stopped doing any work at all. A related bug in the same cache ([#47958](https://github.com/vllm-project/vllm/issues/47958)) hung the engine with no error and `/health` still green. It has since been fixed, but only an outside check that real work completes catches that kind of failure.

One honest limit: a watchdog like this only detects the problem. Wired in as a liveness probe, the same one-token check can trigger the restart too.

---

If I had to put the whole outage in one sentence: most of it was not knowing, some of it was working out what was wrong, and only a sliver was the fix. Everything we built afterwards goes after the first part.

## Further reading

If you want to go deeper, these are the references I'd start with.

**vLLM**

- [Architecture overview](https://docs.vllm.ai/en/latest/design/arch_overview/). The API server and the engine, and how a request moves between them.
- [Optimization and tuning: multi-modal caching](https://docs.vllm.ai/en/latest/configuration/optimization/#multi-modal-caching). The cache this post is about, and the settings that control it, including the shared-memory option.
- [Multi-modal data processing](https://docs.vllm.ai/en/latest/design/mm_processing/). How vLLM processes images and other inputs before the model sees them.
- [Production metrics](https://docs.vllm.ai/en/latest/usage/metrics/). The Prometheus metrics vLLM exposes, including the waiting-requests counter.
- [The cache-drift fix, #46747](https://github.com/vllm-project/vllm/pull/46747). The bug and the fix in the maintainers' own words.

**Health checks, overload and retries**

- [Implementing health checks](https://aws.amazon.com/builders-library/implementing-health-checks/) (Amazon Builders' Library). Shallow and deep checks, and how each kind goes wrong.
- [Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) (Amazon Builders' Library). Why turning work away early beats accepting it and timing out.
- [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) (Amazon Builders' Library). The thundering herd, and how to avoid causing one.
- [Liveness, readiness, and startup probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/) (Kubernetes docs). The distinction this whole post turns on.
- [Addressing cascading failures](https://sre.google/sre-book/addressing-cascading-failures/) (Google SRE book). How one stuck component takes the rest down with it.
