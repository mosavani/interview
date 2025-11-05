
# SI-13057 Final Presentation — Extended Speaker Notes (Natural 3–5 min per slide)

These are conversational, natural-sounding notes designed to be read aloud during the talk. Each section runs roughly 500–700 words — ideal for a 3–5 minute delivery per slide at a natural pace.

---

## **Slide 1 — Title: Debugging RecSys Timeouts: A Deep Dive into Network Saturation**

Alright, let’s start with what this presentation is really about.  
Every once in a while, you come across a production issue that looks small at first — maybe just a few alerts, a couple of retries here and there — but ends up revealing something much deeper about how your systems actually behave under pressure.

This is one of those stories. A few months ago, our Recommendation Service, or “RecSys,” began showing small but consistent timeout spikes during peak hours. The percentage of failures wasn’t huge — maybe one or two percent — but the pattern was unmistakable. These were happening at roughly the same time each morning, and then vanishing. So, of course, curiosity kicked in.

Our alerting system was doing its job, flagging these failures even though most end users didn’t notice anything. But the deeper worry was: *why now?* Nothing major had changed in our deployment or traffic patterns. And yet, these timeouts appeared as if the system was gasping for breath for a few minutes every day.

What makes this incident worth talking about is how it evolved — from what seemed like a transient latency blip to a full-blown investigation into our infrastructure limits. It’s also a reminder that in distributed systems, the line between software behavior and hardware constraints is very, very thin.

So today, I’ll walk you through how we unraveled the mystery, what we found, and the specific engineering and architectural lessons we drew from it.  

---

## **Slide 2 — The Problem**

To set the stage — the RecSys service is our personalized recommendation engine. It runs in one GKE cluster we call the “member” cluster, and it relies heavily on upstream machine-learning services like Prophecy and Alchemy that live in another cluster, the “recsys” cluster. These are connected through an Envoy-based service mesh that manages cross-cluster calls with mutual TLS, load balancing, and observability baked in.

Now, the symptom we started with was a set of timeouts — specifically `IndividualRequestTimeoutException` in the logs. For those unfamiliar with Finagle, that’s a per-attempt timeout, meaning a single network attempt didn’t get a response within its configured limit. It doesn’t necessarily mean the overall request failed — retries often succeed. But it’s still a strong signal that something’s not right.

The strange part was the timing. These weren’t random. They occurred mostly during morning hours, when traffic was ramping up, lasted for a few minutes, and then disappeared as if nothing happened. The system didn’t crash, didn’t throttle, just… stalled.

From an SRE perspective, that’s the worst kind of issue — no clear crash, no single service to blame, and only a vague pattern in timeouts. It’s what we call a “gray failure” — the system is technically up but silently struggling.

So the problem statement became: *Why are inter-cluster requests from RecSys to Prophecy/Alchemy timing out, even though CPU, memory, and mesh metrics look fine?*  

---

## **Slide 3 — Initial Suspects**

Whenever you’re debugging timeouts, there’s a mental checklist you go through.

First — maybe the **client’s timeouts are just too tight**. Finagle clients have per-attempt and overall timeouts, and if you configure them too close to the 95th percentile latency, you’re guaranteed to hit false positives when the system burps. So, was this just aggressive configuration?

Second — perhaps the **upstream services were too slow**. Prophecy and Alchemy are fairly heavy machine-learning systems, with large models and variable compute times. Tail latencies can spike without much warning. So maybe the timeouts were real, but caused by slowness on their end.

And third — could there be a **network or mesh bottleneck**? In a service mesh setup, there are multiple Envoy hops, connection pools, and potential queueing points. If something in the middle starts throttling, even briefly, you’ll see end-to-end latency blow up.

We treated each of these as a hypothesis and started testing from the most likely — the client side.

---

## **Slide 4 — Troubleshooting Client Timeouts & Retries**

We began by digging into the RecSys client behavior. Each Finagle client has a request-timeout (for individual attempts) and a retry budget. The interesting thing about Finagle is that a single logical request — what your code sees — can actually involve multiple network attempts under the hood.

So, when you see `IndividualRequestTimeoutException`, it doesn’t mean the user-facing request failed. It means one *attempt* hit the limit, and Finagle retried, usually successfully. These are what we call “soft failures.”

Our alerting, however, was firing on those soft failures — which made the issue look worse than it was. Still, it’s good that it did, because soft failures are like smoke — they’re early warnings of systemic stress.

We checked latency charts and discovered that p95 latency for downstream calls was almost exactly equal to the timeout value — around 500–600 milliseconds. That’s like setting a speed limit at the same speed everyone drives — you’re bound to trigger it constantly.

So the initial thinking was: let’s reduce the timeout slightly, fail faster, and let the retry mechanism work in our favor. But we held off until we saw what was happening upstream.

---

## **Slide 5 — Troubleshooting Upstream Performance**

We turned our attention to Prophecy and Alchemy. Metrics from those services showed that, yes, Prophecy’s p99 latency was sometimes over 1200 milliseconds. That’s double what the client could tolerate before timing out. But there wasn’t a clear pattern tying this to the specific times we saw issues.

