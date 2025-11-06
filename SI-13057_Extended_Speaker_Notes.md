
# SI-13057 Speaker Notes:

These are conversational, natural-sounding notes designed to be read aloud during the talk. Each section runs roughly 500–700 words — ideal for a 3–5 minute delivery per slide at a natural pace.

---

## **Slide 1 — Title: Debugging RecSys Timeouts: A Deep Dive into Network Saturation**

Every once in a while, we all would have came across a production issue that looks small at first — maybe just a few alerts, a couple of retries here and there — but we end up figuring out something much deeper about how our systems actually behave under pressure. 

So this discussion is no difference. A few months ago, our Recommendation Service, or “RecSys,” began showing small but consistent timeout spikes during peak hours. The percentage of failures wasn’t huge — maybe one or two percent — but the pattern was interesting. These were happening at roughly the same time  on random morning, and then dispears. 

Our alerting worked as expected, flagging these failures. But the concerning part was: why in specific time,  Nothing major had changed in our deployment or infrastructure or configuration patterns. And yet, these timeouts appeared like mysterously.

What makes this incident worth talking about is how it evolved — from what seemed like a transient latency blip to a interesting investigation leading to our infrastructure bottelnecs. It’s also a reminder that in distributed systems, the line between application behavior and hardware constraints is very, very thin.

So , I’ll walk you through journey of resolve intermittent application timeout errors between GKE cluster

---
## **Slide 2 — About Me**

Quick introduction about: My name
------
## **Slide 3 — The Problem**

To set the stage — the RecSys is our personalized recommendation engine, its one of largest service in terms of scale and  equally important for business from revenue perspective. It runs in one  of GKE cluster we call the “member” cluster.

Destination services in the call path are in different GKE cluster, and those services particulary high resrouce intensive, model serving service - Prophecy and related feature store service - Alchemy.  

Problem is - in most simplistic way our recommendation service receives timeout errors from prophecy/alchemy service.  

They occurred mostly during morning hours, when email/push marking campaign start sending email to drive traffic to Creditkarma traffic was ramping up, lasted for a few minutes, and then disappeared as if nothing happened. 

From an SRE point of view, this was one of those “nothing’s really broken, but something’s not right” situations. We initially let it go thinking it is temporary or might disapear — or just a few scattered timeouts, no high impact obvious. Every service looked healthy on its own, there was no change, yet we kept seeing this subtle, repeating pattern.

User impact was unknown but considered to be minimal,  internally about 1% of calls were timing out. That’s small, but in reoccuring nature of error, indicated there more than just timeouts..

Service dashboard (except these timeouts looked) normal. 

---

## **Slide 4 — Environment: **

To really understand what’s happening,  lets take a look at zoomed in version of two cluster.  Member GKE cluster is hosting recommendation service and other services. Traffic routing happens via envoy side car internally.

A request leaves the RecSys app, goes through its local Envoy sidecar, out to a GCP L4 ILB, into an ingress-gateway again running envoy (L7) in the another cluster, and finally into another sidecar and the Prophecy or Alchemy app.

From service perspective, most services are written in scala and custom framework built on top of finagle. While you can think there is lot of network hops, but this design is standard for service mesh setups. 

---


## **Slide 5 — Initial Suspects**
After initially almost ignoring the problem, we focused our investigation in these areas. or this was the mental model when we started. 

First we thought maybe the **client’s timeouts are just too tight**. Finagle clients have per-attempt and overall timeouts, and if you configure them too close to the 95th percentile latency, you’re guaranteed to hit false positives when the system sees tail end issues like this. So, was this just aggressive configuration?

Second is, perhaps the **upstream services were too slow**. Prophecy and Alchemy are fairly heavy model serving or feature store system, with large models and variable compute times. Tail latencies can spike without much warning. So maybe the timeouts were real, but caused by slowness on their end.

And third — could there be a **network or mesh bottleneck**? In a service mesh setup, there are multiple Envoy hops, each hop brings its complexity,  connection pools, and potential queueing points. If something in the middle starts throttling, even briefly, you’ll see end-to-end latency blow up.

