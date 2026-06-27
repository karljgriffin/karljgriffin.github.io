---
layout: post
title: "When Reloads Never End: Investigating HAProxy Process Accumulation in OpenShift Ingress"
date: 2026-06-27
description: "Investigating HAProxy reloads, long-lived connections, and process accumulation in OpenShift ingress."
tags:
  - openshift
  - haproxy
  - ingress
  - debugging
  - infrastructure-investigations
---

---

> A deep dive into OpenShift Ingress, HAProxy reloads, long-lived connections, and why a single annotation dramatically reduced router memory usage.

---

## Introduction

High memory usage? Probably a memory leak.

Intermittent 503s? Probably DNS.

Network timeouts? Probably the firewall.

Most of the time those are reasonable assumptions.

Sometimes they’re completely wrong.

Recently I investigated an OpenShift ingress controller exhibiting some unusual behaviour. Users were intermittently reporting HTTP 503 responses from multiple public endpoints. The failures were sporadic, difficult to reproduce consistently, and didn’t appear to correlate with any obvious infrastructure event.

Our investigation started where most networking investigations begin.

Was DNS failing?

Were security groups blocking traffic?

Was the load balancer unhealthy?

Were backend applications overloaded?

At first glance, everything appeared fine. The ingress controller itself was routing traffic. Backend applications were running normally. The infrastructure showed no obvious signs of distress, yet users continued reporting intermittent failures.

As the investigation progressed we discovered something much more interesting. Inside each router pod were over thirty HAProxy processes. Some router pods were consuming more than **24 GiB of memory**. Despite that, CPU utilisation remained surprisingly modest.

What initially looked like a networking problem eventually became an investigation into how OpenShift performs graceful HAProxy reloads, how long-lived client connections can unintentionally prevent old router generations from exiting, and why a single configuration annotation ultimately returned the entire ingress controller to a stable operating state.

This article walks through that investigation from beginning to end. We’ll examine the evidence that led to the final fix, the hypotheses that were ruled out, the experiments that were performed, and the lessons learned along the way.

> Disclaimer: Certain product names, hostnames and architectural details have been generalised or anonymised, but the investigation and technical findings remain unchanged.

---

## The Environment

The affected ingress controller consisted of:

- OpenShift Router (HAProxy)
- AWS Network Load Balancer (NLB)
- Public ingress
- Multiple router replicas
- Multiple applications exposed through the same ingress controller
- Heavy long-lived ingestion traffic from log forwarding agents to a centralised logging service

At the beginning of the investigation, the ingress controller was configured with:

```yaml
spec:
  replicas: 3
  tuningOptions:
    reloadInterval: 0s
```

In OpenShift, a value of 0s for `reloadInterval` means the ingress controller chooses the default value. In our environment, that meant the router could reload every five seconds.

Early in the investigation we made two mitigations:

- increased the router replicas from 3 → 5 → 7
- changed `reloadInterval` from 0s to 60s

This reduced the amount of reload churn and improved user-facing symptoms, but it did not fully resolve the issue. Memory continued rising, and HAProxy processes continued accumulating.

---

## The First Symptom

The investigation did not begin because we noticed high memory, but because users started reporting intermittent HTTP 503 responses while accessing public services. The failures were frustrating because they weren’t persistent. Refreshing the page would often succeed. Everything pointed towards some kind of transient networking issue.

Naturally we started with the usual suspects:

- DNS resolution
- Underlying network infrastructure
- Backend application health
- Route configuration
- Load balancer behaviour

Everything appeared normal. Applications were healthy. Routes existed. DNS resolved correctly. Pods were running. Nothing immediately explained the intermittent 503 responses.

---

## An Initial Mitigation

Without understanding the root cause, we made two early mitigation changes.

First, we increased ingress capacity. At the time the ingress controller was running only three router replicas. We increased that to five, and eventually seven.

Second, we reduced reload churn by changing the router reload interval from `0s` to `60s`.

Both changes helped.

