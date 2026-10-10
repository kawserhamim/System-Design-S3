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

An **HTTP status code** describes a response. This lab counts `2xx`, `3xx`, and `4xx` as good and `5xx` as bad. Exclude cancellations (`499`) and health checks (`/healthz`) first. Good means no server error; a `404` does not prove the customer placed an order.

## Understand Error Budgets and Burn Rate

<p align="center">
  <a href="./images/burn-rate-alerting.png">
    <img src="./images/burn-rate-alerting.png" alt="Burn thresholds, paired time windows, and notification actions" width="800">
  </a>
</p>

The **error budget** is the allowance for failed requests. A 99.9% target allows **0.1%** failures. **Burn rate** divides the measured failure percentage by this allowance: `1.44% / 0.1% = 14.4x`.

A **window** is the time period we look back over. Both windows must exceed the threshold. A page means an urgent notification; a ticket means a task to review later.

| Threshold | Long + short window | Failure percentage | Action |
| --- | --- | --- | --- |
| 14.4x (fast) | 1 hour + 5 minutes | Above 1.44% | Page now |
| 6x (medium) | 6 hours + 30 minutes | Above 0.60% | Page now |
| 1x (slow) | 3 days + 6 hours | Above 0.10% | Create ticket |

The diagram's exhaustion times assume steady traffic and errors using a full 30-day allowance. This budget counts failed requests, not downtime.

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

---

## Step 2: Write the Budget Calculator

<p align="center">
  <a href="./images/budget-output.png">
    <img src="./images/budget-output.png" alt="Budget calculation showing 10,000 allowed failures and 6,000 remaining" width="800">
  </a>
</p>

Create a practice folder in your terminal:

```bash
mkdir -p m3-l2 && cd m3-l2
```

**Explanation:** Creates and opens the folder for this lab's files.

**Command result:** A successful command normally prints nothing. The terminal is now inside `m3-l2`; save all three files there so the later commands can find them.

Use **10,000,000** counted requests and **4,000** server errors over the most recent 30 days. This is a **rolling window**: tomorrow's calculation adds the newest day and drops the oldest.

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

The rule expects a request counter named `http_requests_total` with `job`, `code`, and `route` labels. `rate()` handles counter resets. `for: 2m` starts a pending period after both conditions become true; either falling below the threshold cancels it.

Lookback windows are not fixed waiting periods. Traffic and evaluation intervals affect detection. Notifications require a running Prometheus setup and notification configuration.

---

## Step 5: Run the Following Commands in the Terminal

<p align="center">
  <a href="./images/budget-output.png">
    <img src="./images/budget-output.png" alt="Expected calculator output with 40% budget used and SLO met Yes" width="800">
  </a>
</p>

From `m3-l2`, run the calculator after saving the files:

```bash
python3 calculate-budget.py
```

**Explanation:** Prints the error budget and actual SLI for this lab's request counts.

**What to expect:** The output shows **10,000** allowed failures, **6,000** remaining, and **40%** budget used. `SLO met: Yes` means the measured **99.96%** is above the **99.9%** target. The YAML file defines the alert; running this calculator does not send notifications.

**Read the output:** `Valid requests` is the total after exclusions. `Actual failures` is the number of server errors. `Failures remaining` subtracts those errors from the allowance. `Budget remaining: 60%` and `Budget used: 40%` describe the same allowance; `Actual SLI: 99.96%` describes good requests out of all counted requests.

### Download the Rule Tool

<p align="center">
  <a href="./images/download-output.png">
    <img src="./images/download-output.png" alt="Prometheus archive download in progress at 11 percent" width="900">
  </a>
</p>

The remaining screenshots show commands using `promtool`, the rule-file tool included with Prometheus. These instructions use the screenshot's Prometheus 3.5.0 release on Linux with an x86-64 processor. Stay in `m3-l2`. You need `curl` and `tar` installed.

```bash
# paste this as one line
curl -fLO https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```

**Explanation:** Downloads the archive containing the rule tool into the current folder.

**Command result:** The progress table shows received bytes, speed, and time. `download-output.png` captures the transfer at **11%**, with about **12.8 MB** received out of **115 MB**. Wait for the received percentage to reach `100` and the prompt to return before extracting. Your speed and timing will differ.

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

**Command result:** `tar` normally completes without output. `ls` prints `promtool`. The screenshot shows the long extraction command wrapping at the filename; keep it as one command when pasting. The `promtool` entry identifies the extracted executable.

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

**Command result:** `Checking orders-api-rules.yaml` names the file being read. `SUCCESS: 1 rules found` means the tool accepted the one fast-burn rule. The screenshot uses a different practice-folder path; use your `m3-l2` folder. This command does not run the alert against live traffic or send a notification.


---

## Conclusion

You defined a reliability target, calculated its failure allowance, and wrote a fast-burn alert rule. Next, explore load balancing and automated failover to reduce failures in the service itself.