We started with each of these as a hypothesis and started testing from the most likely cause  as the error were coming from recommendation-service — the client side.

---


## **Slide 6 — Troubleshooting Client Timeouts & Retries**

The first clear symptom we saw was a pattern of timeouts — specifically, `IndividualRequestTimeoutException` showing up in the RecSys logs.
If you’ve worked with Finagle before, you’ll know this exception. It represents a per-attempt timeout — meaning one RPC attempt didn’t receive a response within its configured limit. It doesn’t necessarily mean the overall request failed; in most cases, Finagle retries automatically and the logical request still succeeds.

But even when those retries succeed, it’s a strong signal that something underneath isn’t keeping up — that the system is starting to show strain.

So, we started by digging into how the recommendation service’s client behaved or how it is configured. In Recsys service, each Finagle client defines both a request timeout (for individual attempts) and a retry budget (how many retries are allowed within a given window).

What’s interesting about Finagle is that a single logical request — what our application sees as one RPC call — might actually involve multiple physical network attempts behind the scenes. That distinction became crucial in understanding what was really happening here.

So, when you see `IndividualRequestTimeoutException`, it doesn’t mean the user-facing request failed. It means one *attempt* hit the limit, and Finagle retried, usually successfully. These are what we called “soft failures.”

Our alerting, however, was firing on failures* - which included this soft failures — which made the issue look worse than it was. Still, it’s good that it did, because soft failures are like early signal — they’re  warnings of some systemic stress.

We also checked latency charts and discovered that p95 latency for downstream calls was almost exactly equal to the timeout value — around 500–600 milliseconds.

So the initial thinking was: let’s reduce the timeout slightly, fail faster, and let the retry mechanism work in our favor. But service team was reluctnt to apply this setting and held off until we saw what was happening upstream.


We turned our attention to Prophecy and Alchemy. Metrics from those services showed that, yes, Prophecy’s p99 latency was sometimes over 1200 milliseconds. That’s double what the client could tolerate before timing out. But there wasn’t a clear pattern tying this to the specific times we saw issues. as I mentioned earlier, this service are inherently require high processing time, so in a way this was expected at p95.

Because this is expensive service - we use HPA. Thout it may look counter intuitve -  what we did is we set the lowering Prophecy’s CPU requests and the HPA to  became more aggressive, scaling up faster when traffic spiked. That did help reduce tail latencies, but not eliminate the problem.

We also found multiple Finagle clients in RecSys pointing to the same upstream service — meaning they each had separate connection pools and retry budgets. That can cause unnecessary retry storms, but again, it wasn’t the smoking gun.

By this point, the usual suspects were ruled out. The app code was fine, the proxies were healthy, and the metrics didn’t show resource starvation. So we zoomed out to the bigger picture.

---

## **Slide 7 — Troubleshooting: Mesh & Network**

At this point, we had gone through the usual suspects — timeouts, retries, and upstream performance — and still hadn’t found a clear cause. So the next step was to dig into the network path itself, especially the service mesh.

Given that all traffic between RecSys and its upstream services — Prophecy and Alchemy — flows through the Envoy-based service mesh, it made sense to look there. The mesh is where a lot of subtle issues can hide: things like connection reuse, queuing, or even circuit-breaking behaviors that don’t show up at the application layer.

So, we started tracing a single request end-to-end through the path: from the RecSys pod’s Envoy sidecar, out of the member cluster, through the GCP internal load balancer, into the mesh ingress gateway in the recsys cluster, and finally to the destination Envoy proxy next to Prophecy or Alchemy. We checked every hop for performance bottlenecks.

The first thing we saw was that both the sidecars and the ingress gateways were extremely healthy. CPU usage on all the Envoy containers was comfortably low — below 25%, even during the incident window. That immediately ruled out CPU starvation or load-induced throttling at the proxy layer.

Next, we looked at connection pools and pending requests inside the mesh. If there’s connection exhaustion or queue buildup, Envoy’s metrics will show it right away. But every pool we checked was well within limits. There were no pending requests waiting for a connection to free up, no visible queuing, and no evidence of saturation in any of the pools.

