# SLO & Error Budget Definition

<p align="center">
  <a href="./images/budget.png">
    <img src="./images/budget.png" alt="Set the SLO, calculate the error budget, and alert on fast burn" width="800">
  </a>
</p>

Define a reliability target for an online shop's Orders API. Calculate allowed failures and write an alert rule. Use a text editor and Python 3; no running service is needed.

## Objective

By the end of this lab, you can:

- Define which requests count as good or bad.
- Set a 99.9% target over the most recent 30 days.
- Calculate an error budget and write a fast-burn alert rule.

## Deliverable

<p align="center">
  <a href="./images/what-you-create.png">
    <img src="./images/what-you-create.png" alt="SLO document and YAML alert-rule file" width="700">
  </a>
</p>

An Orders API plan in `slo-document.md` and `orders-api-rules.yaml`, plus `calculate-budget.py`. Record three thresholds but implement only the fast rule. The diagram's medium rule is optional.

## Understand SLI and SLO

<p align="center">
  <a href="./images/which-requests-orginal.png">
    <img src="./images/which-requests-orginal.png" alt="Exclude cancellations and health checks, then count good and bad requests" width="650">
  </a>
</p>

A **service level indicator (SLI)** measures reliability. A **service level objective (SLO)** sets its target. Here, the SLI is the percentage of counted requests without a server error, and the SLO is at least **99.9%** over the most recent 30 days.

The SLI is the measured result; the SLO is the target. If 9,996 of 10,000 requests are good, the SLI is **99.96%**, above the **99.9%** target.

Lab 01 measures working time. This lab measures good requests, so busy periods count more than quiet periods.

An **HTTP status code** describes a response. This lab counts `2xx`, `3xx`, and `4xx` as good and `5xx` as bad. Exclude cancellations (`499`) and health checks (`/healthz`) first. Good means no server error; a `404` does not prove the customer placed an order.

Read the diagram from top to bottom: exclude skipped requests, split the rest into good and bad, then divide good by the total. `5xx` means codes 500 through 599.

Health checks are not customer orders, and `499` means the client cancelled. Exclude both before counting good and bad requests.

## Understand Error Budgets and Burn Rate

<p align="center">
  <a href="./images/burn-rate-alerting.png">
    <img src="./images/burn-rate-alerting.png" alt="Burn thresholds, paired time windows, and notification actions" width="800">
  </a>
</p>

The **error budget** is the allowance for failed requests. A 99.9% target allows **0.1%** failures. **Burn rate** divides the measured failure percentage by this allowance: `1.44% / 0.1% = 14.4x`.

A 99.9% target allows one failure per 1,000 counted requests, or 10,000 per 10,000,000 requests.

At **1x**, errors use the budget at the allowed rate. At **14.4x**, they use it 14.4 times as fast. Alerts help the team respond before the 30-day target is missed.

A **window** is the time period we look back over. Both windows must exceed the threshold. A page means an urgent notification; a ticket means a task to review later.

The long window checks persistent errors; the short window checks current errors. If the hour shows 2% but the last five minutes show 0.2%, the fast alert does not start. Both must exceed 1.44%.

| Threshold | Long + short window | Failure percentage | Action |
| --- | --- | --- | --- |
| 14.4x (fast) | 1 hour + 5 minutes | Above 1.44% | Page now |
| 6x (medium) | 6 hours + 30 minutes | Above 0.60% | Page now |
| 1x (slow) | 3 days + 6 hours | Above 0.10% | Create ticket |

The diagram's exhaustion times assume steady traffic and errors using a full 30-day allowance. This budget counts failed requests, not downtime.

With steady traffic and errors, `30 / 14.4` gives about **2.08 days** to use a full 30-day allowance. This is not a prediction of the budget currently remaining.

## Step 1: Define Good and Bad Requests

<p align="center">
  <a href="./images/which-requests-orginal.png">
    <img src="./images/which-requests-orginal.png" alt="Rules for choosing which requests enter the SLI" width="650">
  </a>
</p>

Use these rules throughout. Save them in Step 3.

| Request or response | Count as | Reason |
| --- | --- | --- |
| `200` or `302` | Good | No server error |
| `404` | Good | Client error, not server error |
| `499` | Skip | Client cancellation |
| `500` or `504` | Bad | Server error or gateway timeout |
| Any `/healthz` request | Skip | Health check |

Use `SLI = good / (good + bad) * 100`. For 999 good and one bad request, the SLI is **99.9%**.

**Example:** Exclude 30 requests from 1,000, leaving 970. With 965 good and five bad, the SLI is `965 / 970 * 100 = 99.4845%`. Skipped requests are not in the denominator.

---