We tried an experiment — lowering Prophecy’s CPU requests. That sounds counterintuitive, but by requesting less, the Horizontal Pod Autoscaler became more aggressive, scaling up faster when traffic spiked. That did help reduce tail latencies, but not eliminate the problem.

We also found multiple Finagle clients in RecSys pointing to the same upstream service — meaning they each had separate connection pools and retry budgets. That can cause unnecessary retry storms, but again, it wasn’t the smoking gun.

By this point, the usual suspects were ruled out. The app code was fine, the proxies were healthy, and the metrics didn’t show resource starvation. So we zoomed out to the bigger picture.

---

## **Slide 6 — Environment: The Bigger Picture**

To really understand what’s happening, we have to visualize the entire path.

A request leaves the RecSys app, goes through its local Envoy sidecar, out to a GCP internal load balancer, into an ingress gateway in the recsys cluster, and finally into another sidecar and the Prophecy or Alchemy app.

That’s a lot of hops — but this design is standard for service mesh setups. The twist here is the *payload size*. While most of our RPCs are a few kilobytes, these requests were carrying 1 to 2 megabytes of data — user facts, feature vectors, and recommendation context. Every call moved a lot of data.

I confirmed this by querying Envoy’s internal metrics and also doing a tcpdump to capture packets. Sure enough, each transaction was a few megabytes in size. That meant even a moderate increase in traffic could cause a big spike in network throughput.

And that’s when things started to make sense.

---

## **Slide 7 — The “Aha!” Moment**

After correlating several datasets — application metrics, Envoy stats, and GCP node-level monitoring — we found something striking.

Every time these timeouts occurred, the network egress on the ingress gateway nodes flat-lined at exactly 16 Gbps. That’s not a random number; it’s the maximum bandwidth for N1-standard GKE nodes.

Once the NIC hits that cap, it starts dropping packets. TCP then goes into congestion control, retransmits, and the whole thing slows to a crawl. From the client’s perspective, that looks like a timeout.

And because Kubernetes doesn’t schedule pods based on network usage, it had placed multiple high-bandwidth ingress gateway pods on the same few nodes. So even though the cluster had spare capacity, certain nodes were overloaded on network.

That was the “aha!” moment — the system wasn’t failing because of code or configuration, it was hitting a *physical ceiling*. The NIC itself was the bottleneck.

---

## **Slide 8 — The Culprit: Network Saturation**

Let’s unpack that a bit. When a NIC is saturated, its transmit queue starts dropping packets. TCP expects acknowledgements (ACKs) for every packet it sends. When it doesn’t get them in time, it assumes congestion and reduces its sending rate — what we call the congestion window, or cwnd. It then retransmits the missing packets, waits for new ACKs, and gradually ramps back up.

All of that adds delay. Each retransmission adds hundreds of milliseconds, which bubbles up as request latency. When those retransmissions stack up, the client times out — not because the server’s slow, but because the packets are taking too long to make the round trip.

It’s a perfect example of a gray failure: everything’s “up,” but the experience is degraded.

The ironic part? The fix didn’t require touching a single line of code.

---

## **Slide 9 — The Solution**

Once we knew the problem, the fix was straightforward.

We increased the number of ingress gateway pods so that Kubernetes was forced to schedule them across more nodes. That increased the total available network bandwidth from 12 nodes to 16 — roughly a 33% improvement. And just like that, the alerts stopped.

For the long term, we’ve started migrating those nodes to N2D-standard machines, which have double the network bandwidth. We’re also introducing topology spread constraints to make sure high-bandwidth pods never get scheduled on the same node again.

On the application side, we’re exploring payload optimization — maybe compressing large payloads or switching to more efficient formats like Protobuf. That’s an ongoing conversation between RecSys and Prophecy teams.

And we didn’t stop there — we also improved observability. Now we track node-level NIC metrics and correlate them directly with timeout patterns.

---

## **Slide 10 — Q&A: Role-Play with SREs**

Let’s simulate a realistic SRE Q&A.

**SRE:** “Why didn’t you just increase the client timeouts?”  
**Presenter:** Because that would only mask the problem. Longer timeouts would make users wait longer without solving the underlying saturation. Instead, we optimized for faster retries and better spreading of load.

**SRE:** “How do you know it wasn’t an Envoy bug?”  
**Presenter:** Great question. During incidents, Envoy CPU stayed low, pending requests were zero, and no retransmits showed up at the proxy level. The only metric that spiked was node egress. Scaling nodes immediately fixed it. That’s a strong causal link.

**SRE:** “Did lowering CPU requests on Prophecy really help?”  
**Presenter:** Temporarily, yes — it made scaling more responsive, which smoothed tail latency. But it didn’t touch the root cause. The main problem was congestion at ingress.

**SRE:** “What’s preventing this from happening again?”  
**Presenter:** Three layers of prevention: more network headroom (N2D nodes), better pod distribution, and observability. Plus, ongoing payload optimization so we don’t grow into this limit again.

---

**Closing:**  
This entire episode reminded us that not all latency lives in software. Sometimes, the bottleneck is literally the wire.  
And if you’re not watching network throughput at the node level, you’re flying blind.

That’s the story of SI-13057 — a small blip that taught us a big lesson about distributed systems at scale.