We also verified the latency added by the mesh itself, and it was remarkably consistent — a few milliseconds at most per hop. There was no sign of sudden spikes or abnormal variance. If the mesh were the culprit, you’d expect to see erratic delays or outlier latencies between sidecars or ingress gateways. We didn’t see any of that.

Then we went deeper, examining network-level signals like retransmits, connection resets, and 5xx errors. If something was failing in the mesh’s data plane, you’d expect to see those counters moving — but all of them were flat. Zero retransmits, no connection resets, and no spikes in 5xx responses. In other words, the mesh was performing exactly as designed.

At this point, it was both good and bad news. The good news was that our service mesh looked good, but we ran out of easiest explanation so far.  at this stage what i concluded is this wasn’t about configuration or service issue and likely something physical, or something deeper in the infrastructure layer that we hadn’t measured yet.

---

## **Slide 8 — Troubleshooting: Data Volume vs. Request Rate**

So after confirming that the mesh itself wasn’t the bottleneck - we had to take a step back and think differently.

When these  obvious explanations failed, it was a sign that we’re looking at the wrong dimension. We had spent a lot of time analyzing how many requests were being made — the request rate — but not enough time thinking about how much data each of those requests was carrying.

So we went back to the telemetry from the Envoy sidecars and started looking at a specific set of metrics: upstream_rq_payload_size and upstream_rs_payload_size. These metrics showing you how large is request and response payloads for each upstream call from recsys. We dont generally check these metrics cause, in most services, payload size is tiny and fairly consistent. 

The average payload size looked normal, But when we looked at the 99th percentile — they was really large. We were consistently seeing request and response sizes between 1 and 2 megabytes. For an internal RPC path, that’s huge.

Most internal microservice calls are in the range of a few kbs. But in this case, p99  was almost hundreds of times larger.  

So, every RecSys → Prophecy call was moving around one to two megabytes of data per request. That’s because these calls carry  structured recommendation data — user “facts,” feature vectors, and large response payloads from the machine-learning models. 


our thinking shifted to that, problem might not have been how many requests we were sending, but how much data we were moving across the cluster. Even if the request rate hadn’t increased drastically, just a moderate uptick in traffic volume combined with these 1-2Mb payloads could easily push our network bandwidth to its limits.

The system wasn’t failing because of a sudden flood of RPCs; it was failing because each RPC was so heavy that the network couldn’t keep up during spikes.

And the more we correlated metrics, the clearer it became that this was unique to the RecSys → Prophecy path. Other services weren’t hitting the same limits because their payloads were much smaller


This led us straight into the next stage of investigation, where we finally connected those large payloads and organic traffic growth to the network bandwidth. 
---

## **Slide 9 — Anomaly Detection & Correlation**
So by this point, we have good lead to pursue our investigation find the cause.

High payload revealation made us to look back again on traffic growth, and when we zoomed out far enough, we realized - organic growth. Over the past few weeks/months, traffic had been rising steadily — about 15 to 18 percent more than the previous month. 

On its own, that wouldn’t normally cause a any issues. But when we combine that traffic increase with what we’d just discovered  - potentially indicating large data movement across the network.

So to investigate further, we needed visibility deeper down in the stack — beyond the pods, beyond the mesh, down to the GKE node level. Historyically we never needed to look at the GKE node level,  meaning most of our standard service dashboards does not have these metrics; mainly because these data are in stackdrive or GKE's default observability in GCP, to access node metrics like network throughput, we needed to login to cloud console. 

And as soon as we gained the node level observability  it became very clear what going on. 

Every single time we saw a spike in  — those IndividualRequestTimeoutException — we saw the network throughput on specific ingress gateway nodes was hitting a some limits. 

it was flat-lining right at 16 Gbps. That number initially sounded arbitrary, it turned out to be the documented maximum network bandwidth for the N1-standard machines typpe we were using in that cluster. 

Whenever the node’s egress graph went flat at 16 Gbps, a few seconds later, the client timeouts would climb. Then, as traffic eased and the throughput dipped below the limit, the timeouts disappeared. The correlation seemed perfect. Timeout spikes and network saturation on some specific node was aligned almost one-to-one.

