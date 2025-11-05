
# SI-13057 Deep Dive — Final Presentation Package (Flow Review, Speaker Notes, 40‑min Script, and Role‑Play Q&A)

**Author:** You  
**Audience:** Staff/Principal Engineers, SREs, Platform/Traffic, ML service owners  
**Duration:** 35–40 minutes (≈3–4 minutes/slide)  
**Context:** Intermittent client timeouts between RecSys (member cluster) and ML services (Prophecy/Alchemy) in a multi‑cluster GKE environment with Envoy/mesh. Root cause traced to node‑level NIC saturation on N1 nodes and pod placement.  

---

## Part A — Flow & Transition Review (polished for seamless narrative)

**Narrative arc:** Mysterious timeouts → eliminate usual suspects → zoom out to environment → “aha!” bandwidth ceiling → fix and harden.  

| Slide | What it does | Transition you should say | Why it helps |
|---|---|---|---|
| 1. Title / Intro | Sets context, stakes | “This is a real incident. We’ll walk from first symptom to root cause and the fixes we shipped.” | Frames a detective story; primes for learning. |
| 2. The Problem | Quantifies error window and pattern | “With timeouts, you start at the edges: what’s failing, when, how often.” | Builds urgency with facts without overclaiming impact. |
| 3. Initial Suspects | Lays out hypotheses | “Let’s test each layer: client, server, then the network.” | Gives a test plan so the audience can follow the logic tree. |
| 4. Client Timeouts & Retries | ‘Soft’ vs ‘hard’ failures | “Soft failures aren’t user errors, but they are smoke.” | Recalibrates what errors mean; avoids false conclusions. |
| 5. Upstream Performance | Rules‑in server tail, rules‑out server as sole cause | “Quick tunings helped, but didn’t move the needle enough. So we zoomed out.” | Motivates the pivot from software to environment. |
| 6. Environment | Establishes multi‑cluster, high‑bandwidth path | “Here’s where the big picture matters: data size per request.” | Seeds the upcoming ‘aha’ on payload and bandwidth. |
| 7. ‘Aha!’ Moment | Correlates errors with a hard throughput ceiling | “When the NIC flat‑lines, the app times out.” | Reframes a ‘latency’ issue as a ‘throughput’ ceiling. |
| 8. Culprit | Explains NIC saturation → TCP backoff → timeouts | “This is a gray failure: everything ‘up’, just not fast.” | Makes cause→effect concrete. |
| 9. Solution | Immediate + long‑term fixes | “Capacity now, topology and hardware next, plus data hygiene.” | Shows discipline: mitigation and prevention. |
| 10. Q&A | Anticipated questions | “Let’s stress‑test the narrative.” | Demonstrates rigor and prepares for review. |

---

## Part B — Slide‑by‑Slide Speaker Notes (3–4 minutes each)

> Tip: Use the **bold bullets** verbatim; the indented lines are optional elaborations.

### Slide 1 — **Debugging RecSys Timeouts: A Deep Dive into Network Saturation**
- **This is a story about small symptoms revealing a fundamental limit.**  
  Low, intermittent error rates surfaced during morning peaks — enough to trigger alarms and demand answers.
- **Goal for today:** follow the evidence from first alert to root cause and the hardening we shipped.  
  We’ll separate *logical* failures from *physical* behavior, and show why the fix wasn’t in application code.

**Transition:** “Let’s start with exactly what we saw.”

---

### Slide 2 — **The Problem**
- **Symptom:** 1–2% spike in timeout exceptions on RecSys→Prophecy/Alchemy during morning peaks (few minutes, not daily, recurring).  
  Logs dominated by `com.twitter.finagle.IndividualRequestTimeoutException` (retryable, per‑attempt timeout).
- **Impact profile:** Minimal user‑facing failures, but an unhealthy level of soft errors; unmistakable smoke.  
  Alerts fired correctly; the pattern strongly suggested a deeper system issue.
- **Call path:** Cross‑cluster traffic (member → recsys) via sidecars, L4 ILB, and mesh ingress gateway.

**Transition:** “With timeouts, there are three classic suspects. We tested them in order.”

---

### Slide 3 — **Initial Suspects**
- **Client timeouts too tight?** Are p95s kissing our per‑attempt timeout (500–600 ms)?
- **Upstream tail latency?** Are Prophecy/Alchemy p99s flaring under load (1.2s+)?
- **Mesh/network bottleneck?** Any proxy hot spots, pool starvation, retransmits, or queuing?

**Transition:** “We begin where most timeouts start: the client.”

---

### Slide 4 — **Client Timeouts & Retries**
- **Soft vs hard failures:** `IndividualRequestTimeoutException` marks a *single attempt* timing out; Finagle retries under a budget.  
  Soft errors are *signals*, not necessarily member‑visible failures.
