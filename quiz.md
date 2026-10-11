# Quiz — Prometheus + Grafana Observability Stack

Test yourself. Answers are collapsed — no peeking. 😏

---

### Q1. Pull vs push

Prometheus *pulls* metrics from targets instead of having apps *push* them. Name **two** operational advantages of the pull model.

<details>
<summary>Answer</summary>

1. **Service discovery & health detection for free** — if a target stops responding to scrapes, Prometheus knows it's down (`up == 0`) without any extra heartbeat system.
2. **No client-side buffering/backpressure** — the app never blocks or buffers metrics when the backend is slow; a down Prometheus just means a gap in the graph, not a crashed app or lost in-memory queue. (Bonus: scraping is trivially debuggable — `curl` the `/metrics` endpoint yourself.)

</details>

---

### Q2. RED vs USE

Your checkout API's p99 latency tripled, but CPU, memory, and disk on the hosts look fine. Which method — RED or USE — told you about the problem, and what does that tell you about where to look next?

<details>
<summary>Answer</summary>

**RED** (Rate, Errors, Duration) caught it — it's the method for request-driven *services*. USE covers *resources* (hosts, disks, queues), and they're healthy, so the problem isn't saturation: it's in the request path. Look at **traces** next — find which span got slow (downstream DB? a dependency?) — then pull the **logs** for that trace ID.

</details>

---

### Q3. The cardinality bomb 💥

Your Prometheus uses 40 GB of RAM and keeps OOMKilling. `topk(10, count by (__name__)({__name__=~".+"}))` shows one metric with 2 million series. What's the most likely cause — and the one-line fix?

<details>
<summary>Answer</summary>

**Cause:** an unbounded label value — e.g. `user_id`, `email`, or a raw request path with IDs (`/users/12345`) attached as a label. Every unique value × every other label combination = a new time series, and Prometheus holds them all in memory.

**Fix:** drop or aggregate the high-cardinality label at scrape time with `relabel_configs`:

```yaml
relabel_configs:
  - source_labels: [__name__]
    regex: 'my_app_requests_total'
    action: keep
  - regex: 'user_id'        # the offending label
    action: labeldrop
```

Rule of thumb: labels should have **tens** of values, never millions.

</details>

---

### Q4. Alert on symptoms, not causes

Which of these two alerts is better, and why?

- A: `avg(cpu_usage) > 80% for 10m` → page
- B: `error_rate > 5% for 5m` → page

<details>
<summary>Answer</summary>

**B.** A is a *cause*-style alert: high CPU might be totally fine (a batch job, a traffic spike being handled). B is a *symptom*: users are actually seeing failures. Alert on symptoms (error rate, latency, saturation per USE/RED); use cause-style signals on dashboards for diagnosis. Bonus: A pages you for something that may need no action — that's how alert fatigue starts.

</details>

---

### Q5. Correlating the three pillars

It's 3 AM: the `HighErrorRate` alert fired. In 30 seconds or less, describe the click-path in Grafana that takes you from the alert to the root cause using all three pillars.

<details>
<summary>Answer</summary>

1. **Metrics** — open the alert's dashboard panel: confirm the error-rate spike and narrow the time range to it.
2. **Traces** — click a trace exemplar (or jump to Tempo filtered by that time range + service): find the failing spans and spot which downstream call is slow/erroring.
3. **Logs** — take the trace ID into Loki (`{app="checkout"} |~ "trace_id"`) and read the exact error lines around it.

Metrics tell you *something's wrong*, traces tell you *where*, logs tell you *what*. That's the correlation superpower. 🔍

</details>

---

**Scoring:** 5/5 — you're on-call ready. 3–4 — solid, re-read the gotchas section. 0–2 — run `docker compose up -d` and click around; it sticks faster hands-on.