What was especially interesting was that it wasn’t every node. It was random subsets of nodes — whichever ones happened to be hosting multiple ingress gateway pods at that time. Kubernetes had scheduled them unevenly, so a few  nodes were handling a disproportionate amount of high-bandwidth traffic. That explained why the problem appeared sporadic — some mornings you’d hit the overloaded nodes and see timeouts, other days you wouldn’t.

The RecSys → Prophecy requests weren’t failing because of slow code or buggy proxies — they were timing out because the underlying nodes had simply run out of network bandwidth. The system wasn’t broken; it was maxed out.

---

## **Slide 10 — Culprit: GKE? Or Cloud?** 
So once we correlated the spikes in timeouts with those 16-gigabit flat lines on the node-level metrics, we knew the bottleneck was physical — but the next question was, whose bottleneck was it?
Was it something misconfigured on our side, or was this a limitation coming from the underlying cloud infrastructure?

To answer that, we started peeling the layers back one by one.

The first thing we confirmed was that there were no pod-level errors. No TCP retransmits, no connection resets, no queue overflows. Everything at the pod and proxy level was healthy. So the packets were leaving the pods fine — they were just hitting a wall somewhere after that.

When we looked at the GCP documentation and compared it with our metrics, the picture came into focus: these pods were running on N1-standard nodes, which have a hard egress cap of 16 Gbps per virtual machine. That’s a physical networking limit imposed by the cloud instance type, not a setting you can tweak or tune.

And the pattern matched perfectly — only a subset of nodes ever hit that cap. The same ones that were hosting multiple ingress gateway pods. So this wasn’t a global cluster issue; it was localized saturation on a few unlucky nodes carrying too much traffic.

In other words, the network interface cards on those nodes were getting pegged at their maximum throughput. Once that happened, packets started dropping silently. From the outside, everything still looked “up” — the pods were running, health checks were passing — but underneath, the NICs were throttling traffic.

So that raised the obvious question: is this a GKE problem, or is it our configuration?

And the honest answer is — a bit of both.

## **Slide 11 — Culprit: GKE? Or Cloud?** 

Let’s start with the platform side. Kubernetes — at least by default — is not network-aware. The scheduler only looks at CPU and memory when deciding where to place pods. It has no concept of network bandwidth as a resource. So it happily schedules multiple high-throughput ingress gateway pods on the same node, completely unaware that it’s stacking traffic on a finite network pipe.

On our side, that scheduling pattern was amplified by how our workloads were structured. The Prophecy model-serving pods are big — they have heavy CPU and memory footprints. Because of that, Kubernetes can only fit one or two per node. That left a lot of spare CPU room on some nodes, which the scheduler used to pack in multiple Ingress Gateway pods. And that’s where the trouble began.

Essentially, we ended up with a small set of nodes hosting multiple high-bandwidth gateways, all pushing megabytes of traffic per request to heavy model-serving backends. Even though the cluster as a whole had plenty of unused capacity, those particular nodes were maxing out their NICs at 16 Gbps.

This is a perfect example of what we sometimes call a “scheduler blind spot.” Everything looked balanced from a CPU and memory perspective, but from a network perspective, it was completely lopsided.

And because this was happening at the infrastructure level, none of our normal application monitoring caught it. There were no 5xx errors, no CPU spikes, nothing that would’ve triggered an alert. The only symptom visible to us was the client-side timeouts — those soft Finagle exceptions — hinting that something down below was breaking a sweat.

So to summarize: the root cause wasn’t a bug in GKE, and it wasn’t a bug in our code. It was an emergent behavior that came from the combination of (1) a network bandwidth cap in the N1 instance type, and (2) a scheduler that isn’t aware of network saturation when it places pods.

That combination is what produced the “gray failure” — the system was technically healthy, but its performance quietly degraded under load.

Once we understood that dynamic, the fix became much clearer — spread the ingress gateways across more nodes, move to a higher-bandwidth machine type, and introduce topology rules that keep us from over-packing high-traffic pods in the same place again.