Increasing replicas gave the ingress controller more capacity to serve user traffic.

Increasing the reload interval reduced how frequently new HAProxy generations were created.

The user experience improved, and 503 errors became noticeably less frequent.

At first this looked promising. Perhaps the ingress controller had simply been under-provisioned or reloading too aggressively.

However, after continuing to observe the environment, something still didn't fit. Although these changes reduced the frequency of failures, they didn't explain why the failures had happened in the first place.

Even more interestingly, memory consumption across the router pods continued climbing. Some router pods eventually exceeded twenty gigabytes of resident memory, yet CPU usage remained modest.

That combination raised an important question. If HAProxy itself were consuming all of this memory, why wasn't CPU increasing alongside it?

---

## Looking Inside the Router

The first thing we wanted to understand was whether each router pod contained a single HAProxy process (as most people would naturally expect) or something different.

A quick process listing immediately revealed something unexpected:

```
ps -eo stat,cmd | grep '[h]aproxy'
```

Instead of seeing one HAProxy process, we observed numbers like:

```
31
34
35
38
```

per router pod.

The memory wasn’t being consumed by one enormous HAProxy process. It was being consumed by **dozens of HAProxy processes running simultaneously**.

The next question became:

**Why were so many generations of HAProxy still alive?**

Answering that question required thinking about how the router actually performs configuration reloads.

---

## Understanding OpenShift Router Reloads

At this point we’d established two important facts:

1. Router pods contained dozens of HAProxy processes.
2. Memory consumption appeared to increase alongside the number of processes.

So we asked:

> Why doesn’t OpenShift simply restart HAProxy and keep only one process alive?

The answer lies in how OpenShift performs router reloads.

Unlike many applications, the router cannot simply stop HAProxy and start it again whenever configuration changes. Doing so would immediately terminate every active TCP connection passing through the router.

That would mean:

- WebSocket connections would disconnect
- Large downloads would fail
- Long-running API requests would be interrupted
- Clients would experience connection resets

Instead, OpenShift performs what is known as a **graceful** (or **soft**) reload.

The process looks roughly like this:

```
                   Existing traffic
                          │
                          ▼
                ┌──────────────────┐
                │   HAProxy #1     │
                │  (active router) │
                └──────────────────┘
                          │
                  Configuration reload
                          │
                          ▼
                ┌──────────────────┐
                │   HAProxy #2     │
                │ accepts NEW traffic
                └──────────────────┘
                          ▲
                          │
                Existing connections
                 remain on HAProxy #1

Eventually...
HAProxy #1 finishes serving its remaining clients
↓
HAProxy #1 exits
HAProxy #2 becomes the only remaining process
```

This behaviour is one of the reasons OpenShift routers are remarkably resilient. Configuration changes can occur without disconnecting existing users. From a client’s perspective, the reload is effectively invisible. Under normal circumstances, old HAProxy generations drain naturally as client connections complete. After a short period, the old process exits and only the newest HAProxy instance remains. In other words, multiple HAProxy processes during a reload are **completely expected**.

What isn’t expected is having thirty or forty of them. That distinction became important.

---

## Reload Frequency

Once we understood that multiple HAProxy processes were expected during reloads, the next question became:

> How often is the router actually reloading?

This detail mattered because the ingress controller originally had:

```
tuningOptions:
  reloadInterval: 0s
```

In OpenShift, 0s does not mean “never reload”. It means “use the default”. In our environment, that default meant the router could reload as frequently as every five seconds.

That was significant.

If old HAProxy generations were slow to drain, a five-second reload cadence could create new generations much faster than old ones could disappear.

As an early mitigation we changed the reload interval to:

```
tuningOptions:
  reloadInterval: 60s
```

After that change, we confirmed the router was reloading roughly once per minute by inspecting the router logs:

```
oc logs <router-pod> --since=10m | grep "router reloaded"
```

Each router consistently reported approximately:

```
9 reloads in 10 minutes
```

which is consistent with a reload interval of roughly one minute.