- **Evidence:** p95 latencies aligning with configured per‑attempt timeouts; retries firing at the boundary.  
  Larger timeouts would hide the signal and delay healthy retries.
- **Action:** Audited client timeouts vs observed p95/p99; confirmed several were out of tune; tightened where appropriate.

**Transition:** “If the client looks noisy but sane, we check the servers.”

---

### Slide 5 — **Upstream Performance**
- **Observed:** Prophecy’s tail latency (max p99) spiked to ~1200 ms at peaks.  
- **Experiment:** Lowered CPU *requests* to make HPA scale sooner — small relief, not a cure.  
- **Conclusion:** Upstream contributes to tails, but not sufficient to explain the sharp, periodic spikes alone.

**Transition:** “So we zoomed out to the whole path and the data itself.”

---

### Slide 6 — **Environment (Multi‑Cluster, High‑Bandwidth Path)**
- **Architecture:** RecSys (member cluster) → Envoy sidecar → GCP ILB (L4) → Mesh Ingress Gateway (recsys) → server sidecar → Prophecy/Alchemy.
- **Key differentiator:** **Payload size** — 1–2 MB per request/response on this path (unlike typical RPCs).  
  Verified via Envoy histograms and tcpdump/pcap analysis.
- **Hypothesis shift:** The issue might be throughput/volume, not only request rate or proxy CPU.

**Transition:** “Once we asked a bandwidth question, the graphs told a clear story.”

---

### Slide 7 — **The ‘Aha!’ Moment**
- **Correlation:** Spikes in client timeouts coincided with **flat‑lined egress** on specific nodes at **16 Gbps**.  
  That’s the N1 node NIC ceiling.
- **Interpretation:** A hard physical limit masquerading as software latency.  
  When egress hits a cap, packets drop; retries snowball; the app sees timeouts.

**Transition:** “Let’s unpack how a saturated NIC turns into app‑level timeouts.”

---

### Slide 8 — **The Culprit: Node‑Level Network Saturation**
- **Mechanics:** NIC saturation → Tx queue drops → TCP congestion control (cwnd collapse) → retransmissions → inflated latency.  
- **Scheduler effect:** K8s isn’t network‑aware — multiple high‑bandwidth ingress pods co‑located on a few nodes amplified the hotspot.  
- **Gray failure:** Everything remained ‘healthy’; only performance degraded at peak, then self‑recovered as traffic subsided.

**Transition:** “Once we knew the constraint, the fix path was straightforward.”

---

### Slide 9 — **The Solution**
- **Immediate mitigation:** Increase IGW replicas to force wider node distribution → +33% aggregate NIC capacity → alerts stop.  
- **Near‑term:** Migrate IGW nodepool to N2D (higher egress headroom); enable node‑level NIC monitoring dashboards.  
- **Structural:** Apply topology spread constraints / anti‑affinity for ingress; collaborate to reduce payload size (compression, serialization, data diet).

**Transition:** “I’ll wrap with questions we expect from SREs, and the detailed answers.”

---

### Slide 10 — **Q&A (Role‑Play: SRE Panel ↔ Presenter)**

> **Format:** Multiple SREs ask layered, adversarial questions; Presenter answers with data, trade‑offs, and follow‑ups.

#### Thread 1 — **Why not just raise timeouts?**
- **SRE:** “If retries succeed, why not bump the timeouts and stop alerting?”  
- **Presenter:** Increasing timeouts blunts the signal and **extends** the member’s critical path under congestion. With multi‑MB payloads, longer waits during NIC saturation just burn budget while TCP backs off. Tight per‑attempt timeouts + a bounded retry budget harvest healthy replicas faster and reduce tail latency. We also moved alerts to **logical** failures only.

#### Thread 2 — **Prove it’s the NIC, not Envoy bugs**
- **SRE:** “How do you know it’s not an Envoy regression or pool starvation?”  
- **Presenter:** During incidents, Envoy CPU <25%; no pool starvation, pending requests ~0, and mesh‑added latency flat. Node‑level egress pinned **exactly at 16 Gbps** while errors rose; adding nodes cleared both. This is signature **capacity capping**, not proxy thrash.

#### Thread 3 — **Why did HPA tuning help at all?**
- **SRE:** “HPA changes moved metrics. Doesn’t that implicate Prophecy?”  
- **Presenter:** HPA added pods, slightly smoothing tails by spreading work, but didn’t change the **ingress** bottleneck. It was symptomatic relief, not the cause. The strong correlation was with **ingress node** egress saturation.

#### Thread 4 — **Scheduler hot‑spotting**
- **SRE:** “How much did co‑location contribute?”  
- **Presenter:** Materially. Concentrating ingress pods on a few N1 nodes meant each node’s 16 Gbps cap was easier to hit. Topology spread constraints keep per‑node throughput below the ceiling and raise failure isolation.