And that leads us directly into the solution we implemented — how we scaled out, tuned placement, and started building toward a more resilient infrastructure.

---

## **Slide 12 - Summary**
At the end of the day, this isnt about any single bug or misconfiguration. It was the intersection of organic growth, data-heavy payloads, and a physical infrastructure ceiling we hadn’t hit before.

RecSys service was making request with high-payload to Prophecy. 

Combine that with a steady 18% organic traffic growth over the prior month, and we were quietly pushing a large  amount of data through the system — far more than our infrastructure was provisioned to handle.

Those requests traveled across clusters, through Envoy sidecars, load balancers, and ingress gateways  —  running on N1-standard GKE nodes.

And those particular node types have a hard network egress cap of 16 gigabits per second. It’s a limit at the hypervisor level — not tunable, not configurable.

What made this tricky was that the saturation wasn’t uniform. It wasn’t like the whole cluster suddenly hit the wall. It was localized NIC saturation — just a handful of nodes that happened to host multiple ingress gateway pods. 

CPU usage was fine, memory was fine, no crash loops, no restarts. Even the pods themselves reported green across the board. The only signals were at the node's network layer — packet drops and queue delays. 

Once the NIC started dropping packets, TCP’s congestion control kicked in. That added hundreds of milliseconds of latency per impacted request. From Finagle’s point of view, that delay exceeded its per-attempt timeout threshold, which triggered  IndividualRequestTimeoutException errors.

So what looked like a flaky client timeout was actually a symptom of packet loss under network saturation. And because Finagle retried those failed attempts, it unintentionally added even more load on the network, reinforcing the pressure — a subtle feedback loop that made the incident look random and transient.

The final piece was Kubernetes scheduling behavior. The default k8 scheduler,  is completely blind to network bandwidth. It balances pods based on CPU and memory, not throughput. That’s how we ended up with multiple ingress gateway pods co-located on a few  nodes, each pushing multi-megabyte payloads at full tilt. The scheduler thought it was being efficient; in reality, it was concentrating network traffic and amplifying the problem.



**Solution:**  
We structured our response in three layers: 
Immediate mitigation, 
short-term upgrades, and 
medium-term improvements both at the infrastructure and application levels.


first  we expanded the node pool capacity for the ingress gateways. By increasing the number of nodes by roughly 15 percent, we effectively spread the high-traffic pods across more machines. That alone increased the cluster’s aggregate network bandwidth and reduced the pressure on any single node’s NIC. It wasnt real fix but its the most easiest and quickest things we could do without redeployin or rearchitect anything.



At the same time, we enabled node-level monitoring in our observability stack. Previously, we were mostly blind to physical metrics like NIC utilization or packet drops. Now, we collect and visualize network throughput, retransmits, and saturation rates directly from the node layer. That visibility ensures that if this pattern ever starts emerging again, we’ll see it long before it becomes an incident.

okay, Moving to the short-term improvements, we made the decision to migrate the ingress gateway node pool to N2D-standard machines.
These newer instances have roughly double the egress bandwidth capacity — up to 36 Gbps per node — compared to the older N1 types we were using. 

Then, we looked at scheduler-level improvements. Kubernetes, by default, doesn’t take network bandwidth into account when placing pods. It balances based on CPU and memory, and To fix that, we applied Pod Topology Spread Constraints — essentially rules that tell Kubernetes to evenly distribute certain types of pods across available nodes and zones. This guarantees that no single node ends up hosting too many high-traffic gateways.

We’re also exploring a custom controller/ (CRD) that could make the scheduler more network-aware — by factoring in NIC usage or observed throughput as a soft scheduling signal. That’s a longer-term enhancement, but it’s a step toward smarter placement decisions for bandwidth-intensive workloads.

Finally, at  the application layer —  we also wanted to tackle the problem at its source: payload size.

The RecSys → Prophecy and Alchemy requests carry large user-context objects and model responses.  Even modest reductions in payload size — say, 30 to 40 percent — translate directly into saved bandwidth and lower latency at scale.