This told us two things.

First, the router was behaving as configured after the mitigation.

Second, increasing the reload interval reduced reload churn but did not fully solve the problem. Old HAProxy generations were still accumulating as time went on, just more slowly than before.

We knew at this point why new HAProxy processes were being created, but we just weren't sure about why old ones never disappeared.

---

## Measuring the Processes

Rather than simply counting processes, we wanted to understand how long each generation had existed.

Running the following command gave us a much clearer picture:

```
ps -eo pid,ppid,stat,etime,rss,cmd | grep '[h]aproxy'
```

The output looked something like this:

```
PID     ETIME     RSS
2301    00:59     780 MB
2278    01:59     631 MB
2267    02:59     676 MB
2256    06:03     575 MB
2245    07:03     657 MB
2234    08:03     676 MB
2223    11:13     639 MB
```

Several interesting observations immediately stood out:

- Every process had been created roughly one minute apart, matching the configured reload interval
- Each generation still occupied several hundred megabytes of resident memory
- None of the older generations appeared to be exiting, they simply accumulated

The router wasn’t leaking memory in the traditional sense. Each HAProxy process was behaving normally. The problem was that new processes kept arriving while old ones never left.

That shifted our investigation in a different direction. Instead of asking:

> “Why is HAProxy consuming so much memory?”

we started asking:

> “What is preventing old HAProxy generations from terminating?”

That led us to the real root cause.

---

## Following the Connections

For an old HAProxy process to exit, every client connection it owns must first complete. If even a handful of connections remain active indefinitely, the old process has no opportunity to terminate. So we started investigating connections.

Fortunately, HAProxy exposes detailed runtime statistics through its administrative socket. Using the runtime API, we queried the current backend session counts:

```bash
echo "show stat" | socat - /var/lib/haproxy/run/haproxy.sock

```

Among the many metrics available, one in particular caught our attention:

```
scur
```

This represents the current number of active sessions on each backend.

Sorting the busiest backends immediately produced something interesting:

```
frontend-default                   scur=4968
be_secure:logging-backend-a        scur=3377
be_secure:logging-backend-b        scur=1395
```

One observation was immediately obvious. Nearly every backend with a significant number of active sessions belonged to the centralised logging service. Most application routes had only a handful of concurrent connections. The logging ingestion endpoints of that service, however, maintained thousands.

This was our strongest lead so far.

---

## Digging Deeper with Prometheus Metrics

HAProxy also exposes a Prometheus metrics endpoint.

The runtime statistics above identified the busiest backends by their current session counts (`scur`). To validate those findings, I queried the router's Prometheus metrics endpoint, which exports the same underlying information in Prometheus format (`haproxy_backend_current_sessions`). Both interfaces independently confirmed that the overwhelming majority of active connections traversing the ingress controller belonged to the logging ingestion backends:

```
curl -u "$USER:$PASS" \
http://127.0.0.1:1936/metrics
```

Filtering for the `haproxy_backend_current_sessions` metric corroborated what we saw from the `scur` runtime statistics:

```
haproxy_backend_current_sessions
--------------------------------
frontend-default         5984
logging-backend-a        4586
logging-backend-b        1201
```

The pattern was clear. Everything else barely registered.

---

## Why the Logging Backend?

This initially surprised us. After all, the logging backend wasn’t failing. The log forwarding agent wasn’t reporting errors. Log ingestion appeared healthy. But that’s what made it interesting.

One client maintained long-lived persistent HTTP connections to a backend service. Those connections are intentionally long-lived. Under normal circumstances this is desirable. Reusing existing connections avoids repeated TLS handshakes and reduces overall overhead. The side effect, however, is that those connections rarely become idle.

Now consider what happens every minute when the router reloads:

1. A new HAProxy process starts.
2. New client connections begin using the new process.
3. Existing logging backend connections remain attached to the previous process.
4. One minute later another reload occurs.
5. Another HAProxy generation appears.
6. Those long-lived connections remain attached to the HAProxy process that originally accepted them.