## Step 2: Write the Budget Calculator

```text
Counted requests * allowed failure fraction = error budget
Error budget - actual failures = remaining allowance
```

**Explanation:** The calculator first finds the allowance, then subtracts the failures already recorded.

Create a practice folder in your terminal:

```bash
mkdir -p m3-l2 && cd m3-l2
```

**Explanation:** Creates and opens the folder for this lab's files.

**Command result:** A successful command normally prints nothing. The terminal is now inside `m3-l2`; save all three files there so the later commands can find them.

Use **10,000,000** counted requests and **4,000** server errors over the most recent 30 days. This is a **rolling window**: tomorrow's calculation adds the newest day and drops the oldest.

The budget does not reset at the start of a calendar month. Old requests leave the 30-day window as new ones enter it.

Paste this complete code into `calculate-budget.py`:

```python
"""Calculate the request-based error budget used in this lab."""

valid_requests = 10_000_000
bad_requests = 4_000
slo = 0.999
allowed_failures = round(valid_requests * (1 - slo))
remaining = allowed_failures - bad_requests
sli = (valid_requests - bad_requests) / valid_requests * 100

print(f"Valid requests:    {valid_requests:,}")
print(f"SLO target:        {slo * 100:.1f}%")
print(f"Allowed failures:  {allowed_failures:,}")
print(f"Actual failures:   {bad_requests:,}")
print(f"Failures remaining:{remaining:>10,}")
print(f"Budget used:       {bad_requests / allowed_failures * 100:.0f}%")
print(f"Budget remaining:  {remaining / allowed_failures * 100:.0f}%")
print(f"Actual SLI:        {sli:.2f}%")
print(f"SLO met:           {'Yes' if sli >= slo * 100 else 'No'}")
```

**What this file does:** Calculates allowed failures, remaining budget, and the good-request percentage.

| Measure | Calculation | Result |
| --- | --- | --- |
| Allowed failures | 10,000,000 * 0.001 | 10,000 |
| Remaining failures | 10,000 - 4,000 | 6,000 |
| Budget used | 4,000 / 10,000 * 100 | 40% |
| Budget remaining | 6,000 / 10,000 * 100 | 60% |
| Actual SLI | 9,996,000 / 10,000,000 * 100 | 99.96% |

Recalculate when the request total or window changes.

**Different denominators:** `4,000 / 10,000,000 = 0.04%` is the failure rate. `4,000 / 10,000 = 40%` is the budget used. One compares failures with all requests; the other compares them with allowed failures.

If failures rose to 12,000, the 10,000-failure budget would be exceeded by 2,000. Remaining budget would be negative, and the SLI would fall to **99.88%**.

---

## Step 3: Write the SLO Document

<p align="center">
  <a href="./images/burn-rate-alerting.png">
    <img src="./images/burn-rate-alerting.png" alt="Thresholds to record in the SLO document" width="800">
  </a>
</p>

Create `slo-document.md` in `m3-l2` and paste this complete document:

```markdown
# Orders API SLO

## SLI
- Owner: Orders API team.
- Source: production gateway request counts.
- Valid: completed requests excluding 499 and /healthz.
- Good: valid HTTP 2xx, 3xx, or 4xx responses.
- Bad: valid HTTP 5xx responses, including 504.
- SLI = good / valid. Count retries separately.

## SLO
At least 99.9% good requests over the last 30 days.

## Error Budget
Allowed failures = valid requests * 0.001.

| Measure | Value |
| --- | --- |
| Valid requests | 10,000,000 |
| Allowed failures | 10,000 |
| Actual failures | 4,000 |
| Remaining failures | 6,000 |
| Budget used | 40% |
| Actual SLI | 99.96% |

## Alerts

| Speed | Windows | Burn | Errors | Action |
| --- | --- | --- | --- | --- |
| Fast | 1h + 5m | >14.4x | >1.44% | Page |
| Medium | 6h + 30m | >6x | >0.60% | Page |
| Slow | 3d + 6h | >1x | >0.10% | Ticket |
```

**What this file does:** Records request rules, the target, budget calculations, and notification thresholds.

Count retries separately. Requests that never reach the gateway need outside monitoring. Review tickets the next business day; prioritize reliability fixes when the budget runs out.

---

## Step 4: Write the Fast-Burn Alert Rule

<p align="center">
  <a href="./images/fast-alert.png">
    <img src="./images/fast-alert.png" alt="Both windows exceed 1.44% for two minutes before the alert fires" width="700">
  </a>
</p>

**Prometheus** collects service measurements. Its query language, **PromQL**, calculates error rates from request counters. Create `orders-api-rules.yaml` and paste:

```yaml
groups:
  - name: orders-api-slo
    rules:
      - alert: OrdersAPIErrorBudgetBurnFast
        for: 2m
        expr: |
          (
            1 - sum(rate(http_requests_total{job="orders-api",code!~"5..|499",route!="/healthz"}[1h]))
            / sum(rate(http_requests_total{job="orders-api",code!="499",route!="/healthz"}[1h]))
            > 0.0144
          )
          and
          (
            1 - sum(rate(http_requests_total{job="orders-api",code!~"5..|499",route!="/healthz"}[5m]))
            / sum(rate(http_requests_total{job="orders-api",code!="499",route!="/healthz"}[5m]))
            > 0.0144
          )
        labels:
          severity: page
        annotations:
          summary: "orders-api is burning its error budget fast"
```

**What this file does:** Defines an alert when both error percentages stay above 1.44% for two minutes.

The rule expects `http_requests_total` with `job`, `code`, and `route` labels. A **counter** increases as requests arrive.

`rate()` calculates requests per second and handles counter resets. `sum()` combines matching counts. The query calculates `1 - good rate / total rate` to get the error fraction.

The filters skip `499` and `/healthz`; the good count also excludes `5xx`. `0.0144` means 1.44%. `and` requires both windows to exceed it.

**Example:** With 100 requests per second and 98 good ones, `1 - 98 / 100 = 0.02`, or **2%** errors. This exceeds 1.44%; the other window must also exceed it.

`for: 2m` keeps the alert pending for two minutes. It fires if both conditions stay true across evaluations. If either drops to or below the threshold, the pending period clears.

Lookback windows are not fixed waiting periods. Traffic and evaluation intervals affect detection. Notifications require a running Prometheus setup and notification configuration.

---

## Step 5: Run the Following Commands in the Terminal

From `m3-l2`, run the calculator after saving the files:

```bash
python3 calculate-budget.py
```

**Explanation:** Prints the error budget and actual SLI for this lab's request counts.

<p align="center">
  <a href="./images/budget-output.png">
    <img src="./images/budget-output.png" alt="Expected calculator output with 40% budget used and SLO met Yes" width="800">
  </a>
</p>

**What to expect:** The output shows **10,000** allowed failures, **6,000** remaining, and **40%** budget used. `SLO met: Yes` means the measured **99.96%** is above the **99.9%** target. The YAML file defines the alert; running this calculator does not send notifications.

**Read the output:** `Budget remaining: 60%` complements the 40% used. `Actual SLI: 99.96%` describes good requests out of all counted requests.

### Download the Rule Tool

<p align="center">
  <a href="./images/download-output.png">
    <img src="./images/download-output.png" alt="Prometheus archive download in progress at 11 percent" width="900">
  </a>
</p>

The screenshots use `promtool` from Prometheus 3.5.0 on Linux x86-64. Stay in `m3-l2`; these commands need `curl` and `tar`.

```bash
# paste this as one line
curl -fLO https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```

**Explanation:** Downloads the archive containing the rule tool into the current folder.

**Command result:** The screenshot shows **11%** downloaded, about **12.8 MB** out of **115 MB**. Wait until it reaches `100` and the prompt returns before extracting.

### Extract the Tool

<p align="center">
  <a href="./images/extract-output.png">
    <img src="./images/extract-output.png" alt="Extraction command and the promtool filename" width="900">
  </a>
</p>

```bash
# paste the tar command as one line
tar -xzf prometheus-3.5.0.linux-amd64.tar.gz prometheus-3.5.0.linux-amd64/promtool
ls prometheus-3.5.0.linux-amd64
```

**Explanation:** Extracts only `promtool`, then lists the folder containing it.

**Command result:** `tar` normally prints nothing; `ls` lists `promtool`. The screenshot wraps the long filename, but paste the extraction command as one line.

### Run the Rule Tool

<p align="center">
  <a href="./images/run-output.png">
    <img src="./images/run-output.png" alt="Rule tool output showing Checking orders-api-rules.yaml and SUCCESS: 1 rules found" width="900">
  </a>
</p>

Run this from `m3-l2`, where you saved `orders-api-rules.yaml`:

```bash
./prometheus-3.5.0.linux-amd64/promtool check rules orders-api-rules.yaml
```

**Explanation:** Loads the YAML rule file and reports whether its rule syntax is accepted.

**Command result:** `Checking orders-api-rules.yaml` names the file. `SUCCESS: 1 rules found` means its rule syntax was accepted. Use your `m3-l2` folder, not the screenshot's path. This does not send notifications.

---

## Conclusion

You defined a reliability target, calculated its failure allowance, and wrote a fast-burn alert rule. Next, explore load balancing and automated failover to reduce failures in the service itself.