#### Thread 5 — **Why not compression?**
- **SRE:** “Couldn’t we just gzip and keep the same hardware?”  
- **Presenter:** We’re evaluating compression and Protobuf; wins depend on payload entropy and CPU tradeoffs at peak. Even with 30–50% reduction, we still want **headroom** and **even distribution**; infra fixes are non‑negotiable for resilience.

#### Thread 6 — **Proof via reproduction?**
- **SRE:** “Did we reproduce in a testbed?”  
- **Presenter:** Yes — cross‑zone testbed showed the same request→connection assignment and rebuild behavior; saturating egress induced packet loss and timeouts. Mitigations eliminated the pattern.

#### Thread 7 — **Observability gaps**
- **SRE:** “What monitoring would have shortened time‑to‑root‑cause?”  
- **Presenter:** Node NIC throughput and drop counters by default; p99.9+ latency histograms; payload histograms from sidecars; correlation alerts between *soft* failures and node‑level egress.

#### Thread 8 — **Risk of recurrence**
- **SRE:** “What keeps this from coming back next tax season?”  
- **Presenter:** Three layers: (1) **Capacity**—N2D migration and replica spread; (2) **Placement**—topology constraints enforced in CI; (3) **Payload**—compression/format changes tracked as OKRs with load‑tests against bandwidth budgets.

---

## Part C — 40‑Minute Expanded Script (integrated narrative)

Below is a word‑for‑word script you can present. Expect natural pace ~130–150 wpm for 38 minutes including brief pauses.

### Slide 1 — Title (2–3 min)
Hi everyone — thanks for joining. This is a real production incident… [content condensed; see Part B bullets to guide paragraphs]. The important theme is how a small rate of “soft” timeouts exposed a hard capacity constraint. We’ll move quickly from symptom to systems thinking, and finish with actions we shipped.

### Slide 2 — The Problem (3–4 min)
[Expand each bullet from Part B into 2–3 sentences: pattern, timing, exception semantics, why this mattered operationally.]

### Slide 3 — Initial Suspects (3–4 min)
[Walk the hypothesis tree; define success criteria to rule in/out each domain; explain why starting with client avoids premature server blame.]

### Slide 4 — Client Timeouts & Retries (4 min)
[Explain Finagle logical vs physical attempts; why we alert on logical; why earlier retries beat bigger timeouts; how we audited timeouts vs p95/p99 and tuned outliers.]

### Slide 5 — Upstream Performance (3–4 min)
[Discuss Prophecy p99 behavior; the HPA tuning experiment; why incremental improvement did not eliminate peak‑time spikes.]

### Slide 6 — Environment (3–4 min)
[Draw the path verbally: sidecar → L4 ILB → IGW → sidecar → app. Emphasize 1–2 MB payloads; how we measured via Envoy histograms and tcpdump; why data volume alters the dominant bottleneck.]

### Slide 7 — The ‘Aha!’ Moment (3–4 min)
[Describe correlation study: align error spikes with node‑level egress, observe a perfect 16 Gbps plateau, explain why flat‑tops indicate hard caps. Mention no mesh CPU spike.]

### Slide 8 — The Culprit (3–4 min)
[Teach NIC saturation → TCP behavior → application timeouts. Explain Kubernetes scheduler’s lack of network awareness, and how co‑location creates hot nodes.]

### Slide 9 — The Solution (4–5 min)
[Immediate horizontal expansion of IGW, observed 33% capacity increase and alert cessation. Near‑term N2D migration; topology spread. App‑layer payload optimization roadmap; discuss tradeoffs of compression and serialization.]

### Slide 10 — Q&A Role‑Play (6–8 min)
[Run the role‑play threads above. Invite real questions and map them to the prepared themes.]

**Closing (1 min):** Not all latency is software. Watch physical ceilings; treat soft failures as canaries; invest in node‑level observability and topology discipline.

---

## Part D — Visual Aid Prompts (optional, for slide captions)

- A simple diagram of member→recsys path with large payload callouts.  
- Egress flat‑top chart at 16 Gbps overlayed with timeout rate.  
- Before/after IGW replica distribution across nodes.

---

## Part E — Appendix (copy‑paste snippets you can reuse in slides or notes)

### Definitions
- **Soft failure:** A per‑attempt, retryable client timeout. Logical request may still succeed.
- **Hard failure:** Overall deadline exceeded / retry budget exhausted — user‑visible.
- **Gray failure:** System is up; performance degradation causes emergent errors.

### Practical timeout recipe
- Per‑attempt timeout ≈ upstream p95–p97.  
- Overall deadline accommodates one fast retry within SLO.  
- Retry budget to avoid storms; alert on logical failures.

### Mitigation checklist
- Fan out ingress replicas across more nodes.  
- Enforce topology spread constraints.  
- Migrate to higher‑egress nodes.  
- Add node NIC dashboards + correlation alerts.  
- Compress / reformat large payloads where feasible.

---

## License / Sharing

Internal only. Remove business metrics for external sharing.