Repeat that cycle every minute…

```
Minute 0
HAProxy A
│
├── Logging connection
└── User traffic
Minute 1
HAProxy B
│
├── New traffic
HAProxy A
│
└── Existing logging connection
Minute 2
HAProxy C
│
├── New traffic
HAProxy B
│
└── Previous connections
HAProxy A
│
└── Original logging connection
```

As long as those original client connections remain open, older HAProxy generations cannot terminate and the router accumulates multiple generations simultaneously.

Importantly, this **isn’t** a memory leak. Each HAProxy process is behaving exactly as designed.

The unexpected behaviour comes from the interaction between:

- graceful reloads
- frequent router reloads
- extremely long-lived client connections

None of those components are individually “wrong.”

Together, however, they created a situation where retired HAProxy generations never reached the point at which they could safely exit.

---

## Verifying the Hypothesis

At this point, the evidence finally formed a coherent picture.

Every graceful reload created a new HAProxy generation, while long-lived client connections prevented older generations from draining and exiting. Over time, those generations accumulated, with each one consuming several hundred megabytes of memory.

What had initially appeared to be a memory leak was actually the cumulative footprint of **many healthy HAProxy processes remaining alive simultaneously**.

The remaining question was no longer *what* was happening.

It was whether we could safely place an upper bound on how long a retiring HAProxy generation was allowed to survive.

---

## The Fix

Fortunately, OpenShift already provides a mechanism for exactly this scenario.

The ingress controller supports the following annotation:

```yaml
ingress.operator.openshift.io/hard-stop-after
```

This annotation defines the maximum lifetime of a retiring HAProxy generation.

We configured it as:

```
metadata:
  annotations:
    ingress.operator.openshift.io/hard-stop-after: 10m
```

Conceptually, the lifecycle changed from this:

```
Reload
↓
New HAProxy starts
↓
Old HAProxy continues serving existing connections
↓
Old process remains alive indefinitely
```

to this:

```
Reload
↓
New HAProxy starts
↓
Old HAProxy drains existing connections
↓
10 minute limit reached
↓
Old process exits
```

Rather than allowing retired generations to survive indefinitely, every HAProxy process now had a guaranteed maximum lifetime.

If our understanding of the problem was correct, the ingress controller should eventually reach equilibrium, with new HAProxy generations being created at roughly the same rate that older generations were retired.

---

## Measuring the Result

Before changing anything, we’d already established a baseline.

Typical router pods looked something like this:

```
HAProxy processes:	30–38
Memory usage:	        9–24 GiB
Reload frequency:	~1/minute
```

Immediately after applying the annotation we began collecting measurements every five minutes.

The very first sample looked exactly as expected:

```
11:17
Processes: 2–3
Memory: 0.8–1.5 GiB
```

Only a couple of active HAProxy generations existed because the router had only recently been restarted.

Five minutes later:

```
11:23
Processes: 8–9
Memory: 2.2–4.2 GiB
```

Again, exactly what we’d expect. Every minute another HAProxy generation appeared. Memory increased because additional processes existed simultaneously.

If our hypothesis was wrong, we expected those numbers to continue climbing forever.

Ten minutes later:

```
11:28
Processes: 10–12
Memory: 3–5 GiB
```

Still perfectly consistent with the theory. The router was accumulating one new HAProxy process roughly every minute.

The critical observation came after that.

---

## Waiting for the Plateau

The router continued reloading every minute. If the annotation wasn’t working, process counts would continue increasing exactly as before.

Instead, something more interesting happened.

Fifteen minutes passed:

```
Processes: 10–11
```

Twenty minutes passed:

```
Processes: 9–11
```

Thirty minutes:

```
Processes: 8–10
```

The curve had flattened. For the first time, the ingress controller had reached equilibrium. As older generations reached their configured maximum lifetime, they were retired at roughly the same rate that new generations were created.

The system had reached a steady state.

Graphically, the behaviour looked something like this:

```
Before

Processes

40 |                             *
35 |                          *
30 |                       *
25 |                    *
20 |                 *
15 |              *
10 |           *
 5 |        *
 0 +-----------------------------> Time


After

Processes

12 |             ***************
10 |          ****             
 8 |        ***                 
 6 |
 4 |
 2 |
 0 +-----------------------------> Time
```

That plateau was the strongest evidence we could have asked for. The annotation hadn’t merely reduced memory, it had changed the long-term behaviour of the ingress controller. The router now operated within predictable bounds.

---

## Memory Told the Same Story

Perhaps even more satisfying was watching memory usage. Before the change, router pods routinely consumed anywhere between:

```
9 GiB
16 GiB
22 GiB
24 GiB
```

There was no obvious upper limit. Memory simply continued increasing as additional HAProxy generations accumulated. 

After enabling `hard-stop-after`, memory settled into a much narrower range.

```
2–5 GiB
```

Individual pods still fluctuated depending on connection load, but they no longer exhibited the runaway growth that had originally triggered the investigation.

The ingress controller hadn’t become “lighter.” It had become **bounded**. We hadn’t reduced the memory consumed by HAProxy.We’d prevented an unlimited number of HAProxy generations from existing simultaneously.

---

## A Second Experiment: Can We Reduce the Number of Routers?

At this point the original problem appeared solved.

The ingress controller had reached a steady state:

- HAProxy generations no longer accumulated indefinitely
- Memory usage had stabilised
- Process counts remained bounded
- Router reloads continued normally

It would have been easy to stop there, but we asked another question:

> Now that the router is behaving properly, do we still need seven replicas?

Earlier in the investigation we’d increased the ingress controller from three replicas to seven in an attempt to reduce the intermittent 503 responses users were reporting. At the time we didn’t yet understand the underlying issue, so increasing capacity was a reasonable mitigation. Now that the root cause had been addressed, we wondered whether we could safely reduce the replica count again. If five replicas provided the same behaviour, we’d reduce resource consumption without sacrificing reliability.

So we repeated the experiment. The ingress controller was temporarily scaled from 7 replicas to 5 replicas, and we began collecting exactly the same measurements as before.

---

## Everything Looked Healthy…

Initially, the results were encouraging. HAProxy behaved exactly as we’d hoped.

Process counts remained stable:

```
8–9 processes
```

Memory remained bounded:

```
2–5 GiB
```

No uncontrolled accumulation returned. From the router’s perspective, everything looked healthy. If we’d only measured router memory, CPU and process counts, we might have declared success.

Fortunately, we had one more test.

---

## Looking from the Outside

Throughout the investigation we’d also been continuously probing the ingress controller from outside the cluster. The test itself was intentionally simple.

Every second we issued an HTTPS request to a public endpoint and recorded:

- HTTP status code
- Time To First Byte (TTFB)
- Total request time
- Any connection errors

The command looked like this:

```
HOST=<public-endpoint>
for i in {1..300}; do
  TS=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  IP=$(dig +short "$HOST" | head -1)
  curl -skI \
    --connect-timeout 5 \
    --max-time 15 \
    -w "$TS ip=$IP code=%{http_code} \
ttfb=%{time_starttransfer} \
total=%{time_total} \
err=%{errormsg}\n" \
    -o /dev/null \
    "https://$HOST/"
  sleep 1
done
```

Although simple, this became one of the most valuable measurements we collected during the investigation.

Unlike CPU or memory, it measured the only thing users actually care about:

**Can I actually reach the service?**

---

## An Unexpected Regression

With seven router replicas, the probe results were excellent.

Across hundreds of requests we observed:

- No failures
- Consistently low latency
- Responses completing in well under one second

Reducing the ingress controller to five replicas changed that picture almost immediately.

Among otherwise healthy requests we began seeing entries such as:

```
HTTP 000
```

```
Operation timed out after 15 seconds
```

Alongside those failures were occasional latency spikes:

```
TTFB = 5.6 seconds
```

compared to the normal range of only a few hundred milliseconds.

This was unexpected. Nothing inside the ingress controller appeared unhealthy. Memory remained stable. HAProxy process counts remained stable. CPU looked reasonable.

Yet external users were now experiencing noticeably worse performance.

---

## Two Different Problems

This experiment reinforced an important lesson. The original issue and the replica count turned out to be solving different problems.

The `hard-stop-after` annotation solved process accumulation. It prevented old HAProxy generations from surviving indefinitely.

Scaling the router deployment, on the other hand, affected how much traffic each router needed to handle.

Those are related concerns, but they are not the same problem.

A healthy ingress controller can still become more heavily loaded if fewer replicas are available to share incoming connections.

Likewise, an ingress controller with many replicas can still suffer from unlimited HAProxy accumulation if old generations never terminate.

Our experiments showed exactly that.

- Increase replicas:	Reduced user-facing failures, but did not stop HAProxy accumulation
- `hard-stop-after=10m`:	Eliminated unbounded process growth and stabilised memory
- Reduce replicas to 5:	Memory remained stable, but latency increased and intermittent request failures returned

That distinction was a valuable outcome. Without running both experiments, it would have been easy to conclude that one change explained everything.

In reality, the ingress controller was exhibiting two independent behaviours, each requiring its own solution.

---

## Returning to Seven Replicas

Based on the external probe results, we restored the ingress controller to seven replicas. Almost immediately, the latency profile returned to normal. The intermittent timeouts disappeared.

Most importantly, the router retained all of the benefits introduced by the `hard-stop-after` annotation:

- bounded HAProxy process counts
- bounded memory usage
- and consistently healthy request latency

By the end of the investigation we had arrived at a configuration that addressed both dimensions of the problem. The ingress controller no longer accumulated HAProxy processes indefinitely, and users once again experienced stable, reliable access to public services.

---

## Lessons Learned

Looking back, what strikes me is how misleading the initial symptoms were. The issue began as intermittent HTTP 503 responses reported by users.

The natural instinct was to suspect networking:

- DNS
- Firewalls
- Security Groups
- Network Load Balancers
- Backend applications

None of those turned out to be the problem. The ingress controller itself also appeared healthy at first glance. CPU utilisation wasn’t excessive. Pods weren’t crashing. Nothing obvious stood out. Only after digging much deeper into the router did the real picture begin to emerge.

That reinforced a lesson in distributed systems:

> **The observable symptom is often several layers removed from the actual cause.**

---

**Measure Before You Change**

Throughout the investigation we deliberately tried to avoid making configuration changes without first collecting evidence. 

Every hypothesis was accompanied by measurements:

- When we wondered whether the router was repeatedly reloading, we counted reload events
- When memory appeared excessive, we measured individual HAProxy processes
- When we suspected long-lived connections, we inspected backend session counts
- When we believed the fix had worked, we continued measuring for another forty minutes instead of stopping after the first successful result

Only after collecting that evidence did we make further changes. That approach made each subsequent decision significantly easier.

---

**Validate from Both Sides**

We measured the ingress controller from both perspectives.

Internally:

- CPU utilisation
- Memory consumption
- HAProxy process counts
- Process lifetimes
- Router reload frequency
- Backend session counts
- HAProxy runtime statistics

Externally:

- HTTP status codes
- Time To First Byte (TTFB)
- Total request latency
- Connection failures

Those external probes ultimately revealed something that the internal metrics alone never would.

When we reduced the ingress controller from seven replicas back to five, every internal metric looked healthy.

Process counts remained stable, memory remained bounded, CPU looked perfectly reasonable.

If we’d stopped there, we would probably have concluded the experiment was successful.

Instead, the external probe immediately showed intermittent timeouts and significantly increased latency.

That single measurement changed our decision.

It was a useful reminder that infrastructure can appear perfectly healthy while users are experiencing a degraded service.

Ultimately, users experience latency, and not process counts.

---

**One Change Doesn’t Explain Everything**

Another important lesson was resisting the temptation to attribute every improvement to a single change.

Initially, increasing the ingress controller replica count reduced the number of reported HTTP 503 responses.

Later, adding the `hard-stop-after` annotation completely stabilised HAProxy process accumulation.

Finally, reducing the replica count back to five reintroduced user-facing latency despite memory remaining healthy.

Those observations showed that we weren’t solving one problem.

We were solving two different problems that happened to interact.

Understanding that distinction prevented us from drawing the wrong conclusions.

---

**Understand the System Before Optimising It**

One interesting aspect of this investigation is that none of the individual components were actually behaving incorrectly.

HAProxy was performing graceful reloads exactly as designed.

The log forwarding agent was maintaining persistent connections exactly as designed.

The logging backend was accepting long-lived streams exactly as designed.

OpenShift was reloading the router exactly as configured.

It was only when those behaviours were combined that the unexpected outcome emerged.

Failures often arise not because a single component is broken, but because the interaction between several perfectly healthy components produces behaviour that nobody initially anticipated.

---

**Reproducibility Matters**

One request from a teammate during the investigation turned out to be particularly valuable:

> “Please document all of the findings, together with the commands used to extract them, so anyone can reproduce the investigation.”

That changed how we approached the remainder of the debugging session.

Rather than simply collecting screenshots, we documented every command, every measurement, every timestamp and every observation. That is what ultimately enabled me to write this article.

By the end of the investigation we had a repeatable procedure that anyone could execute on another ingress controller.

That documentation is arguably just as valuable as the configuration change itself.

Infrastructure problems rarely occur only once.

The next engineer who encounters similar behaviour won't have to rediscover everything from scratch.

---

## Appendix: Reproducing the Investigation

The following commands were used throughout the investigation and can be repeated against any OpenShift ingress controller.

---

**1. Check Router Memory Usage**

The first indication that something unusual was happening came from the router memory consumption:

```
oc adm top pod -n openshift-ingress
```

Typical problematic output looked like:

```
NAME                                   CPU      MEMORY
router-xxxxx                           1189m    22674Mi
router-yyyyy                           1171m    24574Mi
router-zzzzz                           1549m    11906Mi
```

Memory alone doesn’t prove a leak, but it tells you where to begin looking.

---

**2. Count HAProxy Processes**

Next determine how many HAProxy processes exist inside each router:

```
for POD in $(oc get pods -n openshift-ingress \
-l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=<router> \
-o name); do
echo "=== $POD ==="
oc exec -n openshift-ingress "$POD" -- sh -c \
"ps -eo stat,cmd | grep '[h]aproxy' | wc -l"
done
```

Healthy routers typically have only a handful of HAProxy generations.

During this investigation we observed values as high as:

```
31
34
35
38
```

---

**3. Inspect HAProxy Generations**

Simply counting processes isn’t enough.

Inspecting their age reveals whether old generations are actually disappearing:

```
ps -eo pid,ppid,stat,etime,rss,cmd \
| grep '[h]aproxy'
```

Example:

```
PID     ETIME      RSS
2301    00:59      780 MB
2278    01:59      631 MB
2267    02:59      676 MB
2256    06:03      575 MB
2245    07:03      657 MB
2234    08:03      676 MB
2223    11:13      639 MB
```

Notice the approximately one-minute spacing between processes, matching the configured reload interval.

---

**4. Verify Router Reload Frequency**

Confirm that router reloads are occurring as expected:

```
oc logs <router-pod> --since=10m \
| grep "router reloaded"
```

Typical output:

```
9 reloads in 10 minutes
```

This confirms that reload frequency itself is behaving normally.

---

**5. Inspect Runtime Backend Statistics**

HAProxy exposes a runtime socket containing live backend statistics:

```
echo "show stat" \
| socat - /var/lib/haproxy/run/haproxy.sock
```

Useful fields include:

- Current sessions (scur)
- Request rate
- Errors
- Backend health

During this investigation the busiest backends immediately stood out:

```
be_secure:logging-backend-a     scur=3377
be_secure:logging-backend-b     scur=1395
```

---

**6. Query Prometheus Metrics**

The router exports Prometheus metrics locally.

First obtain the credentials:

```
USER=$(cat /var/lib/haproxy/conf/metrics-auth/statsUsername)
PASS=$(cat /var/lib/haproxy/conf/metrics-auth/statsPassword)
```

Then query:

```
curl -u "$USER:$PASS" \
http://127.0.0.1:1936/metrics
```

One particularly useful metric is:

```
haproxy_backend_current_sessions
```

Filtering that metric immediately identified which services maintained the largest number of active connections.

---

**7. Identify the Busiest Backends**

A convenient one-liner:

```
curl -s -u "$USER:$PASS" \
http://127.0.0.1:1936/metrics \
| grep "^haproxy_backend_current_sessions" \
| sort -k2 -nr \
| head
```

Example output:

```
frontend-default          5984
logging-backend-a         4586
logging-backend-b         1201
application-a             4
application-b             4
```

This became one of the strongest pieces of evidence during the investigation.

---

**8. Monitor External Behaviour**

Infrastructure metrics only tell part of the story.

Always validate behaviour from the client’s perspective.

The following probe repeatedly measures:

- HTTP status
- Time To First Byte
- Total request time
- Connection errors

```
HOST=<hostname>
for i in {1..300}; do
TS=$(date -u +%Y-%m-%dT%H:%M:%SZ)
IP=$(dig +short "$HOST" | head -1)
curl -skI \
--connect-timeout 5 \
--max-time 15 \
-w "$TS ip=$IP code=%{http_code} \
ttfb=%{time_starttransfer} \
total=%{time_total} \
err=%{errormsg}\n" \
-o /dev/null \
"https://$HOST/"
sleep 1
done
```

This probe ultimately demonstrated that reducing router replicas from seven to five negatively affected user-visible latency despite the routers themselves appearing healthy.

---

**9. Capture a Complete Router Snapshot**

The following script gathers nearly every useful datapoint from a router into a single report.

It records:

- CPU
- Memory
- HAProxy process count
- Process lifetimes
- Router reload frequency
- Backend session counts
- Prometheus metrics

This became our primary evidence collection script throughout the investigation.

```
#!/usr/bin/env bash
set -euo pipefail
NS=openshift-ingress
IC=<ingresscontroller-name>
mkdir -p router-snapshot
REPORT=router-snapshot/router-report-$(date +%Y%m%d-%H%M%S).txt
{
echo "Timestamp: $(date)"
echo
echo "=============================="
echo "Pod Resource Usage"
echo "=============================="
oc adm top pod -n $NS | grep "$IC"
echo
for POD in $(oc get pods -n $NS \
-l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=$IC \
-o name); do
echo
echo "############################################"
echo "$POD"
echo "############################################"
oc exec -n $NS "$POD" -- sh -c '
USER=$(cat /var/lib/haproxy/conf/metrics-auth/statsUsername)
PASS=$(cat /var/lib/haproxy/conf/metrics-auth/statsPassword)
echo
echo "Processes"
ps -eo pid,ppid,stat,etime,rss,cmd \
| grep "[h]aproxy"
echo
echo "Reload Count"
grep -c "router reloaded" /proc/1/fd/1 2>/dev/null || true
echo
echo "Top Sessions"
curl -s -u "$USER:$PASS" \
http://127.0.0.1:1936/metrics \
| grep "^haproxy_backend_current_sessions" \
| sort -k2 -nr \
| head -10
'
done
} > "$REPORT"
echo
echo "Report written to: $REPORT"
```

Having a repeatable evidence collection script proved invaluable.

Rather than manually gathering metrics every time, we could produce a complete snapshot of the ingress controller in seconds and compare results before and after configuration changes.


